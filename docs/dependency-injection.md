# Dependency injection (Koin)

DexKot's core libraries don't depend on any DI framework. For Koin users, each infrastructure ships a
small `-di-koin` artifact with a ready-made module, and your app composes the ones it needs. This guide
explains that split, what each module provides, what your app still has to bind, and the conventions
that keep a large Koin graph manageable.

Version 0.27.0, Koin 4.1.1.

- [The philosophy](#the-philosophy)
- [Setup](#setup)
- [The modules](#the-modules)
- [What your app binds](#what-your-app-binds)
- [Putting it together](#putting-it-together)
- [Conventions for your own modules](#conventions-for-your-own-modules)
- [Pitfalls](#pitfalls)
- [API reference](#api-reference)

## The philosophy

1. **Cores have no Koin.** `core-logger`, `core-remoteconfig`, `core-work`, `core-ui`... can be used with
   any DI or none.
2. **One `-di-koin` per infrastructure.** It is multiplatform when its core has an iOS
   implementation (logger, remote config, work) and Android-only otherwise (ui, permission).
   `core-preferences` is the exception: it exposes `settingsFactoryModule` itself.
3. **The app composes.** DexKot modules deliberately leave out bindings that are product decisions:
   which loggers to combine, which remote-config backend to use, which worker class WorkManager should
   run. Instead of picking a default for you, they make those an explicit extension point.

## Setup

```kotlin
// settings.gradle.kts
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
        maven("https://dexkot.github.io/maven")
    }
}
// app/build.gradle.kts
dependencies {
    implementation("dev.dexkot.mobile:core-logger-di-koin:0.27.0")
    implementation("dev.dexkot.mobile:core-remoteconfig-di-koin:0.27.0")
    implementation("dev.dexkot.mobile:core-work-di-koin:0.27.0")
    implementation("dev.dexkot.mobile:core-permission-di-koin:0.27.0")
    implementation("dev.dexkot.mobile:core-permission:0.27.0")   // not exposed by the -di-koin artifact
    implementation("dev.dexkot.mobile:core-ui-di-koin:0.27.0")
    implementation("dev.dexkot.mobile:core-ui:0.27.0")           // not exposed by the -di-koin artifact
    implementation("dev.dexkot.mobile:core-preferences:0.27.0")

    implementation("io.insert-koin:koin-android:4.1.1")
    implementation("io.insert-koin:koin-androidx-compose:4.1.1")
    implementation("io.insert-koin:koin-androidx-workmanager:4.1.1") // with workManagerWorkModule
}
```

Platform support: the logger, remote config and work modules (except `workManagerWorkModule`) and
`core-preferences` are KMP. Android: published. iOS: build from source (`publishToMavenLocal`).

## The modules

### `loggerModule(consoleLoggerTag = "Console", logicSourceComponent = "app")`

Registers `ConsoleLogger`, `FirebaseLogger` and `PostHogLogger` as separate singles (by concrete
type), the logic `ILoggerInteractor` (single, **created at start**, also installed as the default for
the `logger { }` DSL) and the UI `ILoggerInteractor` (factory taking the screen name as a parameter).
It does **not** register `ILogger`; you compose it.

### `remoteConfigModule(dispatcher = Dispatchers.Default)`

Registers `FirebaseRemoteConfigHandler` under `firebaseRemoteConfig()` and `PostHogRemoteConfigHandler`
under `postHogRemoteConfig()` (both singles, created with parameters) and `IRemoteConfigProvider`
(factory) that uses the **unqualified** `IRemoteConfigHandler`, which you bind. The `dispatcher`
parameter configures an internal scope that no binding uses in 0.27.0; you can leave the default.

### `workManagerWorkModule` / `coroutineWorkModule` / `workFactoriesModule`

Two interchangeable backends for background work; both include `workFactoriesModule`, which collects
your per-work-type providers. `workManagerWorkModule` (Android) runs work through WorkManager with
constraints and notifications; `coroutineWorkModule` (common) runs it in-process and is the one iOS
uses. Include one. `workCoroutineScope()` is an optional qualifier for an app-lifetime scope the
in-process engine should use.

### `permissionModule(preferencesName = "dev.dexkot.permissions.preferences")`

Registers `IPermissionChecker`, the `SharedPreferences` used to track which permissions were requested
(under `permissionPreferences()`), and a `PermissionController` factory.

### `uiModule(preferencesName = "dev.dexkot.ui.preferences")`

Registers the `INavActionDispatcher` (built from every `INavActionFactory` you bind), `IUiPreferences`
and `IResourceProvider`.

### `settingsFactoryModule`

A multiplatform `Settings.Factory` (SharedPreferences on Android, NSUserDefaults on iOS) from which you
create named key-value stores.

## What your app binds

| Module | You provide | Why it's yours |
|--------|-------------|----------------|
| `loggerModule` | `single<ILogger>` | Which analytics/crash backends you ship, and in what order. |
| `remoteConfigModule` | unqualified `single<IRemoteConfigHandler>`, plus the `Json` it needs | Which backend serves flags; your serializer settings. |
| `workManagerWorkModule` | worker class + `worker { }`, `single<WorkerClassProvider>`, `IPermissionChecker` (via `permissionModule`), per work type `IWorkExecutorProvider` + `IWorkTypeConfigProvider` (+ optional `IWorkNotificationProvider`) | WorkManager persists the worker's class name, so the class must be owned by the app; the work itself is yours. |
| `coroutineWorkModule` | per work type `IWorkExecutorProvider` + `IWorkTypeConfigProvider` | Same. |
| `uiModule` | every `INavActionFactory` | Your screens and actions. |

```kotlin
val LoggingModule = module {
    includes(loggerModule(consoleLoggerTag = "Notes"))
    single<ILogger> { get<ConsoleLogger>() + get<FirebaseLogger>() + get<PostHogLogger>() }
}

val RemoteConfigModule = module {
    includes(remoteConfigModule())
    single<IRemoteConfigHandler> {
        get<IRemoteConfigHandler>(firebaseRemoteConfig()) { parametersOf(15_000L, get<Json>()) }
    }
}
```

(`plus` is `dev.dexkot.mobile.core.logger.composed.plus`. The `15_000L` is the Firebase sync timeout in
milliseconds.)

## Putting it together

```kotlin
class NotesApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        FirebaseApp.initializeApp(this)   // SDKs first: loggers are created during startKoin
        startKoin {
            androidContext(this@NotesApplication)
            workManagerFactory()
            modules(AppModule)
        }
    }
}

val AppModule = module {
    includes(CoreModule, LoggingModule, RemoteConfigModule, WorkModule, NotesModule, UiModule)
}

val CoreModule = module {
    includes(settingsFactoryModule)
    single { Json { ignoreUnknownKeys = true } }
}

class NotesWorker(context: Context, params: WorkerParameters, private val runner: GenericWorkRunner) :
    CoroutineWorker(context, params) {
    override suspend fun doWork(): Result = runner.run(this)
}

val WorkModule = module {
    includes(workManagerWorkModule)
    worker { NotesWorker(context = get(), params = get(), runner = get()) }
    single<WorkerClassProvider> { WorkerClassProvider { NotesWorker::class.java } }
    factory { SyncNotesConfigProvider() } binds arrayOf(IWorkTypeConfigProvider::class)
    factory { SyncNotesExecutorProvider(syncNotes = get()) } binds arrayOf(IWorkExecutorProvider::class)
}

val UiModule = module {
    includes(
        uiModule(preferencesName = "com.example.notes.ui.preferences"),
        permissionModule(preferencesName = "com.example.notes.permissions"),
        CommonsNavigationModule,
        LauncherModule, NoteListModule, NoteDetailModule,
    )
}
```

With `workManagerFactory()`, also remove WorkManager's default initializer in the manifest
(`androidx.work.WorkManagerInitializer` meta-data with `tools:node="remove"` inside
`androidx.startup.InitializationProvider`), as Koin's WorkManager integration requires.

## Conventions for your own modules

### A module per component, composed with `includes()`

Each entity, screen or feature gets its own module. Parents include children, and `startKoin` only
lists top-level aggregates. Sub-modules inside a library are `internal val`, so the library exposes
exactly one entry point.

```kotlin
val NotesModule = module { includes(NoteModule, NotesUseCaseModule) }
internal val NoteModule = module { /* ... */ }
```

### `single` for state, `factory` for logic

```kotlin
internal val NoteModule = module {
    single<INoteLocalDataSource> { NoteLocalDataSource(database = get(notesDatabase())) }
    factory<INoteRepository> { NoteRepository(localDataSource = get()) }
    factory<IGetNoteUseCase> { GetNoteUseCase(repository = get()).withLogging() }
}
```

Data sources, databases, preference stores and coroutine scopes are `single`: two instances would mean
two caches or two connections to the same file. Repositories, use cases, nav factories and decorators
hold no state and are `factory`.

### ViewModels: generic type plus a qualifier

```kotlin
val NoteDetailModule = module {
    factory { NoteDetailNavActionFactory() } binds arrayOf(INavActionFactory::class)
    factory { NoteDetailNavAction_LogDecorator() } binds arrayOf(INavActionLogDecorator::class)

    viewModel<AndroidViewModel<NoteDetailModel, NoteDetailIntention>>(noteDetailViewModel()) { parameters ->
        NoteDetailViewModel(getNote = get(), savedStateHandle = get())
            .withLogging(logger = get(), sourceComponent = parameters.get())
    }
}

internal fun noteDetailViewModel() = named("note_detail_viewmodel")
```

Every ViewModel is registered as `AndroidViewModel<Model, Intention>`; generics are erased at runtime,
so the qualifier is what tells them apart. `parameters.get()` is the screen identifier the Route
passes in; `withLogging` is generated by `@AutoLogViewModel`.

### Qualifiers live behind functions

```kotlin
fun notesDatabase() = named("notes_database")                        // used by other modules → public
internal fun noteDetailViewModel() = named("note_detail_viewmodel")  // used only here → internal
```

A raw `named("...")` string at a call site compiles even when misspelled and fails at runtime; a
function can't be misspelled. Use snake_case values.

### Multi-binding

Features contribute implementations with `binds arrayOf(Interface::class)`; a collector gathers them
with `getAll<Interface>()`. This is how nav action factories, log decorators and work providers are
registered without a central list.

## Pitfalls

| Symptom | Cause | Fix |
|---------|-------|-----|
| `startKoin` throws `InstanceCreationException ... 'ILoggerInteractor'` | No `single<ILogger>`; the logic interactor is created at start | Bind `single<ILogger>`; read the nested cause. |
| Crash on start inside Firebase/PostHog | Loggers created before the SDK was initialised | Initialise SDKs before `startKoin`. |
| `InstanceCreationException` when a screen opens (`IRemoteConfigProvider`) | No unqualified `IRemoteConfigHandler` | Bind one (see above). |
| Changing the remote config timeout has no effect | Handlers are singles; parameters apply on first creation only | Pass the value in the single place you create the handler. |
| Work never runs / wrong backend | Both work backends included; last one wins | Include exactly one per platform. |
| `IllegalStateException: No INavActionFactory registered` | Factory not bound, or in a module loaded with `loadKoinModules` after the dispatcher was created | Bind it and load it in `startKoin`. Work factories behave the same way. |
| Users lose permission tracking / UI prefs after adopting DexKot | Default preference names differ from yours | Pass your existing names to `permissionModule` / `uiModule`. |
| Unresolved `core-ui` / `core-permission` classes | Their `-di-koin` artifacts depend on them with `implementation` | Add the core artifacts explicitly. |

Tip: a Robolectric test that builds a `koinApplication` with your modules and resolves the key
bindings (`IRemoteConfigProvider`, `INavActionDispatcher`, `getAll<INavActionFactory<*>>()`) catches
most of these in CI. Bind fakes for Firebase/PostHog-backed types, which start their SDKs on creation.

## API reference

```kotlin
// dev.dexkot.mobile.core.logger.di.koin — core-logger-di-koin (KMP)
fun loggerModule(consoleLoggerTag: String = "Console", logicSourceComponent: String = "app"): Module
//   single ConsoleLogger, single FirebaseLogger, single PostHogLogger
//   single<interactors.ddl.ILoggerInteractor>(createdAtStart = true)
//   factory<interactors.ui.ILoggerInteractor> { (sourceComponent: String) }

// dev.dexkot.mobile.core.remoteconfig.di.koin — core-remoteconfig-di-koin (KMP)
fun firebaseRemoteConfig(): Qualifier   // named("firebase_remote_config")
fun postHogRemoteConfig(): Qualifier    // named("posthog_remote_config")
fun remoteConfigModule(dispatcher: CoroutineDispatcher = Dispatchers.Default): Module
//   single<IRemoteConfigHandler>(firebaseRemoteConfig()) { (syncTimeOut: Long, serializer: Json) }
//   single<IRemoteConfigHandler>(postHogRemoteConfig()) { (serializer: Json) }
//   factory<IRemoteConfigProvider> — uses the unqualified IRemoteConfigHandler

// dev.dexkot.mobile.core.work.di.koin — core-work-di-koin (KMP)
val workFactoriesModule: Module     // single IWorkTypeConfigFactory, single IWorkExecutorFactory (collect providers)
val coroutineWorkModule: Module     // CoroutineWorkEngine + Coroutine*WorkUseCase singles
val workManagerWorkModule: Module   // Android: WorkManager, IWorkNotificationFactory, GenericWorkRunner, *WorkUseCase singles
fun workCoroutineScope(): Qualifier // named("dev.dexkot.mobile.core.work.coroutineScope")

// dev.dexkot.mobile.core.permission.di.koin — core-permission-di-koin (Android)
fun permissionPreferences(): Qualifier  // named("permission_preferences")
fun permissionModule(preferencesName: String = "dev.dexkot.permissions.preferences"): Module
//   single<IPermissionChecker>, single<SharedPreferences>(permissionPreferences()),
//   factory { (activity: Activity, single: ActivityResultLauncher<String>, multiple: ActivityResultLauncher<Array<String>>) -> PermissionController }

// dev.dexkot.mobile.core.ui.di.koin — core-ui-di-koin (Android)
fun uiModule(preferencesName: String = "dev.dexkot.ui.preferences"): Module
//   single<INavActionDispatcher>, single<IUiPreferences>, factory<IResourceProvider>

// dev.dexkot.mobile.core.preferences.di — core-preferences (KMP)
expect val settingsFactoryModule: Module   // single<Settings.Factory>

// dev.dexkot.mobile.core.work.domain.worker — core-work (Android)
fun interface WorkerClassProvider { fun workerClass(): Class<out ListenableWorker> }
```
