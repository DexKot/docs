# Full example: a notes list screen

Every Compose-side file of one screen, in the layout DexKot projects use. The ViewModel, Model,
Intention and Koin definition are in the `dexkot-viewmodel` skill; this file assumes:

```kotlin
// com.example.notes.domain
data class Note(val id: Long, val title: String)
enum class NoteStatus { Active, Archived }

// com.example.notes.list.viewmodel
data class NotesModel(
    val notes: Resource<List<Note>>,
    val deletion: Resource<Empty>,
    val status: NoteStatus,
)
sealed interface NotesIntention {
    data object Back : NotesIntention
    data class GoNoteDetail(val noteId: Long) : NotesIntention
    data object Refresh : NotesIntention
    data class FilterByStatus(val status: NoteStatus) : NotesIntention
    data class Delete(val noteId: Long) : NotesIntention
}

// com.example.notes.list.di (NotesModule.kt)
internal fun notesViewModel() = named("notes_viewmodel")   // Koin qualifier
```

## Contents

1. [Package layout](#package-layout)
2. [Route](#route)
3. [ScreenState](#screenstate)
4. [Content UiState](#content-uistate)
5. [Dialog UiState](#dialog-uistate)
6. [Screen](#screen)
7. [Content](#content)
8. [Dialog](#dialog)
9. [Previews](#previews)
10. [App host: providing the locals](#app-host-providing-the-locals)
11. [Permissions, UI preferences and remote config in a screen](#permissions-ui-preferences-and-remote-config-in-a-screen)

## Package layout

```
com/example/notes/list/
├── di/NotesModule.kt                  (dexkot-viewmodel / dexkot-di-koin)
├── route/NotesRoute.kt
├── viewmodel/NotesModel.kt, NotesIntention.kt, NotesViewModel.kt, NotesViewModelState.kt
└── screen/
    ├── NotesScreen.kt
    ├── NotesScreenState.kt
    └── content/
        ├── NotesContent.kt
        ├── NotesContentUiState.kt
        └── dialogs/DeleteNoteDialog.kt, DeleteNoteDialogUiState.kt
```

## Route

```kotlin
package com.example.notes.list.route

import androidx.compose.runtime.saveable.rememberSaveable
import androidx.lifecycle.SavedStateHandle
import androidx.navigation.NavType
import androidx.navigation.navArgument
import com.example.notes.list.di.notesViewModel
import com.example.notes.list.screen.NotesScreen
import com.example.notes.list.screen.NotesScreenState
import com.example.notes.list.viewmodel.NotesIntention
import com.example.notes.list.viewmodel.NotesModel
import dev.dexkot.mobile.core.ui.compose.transitions.ImitationOfActivities
import dev.dexkot.mobile.core.ui.navigation.graph.Route
import dev.dexkot.mobile.core.ui.navigation.navigator.INavigator
import dev.dexkot.mobile.core.ui.viewmodel.ModelUpdates
import dev.dexkot.mobile.core.ui.viewmodel.android.AndroidViewModel
import org.koin.androidx.compose.koinViewModel
import org.koin.core.parameter.parametersOf

object NotesRoute : Route<NotesModel, NotesIntention>(
    identifier = NotesRoute.IDENTIFIER,
    route = NotesRoute.ROUTE,
    arguments = listOf(
        navArgument(NotesRoute.PROJECT_ID_PARAM) { type = NavType.LongType }
    ),
    viewModelFactory = {
        koinViewModel<AndroidViewModel<NotesModel, NotesIntention>>(qualifier = notesViewModel()) {
            parametersOf(NotesRoute.IDENTIFIER)          // sourceComponent for the logging decorator
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

    /** Read by NotesViewModelState; the key never leaves this object. */
    fun SavedStateHandle.getProjectId(): Long = checkNotNull(get<Long>(PROJECT_ID_PARAM))
}
```

`identifier` is also the logger's source component for this destination (the host calls
`loggerFactory(route.identifier)`), which is why it is passed to the ViewModel too.

## ScreenState

```kotlin
package com.example.notes.list.screen

import android.os.Parcelable
import androidx.compose.runtime.Stable
import com.example.notes.list.screen.content.NotesContentUiState
import com.example.notes.list.screen.content.dialogs.DeleteNoteDialogUiState
import com.example.notes.list.viewmodel.NotesModel
import dev.dexkot.mobile.core.ui.compose.screen.UiState
import kotlinx.parcelize.Parcelize

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

## Content UiState

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
    var query: String by fieldMutableStateOf(_query) { _query = it }

    @IgnoredOnParcel
    var loading: Boolean by mutableStateOf(true)

    @IgnoredOnParcel
    var notes: List<Note> by mutableStateOf(emptyList())

    @IgnoredOnParcel
    var status: NoteStatus by mutableStateOf(NoteStatus.Active)

    @IgnoredOnParcel
    var showError: Boolean by mutableStateOf(false)

    @IgnoredOnParcel
    var deleteFailed: Boolean by mutableStateOf(false)

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
        deleteFailed = model.deletion is Resource.Error     // state, not an event: stays until the next attempt
    }
}
```

## Dialog UiState

```kotlin
package com.example.notes.list.screen.content.dialogs

import android.os.Parcelable
import androidx.compose.runtime.Stable
import androidx.compose.runtime.derivedStateOf
import androidx.compose.runtime.getValue
import androidx.compose.runtime.setValue
import com.example.notes.domain.Note
import dev.dexkot.mobile.core.ui.compose.screen.UiState
import dev.dexkot.mobile.core.ui.compose.state.fieldMutableStateOf
import kotlinx.parcelize.IgnoredOnParcel
import kotlinx.parcelize.Parcelize

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

The dialog stores `noteId` and `title` rather than the `Note` itself: primitives parcel without
`@RawValue` and without requiring the domain model to be `Parcelable`.

## Screen

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

## Content

```kotlin
package com.example.notes.list.screen.content

import androidx.compose.foundation.clickable
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.Row
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import androidx.compose.material3.CircularProgressIndicator
import androidx.compose.material3.FilterChip
import androidx.compose.material3.OutlinedTextField
import androidx.compose.material3.Text
import androidx.compose.material3.TextButton
import androidx.compose.runtime.Composable
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import com.example.notes.domain.Note
import com.example.notes.domain.NoteStatus
import dev.dexkot.mobile.core.logger.events.ui.interaction.Interaction
import dev.dexkot.mobile.core.ui.compose.logger.LocalLogger

@Composable
fun NotesContent(
    state: NotesContentUiState,
    onBack: () -> Unit,
    onQueryChange: (String) -> Unit,
    onNoteClick: (Note) -> Unit,
    onDeleteClick: (Note) -> Unit,
    onRefresh: () -> Unit,
    onStatusSelected: (NoteStatus) -> Unit,
) {
    val logger = LocalLogger.current

    Column(modifier = Modifier.fillMaxSize().padding(16.dp)) {
        TextButton(onClick = onBack) { Text("Back") }

        OutlinedTextField(
            value = state.query,
            onValueChange = onQueryChange,
            label = { Text("Search") },
            modifier = Modifier.fillMaxWidth(),
        )

        Row {
            NoteStatus.entries.forEach { status ->
                FilterChip(
                    selected = state.status == status,
                    onClick = { onStatusSelected(status) },
                    label = { Text(status.name) },
                )
            }
        }

        when {
            state.loading && state.notes.isEmpty() -> CircularProgressIndicator()
            state.showError -> TextButton(onClick = onRefresh) { Text("Could not load notes. Retry") }
            state.showEmpty -> Text("No notes")
        }

        if (state.deleteFailed) Text("Could not delete the note")

        LazyColumn {
            items(state.visibleNotes, key = { it.id }) { note ->
                Row(
                    modifier = Modifier
                        .fillMaxWidth()
                        .clickable { onNoteClick(note) }
                        .padding(vertical = 8.dp),
                ) {
                    Text(note.title, modifier = Modifier.weight(1f))
                    TextButton(onClick = {
                        logger.interaction(elementId = "note_delete_button", interaction = Interaction.Tap)
                        onDeleteClick(note)
                    }) { Text("Delete") }
                }
            }
        }
    }
}
```

`LocalLogger` inside a destination is the per-screen `ILoggerInteractor`; use it for UI events that are
not intentions (taps that only open local UI, element shown/hidden). Intentions are logged by the
generated ViewModel decorator. See `dexkot-logging`.

## Dialog

```kotlin
package com.example.notes.list.screen.content.dialogs

import androidx.compose.material3.AlertDialog
import androidx.compose.material3.Text
import androidx.compose.material3.TextButton
import androidx.compose.runtime.Composable

@Composable
fun DeleteNoteDialog(
    state: DeleteNoteDialogUiState,
    onConfirm: (noteId: Long) -> Unit,
    onDismiss: () -> Unit,
) {
    val noteId = state.noteId ?: return

    AlertDialog(
        onDismissRequest = onDismiss,
        title = { Text("Delete note?") },
        text = { Text("\"${state.title}\" will be deleted.") },
        confirmButton = { TextButton(onClick = { onConfirm(noteId) }) { Text("Delete") } },
        dismissButton = { TextButton(onClick = onDismiss) { Text("Cancel") } },
    )
}
```

`AlertDialog` handles the back gesture itself through `onDismissRequest`, so the Screen's
`BackHandler` does not need to know about it. Overlays drawn inside the layout (a custom sheet) do
need `BackHandler(enabled = sheet.isOpen) { sheet.close() }` in the Screen.

## Previews

```kotlin
package com.example.notes.list.screen

import androidx.compose.runtime.Composable
import androidx.compose.runtime.remember
import androidx.compose.ui.tooling.preview.Preview
import com.example.notes.domain.Note
import com.example.notes.domain.NoteStatus
import com.example.notes.list.viewmodel.NotesModel
import dev.dexkot.mobile.core.foundation.errors.ErrorEntity
import dev.dexkot.mobile.core.ui.viewmodel.empty.Empty
import dev.dexkot.mobile.core.ui.viewmodel.resource.Resource

@Preview
@Composable
private fun NotesScreenLoadedPreview() {
    val state = remember {
        NotesScreenState().apply {
            update(
                NotesModel(
                    notes = Resource.Loaded(listOf(Note(1, "Groceries"), Note(2, "Ideas"))),
                    deletion = Resource.None(),
                    status = NoteStatus.Active,
                )
            )
        }
    }
    NotesScreen(screenState = state, onIntention = { })
}

@Preview
@Composable
private fun NotesScreenErrorWithStaleDataPreview() {
    val state = remember {
        NotesScreenState().apply {
            update(
                NotesModel(
                    notes = Resource.Error(data = listOf(Note(1, "Groceries")), error = ErrorEntity.NetworkError()),
                    deletion = Resource.None<Empty>(),
                    status = NoteStatus.Active,
                )
            )
            deleteDialogUiState.open(Note(1, "Groceries"))
        }
    }
    NotesScreen(screenState = state, onIntention = { })
}
```

No providers are needed: `LocalLogger`, `LocalPermissionController`, `LocalRemoteConfig` and
`LocalUiPreference` default to their dummies.

## App host: providing the locals

```kotlin
package com.example.app

import android.content.SharedPreferences
import android.os.Bundle
import androidx.activity.compose.setContent
import androidx.fragment.app.FragmentActivity
import com.example.notes.graph.NotesGraph
import dev.dexkot.mobile.core.logger.interactors.ui.ILoggerInteractor
import dev.dexkot.mobile.core.permission.di.koin.permissionPreferences
import dev.dexkot.mobile.core.ui.compose.Navigation
import dev.dexkot.mobile.core.ui.compose.navigation.rememberNavigator
import dev.dexkot.mobile.core.ui.compose.permission.rememberPermissionController
import dev.dexkot.mobile.core.ui.compose.preferences.Preferences
import org.koin.compose.getKoin
import org.koin.compose.koinInject
import org.koin.core.parameter.parametersOf

class MainActivity : FragmentActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            val koin = getKoin()
            val navigator = rememberNavigator(activity = this, deepLinkScheme = "example")
            // Unconditional, once: registers activity-result launchers.
            val permissionController = rememberPermissionController(
                activity = this,
                preferences = koinInject<SharedPreferences>(qualifier = permissionPreferences()),
            )

            // The navigation host does not provide LocalUiPreference: do it here.
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
    }
}
```

`Navigation` is an extension on `FragmentActivity`. Graphs, NavActions and the dispatcher are covered
by `dexkot-navigation`; the Koin modules that provide these bindings by `dexkot-di-koin`.

## Permissions, UI preferences and remote config in a screen

```kotlin
package com.example.notes.editor.screen.content

import androidx.compose.material3.Button
import androidx.compose.material3.Switch
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.runtime.State
import androidx.compose.runtime.getValue
import dev.dexkot.mobile.core.permission.Permission
import dev.dexkot.mobile.core.permission.PermissionResult
import dev.dexkot.mobile.core.ui.compose.permission.LocalPermissionController
import dev.dexkot.mobile.core.ui.compose.permission.rememberPermissionState
import dev.dexkot.mobile.core.ui.compose.preferences.LocalUiPreference
import dev.dexkot.mobile.core.ui.compose.preferences.getBooleanState
import dev.dexkot.mobile.core.ui.compose.remoteconfig.LocalRemoteConfig
import dev.dexkot.mobile.core.ui.preferences.IUiPreferences
import dev.dexkot.mobile.core.ui.remoteconfig.getBooleanState as getRemoteBooleanState

private const val GRID_LAYOUT_KEY = "notes_grid_layout"

@Composable
fun IUiPreferences.isGridLayout(): State<Boolean> = getBooleanState(GRID_LAYOUT_KEY, defaultValue = false)

fun IUiPreferences.setGridLayout(enabled: Boolean) = setBoolean(GRID_LAYOUT_KEY, enabled)

@Composable
fun EditorToolbar(
    onCameraReady: () -> Unit,
    onOpenSettings: () -> Unit,
) {
    val preferences = LocalUiPreference.current
    val isGrid by preferences.isGridLayout()

    val attachmentsEnabled by LocalRemoteConfig.current
        .getRemoteBooleanState(key = "notes_attachments_enabled", defaultValue = false)

    val permissionController = LocalPermissionController.current
    val cameraState by rememberPermissionState(Permission.Camera)   // refreshed on every ON_RESUME

    Switch(checked = isGrid, onCheckedChange = { preferences.setGridLayout(it) })

    if (attachmentsEnabled) {
        Button(onClick = {
            when {
                cameraState.isGranted -> onCameraReady()
                cameraState.canRequest -> permissionController.requestPermission(Permission.Camera) { result ->
                    if (result == PermissionResult.Granted) onCameraReady()
                }
                else -> onOpenSettings()       // permanently denied: only Settings can grant it
            }
        }) { Text("Attach photo") }
    }
}
```

Why the extension functions: the preference key is private to one file, reads and writes cannot drift
apart, and call sites read like domain code. The permission request goes through the destination's
`LocalPermissionController`, which the host wrapped to log the request and its result.
