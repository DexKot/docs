# core-foundation API reference (0.27.0)

Artifact: `dev.dexkot.mobile:core-foundation:0.27.0` (KMP: Android published; iOS build from source).
Every declaration below is the complete public surface of the module.

## Contents

1. [Result](#result) — `dev.dexkot.mobile.core.foundation.result`
2. [Errors](#errors) — `dev.dexkot.mobile.core.foundation.errors`
3. [Error mappers](#error-mappers) — `dev.dexkot.mobile.core.foundation.errors.mapper`
4. [Cache policy](#cache-policy) — `dev.dexkot.mobile.core.foundation.data.policies`
5. [Dispatchers](#dispatchers) — `dev.dexkot.mobile.core.foundation.coroutines`
6. [Parcelize bridge](#parcelize-bridge) — `dev.dexkot.mobile.core.foundation`
7. [Android-only preferences helper](#android-only-preferences-helper)

## Result

```kotlin
sealed interface ResultOf<out Data> {
    class Success<Data>(val data: Data) : ResultOf<Data>
    class Failure<Data>(val error: ErrorEntity) : ResultOf<Data>
}
```

Plain classes: no `equals`, `hashCode`, `toString`, `copy`. `Success`/`Failure` are invariant in
`Data` even though `ResultOf` is covariant.

```kotlin
fun <Data> ResultOf<Data>.unwrap(): Data?
fun <Data> ResultOf<Data>.getOrDefault(defaultValue: Data): Data
fun <Data, Other> ResultOf<Data>.map(converter: (Data) -> Other): ResultOf<Other>
fun <Data, Other> ResultOf.Success<Data>.map(converter: (Data) -> Other): ResultOf<Other>
fun <Other> ResultOf.Failure<*>.map(): ResultOf.Failure<Other>

fun <T> successOf(data: T): ResultOf.Success<T>
fun successOf(): ResultOf.Success<Unit>
fun <T> failureOf(throwable: Throwable? = null): ResultOf.Failure<T>   // UnexpectedError(throwable)
fun <T> failureOf(error: ErrorEntity): ResultOf.Failure<T>
```

| Function | Behaviour (verified by the library's tests) |
|---|---|
| `unwrap()` | `data` for `Success`, `null` for `Failure` |
| `getOrDefault(d)` | `data`, or `d` for `Failure` **and** for `Success(null)` (implemented as `unwrap() ?: d`) |
| `map { }` | converts `Success` data; for `Failure` keeps the same `ErrorEntity` instance and never calls the converter |
| `Failure.map()` | same error instance, new type parameter |
| `failureOf(throwable)` / `failureOf()` | `Failure(UnexpectedError(throwable))` |
| `failureOf(error)` | `Failure(error)`, same instance |

Not provided: `flatMap`, `fold`, `onSuccess`, `onFailure`, `getOrThrow`, `isSuccess`. Use `when`.

## Errors

Superclass constructor calls are abridged to the severity they pass.

```kotlin
@CommonParcelize
sealed class ErrorEntity(
    open val severity: ErrorSeverity,
    open val exception: Throwable? = null
) : CommonParcelable {

    class DeviceError(override val exception: Throwable? = null) : ErrorEntity(/* ERROR */)
    class UnexpectedError(override val exception: Throwable? = null) : ErrorEntity(/* ERROR */)
    class NetworkError(override val exception: Throwable? = null) : ErrorEntity(/* ERROR */)
    class NoData(override val exception: Throwable? = null) : ErrorEntity(/* WARN */)

    open class BusinessError protected constructor(
        override val exception: Throwable? = null,
        override val severity: ErrorSeverity
    ) : ErrorEntity(severity = severity, exception = exception)
}

enum class ErrorSeverity { INFO, WARN, ERROR }

fun deviceError(throwable: Throwable? = null): ErrorEntity.DeviceError
fun networkError(throwable: Throwable? = null): ErrorEntity.NetworkError
fun unexpectedError(throwable: Throwable? = null): ErrorEntity.UnexpectedError
```

- No `noData()` builder; use `ErrorEntity.NoData()`.
- `ErrorEntity` is sealed, but `BusinessError` is `open`, so any module can subclass it. A `when` over
  `ErrorEntity` is exhaustive with the five branches above; custom errors fall under `BusinessError`.
- `ErrorSeverity` meanings: `INFO` — not really an error, but the operation cannot finish;
  `WARN` — something the user can fix or that is not worrying; `ERROR` — an actual fault.

Custom error template:

```kotlin
@CommonParcelize
class TaskNotFound(val id: Long) : ErrorEntity.BusinessError(severity = ErrorSeverity.WARN)

@CommonParcelize
object ProjectQuotaExceeded : ErrorEntity.BusinessError(severity = ErrorSeverity.WARN)

@CommonParcelize
class SyncRejected(
    val reason: String,
    override val exception: Throwable? = null
) : ErrorEntity.BusinessError(exception = exception, severity = ErrorSeverity.ERROR)
```

## Error mappers

```kotlin
interface ErrorMapper {
    fun <T> handle(throwable: Throwable): ResultOf.Failure<T>   // map() ?: UnexpectedError(throwable)
    fun map(throwable: Throwable): ErrorEntity?                  // null = "not mine"
    operator fun plus(otherMapper: ErrorMapper): ErrorMapper     // chain
}
```

Chain semantics (`a + b + c`): `a.map(t)`; if `null`, `b.map(t)`; if `null`, `c.map(t)`; the same
throwable instance is passed down; evaluation stops at the first non-null; if all return `null`,
`map` returns `null` and `handle` returns `Failure(UnexpectedError(t))`. The implementation class
`ErrorMapperChain` is `internal`.

## Cache policy

```kotlin
enum class CacheFetchPolicy {
    CACHE_ONLY,          // cache or error; no fetch
    FETCH_CURRENT,       // fetch; error if the fetch fails
    CACHED_OR_FETCHED,   // cache if available, else fetch
    FETCHED_OR_CACHED    // fetch; on failure fall back to cache
}
```

Only a vocabulary. Each repository implements the four branches itself.

## Dispatchers

```kotlin
interface PlatformDispatcher {
    val main: CoroutineDispatcher
    val io: CoroutineDispatcher
    val default: CoroutineDispatcher
    val unconfined: CoroutineDispatcher
}

expect object PlatformDispatchers : PlatformDispatcher
```

| | Android | iOS |
|---|---|---|
| `main` | `Dispatchers.Main` | GCD main queue |
| `io` | `Dispatchers.IO` | GCD global queue (default priority) |
| `default` | `Dispatchers.Default` | same GCD global queue as `io` |
| `unconfined` | `Dispatchers.Unconfined` | `Dispatchers.Unconfined` |

Reason it exists: `Dispatchers.IO` is `internal` on Kotlin/Native, so common code has no I/O
dispatcher of its own.

## Parcelize bridge

```kotlin
@Target(AnnotationTarget.CLASS)
@Retention(AnnotationRetention.BINARY)
annotation class CommonParcelize

expect interface CommonParcelable
// Android: actual typealias CommonParcelable = android.os.Parcelable
// iOS:     actual interface CommonParcelable   (empty marker)
```

Consumer modules must apply `org.jetbrains.kotlin.plugin.parcelize` and pass
`-P plugin:org.jetbrains.kotlin.parcelize:additionalAnnotation=dev.dexkot.mobile.core.foundation.CommonParcelize`
to the Android target's compilations (see SKILL.md).

## Android-only preferences helper

Package `dev.dexkot.mobile.core.foundation.data.preferences` (androidMain only):

```kotlin
fun SharedPreferences.Editor.save()          // commit(); throws SavePreferencesException if it returns false
class SavePreferencesException : RuntimeException("Error while saving preferences.")
```

For new multiplatform code prefer `core-preferences` (`Settings`), described in `dexkot-data-layer`.
