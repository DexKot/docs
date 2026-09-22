# Getting started

This guide takes an Android app from zero to one working feature built on DexKot mobile-core. The
feature is a notes list: it loads notes from a local database, shows them in Compose, and navigates
to a note detail screen. Each step links to the guide that covers it in depth. The code in those
guides uses the same notes domain, so you can follow along.

## 1. Add the repository and dependencies

```kotlin
// settings.gradle.kts
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
        maven("https://dexkot.github.io/maven")
    }
}
```

```kotlin
// app/build.gradle.kts
plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
    id("org.jetbrains.kotlin.plugin.compose")
    id("org.jetbrains.kotlin.plugin.parcelize")
    id("com.google.devtools.ksp")            // AutoLog code generation
}

dependencies {
    val dexkot = "0.27.0"
    implementation("dev.dexkot.mobile:core-foundation:$dexkot")
    implementation("dev.dexkot.mobile:core-ui:$dexkot")
    implementation("dev.dexkot.mobile:core-ui-di-koin:$dexkot")
    implementation("dev.dexkot.mobile:core-permission:$dexkot")
    implementation("dev.dexkot.mobile:core-permission-di-koin:$dexkot")
    implementation("dev.dexkot.mobile:core-logger-di-koin:$dexkot")
    implementation("dev.dexkot.mobile:core-annotation:$dexkot")
    ksp("dev.dexkot.mobile:core-ksp-processor:$dexkot")

    // Only if you use a local database with transactions
    implementation("dev.dexkot.mobile:core-database-sqldelight:$dexkot")
}
```

`core-ui` and `core-ksp-processor` must be on the same version, because the generated code calls
into `core-ui`. The Koin modules depend on their cores with `implementation`, so declare the cores
you use (`core-ui`, `core-permission`) yourself. The full list of AndroidX and Koin artifacts your app
needs is in [Presentation → Setup](presentation.md#setup) and
[Dependency injection](dependency-injection.md).

## 2. Understand the layers

```
Screen (Compose)  ──Intention──►  ViewModel  ──►  UseCase  ──►  Repository  ──►  DataSource
                  ◄──Model─────   (Resource)      (ResultOf)    (ResultOf)       (ResultOf, never throws)
```

- **Data and domain** return `ResultOf<T>`: either `Success` or `Failure` with a typed `ErrorEntity`.
  Nothing below the ViewModel throws. See [Results and errors](results-and-errors.md).
- **The ViewModel** turns `ResultOf` into `Resource<T>` (`None`, `Loading`, `Loaded` or `Error`),
  exposes an immutable `Model`, and receives `Intention`s. See [Presentation](presentation.md).
- **Navigation** is data. The ViewModel emits an `INavAction`, and a factory resolves it to an
  executor that navigates. See [Navigation](navigation.md).

## 3. Build the data layer

Create, bottom up:

1. A domain model (`Note`) and, when an operation can fail in an expected way, a `BusinessError`
   subclass (`NoteNotFound`).
2. `INoteLocalDataSource` and its `internal` implementation. Wrap every database call in
   `transactionFactory.withTransaction { }`, catch exceptions and return `ResultOf.Failure`.
3. `INoteRepository` and its implementation, a thin layer over the data sources.
4. One use case per action: `IGetNotesUseCase` and its `internal` implementation, annotated
   `@AutoLogUseCase`.

The [Data layer](data-layer.md) guide walks through each file, the SQLDelight setup and the
transaction pitfalls. The most important pitfall: returning a `Failure` from `withTransaction` does
**not** roll back. Use `rollback()` or `rollBackIfFailure()`.

## 4. Build the screen

A screen is a small set of files in one feature package:

```
notes/list/
├── viewmodel/  NotesModel, NotesIntention, NotesViewModelState, NotesViewModel
├── screen/     NotesScreenState, NotesScreen, content/NotesContent (+ UiStates)
├── route/      NotesRoute, NotesNavAction, NotesNavActionFactory, NotesNavActionExecutor
└── di/         NotesModule
```

1. **Model and Intention.** The Model is a `data class` whose async fields are `Resource<T>`. The
   Intention is a `sealed interface` named after what the user wants: `GoNoteDetail(noteId)`, `Back`
   or `Refresh`, never `ItemClicked`. Annotate each subtype with `@AutoLogIntention`.
2. **ViewModelState.** A small class that wraps `SavedStateHandle`. It reads route parameters and
   creates process-death-safe flows with `savedStateHandle.getMutableStateFlow(key, initial)`.
3. **ViewModel.** Extend `AndroidViewModel<NotesModel, NotesIntention>`, annotate it
   `@AutoLogViewModel`, call use cases and convert their results to `Resource`, and navigate with
   `emit(NoteDetailNavAction(noteId))`.
4. **ScreenState and Screen.** A `@Parcelize @Stable @UiState` class holds UI-only state (dialogs,
   text fields) with `fieldMutableStateOf`, so it survives process death. The `Screen` composable
   receives `(screenState, onIntention)` and never the ViewModel.
5. **Route.** A singleton `object NotesRoute : Route<NotesModel, NotesIntention>(...)` declares the
   route pattern, arguments, ViewModel factory and screen content. It also owns the
   `INavigator.goNotes(...)` and `SavedStateHandle.getProjectId()` extensions.
6. **NavAction triple.** For every destination, write a `NavAction`, a `NavActionFactory` and a
   `NavActionExecutor`.

Complete code for every file: [Presentation](presentation.md) (ViewModel, Resource, ScreenState,
Screen) and [Navigation](navigation.md) (Route, NavAction triple, arguments, results).

## 5. Wire it with Koin

```kotlin
// notes/list/di/NotesModule.kt
val NotesModule = module {
    factory { NotesNavActionFactory() } binds arrayOf(INavActionFactory::class)
    factory { NotesNavAction_LogDecorator() } binds arrayOf(INavActionLogDecorator::class)  // generated by KSP
    viewModel<AndroidViewModel<NotesModel, NotesIntention>>(notesViewModel()) { parameters ->
        NotesViewModel(getNotes = get(), savedStateHandle = get())
            .withLogging(logger = get(), sourceComponent = parameters.get())              // generated by KSP
    }
}

internal fun notesViewModel() = named("notes_viewmodel")
```

At the application level, include `uiModule()`, `permissionModule()` and `loggerModule()`, and bind the
one thing DexKot leaves to you: `single<ILogger>`, the composition of your logging backends. The
[Dependency injection](dependency-injection.md) guide has the complete `Application` and explains
every binding each module expects from you.

## 6. Host the navigation

In your `FragmentActivity`, create the navigator with `rememberNavigator(activity, deepLinkScheme = ...)`
and call `Navigation(graphs = ..., navigator = ..., dispatcher = ..., ...)`. Wrap it in
`Preferences(uiPreferences = ...)`: the host provides the logger, permission controller and remote
config to every screen, but not the UI preferences. See
[Navigation → the root host](navigation.md).

## 7. Add logging

With `@AutoLogUseCase`, `@AutoLogViewModel`, `@AutoLogIntention` and `@AutoLogNavAction` in place,
KSP generates decorators that log every use case call, user intention and navigation. Your business
code never calls the logger. Pick the backends (console, Firebase, PostHog) when you bind
`single<ILogger>`. See [Logging and AutoLog](logging.md).

## Where to go next

- [Background work](background-work.md): WorkManager on Android, in-process coroutines on iOS.
- [Remote config](remote-config.md): typed flags and values from Firebase or PostHog.
- [Permissions](permissions.md): runtime permissions with request tracking and Compose state.

If you use Claude Code, install the skills from this repository
(`/plugin marketplace add DexKot/docs`). Claude will follow these patterns when it writes
DexKot code for you.
