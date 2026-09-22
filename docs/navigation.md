# Navigation

`core-ui` gives you a navigation layer on top of Jetpack Navigation Compose in which **ViewModels never
navigate**. They emit a small data object describing the user's intent; the framework turns it into
navigation. This guide walks through the model, builds a screen destination end to end, and covers
arguments, results, activity results, the root host and the pitfalls.

Artifacts: `dev.dexkot.mobile:core-ui` and `core-ui-di-koin`, version 0.27.0. Android only.

- [Why it works this way](#why-it-works-this-way)
- [Setup](#setup)
- [The pipeline](#the-pipeline)
- [Tutorial: a note detail screen](#tutorial-a-note-detail-screen)
- [Passing data between screens](#passing-data-between-screens)
- [Returning results](#returning-results)
- [Launching other apps for a result](#launching-other-apps-for-a-result)
- [The root host](#the-root-host)
- [Shared actions](#shared-actions)
- [Deciding where to go](#deciding-where-to-go)
- [Pitfalls](#pitfalls)
- [API reference](#api-reference)

## Why it works this way

- **Testable ViewModels.** A ViewModel test asserts "it emitted `NoteDetailNavAction(42)`". No
  `NavController`, no Robolectric.
- **One place for routing decisions.** Whether "open note" goes to the editor or a read-only view,
  or first through onboarding, is decided in an injectable factory, not in every ViewModel.
- **Analytics for free.** Each action carries an `@AutoLogNavAction` annotation; KSP generates a
  decorator that logs the navigation before it runs.
- **One routing mechanism.** Every destination is reached by a deep link URI, so external links and
  in-app navigation resolve through the same patterns.

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
    implementation("dev.dexkot.mobile:core-ui-di-koin:0.27.0")
    implementation("dev.dexkot.mobile:core-permission-di-koin:0.27.0")
    implementation("dev.dexkot.mobile:core-permission:0.27.0")

    // core-ui uses these with `implementation`; your Routes need them directly.
    implementation("androidx.navigation:navigation-compose:2.9.5")
    implementation("androidx.fragment:fragment-ktx:1.8.7")
    implementation("io.insert-koin:koin-androidx-compose:4.1.1")

    // Navigation analytics (optional)
    implementation("dev.dexkot.mobile:core-annotation:0.27.0")
    ksp("dev.dexkot.mobile:core-ksp-processor:0.27.0")
}
```

## The pipeline

```
ViewModel ── emit(NoteDetailNavAction(42)) ──▶ viewModel.nav (StateFlow<Event<INavAction>?>)
                                                   │
NavigationSubscriber (added to every screen) ◀─────┘
   │ dispatcher.dispatch(action)            ← dispatcher wrapped with log decorators per screen
   ▼
NavActionDispatcher ── finds INavActionFactory by action::class ──▶ factory.resolve(action)
   ▼
INavActionExecutor.invoke(navigator)
   ▼
navigator.goNoteDetail(42)  →  navigate("notes/42")  →  NavController.navigate("<scheme>://notes/42")
```

`NavigationSubscriber` also drives the ViewModel lifecycle hooks: `onCreate(args)` on ON_CREATE,
`onResume(result)` on ON_RESUME and `onStop()` on ON_STOP.

## Tutorial: a note detail screen

A screen's navigation code lives in its `route/` folder:

```
notes/detail/
├── route/      NoteDetailRoute, NoteDetailNavAction, NoteDetailNavActionFactory, NoteDetailNavActionExecutor
├── di/         NoteDetailModule
├── viewmodel/  NoteDetailViewModel, NoteDetailModel, NoteDetailIntention
└── screen/     NoteDetailScreen
```

### Step 1: the Route

A `Route` is a singleton object describing one destination: its URI pattern, typed arguments, deep
links, how to create its ViewModel, how to draw it and how to animate it.

```kotlin
package com.example.notes.navigation.notes.detail.route

import androidx.lifecycle.SavedStateHandle
import androidx.navigation.NavType
import androidx.navigation.navArgument
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
    viewModelFactory = {
        koinViewModel<AndroidViewModel<NoteDetailModel, NoteDetailIntention>>(noteDetailViewModel()) {
            parametersOf(NoteDetailRoute.IDENTIFIER)
        }
    },
    screenContent = { viewModel -> NoteDetailScreen(viewModel = viewModel) },
    transitions = ImitationOfActivities,
) {
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

Things to notice:

- The constants are `private const val`. Keeping them private means the only way to navigate here or
  read the argument is through the Route's own extensions, so a typo in a key cannot happen elsewhere.
  They must be `const` because they are read in the super-constructor call, before the object's
  normal properties are initialised.
- `deepLinks` is not set: its default is the route pattern itself, which is what `goNoteDetail`
  navigates to. The host adds the `scheme://` prefix.
- `identifier` names the screen for logging and is handed to the ViewModel definition through
  `parametersOf`.
- Built-in transitions: `DefaultTransitions` (fade), `ImitationOfActivities` (horizontal slide),
  `ModalTransition` (slide up). Or build your own `Transitions(...)`.

### Step 2: the NavAction triple

```kotlin
import dev.dexkot.mobile.core.annotation.AutoLogNavAction
import dev.dexkot.mobile.core.annotation.LogParam
import dev.dexkot.mobile.core.ui.navigation.action.INavAction
import dev.dexkot.mobile.core.ui.navigation.action.INavActionFactory
import dev.dexkot.mobile.core.ui.navigation.navigator.INavActionExecutor
import dev.dexkot.mobile.core.ui.navigation.navigator.INavigator
import com.example.notes.navigation.notes.detail.route.NoteDetailRoute.goNoteDetail

@AutoLogNavAction(identifier = "go_note_detail", params = [LogParam(logName = "note_id", paramName = "noteId")])
data class NoteDetailNavAction(val noteId: Long) : INavAction

internal class NoteDetailNavActionFactory : INavActionFactory<NoteDetailNavAction> {
    override val actionType = NoteDetailNavAction::class
    override fun resolve(action: NoteDetailNavAction): INavActionExecutor = NoteDetailNavActionExecutor(action)
}

internal class NoteDetailNavActionExecutor(private val action: NoteDetailNavAction) : INavActionExecutor {
    override operator fun invoke(navigator: INavigator) {
        navigator.goNoteDetail(noteId = action.noteId)
    }
}
```

The **action** is what the ViewModel knows. The **factory** is created by Koin, so it can receive
extra dependencies and pass them on. The **executor** is the only code that touches `INavigator`.

### Step 3: register in Koin

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

`uiModule()` builds the dispatcher from every factory bound as `INavActionFactory`. The
`_LogDecorator` and `withLogging` are generated by the AutoLog annotations; see the logging guide. More
on modules in the [dependency injection guide](dependency-injection.md).

### Step 4: add it to a Graph

```kotlin
object NotesGraph : Graph(
    route = "notes_graph",
    startDestination = NoteListRoute,
    routes = listOf(NoteListRoute, NoteDetailRoute, NoteEditorRoute),
)
```

### Step 5: emit from the list screen

```kotlin
class NoteListViewModel(/* ... */) : AndroidViewModel<NoteListModel, NoteListIntention>() {
    private val _model = MutableStateFlow(NoteListModel())
    override val model: StateFlow<NoteListModel> = _model.asStateFlow()

    override fun input(intent: NoteListIntention) {
        when (intent) {
            is NoteListIntention.OpenNote -> emit(NoteDetailNavAction(noteId = intent.noteId))
        }
    }
}
```

And read the argument on the other side:

```kotlin
import com.example.notes.navigation.notes.detail.route.NoteDetailRoute.getNoteId

class NoteDetailViewModel(
    private val getNote: IGetNoteUseCase,
    savedStateHandle: SavedStateHandle,
) : AndroidViewModel<NoteDetailModel, NoteDetailIntention>() {
    private val noteId = savedStateHandle.getNoteId()
    // ...
}
```

## Passing data between screens

**Path parameters** for required identifiers (`"notes/{note_id}"`). **Query parameters** for optional
values:

```kotlin
object NoteSearchRoute : Route<NoteSearchModel, NoteSearchIntention>(
    identifier = NoteSearchRoute.IDENTIFIER,
    route = createRoute(NoteSearchRoute.BASE, setOf(NoteSearchRoute.TAG_PARAM)),   // "search/notes?tag={tag}"
    arguments = listOf(
        navArgument(NoteSearchRoute.TAG_PARAM) { type = NavType.StringType; nullable = true; defaultValue = null }
    ),
    viewModelFactory = { /* ... */ },
    screenContent = { /* ... */ },
) {
    private const val IDENTIFIER = "note_search"
    private const val BASE = "search/notes"
    private const val TAG_PARAM = "tag"

    fun INavigator.goNoteSearch(tag: String?) {
        navigate(uri = route.queryParameter(TAG_PARAM, tag))   // encodes the value, or strips the param if null
    }

    fun SavedStateHandle.getTag(): String? = get<String>(TAG_PARAM)
}
```

Both kinds are stored in the destination's `SavedStateHandle`, survive process death and work from
external links.

**Bundle arguments** (`navigate(uri, args = bundle)`) exist for values that cannot be written into a
URI. They are handed to the destination ViewModel's `onCreate(args)` **once**, then removed, and are
not restored after process death. Use them sparingly.

## Returning results

A screen returns data by popping itself with a result; the previous screen gets it in `onResume`:

```kotlin
// editor, after saving
emit(BackNavAction(result = bundleOf("saved_note_id" to noteId)))

// detail screen
override fun onResume(result: Bundle?) {
    if (result?.containsKey("saved_note_id") == true) reload()
}
```

`BackNavAction` is a shared action you define once (see [Shared actions](#shared-actions)); its
executor calls `navigator.popBackStack(result)`. `onResume` runs on every resume, usually with `null`.
If there is nothing to pop, `popBackStack` sets `RESULT_OK` (with the result as extras) and finishes
the Activity.

## Launching other apps for a result

```kotlin
internal class PickAttachmentNavActionExecutor(private val action: PickAttachmentNavAction) : INavActionExecutor {
    override operator fun invoke(navigator: INavigator) {
        navigator.navigateForResult(ActivityResultContracts.OpenDocument()) { _, launch ->
            launch(action.mimeTypes.toTypedArray())
        }
    }
}

fun Bundle.getPickedAttachment(): Uri? {
    val activityResult = BundleCompat.getParcelable(this, INavigator.RESULT_KEY, ActivityResult::class.java)
        ?: return null
    return ActivityResultContracts.OpenDocument().parseResult(activityResult.resultCode, activityResult.data)
}

// ViewModel
override fun onResume(result: Bundle?) {
    result?.getPickedAttachment()?.let { uri -> input(NoteEditorIntention.AttachmentPicked(uri)) }
}
```

The navigator only uses your contract to create the Intent. The raw `ActivityResult` comes back in
`onResume(result)` under `INavigator.RESULT_KEY`, so parse it yourself (reusing the contract as above).

## The root host

One `FragmentActivity` hosts everything. `rememberNavigator` creates the navigator (and its
activity-result launcher); `Navigation` builds the `NavHost` from your graphs.

```kotlin
class MainActivity : FragmentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        var keepSplash by mutableStateOf(true)
        installSplashScreen().setKeepOnScreenCondition { keepSplash }
        super.onCreate(savedInstanceState)

        setContent {
            val navigator = rememberNavigator(activity = this, deepLinkScheme = BuildConfig.APPLICATION_ID)
            Preferences(uiPreferences = koinInject()) {
                AppTheme {
                    AppNavigation(navigator = navigator, removeSplash = { keepSplash = false })
                }
            }
        }
    }
}

@Composable
fun FragmentActivity.AppNavigation(navigator: INavigator, removeSplash: () -> Unit) {
    val koin = getKoin()
    Navigation(
        graphs = listOf(LauncherGraph, NotesGraph),
        navigator = navigator,
        dispatcher = koinInject<INavActionDispatcher>(),
        permissionController = rememberPermissionController(
            activity = this,
            preferences = koinInject<SharedPreferences>(qualifier = permissionPreferences()),
        ),
        remoteConfigProvider = koinInject<IRemoteConfigProvider>(),
        loggerFactory = { source -> koin.get<ILoggerInteractor> { parametersOf(source) } },
        decorators = remember { koin.getAll<INavActionLogDecorator>() },
        removeSplash = removeSplash,
    )
}
```

For each screen, `Navigation` provides `LocalLogger` (named after the Route's `identifier`),
`LocalRemoteConfig` and `LocalPermissionController`, and wraps the dispatcher with your log
decorators. It does **not** provide `LocalUiPreference`; that is why the host is wrapped in
`Preferences(...)`.

**Splash.** `removeSplash` is called when the start graph's start route leaves the back stack. Make
that start route a launcher screen that does the startup work and navigates on with the back stack
cleared:

```kotlin
fun INavigator.goNoteList(clearHistory: Boolean = false) {
    navigate(uri = NOTE_LIST_ROUTE) {
        if (clearHistory) popUpTo(0) { inclusive = true }
    }
}
```

**External links.** The scheme you pass as `deepLinkScheme` is prepended to every Route pattern, so
`com.example.notes://notes/42` opens the detail screen. Declare the same scheme in the Activity's
`<intent-filter>` and navigate to incoming URIs with `navigator.navigate(uri = intent.data)` once the
launcher is done. `https` App Links cannot be declared on a Route (the prefix would produce
`scheme://https://...`); translate them to your scheme in the Activity.

## Shared actions

Actions many screens emit are defined once under `navigation/commons/` and registered in one module:

```kotlin
@AutoLogNavAction(identifier = "back")
data class BackNavAction(val result: Bundle? = null) : INavAction

internal class BackNavActionExecutor(private val action: BackNavAction) : INavActionExecutor {
    override operator fun invoke(navigator: INavigator) = navigator.popBackStack(result = action.result)
}

@AutoLogNavAction(identifier = "open_url")
data class OpenUrlNavAction(val uri: Uri) : INavAction

internal class OpenUrlNavActionExecutor(private val action: OpenUrlNavAction) : INavActionExecutor {
    override operator fun invoke(navigator: INavigator) {
        navigator.navigate { Intent.createChooser(Intent(Intent.ACTION_VIEW, action.uri), null) }
    }
}

@AutoLogNavAction(identifier = "open_app_settings")
data object AppSettingsNavAction : INavAction

internal class AppSettingsNavActionExecutor : INavActionExecutor {
    override operator fun invoke(navigator: INavigator) {
        navigator.navigate {
            Intent(Settings.ACTION_APPLICATION_DETAILS_SETTINGS, Uri.fromParts("package", packageName, null))
        }
    }
}
```

Each also needs its factory, and all of them go into a `CommonsNavigationModule` with `binds arrayOf(...)`.

## Deciding where to go

Keep the ViewModel describing what the user wants and let the factory decide the destination:

```kotlin
internal class OpenNoteNavActionFactory(
    private val remoteConfig: IRemoteConfigProvider,
) : INavActionFactory<OpenNoteNavAction> {
    override val actionType = OpenNoteNavAction::class
    override fun resolve(action: OpenNoteNavAction): INavActionExecutor =
        OpenNoteNavActionExecutor(action, openInEditor = remoteConfig.getBoolean("open_notes_in_editor", defaultValue = false))
}
```

Read the value in `resolve`, not the constructor: the dispatcher creates each factory once and keeps it.

## Pitfalls

| Symptom | Cause | Fix |
|---------|-------|-----|
| `IllegalStateException: No INavActionFactory registered for ...` on tap | Factory not bound, bound in a module loaded later with `loadKoinModules`, or registered for a parent type | Bind one factory per concrete action class, in modules passed to `startKoin`. Matching is by exact `action::class`. |
| Navigation throws "destination ... cannot be found" | Route not in any Graph passed to `Navigation`, or its `deepLinks` override dropped the pattern you navigate to | Add it to a Graph; keep the navigation pattern in `deepLinks`. |
| External link does nothing | `deepLinkScheme` differs from the manifest scheme, or a Route deep link includes a scheme | Use the same scheme in both; keep Route patterns scheme-less. |
| Splash never disappears | The start route is a real screen that stays in the back stack | Add a launcher route that leaves with `popUpTo(...) { inclusive = true }`. |
| Bundle args missing after returning from background | They are delivered once and not restored after process death | Use path or query parameters. |
| Activity result arrives as an `ActivityResult`, not your contract's type | `navigateForResult` doesn't call `parseResult` | Parse from `INavigator.RESULT_KEY`. |
| Every navigation logged twice | You passed a dispatcher already wrapped with `withLogging` | Pass the raw dispatcher; `Navigation` wraps it per screen. |
| Navigation works but isn't logged | `_LogDecorator` not registered | Bind it next to the factory. |
| Screens see default UI preferences | `LocalUiPreference` not provided | Wrap the host in `Preferences(uiPreferences)`. |
| Only the second of two quick emits happens | `nav` is a `StateFlow`; values conflate | Emit one action per user event. |

## API reference

```kotlin
// dev.dexkot.mobile.core.ui.navigation.action
interface INavAction
interface INavActionFactory<T : INavAction> { val actionType: KClass<T>; fun resolve(action: T): INavActionExecutor }
interface INavActionDispatcher { fun dispatch(action: INavAction): INavActionExecutor }
class NavActionDispatcher(factories: List<INavActionFactory<*>>) : INavActionDispatcher
interface INavActionLogDecorator {
    val actionType: KClass<out INavAction>
    fun wrap(executor: INavActionExecutor, action: INavAction, logger: ILoggerInteractor): INavActionExecutor
}
fun INavActionDispatcher.withLogging(logger: ILoggerInteractor, decorators: List<INavActionLogDecorator>): INavActionDispatcher

// dev.dexkot.mobile.core.ui.navigation.navigator
interface INavActionExecutor { operator fun invoke(navigator: INavigator) }
interface INavigator {
    companion object { const val RESULT_KEY = "result" }
    val activity: Activity
    val navHostController: NavHostController
    val deepLinkScheme: String
    fun popBackStack(result: Bundle? = null)
    fun navigate(uri: Uri, args: Bundle? = null, builder: NavOptionsBuilder.() -> Unit = { })
    fun navigate(uri: String, args: Bundle? = null, builder: NavOptionsBuilder.() -> Unit = { }) // prepends "$deepLinkScheme://"
    fun navigate(builder: Activity.() -> Intent)
    fun navigate(intent: Intent)
    fun <Input, Output> navigateForResult(contract: ActivityResultContract<Input, Output>, launch: (Activity, (Input) -> Unit) -> Unit)
    fun onActivityResult(activityResult: ActivityResult)
}
class Navigator(launcher: ActivityResultLauncher<Intent>, navHostController: NavHostController,
                activity: FragmentActivity, deepLinkScheme: String) : INavigator

// dev.dexkot.mobile.core.ui.navigation.graph
open class Route<Model, Intention>(
    val identifier: String,
    val route: String,
    val deepLinks: List<NavDeepLink> = listOf(navDeepLink { uriPattern = route }),
    val arguments: List<NamedNavArgument> = listOf(),
    val viewModelFactory: @Composable () -> IScreenViewModel<Model, Intention>,
    val screenContent: @Composable (IScreenViewModel<Model, Intention>) -> Unit,
    val transitions: Transitions = DefaultTransitions,
)
interface IGraph { val route: String; val startDestination: Route<*, *>; val routes: List<Route<*, *>> }
open class Graph(route: String, startDestination: Route<*, *>, routes: List<Route<*, *>>) : IGraph
fun createRoute(route: String, queryParams: Set<String>): String
fun <T> String.queryParameter(key: String, value: T?): String
fun String.queryParameter(key: String, value: String?): String

// dev.dexkot.mobile.core.ui.compose
@Composable fun FragmentActivity.Navigation(
    graphs: List<Graph>,
    navigator: INavigator,
    dispatcher: INavActionDispatcher,
    permissionController: IPermissionController,
    remoteConfigProvider: IRemoteConfigProvider = DummyRemoteConfigProvider,
    loggerFactory: (sourceComponent: String) -> ILoggerInteractor = { DummyLoggerInteractor },
    decorators: List<INavActionLogDecorator> = emptyList(),
    removeSplash: () -> Unit = { },
    startDestination: Graph = graphs.first(),
)

// dev.dexkot.mobile.core.ui.compose.navigation
@Composable fun rememberNavigator(activity: FragmentActivity, navHostController: NavHostController = rememberNavController(), deepLinkScheme: String): INavigator
@Composable fun NavigationSubscriber(viewModel: IScreenViewModel<*, *>, navigator: INavigator, dispatcher: INavActionDispatcher)

// dev.dexkot.mobile.core.ui.compose.transitions
class Transitions(enterTransition, exitTransition, popEnterTransition, popExitTransition) // AnimatedContentTransitionScope<NavBackStackEntry>.() -> Enter/ExitTransition
val DefaultTransitions: Transitions
val ImitationOfActivities: Transitions
val ModalTransition: Transitions

// dev.dexkot.mobile.core.ui.di.koin (core-ui-di-koin)
fun uiModule(preferencesName: String = "dev.dexkot.ui.preferences"): Module
```
