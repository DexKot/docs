---
name: dexkot-viewmodel
description: "Build ViewModels with DexKot mobile-core (dev.dexkot.mobile:core-ui) using the Model/Intention pattern. Use whenever the user wants to create or change a ViewModel, Model, Intention, or ViewModelState; mentions AndroidViewModel, IScreenViewModel, IViewModel, input(intent), onCreate(args)/onResume(result), emit(INavAction), stateFlow, Resource (None/Loading/Loaded/Error, toLoading, toError, onLoaded), Event, Empty, IResourceProvider, @AutoLogViewModel, @AutoLogIntention or clearTogetherWith; or converts ResultOf into UI state. Also use to review or debug DexKot ViewModel code (lost navigation, stale state, process death)."
---

# DexKot ViewModels (Model / Intention)

`core-ui` gives every screen the same shape: the ViewModel exposes one immutable **Model** as a
`StateFlow`, receives user intent through a single `input(intent)` function, and requests navigation
by emitting an `INavAction`. The Compose side (ScreenState, Screen composable) is covered by the
`dexkot-screen` skill; this skill covers everything up to the `model` flow.

Why this shape: one entry point (`input`) and one output (`model`) make a screen testable as data in
and data out, make every user decision loggable by a generated decorator, and keep the ViewModel
ignorant of how the UI is drawn.

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
// build.gradle.kts (Android application or library module)
plugins {
    id("org.jetbrains.kotlin.plugin.parcelize")
    id("com.google.devtools.ksp")            // only for @AutoLogViewModel
}
dependencies {
    implementation("dev.dexkot.mobile:core-ui:0.27.0")         // brings core-foundation (ResultOf, ErrorEntity)
    implementation("dev.dexkot.mobile:core-annotation:0.27.0") // @AutoLogViewModel, @AutoLogIntention
    ksp("dev.dexkot.mobile:core-ksp-processor:0.27.0")
    // core-ui uses these as implementation dependencies; declare them yourself:
    implementation("androidx.lifecycle:lifecycle-viewmodel-compose:2.10.0")
    implementation("androidx.navigation:navigation-compose:2.9.5")
}
```

`core-ui` is Android-only (AAR). Versions above are the ones the library is built against.

## Core types (package `dev.dexkot.mobile.core.ui.viewmodel`)

| Type | Role |
|---|---|
| `IViewModel<Model, Intention>` | `val model: StateFlow<Model>`, `fun input(intent: Intention)`, and optional `onCreate(args: Bundle?)`, `onResume(result: Bundle?)`, `onStop()` |
| `IScreenViewModel<Model, Intention>` | adds `val nav: StateFlow<Event<INavAction>?>` — what a `Route` needs |
| `android.AndroidViewModel<Model, Intention>` | abstract base: extends `androidx.lifecycle.ViewModel`, implements `IScreenViewModel`; gives you `viewModelScope`, `emit()`, `stateFlow()`, `addCoroutineContext()`, `clearTogetherWith()` |
| `resource.Resource<T>` | async field state: `None`, `Loading`, `Loaded`, `Error` |
| `event.Event<T>` | one-shot wrapper; `handle()` returns the value once, then `null` |
| `empty.Empty` | `@Parcelize object` — a Parcelable "no payload" value |
| `@ViewModel` | marker annotation (runtime retention, nothing processes it) |
| `resources.IResourceProvider` | `getString(id)` / `getString(resId, vararg formatArgs)` without a `Context` |

Import `AndroidViewModel` from `dev.dexkot.mobile.core.ui.viewmodel.android` — androidx has a class
with the same simple name.

## How to build a screen ViewModel

The running example is a notes list inside a project. Files for one screen, in one package
(`com.example.notes.list.viewmodel`):

1. `NotesModel.kt` — the Model
2. `NotesIntention.kt` — the Intention
3. `NotesViewModelState.kt` — the SavedStateHandle wrapper
4. `NotesViewModel.kt` — the ViewModel
5. a Koin definition (see step 5)

### 1. Model: an immutable data class

```kotlin
data class NotesModel(
    val notes: Resource<List<Note>>,        // loaded asynchronously -> Resource
    val deletion: Resource<Empty>,          // an operation without payload -> Resource<Empty>
    val status: NoteStatus,                 // plain value: no loading lifecycle
)
```

Use `Resource<T>` for anything that is fetched or computed asynchronously (it has a lifecycle the UI
must render: spinner, error, stale data). Use plain values for route parameters, configuration and
derived flags. Keep it a `data class` with `val`s: the UI side reacts only when a *new, unequal*
Model is emitted (`StateFlow` drops equal values and `ModelUpdates` keys its effect on the Model),
so mutating a field in place is invisible.

### 2. Intention: what the user wants, not what they touched

```kotlin
sealed interface NotesIntention {

    @AutoLogIntention("go_back")
    data object Back : NotesIntention

    @AutoLogIntention(
        identifier = "go_note_detail",
        params = [LogParam(logName = "note_id", paramName = "noteId")]
    )
    data class GoNoteDetail(val noteId: Long) : NotesIntention

    @AutoLogIntention("refresh")
    data object Refresh : NotesIntention

    @AutoLogIntention(
        identifier = "filter_by_status",
        params = [LogParam(logName = "status", paramName = "status")]
    )
    data class FilterByStatus(val status: NoteStatus) : NotesIntention

    @AutoLogIntention(
        identifier = "delete_note",
        params = [LogParam(logName = "note_id", paramName = "noteId")]
    )
    data class Delete(val noteId: Long) : NotesIntention
}
```

Naming rules and why:

- `Go{Destination}` for navigation (`GoNoteDetail`, `GoSettings`). The prefix makes navigation
  branches obvious in the `when` and removes ambiguity (`GoSync` opens a screen, `Sync` syncs).
- `Back` for leaving the screen without a named destination.
- Plain verbs for domain actions (`Refresh`, `Delete`, `FilterByStatus`).
- Never UI mechanics (`BackClicked`, `PullToRefresh`, `SwipeToDelete`, `FabTapped`). A swipe and a
  menu item that both delete a note are the same intention; the ViewModel should not change when the
  gesture does, and analytics should count the decision, not the widget.
- Keep the sealed interface top-level, in the same package as the ViewModel (the generated logging
  decorator refers to it by simple name).

UI-only decisions (open a dialog, type in a text field, expand a card) are not intentions at all —
they live in the ScreenState (see `dexkot-screen`). An intention is sent when the user commits to
something the ViewModel must act on.

### 3. ViewModelState: the only place that touches SavedStateHandle

This is a convention, not a library class. Create one per ViewModel that needs a `SavedStateHandle`:

```kotlin
import androidx.lifecycle.SavedStateHandle
import com.example.notes.list.route.NotesRoute.getProjectId
import kotlinx.coroutines.flow.MutableStateFlow

class NotesViewModelState(savedStateHandle: SavedStateHandle) {

    /** Route parameter, read through the extension the Route owns. */
    val projectId: Long = savedStateHandle.getProjectId()

    /** Survives process death: written to the saved state on every change. */
    val status: MutableStateFlow<NoteStatus> =
        savedStateHandle.getMutableStateFlow(STATUS_KEY, NoteStatus.Active)

    private companion object {
        const val STATUS_KEY = "status"
    }
}
```

Why: the Route owns its parameter names (they are placeholders in its route string), so reading them
through Route-owned extensions keeps one source of truth. Saved-state keys stay private to one class.
The ViewModel reads typed values and never sees string keys, and tests build the state from
`SavedStateHandle(mapOf("project_id" to 7L))`. Use `getMutableStateFlow(key, initial)` (androidx
lifecycle 2.8+) for any value that must survive process death; values must be Bundle-compatible
(primitives, String, enums, Parcelable, Serializable).

### 4. The ViewModel

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
        // onCreate runs again each time the destination re-enters composition: stay idempotent.
        if (_notes.value is Resource.None) loadNotes()
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
        loadJob?.cancel()                                   // latest request wins
        loadJob = viewModelScope.launch {
            _notes.update { it.toLoading() }                // keeps current data while refreshing
            when (val result = getNotes(projectId = state.projectId, status = _status.value)) {
                is ResultOf.Success -> _notes.update { it.toLoaded(result.data) }
                is ResultOf.Failure -> _notes.update { it.toError(result.error) } // keeps stale data
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

`BackNavAction` and `NoteDetailNavAction` are app-defined `INavAction`s; see `dexkot-navigation`.

Rules this example follows:

- Convert `ResultOf` to `Resource` **in the ViewModel**, with an exhaustive `when`. Use cases and
  repositories return `ResultOf` (see `dexkot-results-errors`); `Resource` is presentation state and
  never leaks below this layer.
- Build `model` by combining private `MutableStateFlow`s and finishing with `stateFlow(initialValue =
  …)`. Compute the initial Model from the sources' current values so the first frame is right.
- Call `emit(action)` directly; it is not `suspend` and needs no coroutine.
- Launch work in the inherited `viewModelScope` (it also carries any context added with
  `addCoroutineContext`, such as the logging context).

### 5. Register it in Koin

```kotlin
import com.example.notes.list.viewmodel.withLogging   // generated by the KSP processor
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

Register the ViewModel under the `AndroidViewModel<Model, Intention>` type because that is what the
generated `withLogging` returns and what the Route's `viewModelFactory` needs. Generic arguments are
erased, so every screen registers the same class: the named qualifier is what keeps them apart. The
`sourceComponent` parameter is the Route identifier, passed by the Route, so ViewModel logs and screen
logs share a source. Details: `dexkot-di-koin`; Route wiring: `dexkot-screen` / `dexkot-navigation`.

## Lifecycle callbacks

`core-ui`'s navigation host calls these from the destination's lifecycle:

| Callback | When | Typical use |
|---|---|---|
| `onCreate(args)` | `ON_CREATE` of the destination, **again every time it re-enters composition** | first load; guard it (`if (x is Resource.None)`) |
| `onResume(result)` | every `ON_RESUME` (also after returning from another screen or from background) | read a result bundle from the screen that just closed; refresh on return |
| `onStop()` | every `ON_STOP` | pause things the user can no longer see (media, sensors) |
| `onCleared()` (androidx) | the destination is popped for good | release resources |

`args` are the extra `Bundle` passed to `INavigator.navigate(uri, args)`; `result` is the `Bundle`
passed to `popBackStack(result)` by the screen above. Both are consumed: they are delivered once and
are `null` on later calls. Route path/query parameters are not in `args` — read them from the
`SavedStateHandle` via the ViewModelState.

## Resource<T> in one table

| State | `data` | Meaning |
|---|---|---|
| `None(data?)` | optional | idle: nothing requested yet |
| `Loading(data?)` | optional | in flight; `data` is the stale value, if any |
| `Loaded(data)` | required | success |
| `Error(data?, error: ErrorEntity)` | optional | failed; `data` is the stale value, if any |

- `toLoading()` and `toError(error)` **keep** the current data — this is how a refresh keeps showing
  the old list under a spinner or an error banner. `toLoading(data = null)` clears it.
- `toNone()` and the no-arg `map()` (→ `Resource<Unit>`) **drop** data. `map { }` and `update { }`
  keep the state type and transform data when present.
- `inProgress()` is `true` for `Loading` **and** `None`. Use `None` only as the pre-request idle
  state; a `None` that means "finished with nothing" will render as loading forever.
- `Flow<Resource<T>>.values()` emits only `Loaded` data; `.finished()` drops in-progress states.
- `onLoaded(flow1[, flow2[, flow3]]) { … }` suspends until every flow is `Loaded`; `onFinished` exists
  only as a 3-flow overload.

Full reference, including the suspending helpers: `references/api.md`.

## Gotchas

- **Two emits, one navigation.** `nav` is a `StateFlow<Event<INavAction>?>`: it keeps only the latest
  value. Two `emit()` calls before the UI collects keep only the second. Emit one action per user
  decision.
- **Navigation waits for the screen.** The event is consumed by the destination's composition. An
  `emit()` that happens while the screen is not composed (the user already navigated away) fires when
  the user comes back. Check whether a late navigation still makes sense before emitting.
- **Events survive config changes exactly once.** `Event.handle()` returns the action once, so a
  rotation re-collecting `nav` does not navigate again. Do not read `nav.value?.handle()` yourself in
  production code: you would steal the event from the host.
- **`onCreate` is not "once per ViewModel".** Returning to a destination recomposes it and calls
  `onCreate(null)` again. Unguarded loads re-run on every back navigation.
- **`stateFlow()` defaults to `SharingStarted.WhileSubscribed()` with no timeout.** Upstream flows stop
  the moment the UI stops collecting (screen covered, rotation) and restart on return. Cheap for
  `combine` over `MutableStateFlow`s; for expensive cold flows (DB queries) pass
  `started = SharingStarted.WhileSubscribed(5_000)`. In unit tests, `model.value` stays at the initial
  value unless something collects `model`.
- **Saved Resources must be Parcelable at runtime.** `Resource` is `@Parcelize` with `@RawValue` data.
  Putting a `Resource<T>` in the `SavedStateHandle` compiles for any `T`, but saving crashes unless `T`
  is Parcelable/primitive/Serializable. `kotlin.Unit` is none of these: persist `Resource<Empty>`, not
  `Resource<Unit>` (and remember `map()` produces `Resource<Unit>`). Prefer persisting small inputs
  (ids, filters, drafts) and reloading data.
- **`onLoaded` can wait forever.** It returns only when every flow reaches `Loaded`; if one errors it
  keeps suspending. Launch it in `viewModelScope` (cancelled on clear) or wrap it with `withTimeout`.
- **`@AutoLogViewModel` needs a direct `AndroidViewModel` subclass** (an intermediate base class fails
  the processor) and a sealed Intention in the same package.
- **Custom decorators must call `clearTogetherWith(delegate)`.** The `ViewModelStore` holds only the
  outermost object; without it the wrapped ViewModel's `viewModelScope` is never cancelled and its
  `onCleared` never runs. Also forward `nav` to the delegate. See `references/api.md`.
- **Context added with `addCoroutineContext` applies to coroutines launched afterwards.** Flows started
  in property initializers (e.g. `.stateFlow(...)`) are launched during construction, before a
  decorator adds its context.

## Strings: IResourceProvider

Resolve display strings in Compose (`stringResource`) whenever possible. Inject `IResourceProvider`
only when the ViewModel itself must produce text that leaves the UI, e.g. a default name that gets
persisted or a share message. It keeps `Context` out of the ViewModel and is trivial to fake in tests.
`uiModule()` from `core-ui-di-koin` registers `ResourceProvider` as a `factory`.

```kotlin
class NoteEditorViewModel(
    private val resources: IResourceProvider,
    private val createNote: ICreateNoteUseCase,
) : AndroidViewModel<NoteEditorModel, NoteEditorIntention>() {
    private fun defaultTitle(): String = resources.getString(R.string.note_untitled)
    // ...
}
```

## Logging

`@AutoLogViewModel` on the ViewModel plus `@AutoLogIntention` on each intention generates a
`<ViewModel>_LogDecorator` and a `withLogging(logger: ILogger, sourceComponent: String)` extension in
the ViewModel's package. The decorator logs each annotated intention before forwarding it and adds a
`LoggerContextElement` to the delegate's `viewModelScope`. Setup and log parameters: `dexkot-logging`.

## References and related skills

- `references/api.md` — every public signature in the viewmodel package, `Resource` semantics per
  method, `Event`, `Empty`, custom decorators, unit-testing a ViewModel.
- `dexkot-screen` — ScreenState, `ModelUpdates`, the Screen composable, composition locals.
- `dexkot-navigation` — `Route`, `INavAction`, factories/executors, passing args and results.
- `dexkot-results-errors` — `ResultOf`, `ErrorEntity`.
- `dexkot-di-koin` — `uiModule()`, ViewModel registration, qualifiers.
- `dexkot-logging` — `@AutoLogViewModel`, `@AutoLogIntention`, `LogParam`, KSP.
