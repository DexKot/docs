# AutoLog reference: annotations, KSP setup, generated code (0.27.0)

Contents
1. Annotations
2. Gradle setup (Android module, KMP module)
3. @AutoLogUseCase: requirements, generated code, Koin
4. @AutoLogViewModel + @AutoLogIntention: requirements, generated code, Koin
5. @AutoLogNavAction: requirements, generated code, Koin
6. Validation matrix (what fails, what is silent)

## 1. Annotations

Package `dev.dexkot.mobile.core.annotation` (artifact `core-annotation`, KMP).

```kotlin
@Target(AnnotationTarget.CLASS) @Retention(AnnotationRetention.SOURCE)
annotation class AutoLogUseCase(val identifier: String, val params: Array<LogParam> = [], val captureEvent: Boolean = false)

@Target(AnnotationTarget.CLASS) @Retention(AnnotationRetention.RUNTIME)
annotation class AutoLogViewModel

@Target(AnnotationTarget.CLASS) @Retention(AnnotationRetention.SOURCE)
annotation class AutoLogIntention(val identifier: String, val params: Array<LogParam> = [])

@Target(AnnotationTarget.CLASS) @Retention(AnnotationRetention.SOURCE)
annotation class AutoLogNavAction(val identifier: String, val params: Array<LogParam> = [])

@Target(AnnotationTarget.ANNOTATION_CLASS) @Retention(AnnotationRetention.SOURCE)
annotation class LogParam(val logName: String, val paramName: String)
```

`LogParam.logName` is the key in the analytics event; `paramName` is the Kotlin name of the invoke
parameter (use cases) or property (intentions, nav actions) whose `toString()` is logged.

## 2. Gradle setup

The processor (`core-ksp-processor`, provider `dev.dexkot.mobile.core.ksp.processor.AutoLogProcessorProvider`)
runs in every module that declares annotated types. Generated files land in the same package as the
annotated symbol.

Generated code references other artifacts, so the module needs them on its classpath:
- use case decorators: `core-foundation` (`ResultOf`) and `core-logger`;
- ViewModel and nav action decorators: `core-ui` and `core-logger` (Android only).

### Android module

```kotlin
plugins {
    id("com.android.library")
    kotlin("android")
    id("com.google.devtools.ksp")     // version matching your Kotlin (library built with Kotlin 2.3.10)
}

dependencies {
    implementation("dev.dexkot.mobile:core-annotation:0.27.0")
    implementation("dev.dexkot.mobile:core-logger:0.27.0")
    implementation("dev.dexkot.mobile:core-ui:0.27.0")        // if the module has ViewModels / nav actions
    ksp("dev.dexkot.mobile:core-ksp-processor:0.27.0")
}
```

### KMP module (use cases in commonMain)

Process common code once, as metadata, and add the output to `commonMain`:

```kotlin
plugins {
    kotlin("multiplatform")
    id("com.google.devtools.ksp")
}

kotlin {
    sourceSets {
        commonMain {
            kotlin.srcDir("build/generated/ksp/metadata/commonMain/kotlin")
            dependencies {
                implementation("dev.dexkot.mobile:core-annotation:0.27.0")
                implementation("dev.dexkot.mobile:core-logger:0.27.0")
                implementation("dev.dexkot.mobile:core-foundation:0.27.0")
            }
        }
    }
}

dependencies {
    add("kspCommonMainMetadata", "dev.dexkot.mobile:core-ksp-processor:0.27.0")
}

// Every compilation must see the generated sources.
tasks.withType<org.jetbrains.kotlin.gradle.tasks.KotlinCompilationTask<*>>().configureEach {
    if (name != "kspCommonMainKotlinMetadata") dependsOn("kspCommonMainKotlinMetadata")
}
// Only if the module publishes sources jars:
tasks.withType<org.gradle.jvm.tasks.Jar>().configureEach { dependsOn("kspCommonMainKotlinMetadata") }
```

Why metadata-only: registering the processor per target as well would generate the same decorator
once per target plus once for common code, producing duplicate declarations. Keep AutoLog ViewModels
and nav actions in Android modules.

## 3. @AutoLogUseCase

Requirements:
- Put it on an **interface**. On a class, KSP emits a warning and generates nothing.
- The interface declares `suspend operator fun invoke(...): ResultOf<T>`. A missing `invoke` is a KSP
  error; a non-suspend or non-`ResultOf` invoke produces a decorator that does not compile.
- One `invoke` per source file, and no parameter types containing a comma. The processor reads the
  signature from the source text with a regex (it strips default values, which interface overrides
  cannot repeat anyway).
- `paramName` is pasted as an expression (`"<logName>" to <paramName>.toString()`). It is not validated;
  a typo becomes a compile error in the generated file. A property path like `note.id` works.

Given:

```kotlin
@AutoLogUseCase(identifier = "delete_note", params = [LogParam(logName = "note_id", paramName = "noteId")], captureEvent = true)
interface IDeleteNoteUseCase {
    suspend operator fun invoke(noteId: Long): ResultOf<Unit>
}
```

KSP generates `IDeleteNoteUseCase_AutoLogDecorator.kt` (shape):

```kotlin
internal class IDeleteNoteUseCase_AutoLogDecorator(private val useCase: IDeleteNoteUseCase) : IDeleteNoteUseCase {
    override suspend fun invoke(noteId: Long): ResultOf<Unit> {
        val startTime = kotlin.time.TimeSource.Monotonic.markNow()
        logger { operation(operationId = "delete_note", result = Status.InProgress, params = mapOf("note_id" to noteId.toString())) }
        val result = useCase(noteId)
        val elapsedMillis = startTime.elapsedNow().inWholeMilliseconds
        logger {
            operation(
                operationId = "delete_note",
                result = when (result) { is ResultOf.Success -> Status.Success; is ResultOf.Failure -> Status.Failure },
                params = mapOf("note_id" to noteId.toString(), "time" to "${elapsedMillis}ms"),
                isBusinessEvent = true,
            )
        }
        return result
    }
}

internal fun IDeleteNoteUseCase.withLogging(): IDeleteNoteUseCase = IDeleteNoteUseCase_AutoLogDecorator(this)
```

It logs through `logger { }`, so the source component is whatever `LoggerContextElement` is current:
the calling screen when invoked from a decorated ViewModel (Android), the work type's element inside
a worker, or the default otherwise.

Koin: call `withLogging()` on the instance, inside the definition, in the declaring module.

```kotlin
factory<IDeleteNoteUseCase> { DeleteNoteUseCase(repository = get()).withLogging() }
```

`factory<I> { Impl(get()) }.withLogging()` does not compile the way you want: `withLogging` would be
applied to the Koin definition, not to the use case.

Name clashes: when two interfaces are declared in the same package, both extensions are called
`withLogging`. Kotlin resolves them by receiver type; if you import them from different packages into
one DI file, alias one (`import com.example.user.withLogging as withUserLogging`).

## 4. @AutoLogViewModel + @AutoLogIntention

Requirements:
- The class's **direct** supertype is `dev.dexkot.mobile.core.ui.viewmodel.android.AndroidViewModel<Model, Intention>`.
  An intermediate base class does not qualify (KSP error).
- Intention logging needs a `sealed` Intention type. A non-sealed Intention, or `Nothing`/`Unit`, is not
  logged (with `Nothing`/`Unit` the decorator does not override `input` at all).
- Annotate every subclass of the sealed Intention with `@AutoLogIntention`. Subclasses without it are
  skipped with no warning, and the generated `when (intent)` has no `else` branch, so a partially
  annotated hierarchy yields a non-exhaustive `when`, which Kotlin rejects.
- `@AutoLogIntention` params are validated: an unknown `paramName` is a KSP error that lists the
  available properties.
- Declare the Intention in the **same package** as the ViewModel, and its subclasses **nested inside** it
  (`sealed interface NotesIntention { data object Back : NotesIntention }`). The generated file refers to
  the Intention by simple name without an import, and to each subclass as `NotesIntention.<Subclass>`.
- The module needs `core-ui` and `core-logger` at the same version as the processor. The 0.27.0 decorator
  calls `AndroidViewModel.clearTogetherWith`, which older core-ui versions do not have.

Generated `NotesViewModel_LogDecorator.kt` (shape):

```kotlin
internal class NotesViewModel_LogDecorator<Model>(
    private val delegate: AndroidViewModel<Model, NotesIntention>,
    logger: ILogger,
    sourceComponent: String,
) : AndroidViewModel<Model, NotesIntention>() {

    private val uiLogger = logger.uiInteractor(sourceComponent)

    init {
        clearTogetherWith(delegate)
        delegate.addCoroutineContext(LoggerContextElement(logger.logicInteractor(sourceComponent)))
    }

    override val model: StateFlow<Model> get() = delegate.model
    override val nav: StateFlow<Event<INavAction>?> get() = delegate.nav
    override fun onCreate(args: Bundle?) = delegate.onCreate(args)
    override fun onResume(result: Bundle?) = delegate.onResume(result)
    override fun onStop() = delegate.onStop()

    override fun input(intent: NotesIntention) {
        when (intent) {
            is NotesIntention.Back -> uiLogger.intention("go_back")
            is NotesIntention.OpenNote -> uiLogger.intention(intentionId = "open_note", params = mapOf("note_id" to intent.noteId.toString()))
        }
        delegate.input(intent)
    }
}

fun <Model> AndroidViewModel<Model, NotesIntention>.withLogging(logger: ILogger, sourceComponent: String): AndroidViewModel<Model, NotesIntention>
```

Why the two `init` calls:
- `clearTogetherWith(delegate)`: the `ViewModelStore` only knows the decorator. Without this, clearing
  the decorator would leave the wrapped ViewModel's `viewModelScope` running and `onCleared` uncalled.
- `addCoroutineContext(...)`: puts a `LoggerContextElement` named after the screen into the wrapped
  ViewModel's `viewModelScope`, so use cases launched from it log under that screen. Coroutines launched
  from the wrapped ViewModel's constructor/`init` start before this and do not get the element; launch
  work from `onCreate` instead.

Koin (the decorator is not a `NotesViewModel`, so bind the abstract type):

```kotlin
import org.koin.core.module.dsl.viewModel
import org.koin.core.qualifier.named

viewModel<AndroidViewModel<NotesModel, NotesIntention>>(named("notes_vm")) { params ->
    NotesViewModel(deleteNote = get()).withLogging(logger = get(), sourceComponent = params.get())
}

// in the route
koinViewModel<AndroidViewModel<NotesModel, NotesIntention>>(named("notes_vm")) { parametersOf(NotesRoute.IDENTIFIER) }
```

Pass the route identifier as `sourceComponent` so ViewModel events and the screen's `LocalLogger` events
share a source. See **dexkot-viewmodel** and **dexkot-screen**.

## 5. @AutoLogNavAction

Requirements:
- `identifier` is non-blank (KSP error otherwise).
- The class lists `dev.dexkot.mobile.core.ui.navigation.action.INavAction` directly among its supertypes.
- Params are validated against the class's properties (KSP error listing available ones).
- Lookup at runtime is by exact `action::class`; subclasses of an annotated action are not logged.
- Make the action a top-level class. The decorator refers to it by simple name, so a nested class
  (`Outer.OpenNote`) does not resolve.

Generated `OpenNoteNavAction_LogDecorator.kt` (public class, no `withLogging`):

```kotlin
class OpenNoteNavAction_LogDecorator : INavActionLogDecorator {
    override val actionType: KClass<out INavAction> = OpenNoteNavAction::class
    override fun wrap(executor: INavActionExecutor, action: INavAction, logger: ILoggerInteractor): INavActionExecutor {
        val typedAction = action as OpenNoteNavAction
        return object : INavActionExecutor {
            override fun invoke(navigator: INavigator) {
                logger.navigate(navigationId = "open_note", params = mapOf("note_id" to typedAction.noteId.toString()))
                executor(navigator)
            }
        }
    }
}
```

Koin: multi-bind every decorator and hand the list to `Navigation`.

```kotlin
factory { OpenNoteNavAction_LogDecorator() } binds arrayOf(INavActionLogDecorator::class)

// root composable
Navigation(
    graphs = graphs,
    navigator = navigator,
    dispatcher = koinInject(),
    permissionController = permissionController,
    loggerFactory = { source -> koin.get<IUiLoggerInteractor> { parametersOf(source) } },
    decorators = koin.getAll<INavActionLogDecorator>(),
)
```

Each screen wraps the dispatcher with `dispatcher.withLogging(LocalLogger.current, decorators)`, so the
`navigate` event carries the originating screen as its source.

## 6. Validation matrix

| Mistake | Result |
|---|---|
| `@AutoLogUseCase` on a class | KSP warning, nothing generated |
| Use case interface without `invoke` | KSP error |
| Non-suspend / non-`ResultOf` `invoke` | generated code fails to compile |
| Typo in use case `paramName` | generated code fails to compile |
| `@AutoLogViewModel` without direct `AndroidViewModel` supertype | KSP error |
| Non-sealed Intention | intentions silently not logged |
| Intention subclass without `@AutoLogIntention` | skipped; non-exhaustive `when` in the decorator |
| Unknown intention / nav action `paramName` | KSP error listing available properties |
| Intention in another package, or subclasses not nested | generated code fails to compile |
| Nested (non top-level) nav action class | generated code fails to compile |
| Blank nav action `identifier` | KSP error |
| Forgot `.withLogging()` / decorator binding | compiles, logs nothing |
| core-ui older than core-ksp-processor 0.27.0 | `clearTogetherWith` unresolved in generated code |
