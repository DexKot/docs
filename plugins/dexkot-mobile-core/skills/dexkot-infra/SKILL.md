---
name: dexkot-infra
description: "Background work, remote config and runtime permissions with DexKot mobile-core (dev.dexkot.mobile core-work, core-remoteconfig, core-permission and their -di-koin / -firebase / -posthog artifacts). Use whenever the task involves IWorkExecutor, IWorkExecutorProvider, IWorkTypeConfigProvider, WorkTypeConfig, IScheduleWorkUseCase, IObserveWorkStatusUseCase, ICancelWorkUseCase, GenericWorkRunner, WorkerClassProvider, IWorkNotificationProvider, CoroutineWorkEngine, WorkManager in a DexKot app; IRemoteConfigHandler, IRemoteConfigProvider, feature flags, FirebaseRemoteConfigHandler, PostHogRemoteConfigHandler, typed `get` / `getFlow`; or Permission, IPermissionChecker, IPermissionController, PermissionState, PermissionResult, rememberPermissionState, LocalPermissionController. Use it when adding a background job, a remote flag or a permission request."
---

# DexKot infrastructure: work, remote config, permissions

Three independent modules share one design idea: **features declare what they need through small
interfaces; the library owns the platform machinery.** A feature that needs background work writes an
executor and a config and never touches WorkManager. A feature that reads a flag calls a typed extension
on `IRemoteConfigProvider` and never knows whether Firebase or PostHog answers. A screen asks
`IPermissionController` for a permission and gets a clear result instead of juggling rationale flags.

Version covered: **0.27.0**. Read the reference for the area you are working on:
- `references/work.md`: core-work, both backends, notifications, full Android + iOS wiring.
- `references/remote-config.md`: handlers, provider, typed values, backend differences.
- `references/permissions.md`: permission model, controller semantics, Compose helpers.

## Setup

```kotlin
// settings.gradle.kts: add maven("https://dexkot.github.io/maven") next to google() and mavenCentral()

dependencies {
    // Background work (KMP)
    implementation("dev.dexkot.mobile:core-work:0.27.0")
    implementation("dev.dexkot.mobile:core-work-di-koin:0.27.0")
    // Android WorkManager backend, app module only:
    implementation("androidx.work:work-runtime-ktx:2.11.1")
    implementation("io.insert-koin:koin-androidx-workmanager:4.1.1")

    // Remote config (KMP)
    implementation("dev.dexkot.mobile:core-remoteconfig:0.27.0")
    implementation("dev.dexkot.mobile:core-remoteconfig-di-koin:0.27.0")   // pulls both backends

    // Permissions (Android only)
    implementation("dev.dexkot.mobile:core-permission:0.27.0")
    implementation("dev.dexkot.mobile:core-permission-di-koin:0.27.0")
}
```

| Area | Android | iOS |
|---|---|---|
| core-work | WorkManager backend or in-process coroutine backend | in-process coroutine backend |
| core-remoteconfig (+ firebase, posthog) | yes | yes (cinterop pods; the app links them) |
| core-permission | yes | none |

Android artifacts are published. iOS klibs: build from source (`publishToMavenLocal`).
Feature modules only need `core-work` / `core-remoteconfig`; the `-di-koin` and backend artifacts belong
in the app (composition root). See **dexkot-di-koin**.

## Background work in five steps

Work ids are `kotlin.uuid.Uuid`; add `optIn("kotlin.uuid.ExperimentalUuidApi")` to modules that use them.

1. **Executor**: the actual job. Return `WorkResult.Success`, `Failure(error, shouldRetry)` or `Interrupted`.
   Cancellation is cooperative: use `try/finally` for cleanup.
2. **`IWorkExecutorProvider`**: `workType` string + `create(workId)`.
3. **`IWorkTypeConfigProvider`**: constraints and backoff for that type (required by WorkManager).
4. **Register both with `binds arrayOf(...)`** in the feature's Koin module; the library aggregates them with `getAll()`.
5. **Schedule / observe / cancel** with `IScheduleWorkUseCase`, `IObserveWorkStatusUseCase`, `ICancelWorkUseCase`,
   always passing the same `workType` and `workId`.

```kotlin
object UploadWork { const val TYPE = "upload_attachment" }

class UploadWorkExecutor(private val workId: Uuid, private val queue: IUploadQueueRepository) : IWorkExecutor {
    override suspend fun doWork(): WorkResult {
        val job = queue.get(workId) ?: return WorkResult.Failure(WorkNotFound(workId), shouldRetry = false)
        return when (val result = queue.upload(job)) {
            is ResultOf.Success -> WorkResult.Success
            is ResultOf.Failure -> WorkResult.Failure(result.error, shouldRetry = true)
        }
    }
}

class UploadWorkExecutorProvider(private val queue: IUploadQueueRepository) : IWorkExecutorProvider {
    override val workType = UploadWork.TYPE
    override fun create(workId: Uuid): IWorkExecutor = UploadWorkExecutor(workId, queue)
}

class UploadWorkConfigProvider : IWorkTypeConfigProvider {
    override val workType = UploadWork.TYPE
    override fun getConfig() = WorkTypeConfig(networkRequirement = NetworkRequirement.CONNECTED, requiresBatteryNotLow = true)
}

val uploadWorkModule = module {
    factory { UploadWorkExecutorProvider(queue = get()) } binds arrayOf(IWorkExecutorProvider::class)
    factory { UploadWorkConfigProvider() } binds arrayOf(IWorkTypeConfigProvider::class)
}
```

The app then picks a backend: `workManagerWorkModule` on Android (durable, constraints, retries,
notifications; needs an app-owned `CoroutineWorker`, a `WorkerClassProvider` and `permissionModule()`),
or `coroutineWorkModule` on iOS / for lightweight in-process work. The full wiring, the backend
comparison table and notifications are in `references/work.md`.

## Remote config in three steps

1. The app binds one **unqualified** `IRemoteConfigHandler` by resolving the backend it wants:

```kotlin
val remoteConfigAppModule = module {
    single { Json { ignoreUnknownKeys = true } }
    includes(remoteConfigModule())
    single<IRemoteConfigHandler> {
        get(qualifier = firebaseRemoteConfig(), parameters = { parametersOf(15_000L, get<Json>()) })
        // or: get(qualifier = postHogRemoteConfig(), parameters = { parametersOf(get<Json>()) })
    }
}
```

2. Features expose typed accessors as extensions, keeping keys and defaults private:

```kotlin
private const val SYNC_ENABLED = "sync_enabled"
private const val SYNC_POLICY = "sync_policy"

@Serializable data class SyncPolicy(val maxBatch: Int = 50, val wifiOnly: Boolean = true)

fun IRemoteConfigProvider.isSyncEnabled(): StateFlow<Boolean> = getBooleanFlow(SYNC_ENABLED, defaultValue = true)
fun IRemoteConfigProvider.syncPolicy(): SyncPolicy = get(SYNC_POLICY, defaultValue = SyncPolicy())
```

3. Inject `IRemoteConfigProvider` (a `factory`) wherever the value is needed. In Compose, read
   `LocalRemoteConfig.current` and use `getBooleanState` / `getLongState` / `getStringState` from core-ui.

Why a default everywhere: a flag must keep working offline, before the first fetch, and when the key is
missing. Every accessor either takes a default or returns a nullable / `ResultOf`. Details and backend
semantics: `references/remote-config.md`.

## Permissions in three steps (Android)

1. `includes(permissionModule(preferencesName = "com.example.notes.permissions"))`: registers
   `IPermissionChecker` and the SharedPreferences used to remember requests.
2. At the root of the Compose tree, build the controller once and pass it to core-ui's `Navigation`, which
   provides it per screen as `LocalPermissionController`, already wrapped to log permission events:

```kotlin
val permissionController = rememberPermissionController(
    activity = this,
    preferences = koinInject(qualifier = permissionPreferences()),
)
```

3. In a screen, observe with `rememberPermissionState(Permission.Camera)` and request with
   `LocalPermissionController.current.requestPermission(Permission.Camera) { result -> ... }`. In
   ViewModels and workers, only check: inject `IPermissionChecker` and call `isGranted(permission)`.

Custom permissions: `Permission("android.permission.READ_CONTACTS")` or subclass `Permission` (it is open).
Details: `references/permissions.md`.

## Rules and rationale

- Keep platform types out of features: executors, configs and flag accessors live in `commonMain` and only
  use the core interfaces. That is what lets the same feature run on WorkManager and on iOS.
- Make executors re-entrant. WorkManager retries and the in-process engine is restarted from your own
  persisted state, so `doWork` may run more than once for the same `workId`.
- Store durable job state (what to upload, progress) in your own database, keyed by `workId`. The work
  layer only schedules and reports status.
- Request permissions only from UI; check them anywhere else. A request needs an Activity and user
  attention; a check needs only a Context.

## Gotchas (summary; each reference has the full list)

1. WorkManager backend: no `IWorkTypeConfigProvider` for the type means `Failure(UnsupportedWorkType)`.
   The returned `Uuid` is the WorkRequest id, not your `workId`; keep using your `workId`.
2. In-process backend: nothing survives process death, constraints and backoff are ignored, and
   `Failure(shouldRetry = true)` is not retried. Re-schedule unfinished work at startup.
3. `ScheduleWorkUseCase` (WorkManager) waits for the enqueue to finish on the calling thread; call it
   from a background dispatcher.
4. The worker class name is persisted by WorkManager: never rename or move it.
5. `remoteConfigModule()` does not bind the unqualified `IRemoteConfigHandler`; `IRemoteConfigProvider`
   fails on first injection without it. Handler parameters apply only on first creation.
6. Flows are cached per key: the first `defaultValue` for a key wins.
7. `rememberPermissionController` is not memoised; call it in a root composable that does not recompose,
   or a request in flight can lose its callback.
8. `requestPermission` always shows the OS dialog unless already granted; check `getState(p).canRequest`
   first if you want to send users to settings instead.

## Related skills

- **dexkot-di-koin**: module layout, `binds`, `getAll`, qualifiers and parameters.
- **dexkot-logging**: `LoggerContextElement` per work type via `WorkerCoroutineContextFactory`.
- **dexkot-results-errors**: `ResultOf`, `ErrorEntity`, used by every use case here.
- **dexkot-screen**: `Navigation(...)`, where the permission controller and remote config provider are passed.
- **dexkot-viewmodel**: checking permissions and reading flags from a ViewModel.
