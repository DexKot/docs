---
name: dexkot-screen
description: "Build Jetpack Compose screens on DexKot mobile-core (dev.dexkot.mobile:core-ui). Use whenever the user wants to create or change a screen, ScreenState or UiState class, Screen/Content composable, dialog state, or a Route's screenContent; mentions fieldMutableStateOf, @UiState, rememberSaveable ScreenState, ModelUpdates, onIntention, LocalLogger, LocalPermissionController, LocalRemoteConfig, LocalUiPreference, Preferences(...), IUiPreferences, getBooleanState, rememberPermissionState, ScreenLifeCycle, DefaultTransitions, ImitationOfActivities, ModalTransition or OnEnterTransitionFinished; or needs previews with Dummy* providers. Also use to debug lost UI state after process death or preferences not updating."
---

# DexKot screens (Compose side)

A DexKot screen has three layers between the ViewModel and pixels:

```
ViewModel.model ──ModelUpdates──▶ ScreenState.update(model)      (Route.screenContent)
                                      │  (Parcelable, Compose-observable)
                                      ▼
                     Screen(screenState, onIntention)             dialogs + BackHandler
                                      │
                                      ▼
                     Content(state = screenState.contentUiState, callbacks)
```

- **ScreenState** turns the Model into UI state and holds UI-only state (dialogs, text being typed,
  selections). It is `@Parcelize`, created with `rememberSaveable`, so it survives rotation and
  process death without the ViewModel knowing about it.
- **Screen** receives `(screenState, onIntention)` — never the ViewModel — so it can be previewed and
  tested with a plain state object and a lambda.
- **Content** composables receive only their own UiState subset and plain callbacks.

The ViewModel side (Model, Intention, `Resource`) is covered by `dexkot-viewmodel`.

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
// build.gradle.kts
plugins {
    id("org.jetbrains.kotlin.plugin.compose")
    id("org.jetbrains.kotlin.plugin.parcelize")
}
dependencies {
    implementation("dev.dexkot.mobile:core-ui:0.27.0")
    // core-ui uses these as implementation dependencies; declare them yourself:
    implementation(platform("androidx.compose:compose-bom:2025.10.01"))
    implementation("androidx.compose.runtime:runtime")
    implementation("androidx.activity:activity-compose:1.12.4")      // BackHandler
    implementation("androidx.navigation:navigation-compose:2.9.5")
    implementation("androidx.lifecycle:lifecycle-runtime-compose:2.10.0")
}
```

`core-ui` is Android-only. It exposes `core-logger`, `core-permission` and `core-remoteconfig` as API
dependencies, so their types (`ILoggerInteractor`, `Permission`, `IRemoteConfigProvider`) are
available without extra lines.

## How to build a screen

Running example: the notes list whose ViewModel is built in `dexkot-viewmodel`
(`NotesModel(notes: Resource<List<Note>>, deletion: Resource<Empty>, status: NoteStatus)`,
`NotesIntention.{Back, GoNoteDetail, Refresh, FilterByStatus, Delete}`). The complete, compilable set
of files (content composable, dialog, preview) is in `references/full-example.md`.

### 1. UiState classes with `fieldMutableStateOf`

```kotlin
import android.os.Parcelable
import androidx.compose.runtime.Stable
import androidx.compose.runtime.derivedStateOf
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.setValue
import dev.dexkot.mobile.core.ui.compose.screen.UiState
import dev.dexkot.mobile.core.ui.compose.state.fieldMutableStateOf
import dev.dexkot.mobile.core.ui.viewmodel.resource.Resource
import dev.dexkot.mobile.core.ui.viewmodel.resource.inProgress
import kotlinx.parcelize.IgnoredOnParcel
import kotlinx.parcelize.Parcelize

@Parcelize
@Stable
@UiState
class NotesContentUiState(
    private var _query: String = "",                  // parcelled backing field
) : Parcelable {

    // UI-only state that must survive process death: backing field + fieldMutableStateOf
    @IgnoredOnParcel
    var query: String by fieldMutableStateOf(_query) { _query = it }

    // Copied from the Model: re-pushed by ModelUpdates after restore, no need to parcel
    @IgnoredOnParcel
    var loading: Boolean by mutableStateOf(true)

    @IgnoredOnParcel
    var notes: List<Note> by mutableStateOf(emptyList())

    @IgnoredOnParcel
    var status: NoteStatus by mutableStateOf(NoteStatus.Active)

    @IgnoredOnParcel
    var showError: Boolean by mutableStateOf(false)

    // Derived: recomputed only when what it reads changes
    @IgnoredOnParcel
    val visibleNotes: List<Note> by derivedStateOf {
        if (query.isBlank()) notes else notes.filter { it.title.contains(query, ignoreCase = true) }
    }

    @IgnoredOnParcel
    val showEmpty: Boolean by derivedStateOf { !loading && !showError && visibleNotes.isEmpty() }

    fun update(model: NotesModel) {
        loading = model.notes.inProgress()
        notes = model.notes.data.orEmpty()        // stale data stays visible while refreshing
        showError = model.notes is Resource.Error
        status = model.status
    }
}
```

How `fieldMutableStateOf(initialValue, setValue)` works: it returns a `MutableState` that Compose
observes, and every write (including the destructured setter `val (v, setV) = state`) first calls
`setValue` — which you use to copy the value into the constructor's `private var` backing field.
Parcelize only saves primary-constructor properties, so the backing field is what gets saved and
restored; on restore the class is rebuilt and `fieldMutableStateOf(_query)` starts from the restored
value. Mark every body property `@IgnoredOnParcel` (Parcelize warns otherwise).

Decide per property:

| Property | Use |
|---|---|
| UI-only and must survive process death (open dialog, typed text, selected tab) | backing field + `fieldMutableStateOf` |
| copied from the Model on every update | plain `mutableStateOf` (the Model re-supplies it) |
| computed from other properties | `derivedStateOf` |

`@Stable` tells Compose the class notifies its own changes (through snapshot state), so composables
taking it can skip recomposition. `@UiState` is a marker only; nothing processes it.

### 2. Dialog state lives in the ScreenState, not in the ViewModel

```kotlin
@Parcelize
@Stable
@UiState
class DeleteNoteDialogUiState(
    private var _noteId: Long? = null,
    private var _title: String = "",
) : Parcelable {

    @IgnoredOnParcel
    var noteId: Long? by fieldMutableStateOf(_noteId) { _noteId = it }

    @IgnoredOnParcel
    var title: String by fieldMutableStateOf(_title) { _title = it }

    @IgnoredOnParcel
    val isOpen: Boolean by derivedStateOf { noteId != null }

    fun open(note: Note) {
        noteId = note.id
        title = note.title
    }

    fun close() {
        noteId = null
        title = ""
    }
}
```

Opening a confirmation dialog is not a user decision the ViewModel must act on; confirming it is. So
`open()`/`close()` are local state changes and only the confirmation becomes `NotesIntention.Delete`.
Keeping it here also means the open dialog is restored after process death for free.

### 3. The ScreenState aggregates UiStates

```kotlin
@Parcelize
@Stable
@UiState
class NotesScreenState(
    val contentUiState: NotesContentUiState = NotesContentUiState(),
    val deleteDialogUiState: DeleteNoteDialogUiState = DeleteNoteDialogUiState(),
) : Parcelable {

    fun update(model: NotesModel) {
        contentUiState.update(model)
    }
}
```

Nested UiStates are constructor `val`s so they are parcelled with the parent. All of them must be
`Parcelable`.

### 4. The Screen composable

```kotlin
import androidx.activity.compose.BackHandler
import androidx.compose.runtime.Composable

@Composable
fun NotesScreen(
    screenState: NotesScreenState,
    onIntention: (NotesIntention) -> Unit,
) {
    BackHandler { onIntention(NotesIntention.Back) }

    NotesContent(
        state = screenState.contentUiState,
        onBack = { onIntention(NotesIntention.Back) },
        onQueryChange = { query -> screenState.contentUiState.query = query },   // UI-only
        onNoteClick = { note -> onIntention(NotesIntention.GoNoteDetail(noteId = note.id)) },
        onDeleteClick = { note -> screenState.deleteDialogUiState.open(note) },
        onRefresh = { onIntention(NotesIntention.Refresh) },
        onStatusSelected = { status -> onIntention(NotesIntention.FilterByStatus(status)) },
    )

    DeleteNoteDialog(
        state = screenState.deleteDialogUiState,
        onConfirm = { noteId ->
            onIntention(NotesIntention.Delete(noteId = noteId))
            screenState.deleteDialogUiState.close()
        },
        onDismiss = { screenState.deleteDialogUiState.close() },
    )
}
```

Rules and why:

- Dialogs, bottom sheets and `BackHandler` are declared at Screen level. The Screen is the only place
  that sees every piece of state, so it is the one that can decide "back closes the sheet" versus
  "back sends `Back`". For in-layout overlays, add `BackHandler(enabled = overlay.isOpen) {
  overlay.close() }` after the screen-wide one (the last enabled handler composed wins).
- Content composables take their UiState subset plus callbacks named after what happened in the UI
  (`onDeleteClick`). The Screen maps those to intentions or local state changes; the Content never
  sees `NotesIntention`, so it is reusable and previewable alone.

### 5. Wire it in the Route

```kotlin
screenContent = { viewModel ->
    val screenState = rememberSaveable { NotesScreenState() }
    viewModel.ModelUpdates { screenState.update(it) }
    NotesScreen(screenState = screenState, onIntention = viewModel::input)
}
```

`rememberSaveable` stores the Parcelable ScreenState in the destination's saved state (per back-stack
entry). `ModelUpdates` collects `viewModel.model` and calls `update` in a `LaunchedEffect(model)`, once
per distinct Model. The full Route (`identifier`, `route`, `arguments`, `viewModelFactory`,
`transitions`) is in `references/full-example.md`; the navigation side is in `dexkot-navigation`.

## Composition locals

The navigation host (`FragmentActivity.Navigation(...)`) wraps **each destination** with:

| Local | Type | Provided per destination | Default outside |
|---|---|---|---|
| `LocalLogger` | `ILoggerInteractor` | `loggerFactory(route.identifier)` | `DummyLoggerInteractor` (no-op) |
| `LocalPermissionController` | `IPermissionController` | your controller, wrapped to log requests/results | `DummyPermissionController()` (everything granted) |
| `LocalRemoteConfig` | `IRemoteConfigProvider` | `remoteConfigProvider` | `DummyRemoteConfigProvider` (defaults) |
| `LocalUiPreference` | `IUiPreferences` | **not provided** | `DummyUiPreference` (no-op writes) |
| `LocalNavAnimatedVisibilityScope` | `AnimatedVisibilityScope?` | the destination's scope | `null` |

`LocalUiPreference` is the one you must provide yourself, around the whole app:

```kotlin
setContent {
    Preferences(uiPreferences = koinInject<IUiPreferences>()) {
        AppTheme {
            Navigation(/* graphs, navigator, dispatcher, permissionController, ... */)
        }
    }
}
```

Each local has a same-named provider composable: `Logger(loggerInteractor) { }`,
`PermissionController(permissionController) { }`, `RemoteConfig(remoteConfigProvider) { }`,
`Preferences(uiPreferences) { }`. Previews need none of them — the dummies are the defaults — but you
can provide a configured dummy to preview edge cases (for example
`DummyPermissionController(mapOf(Permission.Camera to deniedState))`).

## Helpers at a glance

Signatures and details: `references/api.md`.

- **UI preferences** (`dev.dexkot.mobile.core.ui.compose.preferences`):
  `LocalUiPreference.current.getBooleanState(key, defaultValue)` → `State<Boolean>`; also
  `getStringState`, `getStringSetState`, `getFloatState`, and `getState(key, defaultValue, converter)`
  for string-encoded values. Write with `setBoolean`/`setString`/… on the same `IUiPreferences`.
  Wrap keys in extension functions (`fun IUiPreferences.isGridLayout()`) so keys stay private.
- **Remote config** (`dev.dexkot.mobile.core.ui.remoteconfig`):
  `LocalRemoteConfig.current.getBooleanState(key, defaultValue)`, `getLongState`, `getStringState`.
- **Permissions** (`dev.dexkot.mobile.core.ui.compose.permission`): `rememberPermissionState(permission)`
  and `rememberPermissionsState(vararg permissions)` re-read the state on every `ON_RESUME`;
  request through `LocalPermissionController.current.requestPermission(permission) { result -> }`.
  The app creates the controller once with `rememberPermissionController(activity, preferences)` and
  passes it to `Navigation`.
- **Lifecycle** (`dev.dexkot.mobile.core.ui.compose.screen`): `ScreenLifeCycle(onCreate, onStart,
  onResume, onPause, onStop, onDestroy)` observes `LocalLifecycleOwner`.
- **Transitions** (`dev.dexkot.mobile.core.ui.compose.transitions`): pass `DefaultTransitions` (fade,
  the Route default), `ImitationOfActivities` (horizontal slide) or `ModalTransition` (slide up) as a
  Route's `transitions`, or build your own `Transitions(...)`. Inside a screen,
  `OnEnterTransitionFinished { }`, `OnExitTransitionFinished { }` and `rememberIsScreenTransitioning()`
  let you defer heavy work or animations until the screen has settled.

## Gotchas

- **Forgetting `Preferences(...)`.** Without it `LocalUiPreference` is `DummyUiPreference`: writes are
  silently dropped and reads always return the default. There is no error.
- **Checking permissions outside a destination.** Above the navigation host, `LocalPermissionController`
  is the dummy and reports every permission as granted.
- **Preference flows only see writes through the same instance.** `UiPreferences` updates its cached
  `StateFlow` in its own setters. Writes through another `UiPreferences` instance or directly to the
  `SharedPreferences` are not observed. Register `IUiPreferences` as a single instance.
- **`defaultValue` is fixed at the first flow request per key.** The first `getXFlow(key, default)` (or
  `getXState`) creates and caches the flow; later calls with a different default get the cached flow.
- **`remove(key)` detaches existing observers.** It drops the cached flow, so `State`s already
  collecting that key stop updating until they re-request it.
- **Remote config states ignore key changes.** `getBooleanState` / `getLongState` / `getStringState` on
  `IRemoteConfigProvider` use `remember { }` without keys: a different `key` or `defaultValue` on a
  later recomposition is ignored. Use constant keys. (`rememberPermissionsState` behaves the same way
  for its permission list.)
- **Same-named helpers.** `getBooleanState` / `getStringState` exist for both `IUiPreferences`
  (`...ui.compose.preferences`) and `IRemoteConfigProvider` (`...ui.remoteconfig`). Import the right
  one; alias if a file needs both. Likewise the `@UiState` annotation clashes with the logger's
  `UiState` enum (`dev.dexkot.mobile.core.logger.events.ui.state.UiState`).
- **The first frame shows ScreenState defaults.** `ModelUpdates` applies the Model in a
  `LaunchedEffect`, after the first composition. Choose defaults that look right before data arrives
  (`loading = true`, not an empty-state message).
- **`update(model)` receives the whole Model every time anything changes.** Derive state from the
  current Model; do not treat a field as an event (for example "close the dialog when `deletion` is
  `Loaded`" also fires on the next unrelated update, closing a dialog the user just opened again).
- **Model must be an immutable data class.** In-place mutation emits nothing and `ModelUpdates` never
  runs.
- **`@RawValue` types in a UiState must still be Parcelable at runtime**, or `rememberSaveable` crashes
  when the activity saves state.
- **`rememberPermissionController` must be called unconditionally**, once, during composition of the
  activity content: it registers two activity-result launchers, and Compose requires those to be
  registered on every composition in the same order.
- **`ScreenLifeCycle`'s `onDestroy` runs when the composable leaves composition** (on dispose), not on
  `Lifecycle.Event.ON_DESTROY`. It therefore also runs when navigating forward or on configuration
  change.
- **`onCreate` in `ScreenLifeCycle` fires again each time the composable re-enters composition** (a new
  observer receives the catch-up events).
- **`OnEnterTransitionFinished` fires immediately in previews** (no navigation scope);
  `OnExitTransitionFinished` never fires there.

## References and related skills

- `references/full-example.md` — every file of the notes screen: UiStates, Screen, Content, dialog,
  Route, previews, and a small permission + preference example.
- `references/api.md` — signatures for composition locals, dummies, UI preferences, remote config and
  permission helpers, `ScreenLifeCycle`, transitions, `fieldMutableStateOf`.
- `dexkot-viewmodel` — Model, Intention, `Resource`, `ModelUpdates` source of truth.
- `dexkot-navigation` — `Route`, `Graph`, `Navigation(...)`, `rememberNavigator`, NavActions.
- `dexkot-logging` — `LocalLogger` interaction/UI-state events, `loggerFactory`.
- `dexkot-infra` — remote config and permission modules behind the composition locals.
- `dexkot-di-koin` — `uiModule()` (provides `IUiPreferences`), `permissionModule()`.
