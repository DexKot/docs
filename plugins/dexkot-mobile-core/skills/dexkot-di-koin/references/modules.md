# DexKot Koin modules: exact bindings (0.27.0)

What each module registers, with scope and qualifier, and what it expects to find in the graph.

## Contents
- [core-logger-di-koin](#core-logger-di-koin)
- [core-remoteconfig-di-koin](#core-remoteconfig-di-koin)
- [core-work-di-koin](#core-work-di-koin)
- [core-permission-di-koin](#core-permission-di-koin)
- [core-ui-di-koin](#core-ui-di-koin)
- [core-preferences](#core-preferences)
- [Verifying a graph in tests](#verifying-a-graph-in-tests)

## core-logger-di-koin

Package `dev.dexkot.mobile.core.logger.di.koin`. KMP (commonMain). Depends via `api` on
`core-logger`, `core-logger-firebase`, `core-logger-posthog`.

```kotlin
fun loggerModule(
    consoleLoggerTag: String = "Console",
    logicSourceComponent: String = "app",
): Module
```

| Definition | Scope | Notes |
|------------|-------|-------|
| `ConsoleLogger(tag = consoleLoggerTag)` | single | by concrete type |
| `FirebaseLogger()` | single | by concrete type; touches the Firebase SDK when created |
| `PostHogLogger()` | single | by concrete type; touches the PostHog SDK when created |
| `dev.dexkot.mobile.core.logger.interactors.ddl.ILoggerInteractor` | single, **createdAtStart** | `get<ILogger>().logicInteractor(logicSourceComponent)`, also installed with `LoggerContextElement.setDefault(...)` so the `logger { }` DSL works outside a logging coroutine context |
| `dev.dexkot.mobile.core.logger.interactors.ui.ILoggerInteractor` | factory, 1 parameter | `get<ILogger>().uiInteractor(sourceComponent = parameters.get())`; resolve with `get { parametersOf("screen_id") }` |

Requires from the app: `single<ILogger>`. Only the loggers you `get<...>()` in that composition are
ever created, because singles are lazy (except the createdAtStart interactor, which pulls `ILogger`).

```kotlin
single<ILogger> { get<ConsoleLogger>() + get<FirebaseLogger>() }   // `plus` from dev.dexkot.mobile.core.logger.composed
```

Failure mode without it: `startKoin` throws `InstanceCreationException: Could not create instance
for '[Singleton: 'ILoggerInteractor']'`, caused by `NoDefinitionFoundException` for `ILogger`.

## core-remoteconfig-di-koin

Package `dev.dexkot.mobile.core.remoteconfig.di.koin`. KMP (commonMain). Depends via `api` on
`core-remoteconfig`, `core-remoteconfig-firebase`, `core-remoteconfig-posthog`.

```kotlin
fun firebaseRemoteConfig(): Qualifier   // named("firebase_remote_config")
fun postHogRemoteConfig(): Qualifier    // named("posthog_remote_config")

fun remoteConfigModule(dispatcher: CoroutineDispatcher = Dispatchers.Default): Module
```

| Definition | Scope | Parameters |
|------------|-------|-----------|
| internal `CoroutineScope(SupervisorJob() + dispatcher)` | single, internal qualifier | – (no binding in this module consumes it in 0.27.0, so `dispatcher` currently has no observable effect) |
| `IRemoteConfigHandler` @ `firebaseRemoteConfig()` → `FirebaseRemoteConfigHandler(syncTimeOut, serializer)` | single | `Long` sync timeout in **ms**, `Json` |
| `IRemoteConfigHandler` @ `postHogRemoteConfig()` → `PostHogRemoteConfigHandler(serializer)` | single | `Json` |
| `IRemoteConfigProvider` → `RemoteConfigProvider(remoteConfigHandler = get())` | factory | – (uses the **unqualified** handler) |

Requires from the app: an unqualified `single<IRemoteConfigHandler>` (usually an alias to one of the
qualified ones) and the `Json` you pass as a parameter.

```kotlin
single<IRemoteConfigHandler> {
    get<IRemoteConfigHandler>(firebaseRemoteConfig()) { parametersOf(15_000L, get<Json>()) }
}
```

Why singles: each handler keeps its own `StateFlow` per key. With two instances, a `sync()` on one
would not update the flows observed through the other. Parameters are applied on first creation only.
Why the app chooses: the backend (Firebase vs PostHog, or your own `IRemoteConfigHandler`) is a
product decision. Failure mode without it: `InstanceCreationException` when `IRemoteConfigProvider`
is first resolved.

## core-work-di-koin

Package `dev.dexkot.mobile.core.work.di.koin`. KMP. Depends via `api` on `core-work`; the Android
variant uses `core-permission` internally (`implementation`).

```kotlin
val workFactoriesModule: Module          // commonMain; included by both backends
val coroutineWorkModule: Module          // commonMain; in-process backend (iOS, or light Android work)
val workManagerWorkModule: Module        // androidMain; WorkManager backend
fun workCoroutineScope(): Qualifier      // named("dev.dexkot.mobile.core.work.coroutineScope")
```

`workFactoriesModule`:

| Definition | Scope | Collects |
|------------|-------|----------|
| `IWorkTypeConfigFactory` → `WorkTypeConfigFactory` | single | `getAll<IWorkTypeConfigProvider>()` |
| `IWorkExecutorFactory` → `WorkExecutorFactory` | single | `getAll<IWorkExecutorProvider>()` |

`coroutineWorkModule` (includes `workFactoriesModule`):

| Definition | Scope | Optional inputs |
|------------|-------|-----------------|
| `CoroutineWorkEngine` | single | `CoroutineScope` @ `workCoroutineScope()` (else its own `SupervisorJob + Dispatchers.Default`), `WorkBackgroundGrace` (else no-op) |
| `IScheduleWorkUseCase` → `CoroutineScheduleWorkUseCase` | single | `WorkerCoroutineContextFactory` |
| `IObserveWorkStatusUseCase`, `ICancelWorkUseCase` | single | – |

`workManagerWorkModule` (includes `workFactoriesModule`, needs `androidContext`):

| Definition | Scope | Requires |
|------------|-------|----------|
| `WorkManager.getInstance(androidContext())` | single | – |
| `IWorkNotificationFactory` → `WorkNotificationFactory` | single | collects `getAll<IWorkNotificationProvider>()` |
| `GenericWorkRunner` | single | `IWorkExecutorFactory`, `IWorkNotificationFactory`, **`IPermissionChecker`**, optional `WorkerCoroutineContextFactory` |
| `IScheduleWorkUseCase` → `ScheduleWorkUseCase` | single | **`WorkerClassProvider`** |
| `ICancelWorkUseCase`, `IObserveWorkStatusUseCase`, `ConstraintChecker`, `ConstraintChangeListener` | single | – |

Requires from the app with WorkManager:

1. `startKoin { workManagerFactory() }` (`org.koin.androidx.workmanager.koin.workManagerFactory`) and,
   per Koin's WorkManager integration docs, WorkManager's default initializer disabled in the manifest.
2. An app-owned `CoroutineWorker` whose `doWork()` is `runner.run(this)`, registered with
   `worker { AppWorker(get(), get(), get()) }` (`org.koin.androidx.workmanager.dsl.worker`).
   The class is yours because WorkManager persists its fully-qualified name: renaming or moving it
   orphans work already enqueued.
3. `single<WorkerClassProvider> { WorkerClassProvider { AppWorker::class.java } }`.
4. `IPermissionChecker`: include `permissionModule()` (checks notification permission before foreground/completion notifications).
5. Per work type: an `IWorkExecutorProvider` and an `IWorkTypeConfigProvider` (matching `workType`
   strings), and optionally an `IWorkNotificationProvider`, each with `binds arrayOf(...)`.

Include exactly one backend per platform: both bind `IScheduleWorkUseCase`,
`IObserveWorkStatusUseCase` and `ICancelWorkUseCase`.

## core-permission-di-koin

Package `dev.dexkot.mobile.core.permission.di.koin`. Android. Depends on `core-permission` with
`implementation`: add `core-permission` to your app.

```kotlin
fun permissionPreferences(): Qualifier   // named("permission_preferences")
fun permissionModule(preferencesName: String = "dev.dexkot.permissions.preferences"): Module
```

| Definition | Scope | Notes |
|------------|-------|-------|
| `IPermissionChecker` → `PermissionChecker(androidContext())` | single | |
| `SharedPreferences` @ `permissionPreferences()` | single | `getSharedPreferences(preferencesName, MODE_PRIVATE)`; stores `permission_requested_*` tracking |
| `PermissionController` | factory, 3 parameters | `(activity: Activity, singlePermissionLauncher: ActivityResultLauncher<String>, multiplePermissionsLauncher: ActivityResultLauncher<Array<String>>)` |

In Compose you normally don't resolve `PermissionController` from Koin: use core-ui's
`rememberPermissionController(activity, preferences = koinInject(qualifier = permissionPreferences()))`,
which creates the launchers for you.

## core-ui-di-koin

Package `dev.dexkot.mobile.core.ui.di.koin`. Android. Depends on `core-ui` with `implementation`:
add `core-ui` to your app.

```kotlin
fun uiModule(preferencesName: String = "dev.dexkot.ui.preferences"): Module
```

| Definition | Scope | Notes |
|------------|-------|-------|
| `INavActionDispatcher` → `NavActionDispatcher(getAll<INavActionFactory<*>>())` | single | factory list captured on first resolution |
| `IUiPreferences` → `UiPreferences(SharedPreferences named preferencesName)` | single | |
| `IResourceProvider` → `ResourceProvider(context = get())` | factory | uses the `Context` registered by `androidContext(...)` |

Requires from the app: every `INavActionFactory` bound with `binds arrayOf(INavActionFactory::class)`
in modules loaded by `startKoin`. `INavActionLogDecorator`s are not consumed here; the Navigation host
collects them (`getAll<INavActionLogDecorator>()`).

## core-preferences

Package `dev.dexkot.mobile.core.preferences.di`. KMP. The one core that exposes its Koin module
directly (no `-di-koin` artifact).

```kotlin
expect val settingsFactoryModule: Module
// Android: single<Settings.Factory> { SharedPreferencesSettings.Factory(androidContext()) }
// iOS:     single<Settings.Factory> { NSUserDefaultsSettings.Factory() }
```

Define your stores by name in common code:

```kotlin
fun notesSettings() = named("notes_settings")
single<Settings>(notesSettings()) { get<Settings.Factory>().create("com.example.notes.settings") }
```

Koin de-duplicates identical `includes`, so several feature modules may include `settingsFactoryModule`.

## Verifying a graph in tests

Resolve the bindings you depend on in a plain JVM/Robolectric test. It catches missing app bindings
(logger, remote config handler, nav factories) in CI instead of in a user's session.

```kotlin
@RunWith(RobolectricTestRunner::class)
class AppGraphTest {
    @Test
    fun `navigation factories are registered`() {
        val app = koinApplication {
            androidContext(RuntimeEnvironment.getApplication())
            modules(UiModule)
        }
        val types = app.koin.getAll<INavActionFactory<*>>().map { it.actionType }
        assertTrue(NoteDetailNavAction::class in types)
        app.close()
    }
}
```

Avoid resolving `FirebaseRemoteConfigHandler` / `FirebaseLogger` in unit tests: they start their
native SDKs on construction. Bind a fake `IRemoteConfigHandler` / `ILogger` instead.
