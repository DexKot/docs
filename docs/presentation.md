# Presentation layer: ViewModels and screens

This guide shows how to build a screen with `dev.dexkot.mobile:core-ui`: a ViewModel that exposes a
**Model** and accepts **Intentions**, and a Compose side made of a **ScreenState**, a **Screen**
composable and **Content** composables. It walks through one complete screen — a list of notes inside
a project — and ends with an API reference.

Related guides: results and errors (`ResultOf`), navigation (`Route`, NavActions), Koin DI, logging.

## Contents

1. [Setup](#setup)
2. [The shape of a screen](#the-shape-of-a-screen)
3. [Model](#model)
4. [Intention](#intention)
5. [ViewModelState](#viewmodelstate)
6. [The ViewModel](#the-viewmodel)
7. [Resource<T>](#resourcet)
8. [Lifecycle callbacks and navigation](#lifecycle-callbacks-and-navigation)
9. [Registering the ViewModel](#registering-the-viewmodel)
10. [ScreenState and fieldMutableStateOf](#screenstate-and-fieldmutablestateof)
11. [Screen and Content composables](#screen-and-content-composables)
12. [Wiring it in the Route](#wiring-it-in-the-route)
13. [Composition locals](#composition-locals)
14. [UI preferences, remote config and permissions](#ui-preferences-remote-config-and-permissions)
15. [Lifecycle and transitions in Compose](#lifecycle-and-transitions-in-compose)
16. [Testing and previews](#testing-and-previews)
17. [Pitfalls](#pitfalls)
18. [API reference](#api-reference)

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
```

```kotlin
// app/build.gradle.kts
plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
    id("org.jetbrains.kotlin.plugin.compose")
    id("org.jetbrains.kotlin.plugin.parcelize")
    id("com.google.devtools.ksp")                                   // for @AutoLogViewModel
}

dependencies {
    implementation("dev.dexkot.mobile:core-ui:0.27.0")
    implementation("dev.dexkot.mobile:core-ui-di-koin:0.27.0")        // uiModule()
    implementation("dev.dexkot.mobile:core-annotation:0.27.0")
    ksp("dev.dexkot.mobile:core-ksp-processor:0.27.0")

    // core-ui depends on these as implementation dependencies; declare them yourself
    implementation(platform("androidx.compose:compose-bom:2025.10.01"))
    implementation("androidx.compose.runtime:runtime")
    implementation("androidx.compose.material3:material3")
    implementation("androidx.activity:activity-compose:1.12.4")
    implementation("androidx.navigation:navigation-compose:2.9.5")
    implementation("androidx.lifecycle:lifecycle-viewmodel-compose:2.10.0")
    implementation("androidx.lifecycle:lifecycle-runtime-compose:2.10.0")
}
```

Platform support: `core-ui` is Android-only (AAR). It re-exports `core-foundation`, `core-logger`,
`core-permission` and `core-remoteconfig`.

## The shape of a screen

```
            input(intent)                         model: StateFlow<Model>
Screen ─────────────────────▶ ViewModel ─────────────────────────────────▶ ScreenState.update(model)
  ▲          (Intention)          │  emit(INavAction)                            │
  │                               ▼                                              │
  │                        navigation host                                        │
  └──────────────────────── Screen(screenState, onIntention) ◀────────────────────┘
```

- The **ViewModel** owns data and decisions. It exposes one immutable Model and has one entry point,
  `input(intent)`. Navigation is a value it emits, not a call it makes.
- The **ScreenState** is a Parcelable, Compose-observable object that translates the Model into what
  the UI draws, and holds state only the UI cares about (open dialogs, text being typed).
- The **Screen** composable gets the ScreenState and an `onIntention` lambda, never the ViewModel.

Why: a screen becomes "data in, data out" on both sides. ViewModels are tested by sending intentions
and reading the Model; composables are previewed and tested with a plain ScreenState. Every user
decision passes through one function, so a generated decorator can log all of them.

The example uses these domain types and use cases (your own code):

```kotlin
data class Note(val id: Long, val title: String)
enum class NoteStatus { Active, Archived }

interface IGetNotesUseCase {
    suspend operator fun invoke(projectId: Long, status: NoteStatus): ResultOf<List<Note>>
}
interface IDeleteNoteUseCase {
    suspend operator fun invoke(noteId: Long): ResultOf<Unit>
}
```

## Model

```kotlin
package com.example.notes.list.viewmodel

import com.example.notes.domain.Note
import com.example.notes.domain.NoteStatus
import dev.dexkot.mobile.core.ui.viewmodel.empty.Empty
import dev.dexkot.mobile.core.ui.viewmodel.resource.Resource

data class NotesModel(
    val notes: Resource<List<Note>>,   // loaded asynchronously
    val deletion: Resource<Empty>,     // an operation with no payload
    val status: NoteStatus,            // a plain value
)
```

Fields that load asynchronously are `Resource<T>` so the UI can render loading, error and stale data.
Everything else is a plain value. The Model is a `data class` of `val`s: the UI reacts only to a new,
unequal Model, so in-place mutation is invisible.

## Intention

```kotlin
package com.example.notes.list.viewmodel

import com.example.notes.domain.NoteStatus
import dev.dexkot.mobile.core.annotation.AutoLogIntention
import dev.dexkot.mobile.core.annotation.LogParam

sealed interface NotesIntention {

    @AutoLogIntention("go_back")
    data object Back : NotesIntention

    @AutoLogIntention("go_note_detail", params = [LogParam(logName = "note_id", paramName = "noteId")])
    data class GoNoteDetail(val noteId: Long) : NotesIntention

    @AutoLogIntention("refresh")
    data object Refresh : NotesIntention

    @AutoLogIntention("filter_by_status", params = [LogParam(logName = "status", paramName = "status")])
    data class FilterByStatus(val status: NoteStatus) : NotesIntention

    @AutoLogIntention("delete_note", params = [LogParam(logName = "note_id", paramName = "noteId")])
    data class Delete(val noteId: Long) : NotesIntention
}
```

Naming conventions:

| Kind | Convention | Examples |
|---|---|---|
| Navigate to a destination | `Go{Destination}` | `GoNoteDetail`, `GoSettings` |
| Leave the screen | `Back` | `Back` |
| Domain action | verb | `Refresh`, `Delete`, `FilterByStatus` |

Intentions describe what the user wants, never how they expressed it: no `BackClicked`,
`PullToRefresh` or `SwipeToDelete`. A swipe and a menu item that both delete a note are the same
intention; the ViewModel should not change when the gesture does, and analytics should count the
decision. The `Go` prefix keeps navigation branches easy to spot and avoids ambiguity (`GoSync` opens
a screen; `Sync` syncs).

Opening a dialog or typing into a field is not an intention: it is UI state and lives in the
ScreenState. Only committing to something the ViewModel must act on is.

## ViewModelState

A small class per ViewModel that is the only code touching `SavedStateHandle`. It is a convention,
not a library type.

```kotlin
package com.example.notes.list.viewmodel

import androidx.lifecycle.SavedStateHandle
import com.example.notes.domain.NoteStatus
import com.example.notes.list.route.NotesRoute.getProjectId
import kotlinx.coroutines.flow.MutableStateFlow

class NotesViewModelState(savedStateHandle: SavedStateHandle) {

    val projectId: Long = savedStateHandle.getProjectId()

    val status: MutableStateFlow<NoteStatus> =
        savedStateHandle.getMutableStateFlow(STATUS_KEY, NoteStatus.Active)

    private companion object {
        const val STATUS_KEY = "status"
    }
}
```

- Route parameters are read through extensions the Route defines, because the Route owns the
  parameter names in its route string.
- Values that must survive process death use androidx's `getMutableStateFlow(key, initialValue)`:
  every write is saved automatically. Values must be Bundle-compatible (primitives, `String`, enums,
  `Parcelable`, `Serializable`).
- Keys stay private; the ViewModel only sees typed values, and a test can build the state from
  `SavedStateHandle(mapOf("project_id" to 7L))`.

## The ViewModel

```kotlin
package com.example.notes.list.viewmodel

import android.os.Bundle
import androidx.lifecycle.SavedStateHandle
import com.example.navigation.BackNavAction
import com.example.notes.detail.route.NoteDetailNavAction
import com.example.notes.domain.IDeleteNoteUseCase
import com.example.notes.domain.IGetNotesUseCase
import com.example.notes.domain.Note
import com.example.notes.domain.NoteStatus
import dev.dexkot.mobile.core.annotation.AutoLogViewModel
import dev.dexkot.mobile.core.foundation.result.ResultOf
import dev.dexkot.mobile.core.ui.viewmodel.ViewModel
import dev.dexkot.mobile.core.ui.viewmodel.android.AndroidViewModel
import dev.dexkot.mobile.core.ui.viewmodel.empty.Empty
import dev.dexkot.mobile.core.ui.viewmodel.resource.Resource
import kotlinx.coroutines.Job
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.combine
import kotlinx.coroutines.flow.update
import kotlinx.coroutines.launch

@ViewModel
@AutoLogViewModel
class NotesViewModel(
    private val getNotes: IGetNotesUseCase,
    private val deleteNote: IDeleteNoteUseCase,
    savedStateHandle: SavedStateHandle,
) : AndroidViewModel<NotesModel, NotesIntention>() {

    private val state = NotesViewModelState(savedStateHandle)

    private val _notes = MutableStateFlow<Resource<List<Note>>>(Resource.None())
    private val _deletion = MutableStateFlow<Resource<Empty>>(Resource.None())
    private val _status = state.status

    private var loadJob: Job? = null

    override val model: StateFlow<NotesModel> =
        combine(_notes, _deletion, _status) { notes, deletion, status ->
            NotesModel(notes = notes, deletion = deletion, status = status)
        }.stateFlow(
            initialValue = NotesModel(
                notes = _notes.value,
                deletion = _deletion.value,
                status = _status.value,
            )
        )

    override fun onCreate(args: Bundle?) {
        if (_notes.value is Resource.None) loadNotes()     // onCreate can run more than once
    }

    override fun input(intent: NotesIntention) {
        when (intent) {
            NotesIntention.Back -> emit(BackNavAction())
            is NotesIntention.GoNoteDetail -> emit(NoteDetailNavAction(noteId = intent.noteId))
            NotesIntention.Refresh -> loadNotes()
            is NotesIntention.FilterByStatus -> onFilterByStatus(intent.status)
            is NotesIntention.Delete -> onDelete(intent.noteId)
        }
    }

    private fun onFilterByStatus(status: NoteStatus) {
        if (_status.value == status) return
        _status.value = status
        loadNotes()
    }

    private fun loadNotes() {
        loadJob?.cancel()
        loadJob = viewModelScope.launch {
            _notes.update { it.toLoading() }
            when (val result = getNotes(projectId = state.projectId, status = _status.value)) {
                is ResultOf.Success -> _notes.update { it.toLoaded(result.data) }
                is ResultOf.Failure -> _notes.update { it.toError(result.error) }
            }
        }
    }

    private fun onDelete(noteId: Long) {
        viewModelScope.launch {
            _deletion.update { it.toLoading() }
            when (val result = deleteNote(noteId = noteId)) {
                is ResultOf.Success -> {
                    _notes.update { notes -> notes.update { list -> list.filterNot { it.id == noteId } } }
                    _deletion.update { it.toLoaded(Empty) }
                }
                is ResultOf.Failure -> _deletion.update { it.toError(result.error) }
            }
        }
    }
}
```

`BackNavAction` and `NoteDetailNavAction` are app-defined `INavAction`s (see the navigation guide).

Points worth noticing:

- **`ResultOf` becomes `Resource` here**, with an exhaustive `when`. Lower layers never see
  `Resource`; the UI never sees `ResultOf`.
- **`model` combines private flows** and finishes with `stateFlow(initialValue = …)`, a shortcut for
  `stateIn(viewModelScope, SharingStarted.WhileSubscribed(), initialValue)`.
- **`emit()` is a plain function.** No coroutine is needed.
- **`@ViewModel`** is a marker annotation; nothing processes it. Import `AndroidViewModel` from
  `dev.dexkot.mobile.core.ui.viewmodel.android`, not from androidx.

### Strings in a ViewModel

Resolve display text in Compose with `stringResource`. When a ViewModel must produce text itself (a
default title that gets saved, a share message), inject `IResourceProvider`: it offers
`getString(id)` and `getString(resId, vararg formatArgs)` without a `Context`, and is easy to fake.

## Resource\<T\>

| State | `data` | Meaning |
|---|---|---|
| `Resource.None(data?)` | optional | idle, nothing requested yet |
| `Resource.Loading(data?)` | optional | in flight; `data` is the previous value, if any |
| `Resource.Loaded(data)` | required | success |
| `Resource.Error(data?, error)` | optional | failed with an `ErrorEntity`; `data` is the previous value, if any |

Transitions:

```kotlin
_notes.update { it.toLoading() }          // keeps current data: a refresh shows the old list under a spinner
_notes.update { it.toLoaded(fresh) }
_notes.update { it.toError(error) }       // keeps current data: show the old list plus an error
_notes.update { it.toLoading(data = null) } // start over without stale data
_notes.update { it.toNone() }             // back to idle, data dropped
```

- `map { }` transforms data and keeps the state; `map()` without arguments turns it into
  `Resource<Unit>` and drops data. `update { }` transforms data in place.
- `inProgress()` is `true` for `Loading` **and** `None`. Use `None` only for "not requested yet".
- On flows: `values()` emits only loaded data; `finished()` drops in-progress states.
  `onLoaded(flowA, flowB) { a, b -> }` suspends until each flow is `Loaded` (1, 2 and 3-flow
  overloads); `onFinished(a, b, c) { … }` waits for `Loaded` or `Error` and exists only for 3 flows.
- `Empty` is a Parcelable object for operations without a result: prefer `Resource<Empty>` over
  `Resource<Unit>`, because `Unit` cannot be saved in a Bundle.

## Lifecycle callbacks and navigation

The navigation host calls the ViewModel from the destination's lifecycle:

| Callback | Called | Use it for |
|---|---|---|
| `onCreate(args: Bundle?)` | on `ON_CREATE` — and again each time the destination re-enters composition (back from another screen, rotation) | the first load, guarded so it stays idempotent |
| `onResume(result: Bundle?)` | on every `ON_RESUME` | results from a screen that just closed; refresh on return |
| `onStop()` | on every `ON_STOP` | pausing work the user can no longer see |
| `onCleared()` | when the destination leaves the back stack | releasing resources |

`args` is the extra `Bundle` given to `INavigator.navigate(uri, args)`; `result` is the `Bundle` given
to `popBackStack(result)`. Each is delivered once. Route parameters are read from `SavedStateHandle`.

Navigation is requested with `emit(action)`. `nav` is a `StateFlow<Event<INavAction>?>`; the host
takes each `Event` exactly once (`handle()`), so a rotation does not navigate twice. Because it is a
`StateFlow`, two emits before the UI collects keep only the last, and an emit while the screen is not
on screen is performed when the user comes back to it.

## Registering the ViewModel

```kotlin
package com.example.notes.list.di

import com.example.notes.list.viewmodel.NotesIntention
import com.example.notes.list.viewmodel.NotesModel
import com.example.notes.list.viewmodel.NotesViewModel
import com.example.notes.list.viewmodel.withLogging         // generated by the KSP processor
import dev.dexkot.mobile.core.ui.viewmodel.android.AndroidViewModel
import org.koin.core.module.dsl.viewModel
import org.koin.core.qualifier.named
import org.koin.dsl.module

val notesModule = module {
    viewModel<AndroidViewModel<NotesModel, NotesIntention>>(notesViewModel()) { params ->
        NotesViewModel(
            getNotes = get(),
            deleteNote = get(),
            savedStateHandle = get(),
        ).withLogging(logger = get(), sourceComponent = params.get())
    }
}

internal fun notesViewModel() = named("notes_viewmodel")
```

`@AutoLogViewModel` generates `NotesViewModel_LogDecorator` and the `withLogging(logger: ILogger,
sourceComponent: String)` extension. The decorator logs each `@AutoLogIntention` before forwarding it,
adds a logging context to the ViewModel's coroutines and, since 0.27.0, clears the wrapped ViewModel
when it is cleared. Because the decorator's type is `AndroidViewModel<Model, Intention>` and generics
are erased, each screen registers under a named qualifier. The `sourceComponent` is the Route's
identifier, passed by the Route.

## ScreenState and fieldMutableStateOf

```kotlin
package com.example.notes.list.screen.content

import android.os.Parcelable
import androidx.compose.runtime.Stable
import androidx.compose.runtime.derivedStateOf
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.setValue
import com.example.notes.domain.Note
import com.example.notes.domain.NoteStatus
import com.example.notes.list.viewmodel.NotesModel
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
    private var _query: String = "",
) : Parcelable {

    @IgnoredOnParcel
    var query: String by fieldMutableStateOf(_query) { _query = it }       // UI-only, survives process death

    @IgnoredOnParcel
    var loading: Boolean by mutableStateOf(true)                          // copied from the Model

    @IgnoredOnParcel
    var notes: List<Note> by mutableStateOf(emptyList())

    @IgnoredOnParcel
    var status: NoteStatus by mutableStateOf(NoteStatus.Active)

    @IgnoredOnParcel
    var showError: Boolean by mutableStateOf(false)

    @IgnoredOnParcel
    val visibleNotes: List<Note> by derivedStateOf {
        if (query.isBlank()) notes else notes.filter { it.title.contains(query, ignoreCase = true) }
    }

    @IgnoredOnParcel
    val showEmpty: Boolean by derivedStateOf { !loading && !showError && visibleNotes.isEmpty() }

    fun update(model: NotesModel) {
        loading = model.notes.inProgress()
        notes = model.notes.data.orEmpty()
        status = model.status
        showError = model.notes is Resource.Error
    }
}
```

`fieldMutableStateOf(initialValue, setValue)` returns a `MutableState` that Compose observes; each
write first calls `setValue`, which copies the value into the constructor's `private var` backing
field. Parcelize saves only constructor properties, so the backing field is what survives process
death, and the reactive property is rebuilt from it on restore.

| Property kind | Declare it as |
|---|---|
| UI-only, must survive process death (typed text, open dialog, selected tab) | backing field + `fieldMutableStateOf` |
| copied from the Model | `mutableStateOf` (the Model re-supplies it) |
| computed | `derivedStateOf` |

A dialog keeps its own UiState:

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

    fun open(note: Note) { noteId = note.id; title = note.title }

    fun close() { noteId = null; title = "" }
}
```

And the ScreenState aggregates them:

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

`@Stable` tells Compose the object reports its own changes through snapshot state. `@UiState` is a
marker.

## Screen and Content composables

```kotlin
package com.example.notes.list.screen

import androidx.activity.compose.BackHandler
import androidx.compose.runtime.Composable
import com.example.notes.list.screen.content.NotesContent
import com.example.notes.list.screen.content.dialogs.DeleteNoteDialog
import com.example.notes.list.viewmodel.NotesIntention

@Composable
fun NotesScreen(
    screenState: NotesScreenState,
    onIntention: (NotesIntention) -> Unit,
) {
    BackHandler { onIntention(NotesIntention.Back) }

    NotesContent(
        state = screenState.contentUiState,
        onBack = { onIntention(NotesIntention.Back) },
        onQueryChange = { query -> screenState.contentUiState.query = query },
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

- The Screen takes `(screenState, onIntention)` and never a ViewModel.
- Dialogs, bottom sheets and `BackHandler` live at Screen level: only the Screen sees all the state,
  so only it can decide whether back closes an overlay or leaves the screen.
- Content composables receive their UiState subset and UI-named callbacks (`onDeleteClick`); the
  Screen maps those to intentions or local state changes.

`NotesContent` and `DeleteNoteDialog` are ordinary composables (a `LazyColumn`, an `AlertDialog`); the
full versions are in the `dexkot-screen` skill's `references/full-example.md`.

## Wiring it in the Route

```kotlin
object NotesRoute : Route<NotesModel, NotesIntention>(
    identifier = NotesRoute.IDENTIFIER,
    route = NotesRoute.ROUTE,
    arguments = listOf(navArgument(NotesRoute.PROJECT_ID_PARAM) { type = NavType.LongType }),
    viewModelFactory = {
        koinViewModel<AndroidViewModel<NotesModel, NotesIntention>>(qualifier = notesViewModel()) {
            parametersOf(NotesRoute.IDENTIFIER)
        }
    },
    screenContent = { viewModel ->
        val screenState = rememberSaveable { NotesScreenState() }
        viewModel.ModelUpdates { screenState.update(it) }
        NotesScreen(screenState = screenState, onIntention = viewModel::input)
    },
    transitions = ImitationOfActivities,
) {
    private const val IDENTIFIER = "notes"
    private const val PROJECT_ID_PARAM = "project_id"
    private const val ROUTE = "projects/{$PROJECT_ID_PARAM}/notes"

    fun INavigator.goNotes(projectId: Long) {
        navigate(uri = ROUTE.replace("{$PROJECT_ID_PARAM}", projectId.toString()))
    }

    fun SavedStateHandle.getProjectId(): Long = checkNotNull(get<Long>(PROJECT_ID_PARAM))
}
```

`rememberSaveable` keeps the ScreenState across rotation and process death, per back-stack entry.
`ModelUpdates` collects the Model and calls `update` in a `LaunchedEffect` once per distinct Model.

## Composition locals

Inside every destination, the navigation host provides:

| Local | Type | Inside a destination | Default elsewhere |
|---|---|---|---|
| `LocalLogger` | `ILoggerInteractor` | `loggerFactory(route.identifier)` | no-op `DummyLoggerInteractor` |
| `LocalPermissionController` | `IPermissionController` | your controller, wrapped to log requests | `DummyPermissionController()` — everything granted |
| `LocalRemoteConfig` | `IRemoteConfigProvider` | your provider | `DummyRemoteConfigProvider` — defaults |
| `LocalUiPreference` | `IUiPreferences` | **not provided** | `DummyUiPreference` — writes ignored |

Provide `LocalUiPreference` yourself, around the app content:

```kotlin
setContent {
    val navigator = rememberNavigator(activity = this, deepLinkScheme = "example")
    val permissionController = rememberPermissionController(
        activity = this,
        preferences = koinInject<SharedPreferences>(qualifier = permissionPreferences()),
    )
    Preferences(uiPreferences = koinInject()) {
        Navigation(
            graphs = listOf(NotesGraph),
            navigator = navigator,
            dispatcher = koinInject(),
            permissionController = permissionController,
            remoteConfigProvider = koinInject(),
            loggerFactory = { source -> koin.get<ILoggerInteractor> { parametersOf(source) } },
        )
    }
}
```

Each local has a provider composable with a matching name: `Logger(...)`, `PermissionController(...)`,
`RemoteConfig(...)`, `Preferences(...)`. Previews need none of them.

## UI preferences, remote config and permissions

```kotlin
private const val GRID_LAYOUT_KEY = "notes_grid_layout"

@Composable
fun IUiPreferences.isGridLayout(): State<Boolean> = getBooleanState(GRID_LAYOUT_KEY, defaultValue = false)

fun IUiPreferences.setGridLayout(enabled: Boolean) = setBoolean(GRID_LAYOUT_KEY, enabled)

@Composable
fun EditorToolbar(onCameraReady: () -> Unit, onOpenSettings: () -> Unit) {
    val preferences = LocalUiPreference.current
    val isGrid by preferences.isGridLayout()

    val attachmentsEnabled by LocalRemoteConfig.current
        .getRemoteBooleanState(key = "notes_attachments_enabled", defaultValue = false)
        // import dev.dexkot.mobile.core.ui.remoteconfig.getBooleanState as getRemoteBooleanState

    val permissionController = LocalPermissionController.current
    val cameraState by rememberPermissionState(Permission.Camera)

    Switch(checked = isGrid, onCheckedChange = { preferences.setGridLayout(it) })

    if (attachmentsEnabled) {
        Button(onClick = {
            when {
                cameraState.isGranted -> onCameraReady()
                cameraState.canRequest -> permissionController.requestPermission(Permission.Camera) { result ->
                    if (result == PermissionResult.Granted) onCameraReady()
                }
                else -> onOpenSettings()
            }
        }) { Text("Attach photo") }
    }
}
```

- `IUiPreferences` (implementation `UiPreferences`, registered by `uiModule()`) stores small UI-only
  settings in `SharedPreferences`. `getBooleanState`, `getStringState`, `getStringSetState`,
  `getFloatState` and `getState(key, default, converter)` turn a key into Compose `State`.
- Remote config helpers `getBooleanState`, `getLongState`, `getStringState` live in
  `dev.dexkot.mobile.core.ui.remoteconfig` — same names as the preference helpers, different receiver.
- `rememberPermissionState(permission)` / `rememberPermissionsState(vararg)` re-read the state on every
  resume, so the UI updates after the system dialog or a trip to Settings.

## Lifecycle and transitions in Compose

- `ScreenLifeCycle(onCreate, onStart, onResume, onPause, onStop, onDestroy)` observes the current
  lifecycle owner. `onDestroy` runs when the composable is disposed.
- A Route's `transitions` accepts `DefaultTransitions` (fade, the default), `ImitationOfActivities`
  (horizontal slide), `ModalTransition` (slide up/down) or your own `Transitions(enter, exit,
  popEnter, popExit)`.
- `OnEnterTransitionFinished { }` runs once the screen has finished animating in — the place to start
  heavy work or request keyboard focus. `OnExitTransitionFinished { }` and
  `rememberIsScreenTransitioning()` complete the set.

## Testing and previews

ViewModel tests use the undecorated class, `Dispatchers.setMain`, and a subscriber on `model`
(`WhileSubscribed` does not update `model.value` without one):

```kotlin
@Test
fun `onCreate loads notes`() = runTest(dispatcher) {
    val vm = NotesViewModel(FakeGetNotes(), FakeDeleteNote(), SavedStateHandle(mapOf("project_id" to 7L)))
    backgroundScope.launch { vm.model.collect { } }

    vm.onCreate(null)

    assertIs<Resource.Loaded<List<Note>>>(vm.model.value.notes)
}
```

Previews build a ScreenState and feed it a Model:

```kotlin
@Preview
@Composable
private fun NotesScreenPreview() {
    val state = remember {
        NotesScreenState().apply {
            update(NotesModel(Resource.Loaded(listOf(Note(1, "Groceries"))), Resource.None(), NoteStatus.Active))
        }
    }
    NotesScreen(screenState = state, onIntention = { })
}
```

To preview a denied permission, wrap the preview in `PermissionController(DummyPermissionController(
mapOf(Permission.Camera to deniedState))) { … }`.

## Pitfalls

**ViewModel**

- `onCreate` runs again whenever the destination re-enters composition; guard the first load.
- Two `emit()`s before collection keep only the last; an `emit()` while the screen is hidden navigates
  when the user returns.
- `stateFlow()` uses `WhileSubscribed()` without a timeout: upstream stops as soon as the screen is
  covered or rotates. Pass `WhileSubscribed(5_000)` for expensive cold flows.
- `Resource` data is `@RawValue`: saving `Resource<T>` in `SavedStateHandle` crashes at runtime unless
  `T` is Parcelable, primitive or Serializable (`Unit` is not; use `Empty`).
- `inProgress()` is true for `None`: a `None` that means "done, nothing" renders as loading forever.
- `onLoaded` never returns if a flow errors.
- `@AutoLogViewModel` requires a direct `AndroidViewModel` subclass and a sealed Intention in the same
  package. A hand-written decorator must call `clearTogetherWith(delegate)` and forward `nav`.
- Coroutine context added with `addCoroutineContext` (e.g. by the logging decorator) applies only to
  coroutines launched afterwards; flows started in property initializers do not get it.

**Compose**

- Forgetting `Preferences(uiPreferences = …)` silently turns every preference write into a no-op.
- Above the navigation host, `LocalPermissionController` is the dummy and reports "granted".
- Preference flows see only writes through the same `IUiPreferences` instance; the `defaultValue` is
  fixed by the first request for a key; `remove(key)` detaches existing observers.
- Remote config helpers `remember` without keys: changing `key` or `defaultValue` later is ignored.
- `ModelUpdates` runs after the first composition: the first frame shows ScreenState defaults.
- `update(model)` receives the whole Model on every change; derive state, do not react to fields as if
  they were events.
- `rememberPermissionController` must be called unconditionally (it registers activity-result
  launchers).
- `@UiState` (annotation) and the logger's `UiState` enum share a simple name; alias one if needed.

## API reference

### `dev.dexkot.mobile.core.ui.viewmodel`

```kotlin
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
@Composable fun <Model, Intention> IViewModel<Model, Intention>.ModelUpdates(onUpdate: (Model) -> Unit)
annotation class ViewModel   // marker
```

### `dev.dexkot.mobile.core.ui.viewmodel.android`

```kotlin
abstract class AndroidViewModel<Model, Intention> : IScreenViewModel<Model, Intention>, androidx.lifecycle.ViewModel() {
    fun addCoroutineContext(context: CoroutineContext)
    protected val viewModelScope: CoroutineScope
    override val nav: StateFlow<Event<INavAction>?>
    protected fun emit(action: INavAction)
    protected fun clearTogetherWith(delegate: androidx.lifecycle.ViewModel)            // since 0.27.0
    protected fun <T> Flow<T>.stateFlow(
        coroutineScope: CoroutineScope = viewModelScope,
        started: SharingStarted = SharingStarted.WhileSubscribed(),
        initialValue: T
    ): StateFlow<T>
}
```

### `dev.dexkot.mobile.core.ui.viewmodel.resource`

```kotlin
sealed class Resource<T>(open val data: T? = null) : Parcelable {
    class None<T>(data: T? = null); class Loading<T>(data: T? = null)
    class Loaded<T>(data: T); class Error<T>(data: T? = null, val error: ErrorEntity)
    fun <R> map(block: (T) -> R): Resource<R>;  fun map(): Resource<Unit>
    fun update(block: (T) -> T): Resource<T>
    fun toNone(): None<T>;  fun toLoading(data: T? = this.data): Loading<T>
    fun toLoaded(data: T): Loaded<T>;  fun toError(error: ErrorEntity): Error<T>
}
fun <T> Flow<Resource<T>>.values(): Flow<T>
fun <T> Flow<Resource<T>>.finished(): Flow<Resource<T>>
fun Resource<*>.inProgress(): Boolean
suspend fun onLoaded(flow1, [flow2, [flow3,]] block)        // 1, 2 and 3 flows
suspend fun onFinished(flow1, flow2, flow3, block)          // 3 flows only
```

### Other viewmodel types

```kotlin
class Event<T>(data: T, hasBeenHandled: Boolean = false) : Parcelable { fun handle(): T? }   // .viewmodel.event
object Empty : Parcelable                                                                     // .viewmodel.empty
interface IResourceProvider {                                                                 // .resources
    fun getString(@StringRes id: Int): String
    fun getString(@StringRes resId: Int, vararg formatArgs: Any): String
}
class ResourceProvider(context: Context) : IResourceProvider
```

### Compose (`dev.dexkot.mobile.core.ui.compose.*`)

| Symbol | Package |
|---|---|
| `fieldMutableStateOf(initialValue: T, setValue: (T) -> Unit): MutableState<T>` | `compose.state` |
| `@UiState`, `ScreenLifeCycle(onCreate, onStart, onResume, onPause, onStop, onDestroy)` | `compose.screen` |
| `LocalLogger`, `Logger(loggerInteractor, content)`, `DummyLoggerInteractor` | `compose.logger` |
| `LocalPermissionController`, `PermissionController(permissionController, content)`, `rememberPermissionController(activity, preferences)`, `rememberPermissionState(permission)`, `rememberPermissionsState(vararg permissions)`, `IPermissionController.withLogging(logger)` | `compose.permission` |
| `LocalRemoteConfig`, `RemoteConfig(remoteConfigProvider, content)`, `DummyRemoteConfigProvider`, `DummyRemoteConfigHandler` | `compose.remoteconfig` |
| `LocalUiPreference`, `Preferences(uiPreferences, content)`, `DummyUiPreference`, `IUiPreferences.getBooleanState/getStringState/getStringSetState/getFloatState/getState` | `compose.preferences` |
| `Transitions`, `DefaultTransitions`, `ImitationOfActivities`, `ModalTransition`, `LocalNavAnimatedVisibilityScope`, `NavAnimatedVisibility`, `rememberIsScreenTransitioning()`, `OnEnterTransitionFinished { }`, `OnExitTransitionFinished { }` | `compose.transitions` |

### Preferences and remote config (non-Compose packages)

| Symbol | Package |
|---|---|
| `IUiPreferences`, `UiPreferences(sharedPreferences)`, `SharedPreferences.setString/setBoolean/setFloat/setStringSet/delete` | `dev.dexkot.mobile.core.ui.preferences` |
| `IRemoteConfigProvider.getBooleanState/getLongState/getStringState(key, defaultValue)` | `dev.dexkot.mobile.core.ui.remoteconfig` |
