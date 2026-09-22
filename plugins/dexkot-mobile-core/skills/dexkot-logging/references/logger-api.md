# core-logger API reference (0.27.0)

Contents
1. ILogger and helpers
2. Composition and levels
3. Events
4. Interactors
5. LoggerContextElement and the `logger { }` DSL
6. Backends and what each one does with each call
7. Koin: loggerModule and wiring without it
8. Testing

## 1. ILogger and helpers

Package `dev.dexkot.mobile.core.logger`.

```kotlin
interface ILogger {
    fun setUserId(identifier: String)
    fun setUserProperty(key: String, value: String?)
    fun removeUserId()
    fun removeUserProperty(key: String)
    fun log(event: LogEvent)
    fun log(level: Level, tag: String, message: String, params: Map<String, Any> = mapOf())
    fun log(throwable: Throwable, message: String? = null, params: Map<String, Any> = mapOf())
}

class ConsoleLogger(tag: String) : ILogger                 // Android: android.util.Log; iOS: NSLog("[LEVEL] tag: msg")
class PersistentUserIdLogger(settings: com.russhwolf.settings.Settings, delegate: ILogger) : ILogger

fun ILogger.uiInteractor(sourceComponent: String): interactors.ui.ILoggerInteractor
fun ILogger.logicInteractor(sourceComponent: String): interactors.ddl.ILoggerInteractor

// Level shortcuts, each with a one-letter alias (v, d, i, w, e)
fun ILogger.verbose(tag: String, message: String, params: Map<String, Any> = emptyMap())
fun ILogger.debug(tag: String, message: String, params: Map<String, Any> = emptyMap())
fun ILogger.info(tag: String, message: String, params: Map<String, Any> = emptyMap())
fun ILogger.warning(tag: String, message: String, params: Map<String, Any> = emptyMap())
fun ILogger.error(tag: String, message: String, params: Map<String, Any> = emptyMap())
```

`PersistentUserIdLogger` stores the id under the key `"user_id"` in the given `Settings`, re-applies it to
the delegate in its constructor, and forwards everything else unchanged. `core-logger` exposes
`multiplatform-settings` as an `api` dependency, so `SharedPreferencesSettings` (Android) and
`NSUserDefaultsSettings` (iOS) are available without another dependency.

## 2. Composition and levels

```kotlin
// dev.dexkot.mobile.core.logger.composed
operator fun ILogger.plus(logger: ILogger): ILogger       // fans out every call, in order

// dev.dexkot.mobile.core.logger.levels
enum class Level(val value: String) { Verbose("verbose"), Debug("debug"), Info("info"), Warning("warning"), Error("error") }
fun ILogger.withLevel(level: Level): ILogger             // drops log(level, ...) calls below `level`
```

`withLevel` wraps a single logger, so it can apply to one backend only:
`console.withLevel(Level.Debug) + firebase`. It filters developer messages only; events, exceptions
and identity calls always pass.

## 3. Events

`interface LogEvent { val type: String; val parameters: Map<String, Any> }` in
`dev.dexkot.mobile.core.logger.events`.

| Class (package suffix) | `type` | Constructor |
|---|---|---|
| `ui.interaction.InteractionEvent` | `ui_interaction` | `(sourceComponent, interaction: Interaction, elementId, extras: Map<String,String> = mapOf())` |
| `ui.screen.ScreenEvent` | `ui_screen` | `(sourceComponent, action: Action)` |
| `ui.navigation.NavigationEvent` | `ui_navigation` | `(sourceComponent, navigationId, params: Map<String,Any> = emptyMap())` |
| `ui.intention.IntentionEvent` | `ui_intention` | `(sourceComponent, intentionId, params: Map<String,String> = mapOf())` |
| `ui.permission.PermissionEvent` | `ui_permission` | `(sourceComponent, permissions: Map<String, Status>)` |
| `ui.state.UiStateEvent` | `ui_state` | `(sourceComponent, elementId, state: UiState, extras: Map<String,String> = mapOf())` |
| `ddl.operation.OperationEvent` | `operation` | `(sourceComponent, operationId, status: Status, params: Map<String,String> = mapOf(), isBusinessEvent = false)` |

Enums:
- `Interaction`: `Tap, Swipe, ZoomIn, ZoomOut, DragStart, DragEnd, Search, Done`
- `Action`: `Enter, Exit`
- `UiState`: `Show, Hide, Expand, Collapse, FocusCapture, FocusRelease`
- permission `Status`: `Request("requested"), Granted, Denied, RequestIgnored("request_ignored")`
- operation `Status` (`events.ddl.operation.Status`): `InProgress, Success, Failure`

Flattened parameter keys: `source`, `navigation_id`, `intention_id`, `action`, `element_id`,
`interaction`, `state`, `operation_id`, `status`. Caller-supplied params become `parameter_<key>`,
extras `extra_<key>`, permissions `permission_<key>`.

## 4. Interactors

Two interfaces share the simple name `ILoggerInteractor`. Alias them when both are needed:

```kotlin
import dev.dexkot.mobile.core.logger.interactors.ddl.ILoggerInteractor as ILogicLoggerInteractor
import dev.dexkot.mobile.core.logger.interactors.ui.ILoggerInteractor as IUiLoggerInteractor
```

UI (`interactors.ui`):

```kotlin
interface ILoggerInteractor {
    fun enter()
    fun exit()
    fun navigate(navigationId: String, params: Map<String, String> = emptyMap())
    fun permission(permissions: List<String>)                 // logged as Status.Request
    fun permission(permissions: Map<String, Status>)
    fun interaction(elementId: String, interaction: Interaction, extras: Map<String, String> = emptyMap())
    fun uiState(elementId: String, state: UiState, extras: Map<String, String> = emptyMap())
    fun intention(intentionId: String, params: Map<String, String> = emptyMap())
}
class LoggerInteractor(logger: ILogger, sourceComponent: String) : ILoggerInteractor
```

Logic (`interactors.ddl`):

```kotlin
interface ILoggerInteractor {
    fun verbose(message: String, params: Map<String, Any> = emptyMap())
    fun debug(message: String, params: Map<String, Any> = emptyMap())
    fun info(message: String, params: Map<String, Any> = emptyMap())
    fun warning(message: String, params: Map<String, Any> = emptyMap())
    fun error(message: String, params: Map<String, Any>)          // no default for params
    fun error(throwable: Throwable, message: String? = null, params: Map<String, Any> = emptyMap())
    fun operation(operationId: String, result: Status, params: Map<String, String> = emptyMap(), isBusinessEvent: Boolean = false)
}
class LoggerInteractor(logger: ILogger, sourceComponent: String) : ILoggerInteractor
```

The tag for level messages is `"<sourceComponent>/<CallerClass>"`. On Android the caller class is read
from a fixed stack depth, so treat it as a hint; on iOS it is always `Unknown`.

Manual operation event, for work that is not a use case:

```kotlin
import dev.dexkot.mobile.core.logger.events.ddl.operation.Status
import dev.dexkot.mobile.core.logger.interactors.ddl.logger

logger { operation(operationId = "export_notes", result = Status.Success, params = mapOf("count" to "12"), isBusinessEvent = true) }
```

## 5. LoggerContextElement and the `logger { }` DSL

```kotlin
// dev.dexkot.mobile.core.logger.context
expect class LoggerContextElement(loggerInteractor: interactors.ddl.ILoggerInteractor) : CoroutineContext.Element {
    companion object Key : CoroutineContext.Key<LoggerContextElement> {
        fun current(): ILoggerInteractor?
        fun setDefault(logger: ILoggerInteractor)
    }
}

// dev.dexkot.mobile.core.logger.interactors.ddl
fun logger(): ILoggerInteractor?                      // = LoggerContextElement.current()
fun logger(block: ILoggerInteractor.() -> Unit)       // no-op when nothing is available
```

- **Android**: a `ThreadContextElement` backed by a `ThreadLocal`. Inside a coroutine carrying the element,
  `current()` returns its interactor, including from non-suspend functions called on that thread.
  Outside, it returns the default.
- **iOS**: a plain context element. `current()` always returns the default. The element is still in the
  context (`coroutineContext[LoggerContextElement]?.loggerInteractor`) if you need it explicitly.

Where elements come from:
- `loggerModule()` sets the default (source component `logicSourceComponent`, `"app"` by default).
- The generated ViewModel decorator adds one named after the screen to the ViewModel's `viewModelScope`.
- `WorkerCoroutineContextFactory` (core-work) can add one per work type. See **dexkot-infra**.
- You can add one to any scope you create.

## 6. Backends

| Call | ConsoleLogger | FirebaseLogger | PostHogLogger |
|---|---|---|---|
| `setUserId` / `removeUserId` | prints | Analytics + Crashlytics user id | `identify` / `reset` |
| `setUserProperty(k, v)` | prints | Analytics user property | `setPersonProperties`; `null` unsets |
| `log(event)` built-in types | prints one line | `logEvent(event.type, flattened params)` | see below |
| `log(event)` custom `LogEvent` | dropped | `logEvent(type, params)` | dropped |
| `log(level, ...)` | platform log | **ignored** | diagnostic log |
| `log(throwable, ...)` | platform log | Crashlytics `log` + custom keys + `recordException` | `captureException` |

PostHog event routing:
- `InteractionEvent` → capture `"<source>:<elementId>_<interaction>"` with `$screen_name`, `element_id`,
  `interaction` and the extras.
- `ScreenEvent(Enter)` → `screen(source)`. `Exit` is ignored.
- `OperationEvent` → diagnostic log; also capture `"<source>:<operationId>"` when
  `isBusinessEvent && status == Success`.
- `PermissionEvent` → diagnostic log; capture `ui_permission` for results only (not plain requests).
- `NavigationEvent`, `IntentionEvent`, `UiStateEvent` → diagnostic log only.

Platform notes:
- Both SDK-backed loggers have no-arg constructors and use the SDK singletons. Initialise the SDKs first.
- iOS: the modules bind FirebaseAnalytics / FirebaseCrashlytics / PostHog through CocoaPods cinterop. The
  app must link those pods (or SPM packages) itself; the klibs do not embed them.
- PostHog: configure the SDK with `captureScreenViews = false` to avoid duplicate screen events.

## 7. Koin

```kotlin
// dev.dexkot.mobile.core.logger.di.koin
fun loggerModule(consoleLoggerTag: String = "Console", logicSourceComponent: String = "app"): Module
```

Registers:
- `single { ConsoleLogger(consoleLoggerTag) }`, `single { FirebaseLogger() }`, `single { PostHogLogger() }`
  (by concrete type, no qualifiers).
- `single<ddl.ILoggerInteractor>(createdAtStart = true)`: `get<ILogger>().logicInteractor(logicSourceComponent)`,
  also installed with `LoggerContextElement.setDefault`.
- `factory<ui.ILoggerInteractor> { params -> get<ILogger>().uiInteractor(params.get()) }`.

It does not register `ILogger`: the app provides `single<ILogger>`.

### Wiring without loggerModule

If you want only one backend (and not its transitive SDK), depend on `core-logger` plus the backend you
want and register the same things by hand:

```kotlin
val appLoggingModule = module {
    single<ILogger>(createdAtStart = true) { ConsoleLogger("NotesApp") + FirebaseLogger() }
    single<ILogicLoggerInteractor>(createdAtStart = true) {
        get<ILogger>().logicInteractor("app").also { LoggerContextElement.setDefault(it) }
    }
    factory<IUiLoggerInteractor> { params -> get<ILogger>().uiInteractor(sourceComponent = params.get()) }
}
```

## 8. Testing

Implement `ILogger` with a recording fake and build interactors from it (`fake.logicInteractor("test")`).
To exercise `logger { }`, call `LoggerContextElement.setDefault(fake.logicInteractor("test"))` in the test
setup, or run the code under `withContext(LoggerContextElement(...))` on Android/JVM.
