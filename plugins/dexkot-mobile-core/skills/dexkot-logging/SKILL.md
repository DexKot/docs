---
name: dexkot-logging
description: "Logging, analytics and crash reporting with DexKot mobile-core (dev.dexkot.mobile core-logger, core-logger-firebase, core-logger-posthog, core-annotation, core-ksp-processor). Use whenever the task touches ILogger, ConsoleLogger, FirebaseLogger, PostHogLogger, PersistentUserIdLogger, composing loggers with `+`, withLevel, LoggerContextElement, the `logger { }` DSL, ILoggerInteractor (ui or logic), OperationEvent, LocalLogger, or the AutoLog annotations @AutoLogUseCase, @AutoLogViewModel, @AutoLogIntention, @AutoLogNavAction, @LogParam and the KSP-generated `withLogging()` / `*_LogDecorator` classes. Also use when adding analytics to a use case, ViewModel or navigation action, or when setting up KSP for AutoLog in a DexKot project."
---

# DexKot logging and AutoLog

DexKot's logging stack has one guiding principle: **business logic never knows it is being logged.**
Use cases, ViewModels and navigation actions are annotated; a KSP processor generates decorators
that emit start/finish/intention/navigation events; Koin wires the decorated instance in place of
the plain one. Code that does need to log manually (a caught exception in a data source, a custom
operation) reaches the logger through the coroutine context with the `logger { }` DSL, so no
logger has to be threaded through constructors.

Everything funnels into a single `ILogger`, which is usually a composition of several backends
(console + Firebase + PostHog) wrapped in `PersistentUserIdLogger`.

Version covered: **0.27.0**. Detailed API: `references/logger-api.md`. AutoLog details and generated
code: `references/autolog.md`.

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

// build.gradle.kts of the module that composes the logger (usually the app / shared DI module)
dependencies {
    implementation("dev.dexkot.mobile:core-logger:0.27.0")
    implementation("dev.dexkot.mobile:core-logger-firebase:0.27.0")   // optional backend
    implementation("dev.dexkot.mobile:core-logger-posthog:0.27.0")    // optional backend
    implementation("dev.dexkot.mobile:core-logger-di-koin:0.27.0")    // loggerModule()
}
```

| Artifact | Platforms |
|---|---|
| `core-logger`, `core-logger-di-koin`, `core-annotation` | KMP (Android + iOS) |
| `core-logger-firebase`, `core-logger-posthog` | KMP; iOS uses CocoaPods cinterop, the app links the pods itself |
| `core-ksp-processor` | build-time JVM jar |
| ViewModel / NavAction decorators | Android only (they need `core-ui`) |

Android artifacts are published. For iOS, build the klibs from source (`publishToMavenLocal`).

`core-logger-di-koin` depends on both backends, so it pulls the Firebase and PostHog SDKs in. If you
do not want one of them, skip `core-logger-di-koin` and register the pieces yourself (see
`references/logger-api.md`, "Wiring without loggerModule").

## Core concepts

- **`ILogger`**: the sink. Three kinds of calls: analytics events (`log(event: LogEvent)`),
  developer messages (`log(level, tag, message, params)`), exceptions (`log(throwable, message, params)`),
  plus user identity (`setUserId`, `removeUserId`, `setUserProperty`, `removeUserProperty`).
- **Backends**: `ConsoleLogger(tag)` (Logcat / NSLog), `FirebaseLogger()` (Analytics + Crashlytics),
  `PostHogLogger()`. Combine with `a + b + c` (package `dev.dexkot.mobile.core.logger.composed`).
  Filter developer messages with `withLevel(Level.Info)` (package `...logger.levels`).
- **Interactors** give a vocabulary on top of `ILogger`, bound to a `sourceComponent` string:
  - *UI* (`interactors.ui.ILoggerInteractor`): `enter`, `exit`, `navigate`, `interaction`, `uiState`,
    `intention`, `permission`. Obtained with `logger.uiInteractor(sourceComponent)`.
  - *Logic* (`interactors.ddl.ILoggerInteractor`): `verbose/debug/info/warning/error` and
    `operation(operationId, result, params, isBusinessEvent)`. Obtained with `logger.logicInteractor(sourceComponent)`.
  Both interfaces are called `ILoggerInteractor`; import-alias them when a file needs both.
- **`LoggerContextElement`**: a `CoroutineContext.Element` carrying a logic interactor. The
  `logger { }` DSL (package `dev.dexkot.mobile.core.logger.interactors.ddl`) resolves the current one,
  falling back to a process-wide default set with `LoggerContextElement.setDefault(...)`.

## Step 1: compose the application logger

`loggerModule()` registers the three backends by concrete type, the default logic interactor and a UI
interactor factory. It deliberately does **not** bind `ILogger`, because the composition is an app
decision. Bind it yourself:

```kotlin
import android.content.Context
import com.russhwolf.settings.SharedPreferencesSettings
import com.russhwolf.settings.Settings
import org.koin.android.ext.koin.androidContext
import dev.dexkot.mobile.core.logger.ConsoleLogger
import dev.dexkot.mobile.core.logger.FirebaseLogger
import dev.dexkot.mobile.core.logger.ILogger
import dev.dexkot.mobile.core.logger.PersistentUserIdLogger
import dev.dexkot.mobile.core.logger.PostHogLogger
import dev.dexkot.mobile.core.logger.composed.plus
import dev.dexkot.mobile.core.logger.di.koin.loggerModule
import dev.dexkot.mobile.core.logger.levels.Level
import dev.dexkot.mobile.core.logger.levels.withLevel
import org.koin.core.qualifier.named
import org.koin.dsl.module

val appLoggingModule = module {
    includes(loggerModule(consoleLoggerTag = "NotesApp", logicSourceComponent = "app"))

    // Where the user id survives restarts. Android example; on iOS use NSUserDefaultsSettings.
    single<Settings>(named("logger_settings")) {
        SharedPreferencesSettings(androidContext().getSharedPreferences("notes.logger", Context.MODE_PRIVATE))
    }

    single<ILogger>(createdAtStart = true) {
        PersistentUserIdLogger(
            settings = get<Settings>(named("logger_settings")),
            delegate = get<ConsoleLogger>().withLevel(Level.Debug) + get<FirebaseLogger>() + get<PostHogLogger>(),
        )
    }
}
```

Why `PersistentUserIdLogger` wraps the whole composition: it stores the user id under `"user_id"` and
re-applies it to its delegate on construction, so every backend knows the user from the first event of
a cold start, before your login flow runs again.

Initialise the Firebase / PostHog SDKs yourself **before** Koin creates these loggers
(`FirebaseApp` via google-services, `PostHogAndroid.setup(...)` on Android; the native SDKs on iOS).
The library does not configure SDKs. With PostHog, set `captureScreenViews = false`, because
`PostHogLogger` already turns `ScreenEvent(Enter)` into `screen(...)` calls.

## Step 2: log from logic code with `logger { }`

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

Give a long-lived scope its own source component, so its logs are attributable:

```kotlin
single(named("syncScope")) {
    CoroutineScope(Dispatchers.Default + SupervisorJob() +
        LoggerContextElement(get<ILogger>().logicInteractor("note_sync")))
}
```

Identity: inject `ILogger` where login/logout happens and call `setUserId(id)` / `removeUserId()`.

## Step 3: annotate instead of logging by hand (AutoLog)

Add KSP to every module that declares annotated types:

```kotlin
// Android module
plugins { id("com.google.devtools.ksp") }
dependencies {
    implementation("dev.dexkot.mobile:core-annotation:0.27.0")
    ksp("dev.dexkot.mobile:core-ksp-processor:0.27.0")
}
```

KMP modules use `kspCommonMainMetadata` plus a generated source dir; see `references/autolog.md`.
Use a KSP version that matches your Kotlin version (the library is built with Kotlin 2.3.10).

**Use case** (commonMain is fine):

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

// same Gradle module: the generated withLogging() is internal
val notesDomainModule = module {
    factory<IDeleteNoteUseCase> { DeleteNoteUseCase(repository = get()).withLogging() }
}
```

Generated: `IDeleteNoteUseCase_AutoLogDecorator` plus `internal fun IDeleteNoteUseCase.withLogging()`.
It logs `operation("delete_note", InProgress, {note_id})`, runs the use case, then logs
`Success`/`Failure` (from the `ResultOf`) with the elapsed `"time"` (e.g. `"42ms"`).
`captureEvent = true` marks the final event as a business event, which PostHog captures on success.

**ViewModel + intentions** (Android):

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
) : AndroidViewModel<NotesModel, NotesIntention>() { /* ... */ }

val notesUiModule = module {
    viewModel<AndroidViewModel<NotesModel, NotesIntention>>(named("notes_vm")) { params ->
        NotesViewModel(deleteNote = get()).withLogging(logger = get(), sourceComponent = params.get())
    }
}
```

Bind the ViewModel as `AndroidViewModel<Model, Intention>`, not as `NotesViewModel`: the decorator is a
different class. The decorator logs each annotated intention as a UI `intention` event, then forwards it;
it also installs a `LoggerContextElement(sourceComponent)` in the wrapped ViewModel's `viewModelScope`, so
use cases called from that ViewModel log under the screen's name. In 0.27.0 its `init` also calls
`clearTogetherWith(delegate)`, so the wrapped ViewModel's scope is cancelled and `onCleared` runs when the
decorator is cleared.

**Navigation action** (Android):

```kotlin
@AutoLogNavAction(identifier = "open_note", params = [LogParam(logName = "note_id", paramName = "noteId")])
data class OpenNoteNavAction(val noteId: Long) : INavAction

val notesNavModule = module {
    factory { OpenNoteNavAction_LogDecorator() } binds arrayOf(INavActionLogDecorator::class)
}
```

There is no `withLogging()` for nav actions. Pass `getAll<INavActionLogDecorator>()` to core-ui's
`Navigation(decorators = ...)`; its dispatcher looks decorators up by `action::class` and logs a
`navigate` event before executing. See **dexkot-navigation**.

## Step 4: UI interactions

core-ui's `Navigation(loggerFactory = ...)` gives every route a UI interactor, exposed as
`LocalLogger` (package `dev.dexkot.mobile.core.ui.compose.logger`), and logs `enter`/`exit` on
resume/pause automatically. Log taps and visibility changes from composables:

```kotlin
val uiLogger = LocalLogger.current
Button(onClick = {
    uiLogger.interaction(elementId = "save_note", interaction = Interaction.Tap)
    onSave()
}) { Text("Save") }
```

Wire the factory once: `loggerFactory = { source -> koin.get<IUiLoggerInteractor> { parametersOf(source) } }`.
See **dexkot-screen** for the screen structure.

## Rules and rationale

- Annotate, don't log, for use cases, intentions and navigation. Hand-written logging drifts; the
  decorators keep event names and timing uniform and keep domain code free of analytics concerns.
- Use `logger { }` for incidental logging in logic code (caught exceptions, warnings). It is a no-op when
  no interactor is available, so it is safe in tests.
- Keep identifiers and `logName`s short snake_case. Parameters are prefixed (`parameter_<key>`,
  `extra_<key>`), and Firebase Analytics limits event/param names to 40 chars, values to 100 chars and
  25 params per event.
- Never log PII in params: they land in third-party analytics.
- In unit tests, use a fake `ILogger`; `FirebaseLogger()` / `PostHogLogger()` touch native SDKs in their
  constructors.

## Gotchas

1. **No `single<ILogger>` means startup crashes.** `loggerModule()` creates the logic interactor with
   `createdAtStart = true`, and it needs `ILogger`.
2. **`logger { }` silently does nothing** until `LoggerContextElement.setDefault(...)` has run
   (`loggerModule()` does it at start).
3. **iOS resolves only the default interactor** from `logger { }`: `LoggerContextElement` in a scope does
   not change the source component there. On Android it does (thread-local).
4. **`FirebaseLogger` ignores `log(level, ...)`**. Debug messages only reach console and PostHog diagnostics.
   `withLevel` filters only those messages; events and exceptions always pass.
5. **`@AutoLogUseCase` must be on an interface** whose `invoke` is `suspend operator fun invoke(...): ResultOf<T>`.
   On a class you only get a KSP warning and nothing is generated. Keep one `invoke` per file and avoid
   parameter types containing commas (`Map<String, Int>`): the processor reads the signature with a regex.
6. **`withLogging()` for use cases is `internal`**: call it in the module that declares the interface.
   When two interfaces in one file each have `withLogging`, import-alias one.
7. **`@AutoLogViewModel` needs `AndroidViewModel` as a direct supertype** and a `sealed` Intention in the
   same package, with its subclasses nested inside it. Annotate every intention subclass: the generated
   `when` has no `else` branch. Keep annotated nav actions top-level classes.
8. **core-ui and core-ksp-processor must be on the same version**: 0.27.0 decorators call
   `clearTogetherWith`, which older core-ui versions lack.
9. **Only exact nav action classes are logged**: lookup is by `action::class`, so subclasses are skipped.
10. **All logged params are strings**: `LogParam` values go through `toString()`.
11. **PostHog captures an `OperationEvent` only when `isBusinessEvent` and `Success`**; other events are
    diagnostics or screen calls. See `references/logger-api.md` for the full routing table.
12. **Custom `LogEvent` implementations only reach Firebase.** `ConsoleLogger` and `PostHogLogger` switch
    over the built-in event classes and drop anything else.

## Related skills

- **dexkot-di-koin**: module structure, `getAll`, qualifiers.
- **dexkot-viewmodel**: `AndroidViewModel`, Model/Intention.
- **dexkot-screen**: `Navigation`, `LocalLogger`.
- **dexkot-navigation**: `INavAction`, dispatchers, `INavActionLogDecorator`.
- **dexkot-results-errors**: `ResultOf`, which the use case decorator inspects.
- **dexkot-infra**: `WorkerCoroutineContextFactory` for per-work-type loggers.
