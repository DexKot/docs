# core-ui navigation API (0.27.0)

Exact public signatures, grouped by package. Default values are the library's.

## Contents
- [dev.dexkot.mobile.core.ui.navigation.action](#action)
- [dev.dexkot.mobile.core.ui.navigation.navigator](#navigator)
- [dev.dexkot.mobile.core.ui.navigation.graph](#graph)
- [dev.dexkot.mobile.core.ui.compose (host)](#host)
- [dev.dexkot.mobile.core.ui.compose.navigation](#compose-navigation)
- [dev.dexkot.mobile.core.ui.compose.transitions](#transitions)
- [ViewModel side](#viewmodel-side)
- [core-ui-di-koin](#di)

## action

```kotlin
package dev.dexkot.mobile.core.ui.navigation.action

interface INavAction                                   // marker; implement it directly

interface INavActionFactory<T : INavAction> {
    val actionType: KClass<T>                          // matched against action::class (exact)
    fun resolve(action: T): INavActionExecutor
}

interface INavActionDispatcher {
    fun dispatch(action: INavAction): INavActionExecutor
}

class NavActionDispatcher(factories: List<INavActionFactory<*>>) : INavActionDispatcher
// dispatch() throws IllegalStateException("No INavActionFactory registered for <fqcn>") if no factory matches.
// Built with associateBy { it.actionType }: duplicate actionType → last factory wins.

interface INavActionLogDecorator {
    val actionType: KClass<out INavAction>
    fun wrap(executor: INavActionExecutor, action: INavAction, logger: ILoggerInteractor): INavActionExecutor
}

fun INavActionDispatcher.withLogging(
    logger: ILoggerInteractor,                          // dev.dexkot.mobile.core.logger.interactors.ui
    decorators: List<INavActionLogDecorator>,
): INavActionDispatcher
// Returns a dispatcher that wraps the resolved executor with the decorator for action::class, if any.
```

`@AutoLogNavAction(identifier: String, params: Array<LogParam> = [])` (package
`dev.dexkot.mobile.core.annotation`) makes KSP generate `<ActionName>_LogDecorator` (public class,
same package). Its wrapped executor calls `logger.navigate(navigationId = identifier, params = ...)`
and then the real executor. See dexkot-logging.

## navigator

```kotlin
package dev.dexkot.mobile.core.ui.navigation.navigator

interface INavActionExecutor {
    operator fun invoke(navigator: INavigator)
}

interface INavigator {
    companion object { const val RESULT_KEY = "result" }

    val activity: Activity
    val navHostController: NavHostController
    val deepLinkScheme: String

    fun popBackStack(result: Bundle? = null)

    fun navigate(uri: Uri, args: Bundle? = null, builder: NavOptionsBuilder.() -> Unit = { })
    fun navigate(uri: String, args: Bundle? = null, builder: NavOptionsBuilder.() -> Unit = { })
        // default impl: navigate("$deepLinkScheme://$uri".toUri(), args, builder)

    fun navigate(builder: Activity.() -> Intent)        // builds the Intent with the Activity as receiver
    fun navigate(intent: Intent)                        // activity.startActivity(intent)

    fun <Input, Output> navigateForResult(
        contract: ActivityResultContract<Input, Output>,
        launch: (Activity, (Input) -> Unit) -> Unit,
    )

    fun onActivityResult(activityResult: ActivityResult) // called by rememberNavigator's launcher
}

class Navigator(
    launcher: ActivityResultLauncher<Intent>,
    navHostController: NavHostController,
    activity: FragmentActivity,
    deepLinkScheme: String,
) : INavigator
```

`Navigator` behaviour:

- `navigate(uri: Uri, ...)` drops the URI fragment and trailing `/` of the path, then calls
  `navHostController.navigate(deepLink, navOptions(builder))`. If `args != null` they are stored on the
  **new** current back stack entry and delivered once to that screen's `onCreate(args)`.
- `popBackStack(result)`: if the NavController popped, `result` is stored on the now-current entry and
  delivered once to its `onResume(result)`. If there was nothing to pop, it calls
  `activity.setResult(Activity.RESULT_OK, Intent().putExtras(result))` and `activity.finish()`.
- `navigateForResult(contract, launch)`: calls `launch(activity) { input -> launcher.launch(contract.createIntent(activity, input)) }`.
  The launcher is a `StartActivityForResult`; `parseResult` of your contract is **not** called.
- `onActivityResult(r)`: stores `bundleOf(RESULT_KEY to r)` on the current entry → `onResume(result)`.

## graph

```kotlin
package dev.dexkot.mobile.core.ui.navigation.graph

open class Route<Model, Intention>(
    val identifier: String,
    val route: String,
    val deepLinks: List<NavDeepLink> = listOf(navDeepLink { uriPattern = route }),
    val arguments: List<NamedNavArgument> = listOf(),
    val viewModelFactory: @Composable () -> IScreenViewModel<Model, Intention>,
    val screenContent: @Composable (IScreenViewModel<Model, Intention>) -> Unit,
    val transitions: Transitions = DefaultTransitions,
)

interface IGraph {
    val route: String
    val startDestination: Route<*, *>
    val routes: List<Route<*, *>>
}

open class Graph(
    override val route: String,
    override val startDestination: Route<*, *>,
    override val routes: List<Route<*, *>>,
) : IGraph

fun createRoute(route: String, queryParams: Set<String>): String
// createRoute("search/notes", setOf("query", "tag")) == "search/notes?query={query}&tag={tag}"

fun <T> String.queryParameter(key: String, value: T?): String   // value?.toString()
fun String.queryParameter(key: String, value: String?): String
// value != null → replaces "{key}" with Uri.encode(value)
// value == null → removes "&?key={key}", then a trailing "?"
```

`Navigation` registers each Route with `composable(route = route.route, arguments = route.arguments,
deepLinks = route.deepLinks.map { navDeepLink { uriPattern = "${navigator.deepLinkScheme}://${it.uriPattern}" } }, ...)`.
Only `uriPattern` is copied: `action` / `mimeType` on your `NavDeepLink`s are ignored.

## host

```kotlin
package dev.dexkot.mobile.core.ui.compose

@Composable
fun FragmentActivity.Navigation(
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
```

- Builds a `NavHost(navigator.navHostController, startDestination.route)` with one nested
  `navigation(...)` per `Graph`.
- Per Route it composes, in order: `RemoteConfig(remoteConfigProvider)` → `Logger(loggerFactory(route.identifier))`
  → `PermissionController(permissionController.withLogging(logger))` → `route.viewModelFactory()` →
  `NavigationSubscriber(viewModel, navigator, dispatcher.withLogging(logger, decorators))` →
  `route.screenContent(viewModel)`, inside `NavAnimatedVisibility`.
- Splash: calls `removeSplash()` once `startDestination.startDestination.route` is no longer in the back stack.

## compose-navigation

```kotlin
package dev.dexkot.mobile.core.ui.compose.navigation

@Composable
fun rememberNavigator(
    activity: FragmentActivity,
    navHostController: NavHostController = rememberNavController(),
    deepLinkScheme: String,
): INavigator

@Composable
fun NavigationSubscriber(
    viewModel: IScreenViewModel<*, *>,
    navigator: INavigator,
    dispatcher: INavActionDispatcher,
)
// ON_CREATE → viewModel.onCreate(consumed args)
// ON_RESUME → LocalLogger.current.enter(); viewModel.onResume(consumed result)
// ON_PAUSE  → LocalLogger.current.exit()
// ON_STOP   → viewModel.onStop()
// viewModel.nav event → dispatcher.dispatch(action)(navigator)
```

## transitions

```kotlin
package dev.dexkot.mobile.core.ui.compose.transitions

class Transitions(
    val enterTransition: AnimatedContentTransitionScope<NavBackStackEntry>.() -> EnterTransition,
    val exitTransition: AnimatedContentTransitionScope<NavBackStackEntry>.() -> ExitTransition,
    val popEnterTransition: AnimatedContentTransitionScope<NavBackStackEntry>.() -> EnterTransition,
    val popExitTransition: AnimatedContentTransitionScope<NavBackStackEntry>.() -> ExitTransition,
)

val DefaultTransitions: Transitions      // fade in/out, 250 ms, 100 ms delay
val ModalTransition: Transitions         // slide up on enter, slide down on pop exit
val ImitationOfActivities: Transitions   // horizontal slide like Activity transitions
```

## ViewModel side

```kotlin
// dev.dexkot.mobile.core.ui.viewmodel
interface IViewModel<Model, Intention> {
    val model: StateFlow<Model>
    fun input(intent: Intention)
    fun onCreate(args: Bundle? = null) { }
    fun onResume(result: Bundle? = null) { }
    fun onStop() { }
}
interface IScreenViewModel<Model, Intention> : IViewModel<Model, Intention> {
    val nav: StateFlow<Event<INavAction>?>
}

// dev.dexkot.mobile.core.ui.viewmodel.android
abstract class AndroidViewModel<Model, Intention> : IScreenViewModel<Model, Intention>, androidx.lifecycle.ViewModel() {
    protected fun emit(action: INavAction)   // sets nav to a new Event(action)
    // ... see dexkot-viewmodel
}
```

## di

```kotlin
// dev.dexkot.mobile.core.ui.di.koin (core-ui-di-koin)
fun uiModule(preferencesName: String = "dev.dexkot.ui.preferences"): Module
//   single<INavActionDispatcher> { NavActionDispatcher(factories = getAll<INavActionFactory<*>>()) }
//   single<IUiPreferences> { UiPreferences(SharedPreferences named preferencesName) }
//   factory<IResourceProvider> { ResourceProvider(context = get()) }

// dev.dexkot.mobile.core.permission.di.koin (core-permission-di-koin)
fun permissionPreferences(): Qualifier   // named("permission_preferences")

// dev.dexkot.mobile.core.ui.compose.permission (core-ui)
@Composable
fun rememberPermissionController(activity: Activity, preferences: SharedPreferences): IPermissionController
```
