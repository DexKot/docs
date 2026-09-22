# core-ui viewmodel API reference (0.27.0)

Artifact: `dev.dexkot.mobile:core-ui:0.27.0` (Android AAR).

## Contents

1. [IViewModel / IScreenViewModel](#iviewmodel--iscreenviewmodel)
2. [AndroidViewModel](#androidviewmodel)
3. [ModelUpdates](#modelupdates)
4. [Resource](#resource)
5. [Resource extensions](#resource-extensions)
6. [Event](#event)
7. [Empty](#empty)
8. [@ViewModel marker](#viewmodel-marker)
9. [IResourceProvider](#iresourceprovider)
10. [Writing a custom decorator](#writing-a-custom-decorator)
11. [Unit-testing a ViewModel](#unit-testing-a-viewmodel)

## IViewModel / IScreenViewModel

```kotlin
package dev.dexkot.mobile.core.ui.viewmodel

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
```

`Route<Model, Intention>.viewModelFactory` returns an `IScreenViewModel<Model, Intention>`; the
navigation host drives the lifecycle callbacks and consumes `nav`:

- `onCreate(args)`: on the destination's `ON_CREATE`. The host registers its lifecycle observer when
  the destination enters composition, and a new observer receives the catch-up events, so this runs
  again whenever the destination is recomposed from scratch (returning from another screen,
  configuration change). `args` is the `Bundle` given to `INavigator.navigate(uri, args)`, consumed on
  first delivery.
- `onResume(result)`: on every `ON_RESUME`. `result` is the `Bundle` given to
  `INavigator.popBackStack(result)` by the screen that closed (or an activity result under the key
  `INavigator.RESULT_KEY`), consumed on first delivery.
- `onStop()`: on every `ON_STOP`.

The host also calls the screen logger's `enter()` on resume and `exit()` on pause.

## AndroidViewModel

```kotlin
package dev.dexkot.mobile.core.ui.viewmodel.android

abstract class AndroidViewModel<Model, Intention> :
    IScreenViewModel<Model, Intention>, androidx.lifecycle.ViewModel() {

    fun addCoroutineContext(context: CoroutineContext)

    protected val viewModelScope: CoroutineScope

    override val nav: StateFlow<Event<INavAction>?>

    protected fun emit(action: INavAction)

    protected fun clearTogetherWith(delegate: androidx.lifecycle.ViewModel)

    protected fun <T> Flow<T>.stateFlow(
        coroutineScope: CoroutineScope = viewModelScope,
        started: SharingStarted = SharingStarted.WhileSubscribed(),
        initialValue: T
    ): StateFlow<T>
}
```

You still implement `model` and `input` yourself.

- **`viewModelScope`** — androidx's `viewModelScope` plus whatever was added with
  `addCoroutineContext`. Cancelled when the ViewModel is cleared.
- **`addCoroutineContext(context)`** — merges `context` into `viewModelScope` for coroutines launched
  from then on (the scope is rebuilt lazily). Public so a decorator can call it on the ViewModel it
  wraps; the logging decorator uses it to install a `LoggerContextElement`.
- **`emit(action)`** — sets `nav` to `Event(action)`. Not `suspend`; call it from anywhere on the main
  thread. `nav` is a conflating `StateFlow`: only the latest unhandled event is kept.
- **`stateFlow(...)`** — `stateIn` with `viewModelScope` and `WhileSubscribed()` (no stop timeout, no
  replay expiration) as defaults. `initialValue` is named and required.
- **`clearTogetherWith(delegate)`** — ties `delegate`'s lifecycle to this ViewModel: when this one is
  cleared, `delegate`'s closeables are closed, its `viewModelScope` cancelled and its `onCleared`
  called. `ViewModel.clear()` is internal to androidx, so the delegate is registered in a private
  `ViewModelStore` whose `clear()` is invoked from this ViewModel's `addCloseable`. Call it once per
  delegate, typically from the decorator's `init`. New in 0.27.0.

## ModelUpdates

```kotlin
package dev.dexkot.mobile.core.ui.viewmodel

@Composable
fun <Model, Intention> IViewModel<Model, Intention>.ModelUpdates(
    onUpdate: (Model) -> Unit
)
```

Collects `model` with `collectAsState()` and calls `onUpdate` inside `LaunchedEffect(model)`, i.e.
once per distinct Model, after composition. Used in a Route's `screenContent` to push the Model into
the ScreenState (see the `dexkot-screen` skill).

## Resource

```kotlin
package dev.dexkot.mobile.core.ui.viewmodel.resource

@Parcelize
sealed class Resource<T>(open val data: @RawValue T? = null) : Parcelable {

    abstract fun <R> map(block: (data: T) -> R): Resource<R>
    abstract fun map(): Resource<Unit>
    abstract fun update(block: (data: T) -> T): Resource<T>

    class None<T>(override val data: @RawValue T? = null) : Resource<T>
    class Loading<T>(override val data: @RawValue T? = null) : Resource<T>
    class Loaded<T>(override val data: @RawValue T) : Resource<T>
    class Error<T>(override val data: @RawValue T? = null, val error: ErrorEntity) : Resource<T>

    fun toNone(): None<T>
    fun toLoading(data: T? = this.data): Loading<T>
    fun toLoaded(data: T): Loaded<T>
    fun toError(error: ErrorEntity): Error<T>
}
```

Semantics per method:

| Call | Result | Data |
|---|---|---|
| `toNone()` | `None` | dropped |
| `toLoading()` | `Loading` | kept (pass `data = null` to clear) |
| `toLoaded(x)` | `Loaded` | replaced by `x` |
| `toError(e)` | `Error` | kept |
| `map { f(it) }` | same state | `f(data)` if data present, else `null` (`Loaded` always maps) |
| `map()` | same state, `Resource<Unit>` | dropped (`Loaded(Unit)` for `Loaded`) |
| `update { f(it) }` | same state | `f(data)` if data present |

`ErrorEntity` comes from `dev.dexkot.mobile.core.foundation.errors` (see `dexkot-results-errors`).

Note that the state classes are plain classes, not data classes: two `Loaded(list)` instances are not
`equals`, so every `update { }` produces a new Model even if the data is identical.

### Typical ResultOf → Resource conversion

```kotlin
viewModelScope.launch {
    _notes.update { it.toLoading() }
    when (val result = getNotes(projectId)) {
        is ResultOf.Success -> _notes.update { it.toLoaded(result.data) }
        is ResultOf.Failure -> _notes.update { it.toError(result.error) }
    }
}
```

## Resource extensions

```kotlin
fun <T> Flow<Resource<T>>.values(): Flow<T>                 // only Loaded data
fun <T> Flow<Resource<T>>.finished(): Flow<Resource<T>>     // drops None and Loading
fun Resource<*>.inProgress(): Boolean                       // Loading || None

suspend fun <T1> onLoaded(flow1: Flow<Resource<T1>>, block: suspend (T1) -> Unit)
suspend fun <T1, T2> onLoaded(
    flow1: Flow<Resource<T1>>, flow2: Flow<Resource<T2>>,
    block: suspend (T1, T2) -> Unit
)
suspend fun <T1, T2, T3> onLoaded(
    flow1: Flow<Resource<T1>>, flow2: Flow<Resource<T2>>, flow3: Flow<Resource<T3>>,
    block: suspend (T1, T2, T3) -> Unit
)
suspend fun <T1, T2, T3> onFinished(
    flow1: Flow<Resource<T1>>, flow2: Flow<Resource<T2>>, flow3: Flow<Resource<T3>>,
    block: suspend (T1?, T2?, T3?) -> Unit
)
```

- `onLoaded` waits for the first `Loaded` value of each flow, one after another, then runs `block`
  once. It never returns if a flow never reaches `Loaded` (for example it errors).
- `onFinished` waits for the first finished state (`Loaded` or `Error`) of each flow and passes their
  `data` (possibly `null`). Only the 3-flow overload exists.

```kotlin
override fun onCreate(args: Bundle?) {
    viewModelScope.launch {
        onLoaded(_project, _notes) { project, notes ->
            // both are available here
        }
    }
}
```

## Event

```kotlin
package dev.dexkot.mobile.core.ui.viewmodel.event

@Parcelize
class Event<T>(private var data: @RawValue T, private var hasBeenHandled: Boolean = false) : Parcelable {
    fun handle(): T?   // returns data the first time, null afterwards
}
```

`AndroidViewModel.nav` wraps each `INavAction` in an `Event` so the host consumes it once, even if
the same `nav` value is collected again after a configuration change. You can reuse `Event` for your
own one-shot signals in a Model, but prefer modelling them as state (for example a `Resource` that the
ScreenState reacts to) because state survives process death and events do not re-deliver.

## Empty

```kotlin
package dev.dexkot.mobile.core.ui.viewmodel.empty

@Parcelize
object Empty : Parcelable
```

A Parcelable "nothing". Use it as the payload of operations that return no data
(`Resource<Empty>`), especially when that state is saved in a `SavedStateHandle` (`Unit` is not
Parcelable or Serializable).

## @ViewModel marker

```kotlin
package dev.dexkot.mobile.core.ui.viewmodel

@Retention(AnnotationRetention.RUNTIME)
@Target(AnnotationTarget.CLASS)
annotation class ViewModel
```

A marker for tooling and code search. No processor reads it; it does not register or generate
anything. Not to be confused with `androidx.lifecycle.ViewModel` (a class).

## IResourceProvider

```kotlin
package dev.dexkot.mobile.core.ui.resources

interface IResourceProvider {
    fun getString(@StringRes id: Int): String
    fun getString(@StringRes resId: Int, vararg formatArgs: Any): String
}

class ResourceProvider(context: Context) : IResourceProvider
```

`core-ui-di-koin`'s `uiModule()` registers `factory<IResourceProvider> { ResourceProvider(get()) }`.
Pass the application `Context` if you build it by hand: the provider may outlive an Activity.

## Writing a custom decorator

The KSP-generated logging decorator is the reference shape. A hand-written decorator (for example
one that adds tracing) looks like this:

```kotlin
import android.os.Bundle
import dev.dexkot.mobile.core.ui.navigation.action.INavAction
import dev.dexkot.mobile.core.ui.viewmodel.android.AndroidViewModel
import dev.dexkot.mobile.core.ui.viewmodel.event.Event
import kotlinx.coroutines.flow.StateFlow

class TracingViewModel<Model, Intention>(
    private val delegate: AndroidViewModel<Model, Intention>,
    private val tracer: Tracer,
) : AndroidViewModel<Model, Intention>() {

    init {
        clearTogetherWith(delegate)   // otherwise the delegate is never cleared
    }

    override val model: StateFlow<Model> get() = delegate.model
    override val nav: StateFlow<Event<INavAction>?> get() = delegate.nav   // the delegate emits

    override fun input(intent: Intention) {
        tracer.mark("intent $intent")
        delegate.input(intent)
    }

    override fun onCreate(args: Bundle?) = delegate.onCreate(args)
    override fun onResume(result: Bundle?) = delegate.onResume(result)
    override fun onStop() = delegate.onStop()
}
```

Checklist: call `clearTogetherWith(delegate)` once; forward `model`, `nav`, `input` and the three
lifecycle callbacks. Decorators stack (`NotesViewModel(...).withLogging(...).let { TracingViewModel(it,
tracer) }`) because each one is itself an `AndroidViewModel`. Apply `withLogging` directly to the real
ViewModel (innermost): it calls `addCoroutineContext` on the object it wraps, and only the real
ViewModel launches coroutines.

Before 0.27.0 the generated decorator did not clear its delegate: the wrapped ViewModel's
`viewModelScope` leaked and `onCleared` never ran. Regenerate (rebuild) after upgrading.

## Unit-testing a ViewModel

Test the undecorated class. `viewModelScope` runs on `Dispatchers.Main.immediate`, so replace Main.

```kotlin
import androidx.lifecycle.SavedStateHandle
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.ExperimentalCoroutinesApi
import kotlinx.coroutines.launch
import kotlinx.coroutines.test.UnconfinedTestDispatcher
import kotlinx.coroutines.test.resetMain
import kotlinx.coroutines.test.runTest
import kotlinx.coroutines.test.setMain
import kotlin.test.AfterTest
import kotlin.test.BeforeTest
import kotlin.test.Test
import kotlin.test.assertIs

@OptIn(ExperimentalCoroutinesApi::class)
class NotesViewModelTest {

    private val dispatcher = UnconfinedTestDispatcher()

    @BeforeTest fun setUp() = Dispatchers.setMain(dispatcher)
    @AfterTest fun tearDown() = Dispatchers.resetMain()

    private fun viewModel() = NotesViewModel(
        getNotes = FakeGetNotesUseCase(notes = listOf(Note(id = 1, title = "A"))),
        deleteNote = FakeDeleteNoteUseCase(),
        savedStateHandle = SavedStateHandle(mapOf("project_id" to 7L)),
    )

    @Test
    fun `onCreate loads notes`() = runTest(dispatcher) {
        val vm = viewModel()
        backgroundScope.launch { vm.model.collect { } }   // WhileSubscribed needs a subscriber

        vm.onCreate(null)

        assertIs<Resource.Loaded<List<Note>>>(vm.model.value.notes)
    }

    @Test
    fun `Back emits a back navigation`() {
        val vm = viewModel()

        vm.input(NotesIntention.Back)

        assertIs<BackNavAction>(vm.nav.value?.handle())
    }
}
```

`SavedStateHandle(mapOf(...))` must use the same key the Route extension reads (`"project_id"` here),
which is another reason to keep that key as a constant owned by the Route.
