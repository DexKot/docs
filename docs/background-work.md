# Background work

`core-work` lets a feature describe a background job without depending on a platform scheduler. The
feature writes an **executor** (what to do), a **config** (when it may run), and optionally a
**notification provider** (what to show on Android). The app chooses a backend:

- **WorkManager** (Android): durable, honours constraints, retries with backoff, and can run as a
  foreground service with progress notifications.
- **In-process coroutines** (iOS, or lightweight Android work): runs jobs in a `CoroutineScope`. Simple,
  but nothing survives the process.

Feature code is identical for both, so it can live in `commonMain`.

Version: **0.27.0**.

- [Install](#install)
- [1. Write an executor](#1-write-an-executor)
- [2. Describe the work type](#2-describe-the-work-type)
- [3. Schedule, observe, cancel](#3-schedule-observe-cancel)
- [4. Android: WorkManager backend](#4-android-workmanager-backend)
- [5. Android: progress notifications](#5-android-progress-notifications)
- [6. iOS and in-process work](#6-ios-and-in-process-work)
- [Backend differences](#backend-differences)
- [Pitfalls](#pitfalls)
- [API reference](#api-reference)

## Install

```kotlin
// feature modules (commonMain)
implementation("dev.dexkot.mobile:core-work:0.27.0")

// app / composition root
implementation("dev.dexkot.mobile:core-work-di-koin:0.27.0")
// Android + WorkManager backend:
implementation("dev.dexkot.mobile:core-permission-di-koin:0.27.0")
implementation("androidx.work:work-runtime-ktx:2.11.1")
implementation("io.insert-koin:koin-androidx-workmanager:4.1.1")
```

Add the repository `maven("https://dexkot.github.io/maven")` in `settings.gradle.kts`. Android artifacts
are published; for iOS, build the klibs from source (`publishToMavenLocal`).

Work ids are `kotlin.uuid.Uuid`. Opt in where you use them:

```kotlin
kotlin { compilerOptions { optIn.add("kotlin.uuid.ExperimentalUuidApi") } }
```

## 1. Write an executor

```kotlin
object UploadWork { const val TYPE = "upload_attachment" }

class UploadWorkExecutor(
    private val workId: Uuid,
    private val queue: IUploadQueueRepository,
) : IWorkExecutor {

    private val progress = MutableStateFlow<WorkProgressData?>(null)
    override val progressFlow: Flow<WorkProgressData?> = progress     // optional

    override suspend fun doWork(): WorkResult {
        val job = queue.get(workId) ?: return WorkResult.Failure(WorkNotFound(workId), shouldRetry = false)
        job.chunks.forEachIndexed { index, chunk ->
            if (chunk.uploaded) return@forEachIndexed
            progress.value = WorkProgressData(UploadWork.TYPE, workId, current = index + 1, total = job.chunks.size)
            val result = queue.upload(workId, chunk)
            if (result is ResultOf.Failure) return WorkResult.Failure(result.error, shouldRetry = true)
        }
        return WorkResult.Success
    }
}
```

- Return `Success`, `Failure(error, shouldRetry)` or `Interrupted` ("stopped on purpose, state saved").
- Use `shouldRetry = true` for transient failures and `false` when a retry cannot help.
- Make `doWork` **re-entrant**: it may run again for the same `workId` after a retry or a restart. The
  executor above skips chunks that are already uploaded.
- Cancellation is cooperative. There is no `interrupt()`; cancelling the work cancels the coroutine, so
  put cleanup in `finally`.
- Keep the job's durable data (what to upload) in your own storage, keyed by `workId`. The work layer
  only carries the id.

## 2. Describe the work type

```kotlin
class UploadWorkExecutorProvider(private val queue: IUploadQueueRepository) : IWorkExecutorProvider {
    override val workType = UploadWork.TYPE
    override fun create(workId: Uuid): IWorkExecutor = UploadWorkExecutor(workId, queue)
}

class UploadWorkConfigProvider : IWorkTypeConfigProvider {
    override val workType = UploadWork.TYPE
    override fun getConfig() = WorkTypeConfig(
        networkRequirement = NetworkRequirement.UNMETERED,
        requiresBatteryNotLow = true,
        backoffPolicy = BackoffPolicy.EXPONENTIAL,
    )
}

val uploadWorkModule = module {
    factory { UploadWorkExecutorProvider(queue = get()) } binds arrayOf(IWorkExecutorProvider::class)
    factory { UploadWorkConfigProvider() } binds arrayOf(IWorkTypeConfigProvider::class)
}
```

The library collects every provider with `getAll()` and looks them up by `workType`. That is why each
feature can add work types without touching shared code.

`WorkTypeConfig` options: `networkRequirement` (`NONE`, `CONNECTED`, `UNMETERED`),
`requiresBatteryNotLow`, `requiresCharging`, `requiresStorageNotLow`, `requiresDeviceIdle`, and
`backoffPolicy` (`EXPONENTIAL`, `LINEAR`). They apply on WorkManager only.

## 3. Schedule, observe, cancel

```kotlin
class StartUploadUseCase(
    private val queue: IUploadQueueRepository,
    private val scheduleWork: IScheduleWorkUseCase,
) {
    suspend operator fun invoke(file: LocalFile): ResultOf<Uuid> {
        val workId = Uuid.random()
        queue.enqueue(workId, file)
        return when (val scheduled = scheduleWork(workType = UploadWork.TYPE, workId = workId)) {
            is ResultOf.Success -> ResultOf.Success(workId)
            is ResultOf.Failure -> ResultOf.Failure(scheduled.error)
        }
    }
}
```

- Observe with `observeWork(UploadWork.TYPE, workId): Flow<WorkStatus?>`. The statuses are `Queued`,
  `Blocked` (waiting for constraints), `Running`, `Completed`, `Failed` and `Cancelled`. `null` means the
  work is unknown.
- Cancel with `cancelWork(UploadWork.TYPE, workId)`.
- Always pass the same `workType` and `workId` to all three calls.
- On WorkManager, the success value of `scheduleWork` is the WorkRequest id. Keep using your own `workId`.

## 4. Android: WorkManager backend

WorkManager stores the worker's class name in its database, so the worker class has to be yours and must
never be renamed or moved. A library-owned class could change in an upgrade and strand queued work. Write
one tiny worker that delegates to the library:

```kotlin
class AppWorker(
    context: Context,
    params: WorkerParameters,
    private val runner: GenericWorkRunner,
) : CoroutineWorker(context, params) {
    override suspend fun doWork(): Result = runner.run(this)
}
```

```kotlin
val appWorkModule = module {
    worker { AppWorker(context = get(), params = get(), runner = get()) }       // org.koin.androidx.workmanager.dsl.worker
    single<WorkerClassProvider> { WorkerClassProvider { AppWorker::class.java } }
}

class NotesApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        startKoin {
            androidContext(this@NotesApplication)
            workManagerFactory()
            modules(permissionModule(), workManagerWorkModule, appWorkModule, uploadWorkModule /* , ... */)
        }
    }
}
```

- `permissionModule()` provides the `IPermissionChecker` that `GenericWorkRunner` uses to decide whether
  it may post notifications.
- Koin's `workManagerFactory()` requires removing WorkManager's default initializer from the manifest.
  Follow the Koin WorkManager setup guide.
- `ScheduleWorkUseCase` waits for WorkManager to accept the request, so call it from a background
  dispatcher.

Every request is a `OneTimeWorkRequest` with your constraints and a backoff starting at 10 seconds.
`Failure(shouldRetry = true)` and uncaught exceptions become `Result.retry()`. The library sets no retry
limit, so stop retrying yourself when it makes sense.

## 5. Android: progress notifications

Register an `IWorkNotificationProvider` for the work type:

```kotlin
class UploadNotificationProvider(private val context: Context) : IWorkNotificationProvider {
    override val workType = UploadWork.TYPE
    override fun getChannelId() = "uploads"
    override fun getNotification(progress: WorkProgressData) = NotificationContent(
        title = context.getString(R.string.upload_in_progress),
        text = "${progress.current}/${progress.total}",
        iconResId = R.drawable.ic_upload,
        progress = NotificationContent.Progress(progress.current, progress.total, indeterminate = progress.total < 0),
    )
    override fun getCompletedNotification(progress: WorkProgressData, status: WorkCompletionStatus) = NotificationContent(
        title = context.getString(if (status == WorkCompletionStatus.Success) R.string.upload_done else R.string.upload_failed),
        text = "",
        iconResId = R.drawable.ic_upload,
    )
}

// app module
factory { UploadNotificationProvider(androidContext()) } binds arrayOf(IWorkNotificationProvider::class)
```

- The first progress value turns the worker into a foreground service showing `getNotification`.
- When the work ends, `getCompletedNotification` is shown, but only if some progress was emitted.
- Create the notification channels at app start.
- Declare a `foregroundServiceType` for WorkManager's `SystemForegroundService` in your manifest that
  matches `getForegroundServiceType()` (default `dataSync`), plus the matching `FOREGROUND_SERVICE_*`
  permission.
- Without `POST_NOTIFICATIONS` (Android 13+), notifications are skipped and the work still runs.

## 6. iOS and in-process work

```kotlin
val platformWorkModule = module {
    includes(coroutineWorkModule)
    // optional: the scope jobs run in
    single<CoroutineScope>(workCoroutineScope()) { CoroutineScope(SupervisorJob() + Dispatchers.Default) }
}
```

The in-process engine runs `doWork` directly and tracks status in memory. It ignores constraints and
backoff, and it does not retry. Nothing is persisted, so at app start, read unfinished jobs from your
database and schedule them again with their original ids.

For extra background time on iOS, bind a `WorkBackgroundGrace` that wraps the block in
`UIApplication.beginBackgroundTask` / `endBackgroundTask`. The library ships the interface, not an iOS
implementation.

## Backend differences

| | WorkManager | In-process |
|---|---|---|
| Survives process death | yes | no |
| Constraints / backoff | yes | ignored |
| Retry on `Failure(shouldRetry = true)` | yes, unbounded | no, status `Failed` |
| `Interrupted` | `Completed` | `Cancelled` |
| Missing `IWorkTypeConfigProvider` | schedule fails (`UnsupportedWorkType`) | fine |
| Scheduling the same id again | second request enqueued | previous job replaced |
| Notifications | yes | no |

## Pitfalls

- **Don't include both `workManagerWorkModule` and `coroutineWorkModule`** in one Koin graph. Both bind
  the same use cases.
- **Providers loaded late are ignored.** The factories are singletons that capture `getAll()` on first
  use, so load feature modules with the rest at startup.
- **Two providers with the same `workType`:** the last one wins, silently.
- **Scheduling the same `workId` twice on WorkManager** creates two requests. Cancel first.
- **Per-work-type logging:** bind a `WorkerCoroutineContextFactory` that returns a
  `LoggerContextElement` (see [logging](logging.md)). Both backends apply it.

## API reference

```kotlin
// dev.dexkot.mobile.core.work.domain.executor
interface IWorkExecutor { val progressFlow: Flow<WorkProgressData?> get() = emptyFlow(); suspend fun doWork(): WorkResult }
interface IWorkExecutorProvider { val workType: String; fun create(workId: Uuid): IWorkExecutor }
interface IWorkExecutorFactory { fun create(workType: String, workId: Uuid): IWorkExecutor? }

// ...domain.executor.model
data class WorkProgressData(val workType: String, val workId: Uuid, val current: Int, val total: Int, val metadata: Map<String, String> = emptyMap())
sealed class WorkResult { Success; Failure(val error: ErrorEntity, val shouldRetry: Boolean = true); Interrupted }
sealed class WorkStatus { Queued; Blocked; Running; Completed; Failed; Cancelled }

// ...domain.config / .config.model
interface IWorkTypeConfigProvider { val workType: String; fun getConfig(): WorkTypeConfig }
interface IWorkTypeConfigFactory { fun getConfig(workType: String): WorkTypeConfig? }
data class WorkTypeConfig(networkRequirement = NONE, requiresBatteryNotLow = false, requiresCharging = false,
                          requiresStorageNotLow = false, requiresDeviceIdle = false, backoffPolicy = EXPONENTIAL)
enum class NetworkRequirement { NONE, CONNECTED, UNMETERED }
enum class BackoffPolicy { EXPONENTIAL, LINEAR }

// ...domain.usecase
interface IScheduleWorkUseCase { suspend operator fun invoke(workType: String, workId: Uuid): ResultOf<Uuid> }
interface IObserveWorkStatusUseCase { operator fun invoke(workType: String, workId: Uuid): Flow<WorkStatus?> }
interface ICancelWorkUseCase { operator fun invoke(workType: String, workId: Uuid): ResultOf<Unit> }

// ...domain.worker
fun interface WorkerCoroutineContextFactory { fun create(workType: String): CoroutineContext }
fun interface WorkerClassProvider { fun workerClass(): Class<out ListenableWorker> }          // Android

// ...domain.error
UnsupportedWorkType(workType), WorkEnqueueFailed(workType, cause), WorkNotFound(workId),
NonExecutableWorkStatus(workId, currentStatus), NetworkErrorThresholdExceededException(workId, consecutiveErrors)

// ...data.worker (Android)
class GenericWorkRunner(executorFactory, notificationFactory, permissionChecker, coroutineContextFactory: WorkerCoroutineContextFactory? = null) {
    suspend fun run(worker: CoroutineWorker): ListenableWorker.Result
}

// ...domain.notification (Android)
interface IWorkNotificationProvider {
    val workType: String
    fun getChannelId(): String?
    fun getCompletionChannelId(): String? = null
    fun getForegroundServiceType(): Int = ServiceInfo.FOREGROUND_SERVICE_TYPE_DATA_SYNC
    fun getNotification(progress: WorkProgressData): NotificationContent
    fun getCompletedNotification(progress: WorkProgressData, status: WorkCompletionStatus): NotificationContent
}
data class NotificationContent(title: String, text: String, @DrawableRes iconResId: Int, contentIntent: PendingIntent? = null, progress: Progress? = null)
enum class WorkCompletionStatus { Success, Failed, Retrying }

// ...data.coroutine
class CoroutineWorkEngine(scope: CoroutineScope, grace: WorkBackgroundGrace = NoOpWorkBackgroundGrace)
interface WorkBackgroundGrace { suspend fun <T> withGrace(block: suspend () -> T): T }

// dev.dexkot.mobile.core.work.di.koin
val workFactoriesModule: Module
val coroutineWorkModule: Module
fun workCoroutineScope(): StringQualifier
val workManagerWorkModule: Module                     // Android
```
