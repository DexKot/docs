# core-database, core-database-sqldelight, core-preferences API reference (0.27.0)

All three are Kotlin Multiplatform (Android: published; iOS: build from source).

## core-database — `dev.dexkot.mobile:core-database`

Package `dev.dexkot.mobile.core.database.transaction`. Depends on `core-foundation` (`api`).

```kotlin
interface ITransactionFactory {
    suspend fun <Out> withTransaction(
        block: suspend ITransaction<*>.() -> ResultOf<Out>
    ): ResultOf<Out>
}

interface ITransaction<Output> {
    suspend fun <Out> execute(block: suspend ITransaction<*>.() -> ResultOf<Out>): ResultOf<Out>
    suspend fun rollback(failure: ResultOf.Failure<*>): Nothing
    suspend fun <Data> rollBackIfFailure(result: ResultOf<Data>): Data   // default impl
    suspend fun <Out> innerTransaction(block: suspend ITransaction<*>.() -> ResultOf<Out>): ResultOf<Out>
}
```

In application code use `withTransaction`, and inside the block `rollback` and `rollBackIfFailure`.
`execute` and `innerTransaction` are the implementation hooks `TransactionFactory` uses; to nest,
just call `withTransaction` again.

`rollBackIfFailure(result)`: `Success` → returns `data`; `Failure` → `rollback(result)`.

## core-database-sqldelight — `dev.dexkot.mobile:core-database-sqldelight`

Package `dev.dexkot.mobile.core.database.sqldelight`. Exposes `core-database` (`api`). Built
against SQLDelight 2.2.1.

```kotlin
class TransactionFactory(
    sqlDriver: SqlDriver,                 // app.cash.sqldelight.db.SqlDriver
    transacter: SuspendingTransacter      // app.cash.sqldelight.SuspendingTransacter (generateAsync = true)
) : ITransactionFactory
```

Behaviour (from the implementation and its common test suite, run on JVM and iOS):

- **Top level**: takes a single-thread dispatcher from an internal pool (created on demand, reused,
  never closed), switches to it, and runs `transacter.transactionWithResult { }`.
- **Commit**: whenever the block returns normally — `Success` *or* `Failure`.
- **Rollback**: `rollback(failure)` / `rollBackIfFailure(failure)` → rolled back, returns `failure`
  re-typed to `ResultOf<Out>` (same `ErrorEntity`).
- **Exceptions**: any `Throwable` in the block (including `CancellationException`) → rolled back,
  returns `Failure(ErrorEntity.UnexpectedError(throwable))`. `withTransaction` never throws for
  failures inside the block.
- **Nesting**: detected with a coroutine-context element (inherited across `withContext`). A nested
  `withTransaction` runs on the outer transaction's thread inside `SAVEPOINT <random>`; on
  rollback/exception it executes `ROLLBACK TO` that savepoint and returns the failure to the outer
  block; the savepoint is always released. The outer transaction continues unless you call
  `rollBackIfFailure` on the inner result.
- **Detection is per coroutine, not per factory or database.** A `withTransaction` on another
  factory inside the block joins the current transaction.
- **Concurrency**: independent top-level transactions run in parallel on separate threads; SQLite
  serialises writers (use WAL; the tests use a busy timeout on JVM).

`dev.dexkot.mobile.core.database.sqldelight.pools.SuspendablePool<Type>(factory: () -> Type)` with
`suspend fun acquire(): Type` / `suspend fun release(instance: Type)` is public but an implementation
detail of `TransactionFactory`.

## core-preferences — `dev.dexkot.mobile:core-preferences`

Package `dev.dexkot.mobile.core.preferences.di`. Exposes `com.russhwolf:multiplatform-settings`
1.3.0 (`api`); uses Koin 4.1.x.

```kotlin
expect val settingsFactoryModule: org.koin.core.module.Module
// Android: single<Settings.Factory> { SharedPreferencesSettings.Factory(androidContext()) }
// iOS:     single<Settings.Factory> { NSUserDefaultsSettings.Factory() }
```

Usage:

```kotlin
val myModule = module {
    includes(settingsFactoryModule)
    single<Settings>(named("tasks_settings")) { get<Settings.Factory>().create("com.example.tasks") }
}
```

- `Settings.Factory.create(name)` builds a new `Settings` each call; the name is the
  SharedPreferences file (Android) or NSUserDefaults suite (iOS). Keep names stable and unique per
  module. `create(null)` would give the platform default store — avoid it in libraries.
- Android requires `androidContext(...)` in `startKoin`.
- Commonly used `Settings` calls: `getBoolean(key, default)`, `getLongOrNull(key)`,
  `getStringOrNull(key)`, `putBoolean/putLong/putString(key, value)`, `remove(key)`, `clear()`,
  `hasKey(key)`. See multiplatform-settings for the full API.
