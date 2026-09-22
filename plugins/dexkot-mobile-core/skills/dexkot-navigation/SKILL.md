---
name: dexkot-navigation
description: "Compose navigation with DexKot mobile-core (dev.dexkot.mobile core-ui, core-ui-di-koin). Use whenever the task is to create a screen destination, a Route or Graph, a NavAction + NavActionFactory + NavActionExecutor triple, navigate between screens, pass path or query parameters (createRoute, queryParameter), read arguments from SavedStateHandle, return a result with popBackStack(result) / onResume(result), launch an Activity for result (navigateForResult), open a URL or the app settings, set up the root FragmentActivity.Navigation host, rememberNavigator, deep links / deepLinkScheme, splash removal, or INavActionDispatcher / INavigator / @AutoLogNavAction wiring. Use it even if the user just says \"go to screen X\" or \"add navigation\" in a DexKot project."
---

# DexKot navigation

DexKot builds navigation on top of androidx Navigation Compose, but ViewModels never see it. A
ViewModel only **emits a data object** (`INavAction`). The framework resolves that object into an
executor through a factory registered in Koin, and the executor performs the navigation on an
`INavigator`. Every destination is a `Route` object that owns its URI pattern, arguments, ViewModel
factory and screen content, and every in-app navigation goes through a deep link URI.

The payoff: ViewModels are unit-testable without Android (assert the emitted action), navigation
decisions (A/B tests, "has the user seen onboarding?") live in one injectable place, analytics for
every navigation is generated, and internal and external deep links share a single routing path.

Version covered: **0.27.0**. Android only (core-ui is an Android library).
API signatures: `references/api.md`. Longer recipes (host activity, results, activity results,
query params, conditional routing, commons): `references/recipes.md`.

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
    implementation("dev.dexkot.mobile:core-ui:0.27.0")
    implementation("dev.dexkot.mobile:core-ui-di-koin:0.27.0")        // uiModule(): dispatcher + UI prefs
    implementation("dev.dexkot.mobile:core-permission-di-koin:0.27.0") // permission prefs for the host
    // core-ui uses these internally with `implementation`, so your code needs them on its own classpath:
    implementation("androidx.navigation:navigation-compose:2.9.5")      // navArgument, navDeepLink, NavType
    implementation("androidx.fragment:fragment-ktx:1.8.7")              // FragmentActivity
    implementation("io.insert-koin:koin-androidx-compose:4.1.1")        // koinViewModel, koinInject
    // Optional: generated navigation analytics (see dexkot-logging)
    implementation("dev.dexkot.mobile:core-annotation:0.27.0")
    ksp("dev.dexkot.mobile:core-ksp-processor:0.27.0")
}
```

## The dispatch pipeline

```
ViewModel.emit(NoteDetailNavAction(noteId = 42))          // AndroidViewModel.emit, protected
  → viewModel.nav: StateFlow<Event<INavAction>?>
  → NavigationSubscriber (per screen, added by Navigation)
  → INavActionDispatcher.dispatch(action)                  // wrapped per screen with log decorators
      → INavActionFactory<NoteDetailNavAction>.resolve(action): INavActionExecutor
  → executor(navigator)                                    // INavActionExecutor.invoke(INavigator)
  → navigator.navigate("notes/42")                         // Route's INavigator extension
```

`NavigationSubscriber` also forwards the screen lifecycle: `onCreate(args)` once on ON_CREATE,
`onResume(result)` on ON_RESUME, `onStop()` on ON_STOP.

## Build a destination, step by step

Package layout per screen (the triple and the Route live together in `route/`):

```
notes/detail/
├── route/   NoteDetailRoute.kt, NoteDetailNavAction.kt, NoteDetailNavActionFactory.kt, NoteDetailNavActionExecutor.kt
├── di/      NoteDetailModule.kt
├── viewmodel/  NoteDetailViewModel.kt, NoteDetailModel.kt, NoteDetailIntention.kt
└── screen/  NoteDetailScreen.kt
```

### 1. The Route

```kotlin
package com.example.notes.navigation.notes.detail.route

import androidx.lifecycle.SavedStateHandle
import androidx.navigation.NavType
import androidx.navigation.navArgument
import com.example.notes.navigation.notes.detail.di.noteDetailViewModel
import com.example.notes.navigation.notes.detail.screen.NoteDetailScreen
import com.example.notes.navigation.notes.detail.viewmodel.NoteDetailIntention
import com.example.notes.navigation.notes.detail.viewmodel.NoteDetailModel
import dev.dexkot.mobile.core.ui.compose.transitions.ImitationOfActivities
import dev.dexkot.mobile.core.ui.navigation.graph.Route
import dev.dexkot.mobile.core.ui.navigation.navigator.INavigator
import dev.dexkot.mobile.core.ui.viewmodel.android.AndroidViewModel
import org.koin.androidx.compose.koinViewModel
import org.koin.core.parameter.parametersOf

object NoteDetailRoute : Route<NoteDetailModel, NoteDetailIntention>(
    identifier = NoteDetailRoute.IDENTIFIER,
    route = NoteDetailRoute.NOTE_DETAIL_ROUTE,
    arguments = listOf(
        navArgument(NoteDetailRoute.NOTE_ID_PARAM) { type = NavType.LongType }
    ),
    // deepLinks defaults to listOf(navDeepLink { uriPattern = route }), which is what you want here.
    viewModelFactory = {
        koinViewModel<AndroidViewModel<NoteDetailModel, NoteDetailIntention>>(noteDetailViewModel()) {
            parametersOf(NoteDetailRoute.IDENTIFIER)
        }
    },
    screenContent = { viewModel -> NoteDetailScreen(viewModel = viewModel) },
    transitions = ImitationOfActivities,
) {
    // `const` matters: these are read inside the super-constructor call, before the object is initialised.
    private const val IDENTIFIER = "note_detail"
    private const val NOTE_ID_PARAM = "note_id"
    private const val NOTE_DETAIL_ROUTE = "notes/{$NOTE_ID_PARAM}"

    fun INavigator.goNoteDetail(noteId: Long) {
        navigate(uri = createDeepLink(noteId))
    }

    fun SavedStateHandle.getNoteId(): Long = get<Long>(NOTE_ID_PARAM) ?: 0L

    fun createDeepLink(noteId: Long): String =
        NOTE_DETAIL_ROUTE.replace("{$NOTE_ID_PARAM}", noteId.toString())
}
```

Why this shape: the parameter names are private, so nothing outside the Route can build a wrong URI
or read a wrong key. The `INavigator` and `SavedStateHandle` extensions are the only public way in and
out. `identifier` names the screen for logging (`loggerFactory(route.identifier)`) and is passed to the
ViewModel definition through `parametersOf` so its generated logging decorator knows its source.

### 2. The NavAction triple

```kotlin
// NoteDetailNavAction.kt — data only: what the ViewModel knows
@AutoLogNavAction(identifier = "go_note_detail", params = [LogParam(logName = "note_id", paramName = "noteId")])
data class NoteDetailNavAction(val noteId: Long) : INavAction

// NoteDetailNavActionFactory.kt — the injection point
internal class NoteDetailNavActionFactory : INavActionFactory<NoteDetailNavAction> {
    override val actionType = NoteDetailNavAction::class
    override fun resolve(action: NoteDetailNavAction): INavActionExecutor = NoteDetailNavActionExecutor(action)
}

// NoteDetailNavActionExecutor.kt — the only code that touches INavigator
internal class NoteDetailNavActionExecutor(
    private val action: NoteDetailNavAction,
) : INavActionExecutor {
    override operator fun invoke(navigator: INavigator) {
        navigator.goNoteDetail(noteId = action.noteId) // import ...NoteDetailRoute.goNoteDetail
    }
}
```

Imports: `dev.dexkot.mobile.core.ui.navigation.action.{INavAction, INavActionFactory}`,
`dev.dexkot.mobile.core.ui.navigation.navigator.{INavActionExecutor, INavigator}`,
`dev.dexkot.mobile.core.annotation.{AutoLogNavAction, LogParam}`.

Why three classes: the action is a plain value the ViewModel can emit and tests can compare; the
factory is resolved from Koin, so it can receive dependencies (preferences, remote config) the
ViewModel should not know about; the executor holds the navigation code. See "Where decisions belong".

### 3. Register in Koin (screen module)

```kotlin
val NoteDetailModule = module {
    factory { NoteDetailNavActionFactory() } binds arrayOf(INavActionFactory::class)
    factory { NoteDetailNavAction_LogDecorator() } binds arrayOf(INavActionLogDecorator::class) // KSP-generated

    viewModel<AndroidViewModel<NoteDetailModel, NoteDetailIntention>>(noteDetailViewModel()) { parameters ->
        NoteDetailViewModel(getNote = get(), savedStateHandle = get())
            .withLogging(logger = get(), sourceComponent = parameters.get()) // only with @AutoLogViewModel
    }
}

internal fun noteDetailViewModel() = named("note_detail_viewmodel")
```

`uiModule()` builds the dispatcher from `getAll<INavActionFactory<*>>()`; the `binds arrayOf(...)`
is what makes your factory part of that list. Details in **dexkot-di-koin**.

### 4. Add the Route to a Graph

```kotlin
object NotesGraph : Graph(
    route = "notes_graph",
    startDestination = NoteListRoute,
    routes = listOf(NoteListRoute, NoteDetailRoute, NoteEditorRoute),
)
```

A Route that is in no Graph passed to `Navigation(graphs = ...)` does not exist at runtime:
`navigate` to it fails (Navigation throws for an unknown deep link).

### 5. Emit from the ViewModel

```kotlin
class NoteListViewModel(/* use cases */) : AndroidViewModel<NoteListModel, NoteListIntention>() {
    private val _model = MutableStateFlow(NoteListModel())
    override val model: StateFlow<NoteListModel> = _model.asStateFlow()

    override fun input(intent: NoteListIntention) {
        when (intent) {
            is NoteListIntention.OpenNote -> emit(NoteDetailNavAction(noteId = intent.noteId))
            NoteListIntention.Back -> emit(BackNavAction())
        }
    }
}
```

And read the argument in the destination ViewModel through the Route's extension:

```kotlin
import com.example.notes.navigation.notes.detail.route.NoteDetailRoute.getNoteId

class NoteDetailViewModel(private val getNote: IGetNoteUseCase, savedStateHandle: SavedStateHandle) :
    AndroidViewModel<NoteDetailModel, NoteDetailIntention>() {
    private val noteId = savedStateHandle.getNoteId()
    // ...
}
```

## Passing data

| Need | Use | Why |
|------|-----|-----|
| Required id | Path param: `"notes/{note_id}"` | Survives process death (it is in the back stack entry's saved state); works from external links. |
| Optional value / flag | Query param: `createRoute(base, setOf("tag"))` + `route.queryParameter("tag", value)` | Same durability; `null` strips the parameter. |
| Something that cannot be a URI (a `Parcelable`, a content `Uri` grant) | `navigate(uri, args = bundle)` → destination `onCreate(args)` | Delivered **once** and then removed; not rebuilt after process death. Last resort. |
| Result back to the previous screen | `popBackStack(result = bundle)` → previous ViewModel `onResume(result)` | Consumed once on the next resume. |
| Result of an external Activity | `navigateForResult(contract) { _, launch -> launch(input) }` → `onResume(result)` | Raw `ActivityResult` under `INavigator.RESULT_KEY`; parse it yourself. |

Query parameter routes, results and activity results are worked end-to-end in `references/recipes.md`.

## Root host

The app has one `FragmentActivity` that creates the navigator and calls the `Navigation` composable
(an extension on `FragmentActivity`, not `ComponentActivity`). Minimal version:

```kotlin
@Composable
fun FragmentActivity.AppNavigation(navigator: INavigator, removeSplash: () -> Unit) {
    val koin = getKoin()
    Navigation(
        graphs = listOf(LauncherGraph, NotesGraph),   // first graph = start graph by default
        navigator = navigator,
        dispatcher = koinInject<INavActionDispatcher>(),
        permissionController = rememberPermissionController(
            activity = this,
            preferences = koinInject<SharedPreferences>(qualifier = permissionPreferences()),
        ),
        remoteConfigProvider = koinInject<IRemoteConfigProvider>(), // or omit: defaults to a dummy
        loggerFactory = { source -> koin.get<ILoggerInteractor> { parametersOf(source) } },
        decorators = remember { koin.getAll<INavActionLogDecorator>() },
        removeSplash = removeSplash,
    )
}
```

With `val navigator = rememberNavigator(activity = this, deepLinkScheme = BuildConfig.APPLICATION_ID)`
in `setContent`, wrapped in `Preferences(uiPreferences = koinInject()) { ... }`. Full activity with the
splash screen and external deep links: `references/recipes.md#host-activity`.

Per screen, `Navigation` provides `LocalLogger`, `LocalRemoteConfig` and `LocalPermissionController`,
wraps the dispatcher with `withLogging(logger, decorators)`, and attaches `NavigationSubscriber`.

## Shared "commons" actions

Actions that any screen can emit (back, open URL, app settings, share, email) live once in a
`navigation/commons/` package, each as its own triple, registered in one `CommonsNavigationModule`.
Back is the one every app needs:

```kotlin
@AutoLogNavAction(identifier = "back")
data class BackNavAction(val result: Bundle? = null) : INavAction

internal class BackNavActionExecutor(private val action: BackNavAction) : INavActionExecutor {
    override operator fun invoke(navigator: INavigator) = navigator.popBackStack(result = action.result)
}
```

Open-URL and app-settings versions are in `references/recipes.md#commons`.

## Where decisions belong

Put "which screen?" logic in the Factory/Executor, not in the ViewModel. The ViewModel says *what the
user wants* (`OpenNote(noteId)`); the factory, injected with `IUiPreferences`, `IRemoteConfigProvider`
or a use case, decides *where that leads* (editor vs read-only view, onboarding first, experiment
arm). This keeps ViewModels free of routing concerns and lets you change a flow without touching
screens. Read the dependency inside `resolve()` / `invoke()`, not in the factory constructor: factories
are instantiated once (see Gotchas). Example in `references/recipes.md#conditional-routing`.

## Rules

- Emit `INavAction`s from ViewModels; never inject `INavigator`, `NavController` or `Context` to navigate.
- One factory per concrete action class, registered with `binds arrayOf(INavActionFactory::class)`.
- Every destination is an `object : Route<Model, Intention>` in its `route/` folder, listed in a `Graph`.
- Navigate only through the Route's `INavigator` extension, which calls `navigate(uri = createDeepLink(...))`.
- Keep route patterns and `deepLinks` scheme-less (`"notes/{note_id}"`); `Navigation` adds `"$deepLinkScheme://"`.
- Prefer path/query params over `args` Bundles.
- Annotate every action with `@AutoLogNavAction` and register its `_LogDecorator` next to the factory.
- Give the root `Navigation` the raw dispatcher from Koin; it adds logging per screen.

## Gotchas

- **Exact-class lookup.** Factories and log decorators are matched by `action::class`. A factory for a
  sealed parent or a superclass never matches its subclasses. A missing factory throws
  `IllegalStateException("No INavActionFactory registered for ...")` at dispatch time: a runtime
  crash when the user taps, not a compile error. Two factories for the same class: the last one wins silently.
- **Dispatcher snapshot.** `uiModule()` registers the dispatcher as a `single` that calls
  `getAll<INavActionFactory<*>>()` on first resolution. Factories in modules loaded later with
  `loadKoinModules` are never seen; load every navigation module in `startKoin`. It also means each
  factory instance lives for the whole process.
- **Deep links are mandatory and scheme-less.** Navigation always goes through `navigate(deepLink)`.
  The default `deepLinks` (the route pattern) covers it; if you override `deepLinks`, keep the pattern
  you navigate with. `Navigation` prefixes every `uriPattern` with `"${deepLinkScheme}://"`, so an
  `https://` App Link cannot be declared in `Route.deepLinks`; map it to your scheme in the Activity.
- **`deepLinkScheme` must match the manifest** `<data android:scheme=...>` if external links should open
  screens. `navigate(uri: String)` prepends the scheme; `navigate(uri: Uri)` does not, so a `Uri` must be complete.
- **Bundle args are one-shot.** `navigate(uri, args)` stores args on the new entry; they reach
  `onCreate(args)` once and are removed. `onCreate` can run again later (e.g. activity recreation) with `null`.
- **`popBackStack(result)` with nothing to pop** sets `RESULT_OK` with the result as extras and
  finishes the Activity.
- **`navigateForResult` never calls `contract.parseResult`.** The raw `ActivityResult` arrives in the
  current screen's `onResume(result)` under `INavigator.RESULT_KEY` (`"result"`). Avoid that key in
  your own popBackStack results.
- **Splash.** `removeSplash` fires when the start graph's start route leaves the back stack. If that
  route is your home screen, the splash never goes away. Use a launcher route that navigates away with
  `popUpTo(...) { inclusive = true }`. Keep `removeSplash` idempotent (e.g. set a flag): `Navigation`
  registers its listener during composition, so it can be invoked more than once.
- **Don't double-wrap.** `Navigation` already applies `dispatcher.withLogging(...)` per screen.
  Passing a dispatcher you wrapped yourself logs every navigation twice.
- **`LocalUiPreference` is not provided by `Navigation`.** Wrap the host in `Preferences(uiPreferences)`
  or screens read the no-op `DummyUiPreference`.
- **Silent logging failure.** An `@AutoLogNavAction` action whose `_LogDecorator` is not registered
  still navigates; it is just never logged.
- **Nav events are a `StateFlow`.** Two `emit` calls in the same frame conflate: only the last action runs.
- **Missing query placeholders.** A query key you don't pass through `queryParameter` stays as the
  literal `{key}` in the URI. Query arguments need `defaultValue` or `nullable = true` so the link
  still matches when the parameter is stripped.

## Related skills

- **dexkot-viewmodel**: `AndroidViewModel`, `emit`, model/intention, `onCreate`/`onResume`.
- **dexkot-screen**: what goes in `screenContent`, UI state.
- **dexkot-di-koin**: `uiModule`, `permissionModule`, screen modules, qualifiers.
- **dexkot-logging**: `@AutoLogNavAction`, `@LogParam`, `_LogDecorator`, `@AutoLogViewModel` / `withLogging`.
