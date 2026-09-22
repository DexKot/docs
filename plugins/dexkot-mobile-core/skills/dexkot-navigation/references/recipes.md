# Navigation recipes

Complete, copy-ready examples for the less common cases. Imports are listed where they are not obvious.

## Contents
- [Host activity (splash + external deep links)](#host-activity)
- [Launcher route that clears the splash](#launcher-route)
- [Optional values with query parameters](#query-parameters)
- [Returning a result to the previous screen](#results)
- [Launching an Activity for a result](#activity-results)
- [Conditional routing in the factory](#conditional-routing)
- [Commons: open URL and app settings](#commons)
- [Clearing history](#clearing-history)
- [Manifest for external links](#manifest)
- [Testing](#testing)

## Host activity

```kotlin
package com.example.notes

import android.content.Intent
import android.content.SharedPreferences
import android.net.Uri
import android.os.Bundle
import androidx.activity.compose.setContent
import androidx.compose.runtime.Composable
import androidx.compose.runtime.LaunchedEffect
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.setValue
import androidx.core.splashscreen.SplashScreen.Companion.installSplashScreen
import androidx.fragment.app.FragmentActivity
import dev.dexkot.mobile.core.logger.interactors.ui.ILoggerInteractor
import dev.dexkot.mobile.core.permission.di.koin.permissionPreferences
import dev.dexkot.mobile.core.remoteconfig.provider.IRemoteConfigProvider
import dev.dexkot.mobile.core.ui.compose.Navigation
import dev.dexkot.mobile.core.ui.compose.navigation.rememberNavigator
import dev.dexkot.mobile.core.ui.compose.permission.rememberPermissionController
import dev.dexkot.mobile.core.ui.compose.preferences.Preferences
import dev.dexkot.mobile.core.ui.navigation.action.INavActionDispatcher
import dev.dexkot.mobile.core.ui.navigation.action.INavActionLogDecorator
import dev.dexkot.mobile.core.ui.navigation.navigator.INavigator
import dev.dexkot.mobile.core.ui.viewmodel.event.Event
import org.koin.compose.getKoin
import org.koin.compose.koinInject
import org.koin.core.parameter.parametersOf

class MainActivity : FragmentActivity() {

    // Event makes the link consumable once, even across recompositions.
    private var pendingDeepLink by mutableStateOf<Event<Uri>?>(null)

    override fun onCreate(savedInstanceState: Bundle?) {
        var keepSplash by mutableStateOf(true)
        installSplashScreen().setKeepOnScreenCondition { keepSplash }
        super.onCreate(savedInstanceState)

        if (savedInstanceState == null) readDeepLink(intent)

        setContent {
            val navigator = rememberNavigator(activity = this, deepLinkScheme = BuildConfig.APPLICATION_ID)

            Preferences(uiPreferences = koinInject()) {   // Navigation does not provide LocalUiPreference
                AppTheme {
                    AppNavigation(navigator = navigator, removeSplash = { keepSplash = false })
                }
            }

            // Navigate to external links only after the launcher has decided where the user starts.
            LaunchedEffect(keepSplash, pendingDeepLink) {
                if (!keepSplash) {
                    pendingDeepLink?.handle()?.let { uri ->
                        navigator.navigate(uri = uri) { launchSingleTop = true }   // Uri overload: already has the scheme
                    }
                }
            }
        }
    }

    override fun onNewIntent(intent: Intent) {
        super.onNewIntent(intent)
        setIntent(intent)
        readDeepLink(intent)
    }

    private fun readDeepLink(intent: Intent) {
        val uri = intent.data ?: return
        pendingDeepLink = when (uri.scheme) {
            BuildConfig.APPLICATION_ID -> Event(uri)
            // An https App Link cannot be declared on a Route: translate it to your scheme here.
            "https" -> Event(Uri.parse("${BuildConfig.APPLICATION_ID}://${uri.path?.trimStart('/')}"))
            else -> null
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

Notes:
- `ILoggerInteractor` here is the UI interactor (`...interactors.ui`); `loggerModule()` binds it as a
  factory that takes the source component as a parameter (see dexkot-di-koin).
- androidx Navigation may itself handle a matching deep link in the launching intent when the graph is
  first set (standard `NavController` behaviour). Test cold-start links to be sure the result is what
  you want (single navigation, correct back stack).

## Launcher route

The start graph's start route acts as the splash gate. It renders nothing; its ViewModel does the
startup work and then leaves with `popUpTo` inclusive, which is exactly what triggers `removeSplash`.

```kotlin
object LauncherGraph : Graph(route = "launcher_graph", startDestination = LauncherRoute, routes = listOf(LauncherRoute))

object LauncherRoute : Route<LauncherModel, LauncherIntention>(
    identifier = "launcher",
    route = "launcher",
    viewModelFactory = { koinViewModel<AndroidViewModel<LauncherModel, LauncherIntention>>(launcherViewModel()) },
    screenContent = { _ -> },
)

// in LauncherViewModel, once startup work is done:
emit(NoteListNavAction(clearHistory = true))
```

`NoteListRoute.goNoteList(clearHistory = true)` pops everything (see [Clearing history](#clearing-history)),
so `"launcher"` leaves the back stack and the splash is removed.

## Query parameters

```kotlin
import dev.dexkot.mobile.core.ui.navigation.graph.createRoute
import dev.dexkot.mobile.core.ui.navigation.graph.queryParameter

object NoteSearchRoute : Route<NoteSearchModel, NoteSearchIntention>(
    identifier = NoteSearchRoute.IDENTIFIER,
    route = createRoute(NoteSearchRoute.BASE, setOf(NoteSearchRoute.QUERY_PARAM, NoteSearchRoute.TAG_PARAM)),
    // → "search/notes?query={query}&tag={tag}"
    arguments = listOf(
        navArgument(NoteSearchRoute.QUERY_PARAM) { type = NavType.StringType; nullable = true; defaultValue = null },
        navArgument(NoteSearchRoute.TAG_PARAM) { type = NavType.StringType; nullable = true; defaultValue = null },
    ),
    viewModelFactory = {
        koinViewModel<AndroidViewModel<NoteSearchModel, NoteSearchIntention>>(noteSearchViewModel()) {
            parametersOf(NoteSearchRoute.IDENTIFIER)
        }
    },
    screenContent = { viewModel -> NoteSearchScreen(viewModel) },
) {
    private const val IDENTIFIER = "note_search"
    private const val BASE = "search/notes"
    private const val QUERY_PARAM = "query"
    private const val TAG_PARAM = "tag"

    fun INavigator.goNoteSearch(query: String?, tag: String?) {
        navigate(uri = createDeepLink(query, tag))
    }

    // Every key goes through queryParameter: a value is URI-encoded, null removes the parameter.
    fun createDeepLink(query: String?, tag: String?): String =
        route.queryParameter(QUERY_PARAM, query).queryParameter(TAG_PARAM, tag)

    fun SavedStateHandle.getQuery(): String? = get<String>(QUERY_PARAM)
    fun SavedStateHandle.getTag(): String? = get<String>(TAG_PARAM)
}
```

For non-string types use `NavType.BoolType` / `IntType` / `LongType` with a `defaultValue` (they cannot
be nullable) and `queryParameter(key, value)` — the generic overload calls `toString()`.

## Results

The editor saves and returns to the detail screen, which refreshes.

```kotlin
// Shared keys: keep them next to the route that produces the result.
object NoteEditorResult {
    const val SAVED_NOTE_ID = "saved_note_id"
}

// NoteEditorViewModel
private fun onSaved(noteId: Long) {
    emit(BackNavAction(result = bundleOf(NoteEditorResult.SAVED_NOTE_ID to noteId)))
}

// NoteDetailViewModel
override fun onResume(result: Bundle?) {
    if (result?.containsKey(NoteEditorResult.SAVED_NOTE_ID) == true) reload()
}
```

`onResume(result)` is called on every resume, usually with `null`; check the key. The result is removed
after the first delivery.

## Activity results

`navigateForResult` launches the contract's Intent; the raw `ActivityResult` comes back to the screen
that is current when it returns, in `onResume(result)` under `INavigator.RESULT_KEY`. The contract's
`parseResult` is not called, so provide a parser next to the action and reuse the contract for it.

```kotlin
import androidx.activity.result.ActivityResult
import androidx.activity.result.contract.ActivityResultContracts
import androidx.core.os.BundleCompat

@AutoLogNavAction(identifier = "pick_attachment")
data class PickAttachmentNavAction(val mimeTypes: List<String>) : INavAction

internal class PickAttachmentNavActionFactory : INavActionFactory<PickAttachmentNavAction> {
    override val actionType = PickAttachmentNavAction::class
    override fun resolve(action: PickAttachmentNavAction): INavActionExecutor = PickAttachmentNavActionExecutor(action)
}

internal class PickAttachmentNavActionExecutor(private val action: PickAttachmentNavAction) : INavActionExecutor {
    override operator fun invoke(navigator: INavigator) {
        navigator.navigateForResult(ActivityResultContracts.OpenDocument()) { _, launch ->
            launch(action.mimeTypes.toTypedArray())
        }
    }
}

/** Returns the picked document, or null if this bundle is not an activity result / was cancelled. */
fun Bundle.getPickedAttachment(): Uri? {
    val activityResult = BundleCompat.getParcelable(this, INavigator.RESULT_KEY, ActivityResult::class.java)
        ?: return null
    return ActivityResultContracts.OpenDocument().parseResult(activityResult.resultCode, activityResult.data)
}

// In the ViewModel
override fun onResume(result: Bundle?) {
    result?.getPickedAttachment()?.let { uri -> input(NoteEditorIntention.AttachmentPicked(uri)) }
}
```

## Conditional routing

The ViewModel emits "open note"; the factory decides between editor and read-only viewer using a
remote flag, so the experiment never leaks into the ViewModel.

```kotlin
internal class OpenNoteNavActionFactory(
    private val remoteConfig: IRemoteConfigProvider,
) : INavActionFactory<OpenNoteNavAction> {
    override val actionType = OpenNoteNavAction::class

    // Read the flag here, per dispatch: the factory instance lives as long as the dispatcher (process).
    override fun resolve(action: OpenNoteNavAction): INavActionExecutor =
        OpenNoteNavActionExecutor(action, openInEditor = remoteConfig.getBoolean("open_notes_in_editor", defaultValue = false))
}

internal class OpenNoteNavActionExecutor(
    private val action: OpenNoteNavAction,
    private val openInEditor: Boolean,
) : INavActionExecutor {
    override operator fun invoke(navigator: INavigator) {
        if (openInEditor) navigator.goNoteEditor(action.noteId) else navigator.goNoteDetail(action.noteId)
    }
}

// Koin
factory { OpenNoteNavActionFactory(remoteConfig = get()) } binds arrayOf(INavActionFactory::class)
```

Use the two-argument `getBoolean(key, defaultValue)`: the one-argument overload returns `Boolean?`.

## Commons

```kotlin
// web/OpenUrlNavAction.kt
@AutoLogNavAction(identifier = "open_url")
data class OpenUrlNavAction(val uri: Uri) : INavAction

internal class OpenUrlNavActionExecutor(private val action: OpenUrlNavAction) : INavActionExecutor {
    override operator fun invoke(navigator: INavigator) {
        // A chooser avoids ActivityNotFoundException when no default browser is set.
        navigator.navigate { Intent.createChooser(Intent(Intent.ACTION_VIEW, action.uri), null) }
    }
}

// settings/AppSettingsNavAction.kt
@AutoLogNavAction(identifier = "open_app_settings")
data object AppSettingsNavAction : INavAction

internal class AppSettingsNavActionExecutor : INavActionExecutor {
    override operator fun invoke(navigator: INavigator) {
        navigator.navigate {   // receiver is the Activity: packageName is available
            Intent(Settings.ACTION_APPLICATION_DETAILS_SETTINGS, Uri.fromParts("package", packageName, null))
        }
    }
}

// commons/di/CommonsNavigationModule.kt
val CommonsNavigationModule = module {
    factory { BackNavActionFactory() } binds arrayOf(INavActionFactory::class)
    factory { BackNavAction_LogDecorator() } binds arrayOf(INavActionLogDecorator::class)
    factory { OpenUrlNavActionFactory() } binds arrayOf(INavActionFactory::class)
    factory { OpenUrlNavAction_LogDecorator() } binds arrayOf(INavActionLogDecorator::class)
    factory { AppSettingsNavActionFactory() } binds arrayOf(INavActionFactory::class)
    factory { AppSettingsNavAction_LogDecorator() } binds arrayOf(INavActionLogDecorator::class)
}
```

(`Settings` is `android.provider.Settings`.) Each action needs its factory; for a `data object`
the factory's `resolve` simply returns `AppSettingsNavActionExecutor()`.

## Clearing history

`builder` is androidx's `NavOptionsBuilder`, so any NavOptions work:

```kotlin
fun INavigator.goNoteList(clearHistory: Boolean = false) {
    navigate(uri = NOTE_LIST_ROUTE) {
        if (clearHistory) popUpTo(0) { inclusive = true }  // id 0: pop the whole back stack
        launchSingleTop = true
    }
}
```

## Manifest

External links only open screens if the Activity accepts your scheme and it matches `deepLinkScheme`:

```xml
<activity android:name=".MainActivity" android:exported="true" android:launchMode="singleTask">
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="${applicationId}" />
    </intent-filter>
</activity>
```

## Testing

- ViewModel: call `input(...)` and assert `viewModel.nav.value?.handle() == NoteDetailNavAction(42)`.
  NavActions are data classes, so equality is structural.
- Factory/executor: `NavActionDispatcher(listOf(NoteDetailNavActionFactory())).dispatch(action)` returns
  the executor; invoke it with a fake `INavigator` that records `navigate(uri: String, ...)` calls. Note
  that the String overload is a default method that builds a `Uri` via `toUri()`, so either override it
  in the fake or run under Robolectric.
- Registration: start a `koinApplication` with your modules and check that
  `getAll<INavActionFactory<*>>().map { it.actionType }` contains every action class you emit.
