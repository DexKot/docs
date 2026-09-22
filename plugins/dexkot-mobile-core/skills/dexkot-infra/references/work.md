# core-work reference (0.27.0)

Contents
1. Model: providers, factories, backends
2. Common API
3. Defining a work type (full example with progress)
4. Scheduling, observing, cancelling
5. Android: WorkManager backend wiring
6. Android: notifications
7. In-process coroutine backend (iOS, lightweight Android work)
8. Backend comparison
9. Per-work-type coroutine context (logging)
10. Gotchas

## 1. Model

```
feature module (commonMain)          library                               app
IWorkExecutorProvider  ──getAll──▶  IWorkExecutorFactory ─┐
IWorkTypeConfigProvider ─getAll──▶  IWorkTypeConfigFactory├─▶ backend use cases  ◀── chooses backend module
IWorkNotificationProvider (Android)▶ IWorkNotificationFactory┘   (WorkManager | coroutine)
```

Features plug in one provider per work type. The library aggregates providers into factories and runs
them through the backend the app selects. Why: the feature stays platform-neutral and testable, and
the platform-heavy parts (WorkManager requests, foreground notifications, constraint tracking) exist once.

## 2. Common API

Packages under `dev.dexkot.mobile.core.work`.

```kotlin
// domain.executor
interface IWorkExecutor {
    val progressFlow: Flow<WorkProgressData?> get() = emptyFlow()
    suspend fun doWork(): WorkResult
}
interface IWorkExecutorProvider { val workType: String; fun create(workId: Uuid): IWorkExecutor }
interface IWorkExecutorFactory { fun create(workType: String, workId: Uuid): IWorkExecutor? }

// domain.executor.model
data class WorkProgressData(val workType: String, val workId: Uuid, val current: Int, val total: Int, val metadata: Map<String, String> = emptyMap())
sealed class WorkResult {                       // parcelable
    data object Success
    data class Failure(val error: ErrorEntity, val shouldRetry: Boolean = true)
    data object Interrupted
}
sealed class WorkStatus { Queued, Blocked, Running, Completed, Failed, Cancelled }   // data objects

// domain.config
interface IWorkTypeConfigProvider { val workType: String; fun getConfig(): WorkTypeConfig }
interface IWorkTypeConfigFactory { fun getConfig(workType: String): WorkTypeConfig? }

// domain.config.model (+ .network, .backoff)
data class WorkTypeConfig(
    val networkRequirement: NetworkRequirement = NetworkRequirement.NONE,
    val requiresBatteryNotLow: Boolean = false,
    val requiresCharging: Boolean = false,
    val requiresStorageNotLow: Boolean = false,
    val requiresDeviceIdle: Boolean = false,
    val backoffPolicy: BackoffPolicy = BackoffPolicy.EXPONENTIAL,
)
enum class NetworkRequirement { NONE, CONNECTED, UNMETERED }
enum class BackoffPolicy { EXPONENTIAL, LINEAR }

// domain.usecase
interface IScheduleWorkUseCase { suspend operator fun invoke(workType: String, workId: Uuid): ResultOf<Uuid> }
interface IObserveWorkStatusUseCase { operator fun invoke(workType: String, workId: Uuid): Flow<WorkStatus?> }
interface ICancelWorkUseCase { operator fun invoke(workType: String, workId: Uuid): ResultOf<Unit> }

// domain.worker
fun interface WorkerCoroutineContextFactory { fun create(workType: String): CoroutineContext }

// domain.work (tags used by the WorkManager backend)
fun workTag(workType: String, workId: Uuid): String      // "<workType>_<workId>"
fun workIdTag(workId: Uuid): String                      // "workId_<workId>"
fun workTypeTag(workType: String): String                // "workType_<workType>"

// domain.error: ErrorEntity.BusinessError subclasses
class UnsupportedWorkType(val workType: String)
class WorkEnqueueFailed(val workType: String, val cause: Throwable)
class WorkNotFound(val workId: Uuid)
class NonExecutableWorkStatus(val workId: Uuid, val currentStatus: String)
class NetworkErrorThresholdExceededException(val workId: Uuid, val consecutiveErrors: Int) : Exception
```

`WorkProgressData.total = -1` means indeterminate progress. The error classes are provided for executors
to use; the library itself only returns `UnsupportedWorkType` and `WorkEnqueueFailed`.

## 3. Defining a work type

```kotlin
package com.example.upload.work

import com.example.upload.data.IUploadQueueRepository
import dev.dexkot.mobile.core.foundation.result.ResultOf
import dev.dexkot.mobile.core.work.domain.config.IWorkTypeConfigProvider
import dev.dexkot.mobile.core.work.domain.config.model.WorkTypeConfig
import dev.dexkot.mobile.core.work.domain.config.model.backoff.BackoffPolicy
import dev.dexkot.mobile.core.work.domain.config.model.network.NetworkRequirement
import dev.dexkot.mobile.core.work.domain.error.WorkNotFound
import dev.dexkot.mobile.core.work.domain.executor.IWorkExecutor
import dev.dexkot.mobile.core.work.domain.executor.IWorkExecutorProvider
import dev.dexkot.mobile.core.work.domain.executor.model.WorkProgressData
import dev.dexkot.mobile.core.work.domain.executor.model.WorkResult
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.MutableStateFlow
import kotlin.uuid.Uuid

object UploadWork { const val TYPE = "upload_attachment" }

class UploadWorkExecutor(
    private val workId: Uuid,
    private val queue: IUploadQueueRepository,
) : IWorkExecutor {

    private val progress = MutableStateFlow<WorkProgressData?>(null)
    override val progressFlow: Flow<WorkProgressData?> = progress

    override suspend fun doWork(): WorkResult {
        val job = queue.get(workId) ?: return WorkResult.Failure(WorkNotFound(workId), shouldRetry = false)
        job.chunks.forEachIndexed { index, chunk ->
            if (chunk.uploaded) return@forEachIndexed                     // re-entrant: skip finished parts
            progress.value = WorkProgressData(UploadWork.TYPE, workId, current = index + 1, total = job.chunks.size)
            val result = queue.upload(workId, chunk)
            if (result is ResultOf.Failure) return WorkResult.Failure(result.error, shouldRetry = true)
        }
        return WorkResult.Success
    }
}

class UploadWorkExecutorProvider(private val queue: IUploadQueueRepository) : IWorkExecutorProvider {
    override val workType: String = UploadWork.TYPE
    override fun create(workId: Uuid): IWorkExecutor = UploadWorkExecutor(workId, queue)
}

class UploadWorkConfigProvider : IWorkTypeConfigProvider {
    override val workType: String = UploadWork.TYPE
    override fun getConfig() = WorkTypeConfig(
        networkRequirement = NetworkRequirement.UNMETERED,
        requiresBatteryNotLow = true,
        backoffPolicy = BackoffPolicy.EXPONENTIAL,
    )
}
```

```kotlin
// feature DI (commonMain)
import org.koin.dsl.binds
import org.koin.dsl.module

val uploadWorkModule = module {
    factory { UploadWorkExecutorProvider(queue = get()) } binds arrayOf(IWorkExecutorProvider::class)
    factory { UploadWorkConfigProvider() } binds arrayOf(IWorkTypeConfigProvider::class)
}
```

Choosing `Failure.shouldRetry`: `true` for transient problems (network, server 5xx), `false` when a retry
cannot succeed (job deleted, validation error). `Interrupted` means "stopped on purpose, state saved";
WorkManager treats it as success.

Cancellation: there is no `interrupt()` method. Cancelling the work cancels the coroutine running
`doWork`; suspend calls throw `CancellationException`. Put cleanup in `finally` and do not swallow
`CancellationException`.

## 4. Scheduling, observing, cancelling

```kotlin
class StartUploadUseCase(
    private val queue: IUploadQueueRepository,
    private val scheduleWork: IScheduleWorkUseCase,
) {
    suspend operator fun invoke(file: LocalFile): ResultOf<Uuid> {
        val workId = Uuid.random()
        queue.enqueue(workId, file)                                // durable state first
        return when (val scheduled = scheduleWork(workType = UploadWork.TYPE, workId = workId)) {
            is ResultOf.Success -> ResultOf.Success(workId)       // keep your own id, not scheduled.data
            is ResultOf.Failure -> ResultOf.Failure(scheduled.error)
        }
    }
}

class ObserveUploadUseCase(private val observeWork: IObserveWorkStatusUseCase) {
    operator fun invoke(workId: Uuid): Flow<WorkStatus?> = observeWork(UploadWork.TYPE, workId)
}
```

`null` from the status flow means the work is unknown to the backend (never scheduled, pruned by
WorkManager, or lost with the process on the in-process backend).

## 5. Android: WorkManager backend wiring

Android-only API (`core-work` androidMain):

```kotlin
// data.worker
class GenericWorkRunner(
    executorFactory: IWorkExecutorFactory,
    notificationFactory: IWorkNotificationFactory,
    permissionChecker: IPermissionChecker,
    coroutineContextFactory: WorkerCoroutineContextFactory? = null,
) {
    suspend fun run(worker: CoroutineWorker): ListenableWorker.Result
    companion object { fun createInputData(workType: String, workId: Uuid): androidx.work.Data }
}
// domain.worker
fun interface WorkerClassProvider { fun workerClass(): Class<out ListenableWorker> }
```

The library does not ship a `Worker` class. WorkManager stores the worker's fully-qualified class name in
its database, so the class must live in your app and keep its name forever; a library class could be
renamed by an upgrade and strand queued work. Write a one-line worker that delegates to `GenericWorkRunner`:

```kotlin
package com.example.notes.work

import android.content.Context
import androidx.work.CoroutineWorker
import androidx.work.WorkerParameters
import dev.dexkot.mobile.core.work.data.worker.GenericWorkRunner

class AppWorker(
    context: Context,
    params: WorkerParameters,
    private val runner: GenericWorkRunner,
) : CoroutineWorker(context, params) {
    override suspend fun doWork(): Result = runner.run(this)
}
```

```kotlin
package com.example.notes.di

import com.example.notes.work.AppWorker
import dev.dexkot.mobile.core.logger.ILogger
import dev.dexkot.mobile.core.logger.context.LoggerContextElement
import dev.dexkot.mobile.core.logger.logicInteractor
import dev.dexkot.mobile.core.work.domain.notification.IWorkNotificationProvider
import dev.dexkot.mobile.core.work.domain.worker.WorkerClassProvider
import dev.dexkot.mobile.core.work.domain.worker.WorkerCoroutineContextFactory
import org.koin.android.ext.koin.androidContext
import org.koin.androidx.workmanager.dsl.worker
import org.koin.dsl.binds
import org.koin.dsl.module

val appWorkModule = module {
    worker { AppWorker(context = get(), params = get(), runner = get()) }
    single<WorkerClassProvider> { WorkerClassProvider { AppWorker::class.java } }

    // optional: every execution logs under "worker_<type>"
    single<WorkerCoroutineContextFactory> {
        WorkerCoroutineContextFactory { type -> LoggerContextElement(get<ILogger>().logicInteractor("worker_$type")) }
    }

    // optional: notifications for a type (section 6)
    factory { UploadNotificationProvider(androidContext()) } binds arrayOf(IWorkNotificationProvider::class)
}
```

```kotlin
// Application.onCreate
startKoin {
    androidContext(this@NotesApplication)
    workManagerFactory()                                  // org.koin.androidx.workmanager.koin.workManagerFactory
    modules(
        permissionModule(),                               // GenericWorkRunner needs IPermissionChecker
        workManagerWorkModule,
        appWorkModule,
        uploadWorkModule,
        /* ... */
    )
}
```

Koin's `workManagerFactory()` requires disabling WorkManager's default initializer in the manifest (see
the Koin WorkManager documentation). `core-work` declares WorkManager as an implementation dependency, so
add `androidx.work:work-runtime-ktx` to the app yourself.

`workManagerWorkModule` (`dev.dexkot.mobile.core.work.di.koin`, androidMain) registers:
the factories (`workFactoriesModule`), `WorkManager.getInstance(context)`, `WorkNotificationFactory`,
`GenericWorkRunner`, `ScheduleWorkUseCase`, `CancelWorkUseCase`, `ObserveWorkStatusUseCase`,
`ConstraintChecker` and `ConstraintChangeListener`. You provide `WorkerClassProvider`, `IPermissionChecker`
and the worker registration.

Scheduling details: one `OneTimeWorkRequest` per call, with the constraints from `WorkTypeConfig`,
backoff `config.backoffPolicy` starting at `WorkRequest.MIN_BACKOFF_MILLIS` (10 s), and three tags
(`workTypeTag`, `workIdTag`, `workTag`). Status is observed through the `workTag` and re-evaluated when
connectivity, battery, charging or storage change, so `Blocked` (enqueued with unmet constraints) is
distinguished from `Queued`.

## 6. Android: notifications

```kotlin
interface IWorkNotificationProvider {                 // domain.notification
    val workType: String
    fun getChannelId(): String?                       // null: no foreground / progress notification
    fun getCompletionChannelId(): String? = null      // null: reuse getChannelId()
    fun getForegroundServiceType(): Int = ServiceInfo.FOREGROUND_SERVICE_TYPE_DATA_SYNC
    fun getNotification(progress: WorkProgressData): NotificationContent
    fun getCompletedNotification(progress: WorkProgressData, status: WorkCompletionStatus): NotificationContent
}
data class NotificationContent(
    val title: String, val text: String, @DrawableRes val iconResId: Int,
    val contentIntent: PendingIntent? = null, val progress: Progress? = null,
) { data class Progress(val current: Int, val total: Int, val indeterminate: Boolean = false) }
enum class WorkCompletionStatus { Success, Failed, Retrying }
```

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
```

How it behaves:
- The first non-null value on `progressFlow` promotes the worker to a foreground service with the
  in-progress notification; later values update it.
- When `doWork` returns, a completion notification is shown **only if at least one progress value was
  emitted**. `Failure(shouldRetry = true)` maps to `Retrying`.
- Without `POST_NOTIFICATIONS` on API 33+, both notifications are skipped silently; the work still runs.
- `CancelWorkUseCase` dismisses the completion notification.

What you must provide: the notification channels (create them at app start, before any work runs), and a
`foregroundServiceType` in the manifest matching `getForegroundServiceType()` (for
`androidx.work.impl.foreground.SystemForegroundService`) plus the matching `FOREGROUND_SERVICE_*`
permission.

## 7. In-process coroutine backend

```kotlin
// data.coroutine
class CoroutineWorkEngine(scope: CoroutineScope, grace: WorkBackgroundGrace = NoOpWorkBackgroundGrace) {
    fun schedule(workId: Uuid, executor: IWorkExecutor, context: CoroutineContext = EmptyCoroutineContext)
    fun observe(workId: Uuid): Flow<WorkStatus?>
    fun cancel(workId: Uuid)
}
interface WorkBackgroundGrace { suspend fun <T> withGrace(block: suspend () -> T): T }
object NoOpWorkBackgroundGrace : WorkBackgroundGrace
class CoroutineScheduleWorkUseCase(engine, executorFactory: IWorkExecutorFactory, contextFactory: WorkerCoroutineContextFactory? = null)
class CoroutineObserveWorkStatusUseCase(engine)
class CoroutineCancelWorkUseCase(engine)
```

Koin (`coroutineWorkModule`, commonMain) registers the engine and the three use cases. Optional
overrides it picks up if present:
- `single<CoroutineScope>(workCoroutineScope()) { ... }`: the scope jobs run in (default
  `SupervisorJob() + Dispatchers.Default`).
- `single<WorkBackgroundGrace> { ... }`: wraps each `doWork`, e.g. to request background execution time
  on iOS with `UIApplication.beginBackgroundTask`. The library ships no iOS implementation; write one if
  you need it.
- `single<WorkerCoroutineContextFactory> { ... }`.

```kotlin
// iosMain platform module
val platformWorkModule = module {
    includes(coroutineWorkModule)
    single<CoroutineScope>(workCoroutineScope()) { CoroutineScope(SupervisorJob() + Dispatchers.Default) }
}
```

Because nothing is persisted, run a startup step that reads unfinished jobs from your database and
schedules them again with their original `workId`.

## 8. Backend comparison

| | WorkManager (Android) | In-process coroutines |
|---|---|---|
| Needs `IWorkTypeConfigProvider` | yes, else `Failure(UnsupportedWorkType)` | no (config ignored) |
| Needs `IWorkExecutorProvider` | yes; missing at run time fails the work | yes, else `Failure(UnsupportedWorkType)` at schedule |
| `Success` value | the WorkRequest id | your `workId` |
| Constraints and backoff | applied | ignored; never `Blocked` |
| `Failure(shouldRetry = true)` | `Result.retry()` (no cap from the library) | `Failed` |
| `Failure(shouldRetry = false)` | `Failed` | `Failed` |
| `Interrupted` | `Result.success()` → `Completed` | `Cancelled` |
| Exception thrown from `doWork` | caught → retry | `Failed` |
| Same `workId` scheduled twice | a second request is enqueued | previous job cancelled and replaced |
| Survives process death | yes | no |
| Status after restart | from WorkManager's database | `null` |
| Notifications | yes (section 6) | no |

## 9. Per-work-type coroutine context

Both backends wrap each execution in `WorkerCoroutineContextFactory.create(workType)` when one is bound.
The usual content is a `LoggerContextElement`, so `logger { }` calls and AutoLog use case decorators inside
an executor are attributed to the work type. See **dexkot-logging**.

## 10. Gotchas

1. **The worker class is app-owned and permanent.** Renaming or moving it breaks already-queued work.
2. **`ScheduleWorkUseCase` blocks while enqueuing** (`Operation.result.get()`); invoke it off the main thread.
3. **Keep using your `workId`.** The WorkManager backend returns the request id, but status and cancel
   look work up by `workTag(workType, workId)`.
4. **Don't schedule the same `workId` twice on WorkManager.** A second request is enqueued (not unique
   work) and `observe` reports whichever WorkManager lists first. Cancel before re-scheduling.
5. **Factories are resolved once.** `workFactoriesModule` binds the factories as `single` and captures
   `getAll()` at first use: providers from modules loaded later (`loadKoinModules`) are not seen.
6. **One provider per work type.** A second provider with the same `workType` silently replaces the first.
7. **Retries are unbounded on WorkManager.** Track attempts in your own state (or use
   `worker.runAttemptCount` in your worker) and return `shouldRetry = false` when you want to stop.
8. **`Blocked` is an approximation** recomputed from system broadcasts; don't build logic that depends on
   exact constraint timing.
9. **Don't include both backend modules** in one Koin graph: both bind `IScheduleWorkUseCase`,
   `IObserveWorkStatusUseCase` and `ICancelWorkUseCase`.
10. **`kotlin.uuid.Uuid`, not `java.util.UUID`.** Opt in with `optIn("kotlin.uuid.ExperimentalUuidApi")`.
