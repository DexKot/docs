# Results and errors

This guide covers `core-foundation`, the base module of DexKot mobile-core: how operations report
success and failure (`ResultOf`), how failures are described (`ErrorEntity`), how third-party
exceptions become typed errors (`ErrorMapper`), and a few cross-platform helpers
(`CacheFetchPolicy`, `PlatformDispatchers`, `CommonParcelize`).

- [Why results instead of exceptions](#why-results-instead-of-exceptions)
- [Installation](#installation)
- [ResultOf](#resultof)
- [ErrorEntity](#errorentity)
- [Custom business errors](#custom-business-errors)
- [Mapping exceptions with ErrorMapper](#mapping-exceptions-with-errormapper)
- [CacheFetchPolicy](#cachefetchpolicy)
- [PlatformDispatchers](#platformdispatchers)
- [CommonParcelize in KMP modules](#commonparcelize-in-kmp-modules)
- [From ResultOf to Resource](#from-resultof-to-resource)
- [Pitfalls](#pitfalls)
- [API reference](#api-reference)

## Why results instead of exceptions

Kotlin has no checked exceptions, so nothing tells a caller that `repository.getTask(id)` can fail
because the disk is full, the row does not exist, or the server timed out. In DexKot apps the data
layer catches exceptions where they happen and returns a value instead:

```kotlin
suspend fun getTask(id: Long): ResultOf<Task>
```

Now every caller has to handle both outcomes to get at the data, failures carry a type the UI can
react to (`TaskNotFound` vs. `NetworkError`), and a crash from an unexpected exception deep in the
stack becomes a visible, logged `UnexpectedError`.

## Installation

```kotlin
// settings.gradle.kts
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
        maven("https://dexkot.github.io/maven")
    }
}

// build.gradle.kts (in commonMain.dependencies for a KMP module)
dependencies {
    implementation("dev.dexkot.mobile:core-foundation:0.27.0")
}
```

`core-foundation` is a Kotlin Multiplatform library. Android: published. iOS: build from source
(`publishToMavenLocal`). It brings `kotlinx-coroutines-core` with it as an `api` dependency.

## ResultOf

```kotlin
package dev.dexkot.mobile.core.foundation.result

sealed interface ResultOf<out Data> {
    class Success<Data>(val data: Data) : ResultOf<Data>
    class Failure<Data>(val error: ErrorEntity) : ResultOf<Data>
}
```

### Creating results

```kotlin
ResultOf.Success(task)          // or successOf(task)
successOf()                     // Success<Unit>
ResultOf.Failure(TaskNotFound(id))   // or failureOf(TaskNotFound(id))
failureOf<Task>(exception)      // Failure(UnexpectedError(exception))
```

### Consuming results

Branch with `when` and `is`:

```kotlin
when (val result = getTask(id)) {
    is ResultOf.Success -> show(result.data)
    is ResultOf.Failure -> showError(result.error)
}
```

The library ships a deliberately small set of helpers:

| Helper | What it does |
|---|---|
| `result.map { data -> other }` | Transforms the success value; a failure passes through untouched |
| `failure.map()` | Same error, different type parameter — for propagating a failure |
| `result.getOrDefault(fallback)` | The data, or `fallback` on failure (and on `Success(null)`) |
| `result.unwrap()` | The data, or `null` on failure |

There is no `flatMap` or `fold`. Chaining dependent calls is written with `when` and an early
return, which reads top to bottom:

```kotlin
import dev.dexkot.mobile.core.foundation.result.ResultOf
import dev.dexkot.mobile.core.foundation.result.map

suspend fun projectSummary(projectId: Long): ResultOf<ProjectSummary> {
    val project = when (val result = projectRepository.getProject(projectId)) {
        is ResultOf.Success -> result.data
        is ResultOf.Failure -> return result.map()      // Failure<Project> -> Failure<ProjectSummary>
    }
    return taskRepository.getTasks(projectId).map { tasks ->
        ProjectSummary(title = project.title, openTasks = tasks.count { !it.done })
    }
}
```

`result.map()` is needed because `Success` and `Failure` are invariant: a `Failure<Project>` is not a
`ResultOf<ProjectSummary>`, even though it carries no project.

## ErrorEntity

```kotlin
package dev.dexkot.mobile.core.foundation.errors

sealed class ErrorEntity(open val severity: ErrorSeverity, open val exception: Throwable? = null)
```

| Type | Severity | Typical source |
|---|---|---|
| `DeviceError` | ERROR | Database, file system, preferences |
| `NetworkError` | ERROR | Connection lost, timeout, server unreachable |
| `UnexpectedError` | ERROR | Bugs, impossible states, unmapped exceptions |
| `NoData` | WARN | A cache or store that has never been filled |
| `BusinessError` | chosen by you | Expected domain outcomes |

`ErrorSeverity` is `INFO` (not really an error, but the operation cannot finish), `WARN` (something
the user can fix, or not worrying) and `ERROR` (an actual fault). The UI and logging can use it to
decide between an inline message and an error screen, or between a warning and an error log,
without enumerating every error type.

Short builders exist for the common types: `deviceError(e)`, `networkError(e)`,
`unexpectedError(e)` (the argument is optional). For `NoData` use the constructor,
`ErrorEntity.NoData()`.

Always pass the original exception when there is one — it keeps the stack trace for logging.

## Custom business errors

Expected outcomes of your domain ("task not found", "quota exceeded") are subclasses of
`ErrorEntity.BusinessError`:

```kotlin
package com.example.tasks.domain.error

import dev.dexkot.mobile.core.foundation.CommonParcelize
import dev.dexkot.mobile.core.foundation.errors.ErrorEntity
import dev.dexkot.mobile.core.foundation.errors.ErrorSeverity

@CommonParcelize
class TaskNotFound(val id: Long) : ErrorEntity.BusinessError(severity = ErrorSeverity.WARN)

@CommonParcelize
object ProjectQuotaExceeded : ErrorEntity.BusinessError(severity = ErrorSeverity.WARN)
```

Notes:

- `severity` has no default — decide it for each error. `exception` defaults to `null`.
- The constructor is `protected`: `BusinessError` is only meant to be subclassed.
- `ErrorEntity` is sealed, but `BusinessError` is open, so subclasses can live in any module.
  Keep them in the domain package of the entity they describe.
- Use `object` for errors without data.
- Annotate each subclass with `@CommonParcelize`. On Android, `ErrorEntity` is `Parcelable` because
  errors are stored in UI state (`Resource.Error`) that survives process death. An un-annotated
  subclass only has the parcel code generated for `BusinessError` itself, which does not know your
  fields, so it will not come back as your type. Every property must be a parcelable type.

Callers check the type:

```kotlin
if (result is ResultOf.Failure && result.error is TaskNotFound) navigateBack()
```

## Mapping exceptions with ErrorMapper

Remote data sources receive exceptions from HTTP clients and SDKs. An `ErrorMapper` translates them:

```kotlin
interface ErrorMapper {
    fun map(throwable: Throwable): ErrorEntity?                 // null = "I don't know this one"
    fun <T> handle(throwable: Throwable): ResultOf.Failure<T>   // map(), else UnexpectedError
    operator fun plus(otherMapper: ErrorMapper): ErrorMapper    // chain
}
```

Write small mappers that each understand one thing, then chain them:

```kotlin
import dev.dexkot.mobile.core.foundation.errors.ErrorEntity
import dev.dexkot.mobile.core.foundation.errors.mapper.ErrorMapper
import dev.dexkot.mobile.core.foundation.errors.networkError

class ApiException(val code: String, message: String? = null) : Exception(message)
class TransportException(cause: Throwable? = null) : Exception(cause)

class ProjectApiErrorMapper : ErrorMapper {
    override fun map(throwable: Throwable): ErrorEntity? = when {
        throwable is ApiException && throwable.code == "project/quota-exceeded" -> ProjectQuotaExceeded
        else -> null
    }
}

class TransportErrorMapper : ErrorMapper {
    override fun map(throwable: Throwable): ErrorEntity? = when (throwable) {
        is TransportException -> networkError(throwable)
        else -> null
    }
}

val projectErrorMapper: ErrorMapper = ProjectApiErrorMapper() + TransportErrorMapper()
```

Use it in the data source's `catch`:

```kotlin
try {
    ResultOf.Success(api.getProjects().map { it.toDomain() })
} catch (exception: CancellationException) {
    throw exception
} catch (exception: Exception) {
    projectErrorMapper.handle(exception)
}
```

How the chain behaves:

1. Mappers run left to right with the same exception; the first non-null result wins.
2. If none recognises it, `handle` returns `Failure(UnexpectedError(exception))` — nothing is lost.
3. Therefore put the most specific mapper first. A generic mapper in front would claim exceptions the
   specific one should classify.

Inject the composed mapper into the data source (typed as `ErrorMapper`) so it can be replaced in tests.

## CacheFetchPolicy

```kotlin
enum class CacheFetchPolicy { CACHE_ONLY, FETCH_CURRENT, CACHED_OR_FETCHED, FETCHED_OR_CACHED }
```

| Value | Behaviour the repository should implement |
|---|---|
| `CACHE_ONLY` | Return cached data or a failure; never fetch |
| `FETCH_CURRENT` | Always fetch; fail if the fetch fails |
| `CACHED_OR_FETCHED` | Return cached data if present, otherwise fetch |
| `FETCHED_OR_CACHED` | Fetch; if it fails, fall back to the cache |

It is a shared vocabulary only — there is no caching engine behind it. Each repository implements
the four cases; see the [data layer guide](data-layer.md#caching-repositories).

## PlatformDispatchers

```kotlin
import dev.dexkot.mobile.core.foundation.coroutines.PlatformDispatchers

withContext(PlatformDispatchers.io) { /* blocking I/O */ }
```

`Dispatchers.IO` is not public API on Kotlin/Native, so shared code cannot name it.
`PlatformDispatchers` (an object implementing the `PlatformDispatcher` interface) fills the gap:

| | Android | iOS |
|---|---|---|
| `main` | `Dispatchers.Main` | GCD main queue |
| `io` | `Dispatchers.IO` | GCD global queue |
| `default` | `Dispatchers.Default` | the same GCD global queue |
| `unconfined` | `Dispatchers.Unconfined` | `Dispatchers.Unconfined` |

On iOS there is no separate I/O pool: `io` and `default` share one queue managed by GCD. For tests,
depend on the `PlatformDispatcher` interface and default the parameter to `PlatformDispatchers`.

## CommonParcelize in KMP modules

`CommonParcelable` is `android.os.Parcelable` on Android and an empty interface on iOS, so common
code can declare types that are parcelable on Android. `@CommonParcelize` marks them — but the
Kotlin parcelize plugin only honours it when configured to:

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

Without the flag, Android compilation fails for any `CommonParcelable` class because
`writeToParcel` is never generated. Then:

```kotlin
@CommonParcelize
data class Task(val id: Long, val projectId: Long, val title: String, val done: Boolean) : CommonParcelable
```

## From ResultOf to Resource

`ResultOf` describes the outcome of one call and belongs to data sources, repositories and use cases.
It reaches the ViewModel, which translates it into `Resource<T>` from `core-ui`
(`dev.dexkot.mobile.core.ui.viewmodel.resource.Resource`: `None`, `Loading`, `Loaded`, `Error`) —
a value that describes a loading lifecycle and is what UI state exposes:

```kotlin
_model.update { it.copy(tasks = it.tasks.toLoading()) }
when (val result = getProjectTasks(projectId)) {
    is ResultOf.Success -> _model.update { it.copy(tasks = it.tasks.toLoaded(result.data)) }
    is ResultOf.Failure -> _model.update { it.copy(tasks = it.tasks.toError(result.error)) }
}
```

Keep `ResultOf` out of UI state and `Resource` out of the domain. The ViewModel guide covers this
in detail.

## Pitfalls

- **No `equals`.** `ResultOf.Success`, `ResultOf.Failure` and the `ErrorEntity` types are plain
  classes. `result == successOf(1)` and `error == ErrorEntity.NoData()` are always `false`. Use `is`,
  and in tests compare `.data` or check `.error is X`.
- **Nullable data.** For `ResultOf<Task?>`, `unwrap()` returns `null` both for a failure and for a
  successful `null`, and `getOrDefault` returns the fallback for both. Use `when`.
- **Type inference.** `failureOf(error)` and `ResultOf.Failure(error)` need a type argument when
  there is no expected type: `failureOf<Task>(TaskNotFound(id))`.
- **`failureOf(throwable)` always means `UnexpectedError`.** For a specific error, pass the
  `ErrorEntity`.
- **Propagating failures across types** requires `failure.map()`.
- **Mapper order.** `generic + specific` never lets `specific` see what `generic` already mapped.
- **`catch (e: Exception)` catches cancellation too.** Rethrow `CancellationException` first in
  non-transactional code so cancelled coroutines stop.
- **Parcelize setup.** Missing compiler flag → compile error on Android; missing `@CommonParcelize`
  on an error subclass → it does not survive a parcel round trip as itself.

## API reference

```kotlin
// dev.dexkot.mobile.core.foundation.result
sealed interface ResultOf<out Data> {
    class Success<Data>(val data: Data) : ResultOf<Data>
    class Failure<Data>(val error: ErrorEntity) : ResultOf<Data>
}
fun <Data> ResultOf<Data>.unwrap(): Data?
fun <Data> ResultOf<Data>.getOrDefault(defaultValue: Data): Data
fun <Data, Other> ResultOf<Data>.map(converter: (Data) -> Other): ResultOf<Other>
fun <Data, Other> ResultOf.Success<Data>.map(converter: (Data) -> Other): ResultOf<Other>
fun <Other> ResultOf.Failure<*>.map(): ResultOf.Failure<Other>
fun <T> successOf(data: T): ResultOf.Success<T>
fun successOf(): ResultOf.Success<Unit>
fun <T> failureOf(throwable: Throwable? = null): ResultOf.Failure<T>
fun <T> failureOf(error: ErrorEntity): ResultOf.Failure<T>

// dev.dexkot.mobile.core.foundation.errors
@CommonParcelize
sealed class ErrorEntity(open val severity: ErrorSeverity, open val exception: Throwable? = null) : CommonParcelable {
    class DeviceError(override val exception: Throwable? = null)       // ERROR
    class UnexpectedError(override val exception: Throwable? = null)   // ERROR
    class NetworkError(override val exception: Throwable? = null)      // ERROR
    class NoData(override val exception: Throwable? = null)            // WARN
    open class BusinessError protected constructor(
        override val exception: Throwable? = null,
        override val severity: ErrorSeverity
    )
}
enum class ErrorSeverity { INFO, WARN, ERROR }
fun deviceError(throwable: Throwable? = null): ErrorEntity.DeviceError
fun networkError(throwable: Throwable? = null): ErrorEntity.NetworkError
fun unexpectedError(throwable: Throwable? = null): ErrorEntity.UnexpectedError

// dev.dexkot.mobile.core.foundation.errors.mapper
interface ErrorMapper {
    fun <T> handle(throwable: Throwable): ResultOf.Failure<T>
    fun map(throwable: Throwable): ErrorEntity?
    operator fun plus(otherMapper: ErrorMapper): ErrorMapper
}

// dev.dexkot.mobile.core.foundation.data.policies
enum class CacheFetchPolicy { CACHE_ONLY, FETCH_CURRENT, CACHED_OR_FETCHED, FETCHED_OR_CACHED }

// dev.dexkot.mobile.core.foundation.coroutines
interface PlatformDispatcher {
    val main: CoroutineDispatcher
    val io: CoroutineDispatcher
    val default: CoroutineDispatcher
    val unconfined: CoroutineDispatcher
}
expect object PlatformDispatchers : PlatformDispatcher

// dev.dexkot.mobile.core.foundation
@Target(AnnotationTarget.CLASS) @Retention(AnnotationRetention.BINARY)
annotation class CommonParcelize
expect interface CommonParcelable   // Android: android.os.Parcelable; iOS: empty interface

// dev.dexkot.mobile.core.foundation.data.preferences (Android only)
fun SharedPreferences.Editor.save()   // commit(), throws SavePreferencesException on false
class SavePreferencesException : RuntimeException
```
