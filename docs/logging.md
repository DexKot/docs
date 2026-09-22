# Logging, analytics and AutoLog

DexKot's logging modules give you one `ILogger` for developer messages, analytics events, crash reports
and user identity, and a KSP processor that logs use cases, screen intentions and navigation for you.

The principle behind the design: **business logic never knows it is being logged.** You annotate an
interface or class, KSP writes a decorator, and Koin hands out the decorated instance. The use case stays
a use case. Code that really does need to log by hand (say, a caught exception in a data source) gets the
logger from the coroutine context, so you never add a logger parameter to a constructor just for that.

Version: **0.27.0**.

- [Modules](#modules)
- [1. Compose your logger](#1-compose-your-logger)
- [2. Log from logic code](#2-log-from-logic-code)
- [3. Set up AutoLog (KSP)](#3-set-up-autolog-ksp)
- [4. Log use cases](#4-log-use-cases)
- [5. Log ViewModel intentions](#5-log-viewmodel-intentions)
- [6. Log navigation](#6-log-navigation)
- [7. Log UI interactions](#7-log-ui-interactions)
- [What reaches each backend](#what-reaches-each-backend)
- [Pitfalls](#pitfalls)
- [API reference](#api-reference)

## Modules

| Artifact (`dev.dexkot.mobile:`) | What it gives you | Platforms |
|---|---|---|
| `core-logger` | `ILogger`, `ConsoleLogger`, `PersistentUserIdLogger`, events, interactors, `logger { }` | Android, iOS |
| `core-logger-firebase` | `FirebaseLogger` (Analytics + Crashlytics) | Android, iOS* |
| `core-logger-posthog` | `PostHogLogger` | Android, iOS* |
| `core-logger-di-koin` | `loggerModule()` | Android, iOS |
| `core-annotation` | `@AutoLog*` annotations, `@LogParam` | Android, iOS |
| `core-ksp-processor` | the KSP processor | build time |

\* On iOS, the backends call the native SDKs through CocoaPods cinterop; your app links the Firebase /
PostHog pods (or SPM packages) itself. Android artifacts are published to the Maven repo; for iOS, build
the klibs from source with `publishToMavenLocal`.

```kotlin
// settings.gradle.kts
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
        maven("https://dexkot.github.io/maven")
    }
}

// build.gradle.kts (app / composition module)
dependencies {
    implementation("dev.dexkot.mobile:core-logger:0.27.0")
    implementation("dev.dexkot.mobile:core-logger-firebase:0.27.0")
    implementation("dev.dexkot.mobile:core-logger-posthog:0.27.0")
    implementation("dev.dexkot.mobile:core-logger-di-koin:0.27.0")
}
```

`core-logger-di-koin` depends on both backends. If you only want one SDK, skip it and register the pieces
yourself (shown in the API reference).

## 1. Compose your logger

Loggers compose with `+`: every call fans out to each backend in order. `withLevel` filters developer
messages for one backend. `PersistentUserIdLogger` remembers the user id across launches.

```kotlin
import android.content.Context
import com.russhwolf.settings.Settings
import com.russhwolf.settings.SharedPreferencesSettings
import dev.dexkot.mobile.core.logger.ConsoleLogger
import dev.dexkot.mobile.core.logger.FirebaseLogger
import dev.dexkot.mobile.core.logger.ILogger
import dev.dexkot.mobile.core.logger.PersistentUserIdLogger
import dev.dexkot.mobile.core.logger.PostHogLogger
import dev.dexkot.mobile.core.logger.composed.plus
import dev.dexkot.mobile.core.logger.di.koin.loggerModule
import dev.dexkot.mobile.core.logger.levels.Level
import dev.dexkot.mobile.core.logger.levels.withLevel
import org.koin.android.ext.koin.androidContext
import org.koin.core.qualifier.named
import org.koin.dsl.module

val appLoggingModule = module {
    includes(loggerModule(consoleLoggerTag = "NotesApp", logicSourceComponent = "app"))

    single<Settings>(named("logger_settings")) {
        SharedPreferencesSettings(androidContext().getSharedPreferences("notes.logger", Context.MODE_PRIVATE))
    }

    single<ILogger>(createdAtStart = true) {
        PersistentUserIdLogger(
            settings = get(named("logger_settings")),
            delegate = get<ConsoleLogger>().withLevel(Level.Debug) + get<FirebaseLogger>() + get<PostHogLogger>(),
        )
    }
}
```

Why the app binds `ILogger` itself: which backends you use, in which order, and with which filters is an
app decision, so `loggerModule()` registers the parts (`ConsoleLogger`, `FirebaseLogger`, `PostHogLogger`
by concrete type, plus the interactors) and leaves the composition to you.

Why `PersistentUserIdLogger` goes on the outside: on a cold start it re-applies the stored id to every
backend before the first event, so early events are attributed to the user.

On iOS, use `NSUserDefaultsSettings` instead of `SharedPreferencesSettings` (both come with
`multiplatform-settings`, which `core-logger` exposes).

**SDK setup is yours.** Initialise Firebase and PostHog before Koin creates the loggers. For PostHog, set
`captureScreenViews = false`: `PostHogLogger` already records screens from DexKot's screen events.

**Identity.** Wherever login and logout happen, inject `ILogger` and call `setUserId(id)` /
`removeUserId()`.

## 2. Log from logic code

```kotlin
import dev.dexkot.mobile.core.logger.interactors.ddl.logger

class NoteLocalDataSource(private val queries: NoteQueries) {
    fun read(id: Long): NoteEntity? = try {
        queries.selectById(id).executeAsOneOrNull()
    } catch (e: Exception) {
        logger { error(throwable = e, message = "Failed to read note", params = mapOf("note_id" to id)) }
        null
    }
}
```

`logger { }` finds the current *logic interactor*:
1. the `LoggerContextElement` of the running coroutine (Android), or
2. the process-wide default, which `loggerModule()` installs at startup.

If neither exists, the block is skipped. That makes it safe in unit tests.

Give long-lived scopes their own name so their logs are easy to filter:

```kotlin
single(named("syncScope")) {
    CoroutineScope(Dispatchers.Default + SupervisorJob() +
        LoggerContextElement(get<ILogger>().logicInteractor("note_sync")))
}
```

The generated ViewModel decorator does the same for each screen, and core-work can do it for each work
type.

To record a business operation that is not a use case:

```kotlin
logger { operation(operationId = "export_notes", result = Status.Success, params = mapOf("count" to "12"), isBusinessEvent = true) }
```

## 3. Set up AutoLog (KSP)

Apply KSP to every module that declares annotated types.

**Android module:**

```kotlin
plugins {
    id("com.google.devtools.ksp")   // the version matching your Kotlin version
}

dependencies {
    implementation("dev.dexkot.mobile:core-annotation:0.27.0")
    implementation("dev.dexkot.mobile:core-logger:0.27.0")
    implementation("dev.dexkot.mobile:core-ui:0.27.0")         // for ViewModels / nav actions
    ksp("dev.dexkot.mobile:core-ksp-processor:0.27.0")
}
```

**KMP module** (use cases in `commonMain`):

```kotlin
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

tasks.withType<org.jetbrains.kotlin.gradle.tasks.KotlinCompilationTask<*>>().configureEach {
    if (name != "kspCommonMainKotlinMetadata") dependsOn("kspCommonMainKotlinMetadata")
}
```

Keep `core-ui` and `core-ksp-processor` on the same version. The 0.27.0 ViewModel decorator calls
`clearTogetherWith`, which was added to `AndroidViewModel` in core-ui 0.27.0.

## 4. Log use cases

```kotlin
@AutoLogUseCase(
    identifier = "delete_note",
    params = [LogParam(logName = "note_id", paramName = "noteId")],
    captureEvent = true,
)
interface IDeleteNoteUseCase {
    suspend operator fun invoke(noteId: Long): ResultOf<Unit>
}

internal class DeleteNoteUseCase(private val repository: INoteRepository) : IDeleteNoteUseCase {
    override suspend fun invoke(noteId: Long): ResultOf<Unit> = repository.delete(noteId)
}

val notesDomainModule = module {
    factory<IDeleteNoteUseCase> { DeleteNoteUseCase(repository = get()).withLogging() }
}
```

On every call, the generated decorator:
1. logs an `operation` event `delete_note` with status `in_progress` and `note_id`;
2. runs the real use case;
3. logs `success` or `failure` (from the `ResultOf`), with `note_id` and `time` (for example `"42ms"`).

`captureEvent = true` marks the final event as a business event. PostHog records business events when
they succeed, so use it for things a product team would chart (a note deleted, a purchase made), not for
every read.

Rules the processor enforces or relies on:
- the annotation goes on the **interface**;
- `invoke` must be `suspend operator fun invoke(...): ResultOf<T>`;
- keep one `invoke` per file, with no parameter types containing commas (the signature is read from the
  source text);
- `withLogging()` is `internal`, so call it from the module that declares the interface.

## 5. Log ViewModel intentions

```kotlin
sealed interface NotesIntention {
    @AutoLogIntention("go_back")
    data object Back : NotesIntention

    @AutoLogIntention(identifier = "open_note", params = [LogParam(logName = "note_id", paramName = "noteId")])
    data class OpenNote(val noteId: Long) : NotesIntention
}

@AutoLogViewModel
class NotesViewModel(
    private val deleteNote: IDeleteNoteUseCase,
) : AndroidViewModel<NotesModel, NotesIntention>() {
    /* ... */
}

val notesUiModule = module {
    viewModel<AndroidViewModel<NotesModel, NotesIntention>>(named("notes_vm")) { params ->
        NotesViewModel(deleteNote = get()).withLogging(logger = get(), sourceComponent = params.get())
    }
}
```

The generated `NotesViewModel_LogDecorator`:
- logs each annotated intention, then forwards it to your ViewModel;
- puts a `LoggerContextElement` named after the screen into your ViewModel's `viewModelScope`, so the use
  case events above carry the screen as their source;
- clears your ViewModel when it is cleared (`clearTogetherWith`).

Requirements:
- `AndroidViewModel` must be the **direct** superclass.
- The Intention must be `sealed`, in the same package as the ViewModel, with its subclasses nested
  inside it.
- Every subclass needs `@AutoLogIntention`. The generated `when` has no `else` branch.
- Bind the ViewModel as `AndroidViewModel<Model, Intention>`, because the decorator is a different class.

## 6. Log navigation

```kotlin
@AutoLogNavAction(identifier = "open_note", params = [LogParam(logName = "note_id", paramName = "noteId")])
data class OpenNoteNavAction(val noteId: Long) : INavAction

val notesNavModule = module {
    factory { OpenNoteNavAction_LogDecorator() } binds arrayOf(INavActionLogDecorator::class)
}
```

Hand all decorators to core-ui's `Navigation(decorators = koin.getAll())`. When a screen dispatches an
`OpenNoteNavAction`, a `navigate` event is logged under that screen before the navigation runs.

## 7. Log UI interactions

`Navigation(loggerFactory = { source -> koin.get<IUiLoggerInteractor> { parametersOf(source) } })` gives
each screen a UI interactor, available as `LocalLogger.current`. Screens log `enter` and `exit`
automatically. Taps and visibility changes are yours to log:

```kotlin
val uiLogger = LocalLogger.current
IconButton(onClick = {
    uiLogger.interaction(elementId = "delete_note", interaction = Interaction.Tap)
    onDelete()
}) { Icon(Icons.Default.Delete, contentDescription = null) }
```

## What reaches each backend

| | Console | Firebase | PostHog |
|---|---|---|---|
| Developer messages (`log(level, …)`) | yes | no | diagnostic log |
| Exceptions | yes | Crashlytics non-fatal | `captureException` |
| Interaction events | yes | Analytics event `ui_interaction` | event `<screen>:<element>_<interaction>` |
| Screen enter / exit | yes | `ui_screen` | `screen(<screen>)` on enter |
| Operation events | yes | `operation` | diagnostic; event `<source>:<operation>` if business + success |
| Permission events | yes | `ui_permission` | results only |
| Navigation / intention / UI state | yes | `ui_navigation` / `ui_intention` / `ui_state` | diagnostic |
| Your own `LogEvent` classes | no | yes | no |

Firebase flattens parameters. Your params become `parameter_<key>`, extras `extra_<key>`, and
permissions `permission_<key>`. Firebase allows at most 40 characters in event and parameter names, 100 in
values, and 25 parameters per event, so keep ids short.

## Pitfalls

- **Forgetting `single<ILogger>`** crashes startup: `loggerModule()` creates the logic interactor eagerly.
- **iOS has one logic interactor.** `logger { }` always uses the default one; per-scope elements only work
  on Android.
- **Firebase drops developer messages.** Use console or PostHog to see them.
- **Missing `.withLogging()` / decorator binding** compiles fine and logs nothing.
- **Params are strings.** Every `LogParam` goes through `toString()`.
- **Don't log personal data.** Params end up in third-party analytics.
- **Tests:** `FirebaseLogger()` and `PostHogLogger()` touch native SDKs. Use a fake `ILogger`, and call
  `LoggerContextElement.setDefault(fake.logicInteractor("test"))` if the code under test uses `logger { }`.

## API reference

```kotlin
// dev.dexkot.mobile.core.logger
interface ILogger {
    fun setUserId(identifier: String)
    fun setUserProperty(key: String, value: String?)
    fun removeUserId()
    fun removeUserProperty(key: String)
    fun log(event: LogEvent)
    fun log(level: Level, tag: String, message: String, params: Map<String, Any> = mapOf())
    fun log(throwable: Throwable, message: String? = null, params: Map<String, Any> = mapOf())
}
class ConsoleLogger(tag: String) : ILogger
class PersistentUserIdLogger(settings: Settings, delegate: ILogger) : ILogger
class FirebaseLogger() : ILogger                          // core-logger-firebase
class PostHogLogger() : ILogger                           // core-logger-posthog
fun ILogger.uiInteractor(sourceComponent: String): interactors.ui.ILoggerInteractor
fun ILogger.logicInteractor(sourceComponent: String): interactors.ddl.ILoggerInteractor
fun ILogger.verbose|debug|info|warning|error(tag: String, message: String, params: Map<String, Any> = emptyMap())   // + v, d, i, w, e

// .composed / .levels
operator fun ILogger.plus(logger: ILogger): ILogger
fun ILogger.withLevel(level: Level): ILogger
enum class Level { Verbose, Debug, Info, Warning, Error }

// .interactors.ui
interface ILoggerInteractor {
    fun enter(); fun exit()
    fun navigate(navigationId: String, params: Map<String, String> = emptyMap())
    fun permission(permissions: List<String>)
    fun permission(permissions: Map<String, Status>)
    fun interaction(elementId: String, interaction: Interaction, extras: Map<String, String> = emptyMap())
    fun uiState(elementId: String, state: UiState, extras: Map<String, String> = emptyMap())
    fun intention(intentionId: String, params: Map<String, String> = emptyMap())
}

// .interactors.ddl
interface ILoggerInteractor {
    fun verbose|debug|info|warning(message: String, params: Map<String, Any> = emptyMap())
    fun error(message: String, params: Map<String, Any>)
    fun error(throwable: Throwable, message: String? = null, params: Map<String, Any> = emptyMap())
    fun operation(operationId: String, result: Status, params: Map<String, String> = emptyMap(), isBusinessEvent: Boolean = false)
}
fun logger(): ILoggerInteractor?
fun logger(block: ILoggerInteractor.() -> Unit)

// .context
class LoggerContextElement(loggerInteractor: interactors.ddl.ILoggerInteractor) : CoroutineContext.Element {
    companion object Key { fun current(): ILoggerInteractor?; fun setDefault(logger: ILoggerInteractor) }
}

// .events (LogEvent: type + parameters)
InteractionEvent, ScreenEvent, NavigationEvent, IntentionEvent, PermissionEvent, UiStateEvent, OperationEvent
enum Interaction { Tap, Swipe, ZoomIn, ZoomOut, DragStart, DragEnd, Search, Done }
enum UiState { Show, Hide, Expand, Collapse, FocusCapture, FocusRelease }
enum operation.Status { InProgress, Success, Failure }
enum permission.Status { Request, Granted, Denied, RequestIgnored }

// dev.dexkot.mobile.core.logger.di.koin
fun loggerModule(consoleLoggerTag: String = "Console", logicSourceComponent: String = "app"): Module

// dev.dexkot.mobile.core.annotation
annotation class AutoLogUseCase(val identifier: String, val params: Array<LogParam> = [], val captureEvent: Boolean = false)
annotation class AutoLogViewModel
annotation class AutoLogIntention(val identifier: String, val params: Array<LogParam> = [])
annotation class AutoLogNavAction(val identifier: String, val params: Array<LogParam> = [])
annotation class LogParam(val logName: String, val paramName: String)

// Generated
internal fun IFoo.withLogging(): IFoo                                           // per @AutoLogUseCase interface
fun <Model> AndroidViewModel<Model, FooIntention>.withLogging(logger: ILogger, sourceComponent: String): AndroidViewModel<Model, FooIntention>
class FooNavAction_LogDecorator : INavActionLogDecorator                         // per @AutoLogNavAction
```

Registering the pieces without `loggerModule()` (for example, with one backend only):

```kotlin
val appLoggingModule = module {
    single<ILogger>(createdAtStart = true) { ConsoleLogger("NotesApp") + FirebaseLogger() }
    single<ILogicLoggerInteractor>(createdAtStart = true) {
        get<ILogger>().logicInteractor("app").also { LoggerContextElement.setDefault(it) }
    }
    factory<IUiLoggerInteractor> { params -> get<ILogger>().uiInteractor(sourceComponent = params.get()) }
}
```

(`ILogicLoggerInteractor` and `IUiLoggerInteractor` are import aliases for the two `ILoggerInteractor`
interfaces.)
