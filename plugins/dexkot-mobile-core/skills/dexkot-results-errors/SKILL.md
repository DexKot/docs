---
name: dexkot-results-errors
description: "Error and result handling with DexKot mobile-core (dev.dexkot.mobile:core-foundation). Use whenever code returns or consumes ResultOf, successOf/failureOf, unwrap/getOrDefault/map, builds ErrorEntity values (DeviceError, NetworkError, UnexpectedError, NoData), defines a custom BusinessError, writes an ErrorMapper or chains mappers with `+`, picks a CacheFetchPolicy, sets up @CommonParcelize/CommonParcelable in a KMP module, uses PlatformDispatchers.io, or converts ResultOf to Resource in a ViewModel. Trigger even for small edits like \"return a failure here\" or \"add a not-found error\" in a project that depends on DexKot."
---

# DexKot results & errors (core-foundation)

`core-foundation` is the dependency-free base of DexKot mobile-core. It gives every layer the same
vocabulary for "this operation worked / failed and why": `ResultOf<T>` for outcomes, `ErrorEntity`
for typed failures, `ErrorMapper` for turning third-party exceptions into `ErrorEntity`, plus a few
cross-platform helpers (`CommonParcelize`, `PlatformDispatchers`, `CacheFetchPolicy`).

The core idea: **data and domain code never throw across a layer boundary.** Exceptions are caught
where they happen (data sources), translated into an `ErrorEntity`, and travel up as a value. The
compiler then forces every caller to decide what a failure means, instead of discovering an uncaught
exception in production.

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

// build.gradle.kts (KMP module: put it in commonMain.dependencies)
dependencies {
    implementation("dev.dexkot.mobile:core-foundation:0.27.0")
}
```

Kotlin Multiplatform module. Android: published. iOS: build from source (`publishToMavenLocal`).
It exposes `kotlinx-coroutines-core` as an `api` dependency.

If the module defines its own error types or parcelable models, also apply the parcelize setup in
[CommonParcelize setup](#commonparcelize-setup-for-kmp-modules).

## Core concepts

### ResultOf

```kotlin
sealed interface ResultOf<out Data> {
    class Success<Data>(val data: Data) : ResultOf<Data>
    class Failure<Data>(val error: ErrorEntity) : ResultOf<Data>
}
```

Package `dev.dexkot.mobile.core.foundation.result`. Helpers (all top-level functions/extensions in the
same package): `successOf(data)`, `successOf()` (→ `Success<Unit>`), `failureOf(throwable?)`
(→ `UnexpectedError`), `failureOf(error)`, `unwrap()`, `getOrDefault(default)`, `map { }`, and
`Failure.map()` to re-type a failure. That is the complete list — there is no `flatMap`, `fold`,
`onSuccess` or `getOrThrow`. Compose with `when`.

### ErrorEntity

Package `dev.dexkot.mobile.core.foundation.errors`. A sealed class with `severity: ErrorSeverity`
and `exception: Throwable?`:

| Type | Severity | Use it for |
|---|---|---|
| `DeviceError(exception?)` | ERROR | Local I/O: database, files, preferences |
| `NetworkError(exception?)` | ERROR | Transport failures talking to a server |
| `UnexpectedError(exception?)` | ERROR | Bugs and inconsistencies; the fallback for anything unmapped |
| `NoData(exception?)` | WARN | A source that has not been populated yet (e.g. empty cache) |
| `BusinessError` (open, protected ctor) | you choose | Expected domain outcomes: not found, quota exceeded, validation |

`ErrorSeverity` is `INFO`, `WARN`, `ERROR`. Builders: `deviceError(t?)`, `networkError(t?)`,
`unexpectedError(t?)`. There is no `noData()` builder — write `ErrorEntity.NoData()`.

Severity lets the UI and logging treat an expected outcome (WARN "project not found") differently
from a real fault (ERROR "disk full") without knowing every concrete error type.

## How to

### Return results from data code

```kotlin
import dev.dexkot.mobile.core.foundation.errors.deviceError
import dev.dexkot.mobile.core.foundation.result.ResultOf

internal class TaskFileDataSource(private val store: TaskFileStore) {

    suspend fun readTitle(id: Long): ResultOf<String> {
        return try {
            val title = store.readTitle(id)                        // may throw, may return null
                ?: return ResultOf.Failure(TaskNotFound(id = id))  // expected outcome -> BusinessError
            ResultOf.Success(title)
        } catch (exception: Exception) {
            ResultOf.Failure(deviceError(exception))               // local I/O fault -> DeviceError
        }
    }
}
```

Catch at the lowest layer that talks to the outside world (the data source), pick the error type
there, and return. Repositories and use cases only pass results along or combine them.

### Consume and compose results

Use `when` with `is` — the subclasses are plain classes, not data classes.

```kotlin
import dev.dexkot.mobile.core.foundation.result.ResultOf
import dev.dexkot.mobile.core.foundation.result.map

suspend fun projectSummary(projectId: Long): ResultOf<ProjectSummary> {
    val project = when (val result = projectRepository.getProject(projectId)) {
        is ResultOf.Success -> result.data
        is ResultOf.Failure -> return result.map()   // re-type Failure<Project> -> Failure<ProjectSummary>
    }
    return taskRepository.getTasks(projectId).map { tasks ->
        ProjectSummary(title = project.title, openTasks = tasks.count { !it.done })
    }
}
```

- `result.map { }` transforms success data and passes failures through (the converter is not called).
- `failure.map()` (no lambda) keeps the same `ErrorEntity` instance but changes the type parameter.
  `Success`/`Failure` are invariant, so a `Failure<Project>` is not a `ResultOf<ProjectSummary>` —
  this is how you early-return it.
- `getOrDefault(x)` for "use a fallback on failure"; `unwrap()` only when `Data` is non-nullable.

### Define a business error

```kotlin
import dev.dexkot.mobile.core.foundation.CommonParcelize
import dev.dexkot.mobile.core.foundation.errors.ErrorEntity
import dev.dexkot.mobile.core.foundation.errors.ErrorSeverity

@CommonParcelize
class TaskNotFound(
    val id: Long
) : ErrorEntity.BusinessError(severity = ErrorSeverity.WARN)

@CommonParcelize
object ProjectQuotaExceeded : ErrorEntity.BusinessError(severity = ErrorSeverity.WARN)
```

- Put them in `domain/error/` next to the entity they describe; they are part of the domain contract.
- `severity` is required (no default); `exception` defaults to `null`. The constructor is `protected`,
  so you can only subclass, not instantiate `BusinessError` directly.
- Annotate every subclass with `@CommonParcelize`. On Android `ErrorEntity` is `Parcelable` (it ends
  up in `Resource.Error`, `SavedStateHandle`, navigation results). Without the annotation the subclass
  only inherits `BusinessError`'s generated parcel code, which knows nothing about your fields, so a
  round trip through a `Bundle` will not give your type back. All properties must be parcelable types.
- Use an `object` when the error carries no data.

Callers match on type: `if (result is ResultOf.Failure && result.error is TaskNotFound) ...`.

### Map third-party exceptions with ErrorMapper

```kotlin
import dev.dexkot.mobile.core.foundation.errors.ErrorEntity
import dev.dexkot.mobile.core.foundation.errors.mapper.ErrorMapper
import dev.dexkot.mobile.core.foundation.errors.networkError

class ApiException(val code: String, message: String? = null) : Exception(message)
class TransportException(cause: Throwable? = null) : Exception(cause)

/** Knows only the error codes of the projects endpoint. */
class ProjectApiErrorMapper : ErrorMapper {
    override fun map(throwable: Throwable): ErrorEntity? = when {
        throwable is ApiException && throwable.code == "project/quota-exceeded" -> ProjectQuotaExceeded
        else -> null      // not mine: let the next mapper try
    }
}

/** Generic transport failures, shared by every remote data source. */
class TransportErrorMapper : ErrorMapper {
    override fun map(throwable: Throwable): ErrorEntity? = when (throwable) {
        is TransportException -> networkError(throwable)
        else -> null
    }
}

val projectErrorMapper: ErrorMapper = ProjectApiErrorMapper() + TransportErrorMapper()
```

In the data source: `catch (exception: Exception) { errorMapper.handle(exception) }`.

- `map()` returns `null` for exceptions the mapper does not understand. Return `null` rather than a
  guess so the chain can continue.
- `handle<T>(throwable)` returns `ResultOf.Failure<T>`; if `map()` gives `null` it falls back to
  `UnexpectedError(throwable)`, so an unmapped exception is never lost.
- `a + b + c` evaluates left to right and stops at the first non-null result. Put the most specific
  mapper first; a generic mapper placed first would swallow errors the specific one should classify.
- `+` returns `ErrorMapper` (the chain class is internal) — type properties and constructor
  parameters as `ErrorMapper`.
- Inject the composed mapper into the data source constructor so tests can pass a fake.

### Choose a CacheFetchPolicy

`dev.dexkot.mobile.core.foundation.data.policies.CacheFetchPolicy` is only an enum; the library has
no cache engine. It gives repositories and callers a shared name for four strategies:

| Value | Meaning |
|---|---|
| `CACHE_ONLY` | Cache or failure; never fetch |
| `FETCH_CURRENT` | Always fetch; fail if the fetch fails |
| `CACHED_OR_FETCHED` | Cache if available, otherwise fetch |
| `FETCHED_OR_CACHED` | Fetch; fall back to cache if the fetch fails |

Each repository implements all four branches with a `when` (see the `dexkot-data-layer` skill).
Take the policy as a parameter with a sensible default so use cases can override it.

### Run blocking work on the right dispatcher

```kotlin
import dev.dexkot.mobile.core.foundation.coroutines.PlatformDispatchers
import kotlinx.coroutines.withContext

suspend fun readFile(path: String): ResultOf<ByteArray> = withContext(PlatformDispatchers.io) { /* ... */ }
```

`PlatformDispatchers` has `main`, `io`, `default`, `unconfined`. It exists because `Dispatchers.IO`
is not public API on Kotlin/Native, so common code could not otherwise name an I/O dispatcher. On
Android these are the regular `Dispatchers.*`. On iOS, `main` is the GCD main queue and `io` and
`default` both dispatch to the same GCD global queue (no separate I/O pool, no 64-thread cap).
For testability, depend on the `PlatformDispatcher` interface and default it to `PlatformDispatchers`.

## CommonParcelize setup for KMP modules

`@CommonParcelize` and `CommonParcelable` let common code declare Android-parcelable types:
`CommonParcelable` is `android.os.Parcelable` on Android and an empty interface on iOS.
`@CommonParcelize` is a plain annotation — the Kotlin parcelize plugin only honours it if you tell
it to. Add this to every KMP module that declares `@CommonParcelize` classes:

```kotlin
plugins {
    // your KMP + Android library plugins
    id("org.jetbrains.kotlin.plugin.parcelize")
}

kotlin {
    androidTarget {
        compilations.configureEach {
            compileTaskProvider.configure {
                compilerOptions {
                    freeCompilerArgs.addAll(
                        "-P",
                        "plugin:org.jetbrains.kotlin.parcelize:additionalAnnotation=dev.dexkot.mobile.core.foundation.CommonParcelize"
                    )
                }
            }
        }
    }
}
```

Domain models that travel through `SavedStateHandle` or navigation use the same pair:
`@CommonParcelize data class Task(...) : CommonParcelable`. Android-only modules can keep using
`@Parcelize`/`Parcelable` directly.

## The ResultOf → Resource boundary

`ResultOf` is the language of data sources, repositories and use cases, and it reaches the
ViewModel. The ViewModel translates it into `Resource<T>` (from `core-ui`,
`dev.dexkot.mobile.core.ui.viewmodel.resource.Resource`: `None`, `Loading`, `Loaded`, `Error`),
which is what the model/UI state exposes:

```kotlin
_model.update { it.copy(tasks = it.tasks.toLoading()) }
when (val result = getProjectTasks(projectId)) {
    is ResultOf.Success -> _model.update { it.copy(tasks = it.tasks.toLoaded(result.data)) }
    is ResultOf.Failure -> _model.update { it.copy(tasks = it.tasks.toError(result.error)) }
}
```

Keep `ResultOf` out of UI state and composables (it describes one call, not a loading lifecycle),
and keep `Resource` out of use cases and repositories (they should not know about UI loading
states). See the `dexkot-viewmodel` skill for the full pattern.

## Rules

- Return `ResultOf` from every data source, repository and use case function that can fail. Catch
  exceptions in data sources; do not let them escape.
- Pick the most specific error: `DeviceError` for local I/O, `NetworkError` for transport,
  `BusinessError` subclasses for expected domain outcomes, `UnexpectedError` for "should not happen".
- Keep the original exception in the error (`deviceError(exception)`) so logs keep the stack trace.
- Branch with `when`/`is`; propagate failures with `failure.map()`.
- Build remote error handling as small focused `ErrorMapper`s composed with `+`.

## Gotchas

- **No `equals`/`toString`.** `ResultOf.Success`, `ResultOf.Failure` and every `ErrorEntity` are plain
  classes. `result == successOf(1)` or `error == ErrorEntity.NoData()` is always `false`. In tests,
  assert with `is` and compare `.data` / `.error` fields.
- **`unwrap()` is ambiguous for nullable data.** For `ResultOf<Task?>`, `null` may mean "failed" or
  "succeeded with null". `getOrDefault(x)` likewise returns `x` for a `Success(null)`. Use `when`.
- **`failureOf(...)` may need a type argument** when there is no expected type:
  `val f = failureOf<Task>(TaskNotFound(id))`. `failureOf()` / `failureOf(throwable)` always produce
  `UnexpectedError`; to fail with a specific error call `failureOf(error)` or `ResultOf.Failure(error)`.
- **Re-typing failures**: returning `result` where `result: ResultOf.Failure<A>` from a function
  returning `ResultOf<B>` does not compile; use `result.map()`.
- **Forgotten `@CommonParcelize` or compiler flag**: without the `additionalAnnotation` flag, Android
  compilation fails for `CommonParcelable` classes (missing `writeToParcel`); without the annotation on
  a `BusinessError` subclass, the subclass does not survive parcelling as itself.
- **Mapper order matters**: `generic + specific` never reaches `specific` for exceptions `generic`
  understands.
- **Catching `Exception` also catches `CancellationException`.** In non-transactional data sources,
  rethrow it (`catch (e: CancellationException) { throw e }` before the generic catch) so coroutine
  cancellation keeps working.

## References

- `references/api.md` — every public signature in core-foundation with notes.
- Sibling skills: `dexkot-data-layer` (data sources, repositories, use cases, transactions),
  `dexkot-viewmodel` (ResultOf → Resource), `dexkot-logging` (logging failures), `dexkot-di-koin`.
