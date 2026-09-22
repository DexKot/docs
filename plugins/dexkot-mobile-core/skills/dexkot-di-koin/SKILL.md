---
name: dexkot-di-koin
description: "Koin dependency injection for apps built on DexKot mobile-core (dev.dexkot.mobile). Use whenever the task involves startKoin, a Koin module, or wiring DexKot infrastructure: loggerModule / single ILogger, remoteConfigModule / firebaseRemoteConfig() / postHogRemoteConfig() / IRemoteConfigHandler, workManagerWorkModule / coroutineWorkModule / WorkerClassProvider / workCoroutineScope(), permissionModule / permissionPreferences(), uiModule / INavActionDispatcher, settingsFactoryModule; registering ViewModels, data sources, repositories, use cases or NavActionFactories; single vs factory vs viewModel; named qualifiers; includes(); binds arrayOf / getAll multi-binding; or debugging InstanceCreationException / NoDefinitionFoundException in a DexKot project."
---

# DexKot dependency injection with Koin

DexKot's core libraries have **no Koin dependency**. Each infrastructure that needs wiring ships a
separate `-di-koin` artifact with a module function, and the **app composes** them. Where a choice
is genuinely the app's (which loggers to combine, which remote-config backend, which worker class),
the DexKot module leaves that binding out on purpose and the app provides it. This keeps the cores
usable with any DI, lets iOS share the KMP modules, and stops the library from making product
decisions for you.

Version covered: **0.27.0**, Koin **4.1.1**. Per-module bindings and requirements:
`references/modules.md`. A complete app graph (Application, logging, remote config, WorkManager,
navigation, screens): `references/app-setup.md`.

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
// app/build.gradle.kts — pick the ones you use
dependencies {
    implementation("dev.dexkot.mobile:core-logger-di-koin:0.27.0")        // KMP; brings core-logger(-firebase/-posthog) via api
    implementation("dev.dexkot.mobile:core-remoteconfig-di-koin:0.27.0")  // KMP; brings core-remoteconfig(-firebase/-posthog) via api
    implementation("dev.dexkot.mobile:core-work-di-koin:0.27.0")          // KMP; brings core-work via api
    implementation("dev.dexkot.mobile:core-permission-di-koin:0.27.0")    // Android
    implementation("dev.dexkot.mobile:core-permission:0.27.0")            //   ↳ its core is `implementation`: add it yourself
    implementation("dev.dexkot.mobile:core-ui-di-koin:0.27.0")            // Android
    implementation("dev.dexkot.mobile:core-ui:0.27.0")                    //   ↳ its core is `implementation`: add it yourself
    implementation("dev.dexkot.mobile:core-preferences:0.27.0")           // KMP; exposes settingsFactoryModule directly

    implementation("io.insert-koin:koin-android:4.1.1")
    implementation("io.insert-koin:koin-androidx-compose:4.1.1")          // koinViewModel / koinInject
    implementation("io.insert-koin:koin-androidx-workmanager:4.1.1")      // only with workManagerWorkModule
}
```

Platform support: `core-logger-di-koin`, `core-remoteconfig-di-koin`, `core-work-di-koin`
(`workFactoriesModule`, `coroutineWorkModule`) and `core-preferences` are KMP. Android: published.
iOS: build from source (`publishToMavenLocal`). `core-ui-di-koin`, `core-permission-di-koin` and
`workManagerWorkModule` are Android only.

## Entry points at a glance

| Artifact | Entry point | The app must also bind |
|----------|-------------|------------------------|
| core-logger-di-koin | `loggerModule(consoleLoggerTag = "Console", logicSourceComponent = "app")` | `single<ILogger>` (the composition) |
| core-remoteconfig-di-koin | `remoteConfigModule(dispatcher = Dispatchers.Default)`; qualifiers `firebaseRemoteConfig()`, `postHogRemoteConfig()` | unqualified `single<IRemoteConfigHandler>`; a `Json` to pass in |
| core-work-di-koin | `workManagerWorkModule` (Android) **or** `coroutineWorkModule`; both include `workFactoriesModule`; qualifier `workCoroutineScope()` | per work type: `IWorkExecutorProvider` + `IWorkTypeConfigProvider`; WorkManager also: worker + `WorkerClassProvider` + `IPermissionChecker` |
| core-permission-di-koin | `permissionModule(preferencesName = "dev.dexkot.permissions.preferences")`; qualifier `permissionPreferences()` | nothing (needs `androidContext`) |
| core-ui-di-koin | `uiModule(preferencesName = "dev.dexkot.ui.preferences")` | your `INavActionFactory`s (multi-bound) |
| core-preferences | `settingsFactoryModule` | your named `Settings` stores |

## The two bindings DexKot leaves to you

```kotlin
import dev.dexkot.mobile.core.logger.ConsoleLogger
import dev.dexkot.mobile.core.logger.FirebaseLogger
import dev.dexkot.mobile.core.logger.ILogger
import dev.dexkot.mobile.core.logger.PostHogLogger
import dev.dexkot.mobile.core.logger.composed.plus
import dev.dexkot.mobile.core.logger.di.koin.loggerModule
import dev.dexkot.mobile.core.remoteconfig.di.koin.firebaseRemoteConfig
import dev.dexkot.mobile.core.remoteconfig.di.koin.remoteConfigModule
import dev.dexkot.mobile.core.remoteconfig.handler.IRemoteConfigHandler
import kotlinx.serialization.json.Json
import org.koin.core.parameter.parametersOf
import org.koin.dsl.module

val LoggingModule = module {
    includes(loggerModule(consoleLoggerTag = "Notes"))
    // Which backends, in which order, wrapped how: the app's decision.
    single<ILogger> { get<ConsoleLogger>() + get<FirebaseLogger>() + get<PostHogLogger>() }
}

val RemoteConfigModule = module {
    includes(remoteConfigModule())
    // Pick the backend: the unqualified handler is what IRemoteConfigProvider uses.
    single<IRemoteConfigHandler> {
        get<IRemoteConfigHandler>(firebaseRemoteConfig()) {
            parametersOf(15_000L /* syncTimeOut, ms */, get<Json>())
        }
    }
}
```

The PostHog variant is `get<IRemoteConfigHandler>(postHogRemoteConfig()) { parametersOf(get<Json>()) }`.

## Project conventions

**Module hierarchy.** One module per component (entity, screen, feature); parents `includes()`
children, and `startKoin` only sees top-level aggregates. Sub-modules inside a library are
`internal val`. This makes a feature a single include and keeps its internals private.

```kotlin
val NotesModule = module { includes(NoteModule, TagModule, NotesUseCaseModule) }   // aggregate, public
internal val NoteModule = module { /* data source + repository */ }                 // entity, internal
```

**Scopes.**

| Scope | For | Why |
|-------|-----|-----|
| `single` | Data sources, databases, SharedPreferences / `Settings`, `CoroutineScope`s, anything with state or a cache | Two instances would mean two caches, two DB connections or two in-memory views of the same file. |
| `factory` | Repositories, use cases, NavActionFactories, `_LogDecorator`s, mappers | Stateless logic; cheap to create, nothing to share. |
| `viewModel` | Every screen ViewModel, typed `AndroidViewModel<Model, Intention>` with a qualifier | Lifecycle is owned by the back stack entry; the qualifier disambiguates the generic type. |

```kotlin
internal val NoteModule = module {
    single<INoteLocalDataSource> { NoteLocalDataSource(database = get()) }
    factory<INoteRepository> { NoteRepository(localDataSource = get(), remoteDataSource = get()) }
    factory<IGetNoteUseCase> { GetNoteUseCase(repository = get()).withLogging() } // @AutoLogUseCase
}
```

**Named qualifiers, always behind a function.** Never write `named("...")` at call sites: a typo
compiles and fails at runtime. Declare `internal fun` when only this module uses it (ViewModels) and
public `fun` when other modules resolve it (a shared database, a shared scope). Use snake_case values.

```kotlin
fun notesDatabase() = named("notes_database")              // shared → public
internal fun noteDetailViewModel() = named("note_detail_viewmodel") // module-scoped → internal
```

**ViewModels.** Every ViewModel is registered as its generic base type with its qualifier; without
the qualifier all `AndroidViewModel<*, *>` definitions collide on type erasure.

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

`parameters.get()` is the Route identifier passed by `koinViewModel(...) { parametersOf(IDENTIFIER) }`
(see dexkot-navigation). `withLogging` is generated by `@AutoLogViewModel` (see dexkot-logging); drop
it if the ViewModel is not annotated. `viewModel` is `org.koin.core.module.dsl.viewModel`.

**Multi-binding.** Plug-in style contributions (nav action factories, log decorators, work providers)
are registered by the feature that owns them with `binds arrayOf(Interface::class)` and collected by
the consumer with `getAll<Interface>()`. Decentralised registration means adding a screen never
touches a central list. Import `org.koin.dsl.binds`.

## Application startup

```kotlin
class NotesApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        // SDKs before Koin: loggerModule builds the logic interactor (and so your ILogger, including
        // FirebaseLogger / PostHogLogger) during startKoin.
        FirebaseApp.initializeApp(this)
        // initialise the PostHog SDK here if you use PostHogLogger / PostHogRemoteConfigHandler

        startKoin {
            androidContext(this@NotesApplication)   // required by uiModule, permissionModule, settingsFactoryModule, WorkManager
            workManagerFactory()                    // only with workManagerWorkModule
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

val UiModule = module {
    includes(
        uiModule(preferencesName = "com.example.notes.ui.preferences"),
        permissionModule(preferencesName = "com.example.notes.permissions"),
        CommonsNavigationModule,
        LauncherModule, NoteListModule, NoteDetailModule,
    )
}
```

`WorkModule` (worker, `WorkerClassProvider`, providers) is in `references/app-setup.md`.

## Gotchas

- **Missing `single<ILogger>` fails at `startKoin`**, because `loggerModule`'s logic interactor is
  `createdAtStart`. The error names `'ILoggerInteractor'` (`InstanceCreationException`); read the
  nested cause (`NoDefinitionFoundException` for `ILogger`).
- **Initialise Firebase / PostHog before `startKoin`** for the same reason: the loggers are created then.
- **Missing unqualified `IRemoteConfigHandler`** fails later, when `IRemoteConfigProvider` (a `factory`)
  is first resolved, often from the Navigation host. Test it with a Koin resolution test.
- **Remote config handlers are `single`s with state.** The injected `syncTimeOut` (milliseconds) and
  `Json` apply only on first creation; resolving the same qualifier again with other parameters
  returns the original instance and ignores them.
- **One work backend per platform.** `coroutineWorkModule` and `workManagerWorkModule` bind the same
  use-case interfaces; including both makes the last one loaded win silently (or fails if you disabled overrides).
- **WorkManager backend requirements:** `startKoin { workManagerFactory() }`, your own worker class
  registered with `worker { }` that delegates to `GenericWorkRunner`, `single<WorkerClassProvider>`,
  an `IPermissionChecker` (include `permissionModule`), and per work type an `IWorkExecutorProvider`
  and `IWorkTypeConfigProvider` (plus an optional `IWorkNotificationProvider` for progress
  notifications), each with `binds arrayOf(...)`. Details: `references/modules.md#core-work-di-koin`.
- **Collectors snapshot at first resolution.** `INavActionDispatcher`, `IWorkExecutorFactory`,
  `IWorkTypeConfigFactory` and `IWorkNotificationFactory` are `single`s built from `getAll`. Anything
  registered later with `loadKoinModules` is invisible to them; load all modules in `startKoin`.
- **Keep your existing preference file names.** `permissionModule(preferencesName)` and
  `uiModule(preferencesName)` default to DexKot names. If your app already stored permission tracking
  or UI prefs, pass the old name or users lose them on upgrade.
- **`"permission_preferences"`** (the value behind `permissionPreferences()`) is effectively public
  API; resolve it through the function, never the string.
- **Transitive dependencies:** `core-ui-di-koin` and `core-permission-di-koin` depend on their cores with
  `implementation`, so add `core-ui` / `core-permission` yourself. `core-logger-di-koin` and
  `core-remoteconfig-di-koin` expose Firebase and PostHog artifacts via `api`, so those SDKs come along.
- **`factory` for multi-bound items does not make them per-call** when the collector is a `single`:
  the dispatcher keeps the instances it got. Read changing values inside methods, not constructors.

## Related skills

- **dexkot-navigation**: Routes, NavAction triples, the Navigation host that consumes this graph.
- **dexkot-viewmodel**: what goes inside the `viewModel { }` definition.
- **dexkot-logging**: `ILogger` composition, `@AutoLog*` and generated `withLogging()`.
- **dexkot-data-layer**: data sources, repositories and `withTransaction {}` behind the `single`/`factory` rules.
- **dexkot-infra**: remote config, permissions, preferences and background work APIs.
