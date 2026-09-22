# core-ui Compose API reference (0.27.0)

Artifact: `dev.dexkot.mobile:core-ui:0.27.0` (Android AAR). All packages below are under
`dev.dexkot.mobile.core.ui`.

## Contents

1. [State: fieldMutableStateOf, @UiState](#state-fieldmutablestateof-uistate)
2. [ModelUpdates](#modelupdates)
3. [Composition locals and providers](#composition-locals-and-providers)
4. [Dummies for previews](#dummies-for-previews)
5. [UI preferences](#ui-preferences)
6. [Remote config helpers](#remote-config-helpers)
7. [Permission helpers](#permission-helpers)
8. [ScreenLifeCycle](#screenlifecycle)
9. [Transitions](#transitions)

## State: fieldMutableStateOf, @UiState

```kotlin
package dev.dexkot.mobile.core.ui.compose.state

fun <T> fieldMutableStateOf(
    initialValue: T,
    setValue: (T) -> Unit,
): MutableState<T>
```

Returns a `MutableState<T>` backed by a regular `mutableStateOf(initialValue)`. Every write calls
`setValue(newValue)` **before** updating the observable state — through `value =`, a delegated
property (`var x by fieldMutableStateOf(...)`), or the destructured setter
(`val (x, setX) = fieldMutableStateOf(...)`). Creating it does not call `setValue`.

Pattern:

```kotlin
@Parcelize
@Stable
@UiState
class ExpandableUiState(private var _expanded: Boolean = false) : Parcelable {
    @IgnoredOnParcel
    var expanded: Boolean by fieldMutableStateOf(_expanded) { _expanded = it }
}
```

The same callback can do more than sync the field (for example reset an error flag when the text
changes): `fieldMutableStateOf(_name) { _name = it; error = false }`.

```kotlin
package dev.dexkot.mobile.core.ui.compose.screen

@Retention(AnnotationRetention.RUNTIME)
@Target(AnnotationTarget.CLASS)
annotation class UiState
```

Marker for ScreenState/UiState classes. Nothing processes it.

## ModelUpdates

```kotlin
package dev.dexkot.mobile.core.ui.viewmodel

@Composable
fun <Model, Intention> IViewModel<Model, Intention>.ModelUpdates(onUpdate: (Model) -> Unit)
```

`model.collectAsState()` + `LaunchedEffect(model) { onUpdate(model) }`.

## Composition locals and providers

| Local (package) | Type | Default | Provider composable |
|---|---|---|---|
| `LocalLogger` (`compose.logger`) | `ILoggerInteractor` | `DummyLoggerInteractor` | `Logger(loggerInteractor, content)` |
| `LocalPermissionController` (`compose.permission`) | `IPermissionController` | `DummyPermissionController()` | `PermissionController(permissionController, content)` |
| `LocalRemoteConfig` (`compose.remoteconfig`) | `IRemoteConfigProvider` | `DummyRemoteConfigProvider` | `RemoteConfig(remoteConfigProvider, content)` |
| `LocalUiPreference` (`compose.preferences`) | `IUiPreferences` | `DummyUiPreference` | `Preferences(uiPreferences, content)` |
| `LocalNavAnimatedVisibilityScope` (`compose.transitions`) | `AnimatedVisibilityScope?` | `null` | `NavAnimatedVisibility(scope, content)` |

All are `staticCompositionLocalOf`: changing the provided value recomposes the whole subtree, which is
fine because they are set once per destination/app.

What `FragmentActivity.Navigation(...)` provides around each destination, in order:
`RemoteConfig(remoteConfigProvider)` → `Logger(loggerFactory(route.identifier))` →
`PermissionController(permissionController.withLogging(logger))` → the ViewModel, the navigation
subscriber and `route.screenContent(viewModel)`. `NavAnimatedVisibility` wraps it all with the
destination's `AnimatedContentScope`. `LocalUiPreference` is never provided by the host.

`IPermissionController.withLogging(logger: ILoggerInteractor): IPermissionController` (package
`compose.permission`) is public: it logs each request (`logger.permission(keys)`) and each result
(`logger.permission(key to status)`, with `AutoDenied` logged as `RequestIgnored`).

## Dummies for previews

| Dummy | Behaviour |
|---|---|
| `compose.logger.DummyLoggerInteractor` (object) | every call is a no-op |
| `dev.dexkot.mobile.core.permission.controller.DummyPermissionController(permissionStates: Map<Permission, PermissionState> = emptyMap())` | unknown permissions are `PermissionState.Granted`; requests answer immediately from the map (`Denied` → `Denied`, `PermanentlyDenied` → `AutoDenied`) |
| `compose.remoteconfig.DummyRemoteConfigProvider` (object) | `getBoolean(key)` → `false`, `getString(key)` → `""`, `getLong(key)` → `0`; flows emit the given default |
| `compose.remoteconfig.DummyRemoteConfigHandler` (object) | backing handler of the provider above; `sync()` → `Success(true)` |
| `compose.preferences.DummyUiPreference` (object) | writes ignored; reads and flows return the default |

Preview with a denied permission:

```kotlin
@Preview
@Composable
private fun CameraDeniedPreview() {
    PermissionController(
        permissionController = DummyPermissionController(
            mapOf(
                Permission.Camera to PermissionState(
                    status = PermissionStatus.Denied,
                    requiresRationale = true,
                    isFirstTimeEver = false,
                    isFirstTimeThisSession = true,
                )
            )
        )
    ) {
        EditorToolbar(onCameraReady = { }, onOpenSettings = { })
    }
}
```

## UI preferences

```kotlin
package dev.dexkot.mobile.core.ui.preferences

interface IUiPreferences {
    fun setBoolean(key: String, value: Boolean)
    fun getBoolean(key: String, defaultValue: Boolean = false): Boolean
    fun getBooleanFlow(key: String, defaultValue: Boolean = false): StateFlow<Boolean>
    fun getString(key: String, defaultValue: String = ""): String
    fun setString(key: String, value: String)
    fun getStringFlow(key: String, defaultValue: String = ""): StateFlow<String>
    fun getStringSet(key: String, defaultValue: Set<String> = emptySet()): Set<String>
    fun setStringSet(key: String, value: Set<String>)
    fun getStringSetFlow(key: String, defaultValue: Set<String> = emptySet()): StateFlow<Set<String>>
    fun setFloat(key: String, value: Float)
    fun getFloat(key: String, defaultValue: Float = 0f): Float
    fun getFloatFlow(key: String, defaultValue: Float = 0f): StateFlow<Float>
    fun remove(key: String)
}

class UiPreferences(sharedPreferences: SharedPreferences) : IUiPreferences
```

Purpose: small, UI-only settings (layout mode, sort order, "coach mark seen", theme). Domain settings
belong in the data layer.

`UiPreferences` behaviour:

- Setters write to `SharedPreferences` (`apply()`, asynchronous) and update the cached flow for that
  key, if one exists.
- `getXFlow(key, default)` creates a `MutableStateFlow` initialised from `SharedPreferences` on first
  call per key and returns the cached one afterwards (the `defaultValue` of later calls is ignored).
- Flows see only writes made through the same `UiPreferences` instance.
- `remove(key)` deletes the value and drops the cached flows for that key; existing collectors keep
  the old flow and stop receiving updates.
- Keys share one `SharedPreferences` file: reading a key with a different type than it was written
  with throws `ClassCastException`.

`core-ui-di-koin`'s `uiModule(preferencesName: String = "dev.dexkot.ui.preferences")` registers a
single `UiPreferences` on that file. Pass your old file name when migrating, so existing users keep
their settings.

Compose helpers (package `dev.dexkot.mobile.core.ui.compose.preferences`):

```kotlin
@Composable fun IUiPreferences.getBooleanState(key: String, defaultValue: Boolean = false): State<Boolean>
@Composable fun IUiPreferences.getStringState(key: String, defaultValue: String = ""): State<String>
@Composable fun IUiPreferences.getStringSetState(key: String, defaultValue: Set<String> = emptySet()): State<Set<String>>
@Composable fun IUiPreferences.getFloatState(key: String, defaultValue: Float = 0f): State<Float>
@Composable fun <T> IUiPreferences.getState(
    key: String,
    defaultValue: String = "",
    converter: (String) -> T
): State<T>
```

Each remembers the flow keyed by `key` and collects it. `getState` stores a string and maps it with
`converter` through `derivedStateOf` (for example an enum: `getState(KEY, SortOrder.Date.name) {
SortOrder.valueOf(it) }`); the converter is captured on first composition for a key.

`SharedPreferences` extensions in `dev.dexkot.mobile.core.ui.preferences`: `setString`, `setBoolean`,
`setFloat`, `setStringSet`, `delete(key)` — each a one-line `edit { }`.

## Remote config helpers

Package `dev.dexkot.mobile.core.ui.remoteconfig`:

```kotlin
@Composable fun IRemoteConfigProvider.getBooleanState(key: String, defaultValue: Boolean): State<Boolean>
@Composable fun IRemoteConfigProvider.getLongState(key: String, defaultValue: Long): State<Long>
@Composable fun IRemoteConfigProvider.getStringState(key: String, defaultValue: String): State<String>
```

Each calls `getXFlow(key, defaultValue)` inside `remember { }` **without keys** and collects it; key or
default changes after the first composition are ignored. Use with `LocalRemoteConfig.current`.
`IRemoteConfigProvider` itself (sync, non-Compose getters) is covered by `dexkot-infra`.

## Permission helpers

Package `dev.dexkot.mobile.core.ui.compose.permission`:

```kotlin
@Composable
fun rememberPermissionController(activity: Activity, preferences: SharedPreferences): IPermissionController

@Composable
fun rememberPermissionState(permission: Permission): State<PermissionState>

@Composable
fun rememberPermissionsState(vararg permissions: Permission): State<Map<Permission, PermissionState>>
```

- `rememberPermissionController` registers two activity-result launchers
  (`RequestPermission`, `RequestMultiplePermissions`) and builds a `PermissionController`. Call it
  once, unconditionally, in the activity's content and pass it to `Navigation(permissionController =
  ...)`. `preferences` stores request history (first time ever / this session); `permissionModule()`
  from `core-permission-di-koin` provides one under the `permissionPreferences()` qualifier.
- `rememberPermissionState` reads `LocalPermissionController.current.getState(permission)` initially
  and on every `ON_RESUME` of the current lifecycle owner (returning from the system dialog or from
  Settings triggers it).
- `rememberPermissionsState` does the same for several permissions; the list is captured on first
  composition.

`PermissionState` (from `core-permission`): `status: PermissionStatus` (`Granted`, `Denied`,
`PermanentlyDenied`), `requiresRationale`, `isFirstTimeEver`, `isFirstTimeThisSession`, plus
`isGranted`, `canRequest` (not permanently denied) and `shouldShowDisclosure` (not granted and first
time or rationale required). Request results are `PermissionResult.Granted`, `Denied`, `AutoDenied`
(not asked because permanently denied).

## ScreenLifeCycle

```kotlin
package dev.dexkot.mobile.core.ui.compose.screen

@Composable
fun ScreenLifeCycle(
    onCreate: () -> Unit = { },
    onStart: () -> Unit = { },
    onResume: () -> Unit = { },
    onPause: () -> Unit = { },
    onStop: () -> Unit = { },
    onDestroy: () -> Unit = { },
)
```

Adds a `LifecycleEventObserver` to `LocalLifecycleOwner.current` (the `NavBackStackEntry` inside a
destination) in a `DisposableEffect(lifecycleOwner)`. `ON_CREATE`…`ON_STOP` map to the callbacks;
`ON_DESTROY` is ignored. `onDestroy` runs in `onDispose`, when the composable leaves composition. The
lambdas are captured by the effect: they are not updated on recomposition, so read changing values
through state (`rememberUpdatedState`) if needed. The navigation host itself uses `ScreenLifeCycle` to
call the ViewModel's `onCreate`/`onResume`/`onStop`.

## Transitions

Package `dev.dexkot.mobile.core.ui.compose.transitions`:

```kotlin
class Transitions(
    val enterTransition: AnimatedContentTransitionScope<NavBackStackEntry>.() -> EnterTransition,
    val exitTransition: AnimatedContentTransitionScope<NavBackStackEntry>.() -> ExitTransition,
    val popEnterTransition: AnimatedContentTransitionScope<NavBackStackEntry>.() -> EnterTransition,
    val popExitTransition: AnimatedContentTransitionScope<NavBackStackEntry>.() -> ExitTransition,
)

val DefaultTransitions: Transitions     // fade in/out, 250 ms after a 100 ms delay (Route default)
val ImitationOfActivities: Transitions  // enters from the right; when covered slides half-way left; pop reverses
val ModalTransition: Transitions        // slides up on enter, down on pop; fades out (after 350 ms) when covered; no pop-enter

@Composable fun rememberIsScreenTransitioning(): State<Boolean>
@Composable fun OnEnterTransitionFinished(onFinished: () -> Unit)
@Composable fun OnExitTransitionFinished(onFinished: () -> Unit)
```

Set per Route with `transitions = ModalTransition`. A Route's `enter`/`popExit` animate that screen
appearing and being popped; its `exit`/`popEnter` animate it being covered by, and uncovered from,
the next screen (whose own `enter`/`popExit` run at the same time). All durations are 250 ms after a
100 ms delay unless noted.

- `rememberIsScreenTransitioning()` — `true` while the destination's enter/exit transition runs;
  always `false` outside a destination (previews).
- `OnEnterTransitionFinished { }` — calls `onFinished` once the enter transition has settled on
  `Visible`. Outside a destination it fires immediately, so previews and tests do not wait forever.
  Use it to start heavy work (large lists, video, keyboard focus) after the animation, which keeps the
  transition smooth.
- `OnExitTransitionFinished { }` — calls `onFinished` when the exit transition settles on `PostExit`,
  just before the destination is removed. No-op outside a destination.

```kotlin
@Composable
fun SearchContent(state: SearchContentUiState) {
    val focusRequester = remember { FocusRequester() }
    OnEnterTransitionFinished { focusRequester.requestFocus() }   // keyboard after the slide, not during
    TextField(
        value = state.query,
        onValueChange = { state.query = it },
        modifier = Modifier.focusRequester(focusRequester),
    )
}
```
