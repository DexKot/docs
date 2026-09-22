# Data layer

How to structure data access in an app built on DexKot mobile-core: data sources, repositories and
use cases; SQLite transactions with `core-database` and `core-database-sqldelight`; and key-value
stores with `core-preferences`. Results and errors are covered in
[Results and errors](results-and-errors.md).

- [Architecture](#architecture)
- [Installation](#installation)
- [Setting up a SQLDelight database](#setting-up-a-sqldelight-database)
- [Data sources](#data-sources)
- [DB mappers](#db-mappers)
- [Repositories](#repositories)
- [Caching repositories](#caching-repositories)
- [Use cases](#use-cases)
- [Transactions](#transactions)
- [Atomic operations across repositories](#atomic-operations-across-repositories)
- [Key-value stores](#key-value-stores)
- [Testing](#testing)
- [Pitfalls](#pitfalls)
- [API reference](#api-reference)

## Architecture

```
ViewModel ──► UseCase ──► Repository ──► DataSource ──► SQLite / key-value store / HTTP
```

| Layer | Responsibility | Visibility |
|---|---|---|
| DataSource | Talk to one backend; catch every exception; map storage ↔ domain models | `internal` interface + `internal` class |
| Repository | The domain's access point for an entity; delegate to one data source, or orchestrate several | `internal` interface (domain) + `internal` class (data) |
| UseCase | One domain operation: business rules, composition, transactions across repositories | public interface + `internal` class |

Everything returns `ResultOf`. ViewModels only depend on use case interfaces. That gives each rule
one home (the use case), keeps storage details out of the domain, and lets a module expose a small
public API — its use case interfaces — while the data layer stays `internal`.

A package layout that works well:

```
tasks/
├── domain/   model/  error/  repository/
├── data/     datasource/local/  datasource/local/model/  datasource/remote/  repository/
├── usecase/  <operation>/I<Operation>UseCase.kt + <Operation>UseCase.kt
└── di/
```

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
```

| Artifact | Contains | Add it to |
|---|---|---|
| `dev.dexkot.mobile:core-database:0.27.0` | `ITransactionFactory`, `ITransaction` | domain and data code |
| `dev.dexkot.mobile:core-database-sqldelight:0.27.0` | `TransactionFactory` for SQLDelight | only the module that builds the database |
| `dev.dexkot.mobile:core-preferences:0.27.0` | `settingsFactoryModule` (Koin) | modules with key-value stores |

All are Kotlin Multiplatform. Android: published. iOS: build from source (`publishToMavenLocal`).
`core-database` brings `core-foundation`; `core-preferences` brings `com.russhwolf:multiplatform-settings`
1.3.0 and uses Koin. The SQLDelight integration is built against SQLDelight 2.2.1.

Splitting the interface (`core-database`) from the implementation (`core-database-sqldelight`)
means use cases and data sources can be unit-tested with a fake `ITransactionFactory` and do not
depend on SQLDelight types.

## Setting up a SQLDelight database

### Gradle

```kotlin
import org.jetbrains.kotlin.gradle.plugin.mpp.KotlinNativeTarget

plugins {
    // your KMP + Android library plugins
    id("app.cash.sqldelight") version "2.2.1"
}

sqldelight {
    databases {
        create("TasksSqlDatabase") {
            packageName.set("com.example.tasks.data.database")
            generateAsync.set(true)
        }
    }
}

kotlin {
    targets.withType<KotlinNativeTarget>().configureEach {
        binaries.all { linkerOpts("-lsqlite3") }   // NativeSqliteDriver needs the system SQLite linked
    }
    sourceSets {
        commonMain.dependencies {
            implementation("dev.dexkot.mobile:core-database-sqldelight:0.27.0")
            implementation("app.cash.sqldelight:runtime:2.2.1")
            implementation("app.cash.sqldelight:async-extensions:2.2.1")
            implementation("app.cash.sqldelight:coroutines-extensions:2.2.1")  // optional: Flow queries
        }
        androidMain.dependencies { implementation("app.cash.sqldelight:android-driver:2.2.1") }
        iosMain.dependencies { implementation("app.cash.sqldelight:native-driver:2.2.1") }
    }
}
```

`generateAsync = true` is required. `TransactionFactory` takes a `SuspendingTransacter`, which the
generated database only implements in async mode. As a consequence, queries are read with
`awaitAsOne()` / `awaitAsOneOrNull()` / `awaitAsList()` and mutators are `suspend` functions.

### Schema

`src/commonMain/sqldelight/com/example/tasks/data/database/TaskEntity.sq`:

```sql
CREATE TABLE TaskEntity (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    project_id INTEGER NOT NULL,
    title TEXT NOT NULL,
    done INTEGER NOT NULL DEFAULT 0
);

get:
SELECT * FROM TaskEntity WHERE id = ?;

getByProject:
SELECT * FROM TaskEntity WHERE project_id = ? ORDER BY id;

insert:
INSERT INTO TaskEntity(project_id, title, done) VALUES (?, ?, ?);

lastInsertedId:
SELECT last_insert_rowid();
```

Naming the table `TaskEntity` keeps the generated row class from clashing with the domain `Task`.

### Database object

```kotlin
package com.example.tasks.data.database

import app.cash.sqldelight.db.SqlDriver
import dev.dexkot.mobile.core.database.sqldelight.TransactionFactory
import dev.dexkot.mobile.core.database.transaction.ITransactionFactory

class TasksDatabase internal constructor(
    private val database: TasksSqlDatabase,
    private val transactionFactory: ITransactionFactory
) : TasksSqlDatabase by database, ITransactionFactory by transactionFactory

internal fun createTasksDatabase(driver: SqlDriver): TasksDatabase {
    val database = TasksSqlDatabase(driver)
    return TasksDatabase(
        database = database,
        transactionFactory = TransactionFactory(sqlDriver = driver, transacter = database)
    )
}
```

One object gives data sources both the generated queries and the transactions, and guarantees both
use the same driver.

### Drivers

The Android and native drivers expect a synchronous schema, so wrap the async one with
`.synchronous()`:

```kotlin
// androidMain
import android.content.Context
import androidx.sqlite.db.SupportSQLiteDatabase
import app.cash.sqldelight.async.coroutines.synchronous
import app.cash.sqldelight.db.SqlDriver
import app.cash.sqldelight.driver.android.AndroidSqliteDriver

internal fun createTasksDriver(context: Context): SqlDriver {
    val schema = TasksSqlDatabase.Schema.synchronous()
    return AndroidSqliteDriver(
        schema = schema,
        context = context,
        name = "tasks.db",
        callback = object : AndroidSqliteDriver.Callback(schema) {
            override fun onConfigure(db: SupportSQLiteDatabase) {
                db.enableWriteAheadLogging()
                db.setForeignKeyConstraintsEnabled(true)
            }
        }
    )
}
```

```kotlin
// iosMain
import app.cash.sqldelight.async.coroutines.synchronous
import app.cash.sqldelight.db.SqlDriver
import app.cash.sqldelight.driver.native.NativeSqliteDriver

internal fun createTasksDriver(): SqlDriver = NativeSqliteDriver(
    schema = TasksSqlDatabase.Schema.synchronous(),
    name = "tasks.db",
    onConfiguration = { config ->
        config.copy(extendedConfig = config.extendedConfig.copy(foreignKeyConstraints = true))
    }
)
```

Write-ahead logging matters because `TransactionFactory` runs independent transactions on separate
threads at the same time: with WAL, reads do not wait for the writer. The native driver uses WAL by
default. Foreign keys are off in SQLite unless enabled per connection.

### Koin

```kotlin
import org.koin.core.module.Module
import org.koin.core.qualifier.named
import org.koin.dsl.module

fun tasksTransaction() = named("tasks_transactionFactory")    // public: other modules may join
internal fun tasksDriver() = named("tasks_sqlDriver")

internal expect val tasksPlatformModule: Module              // single<SqlDriver>(tasksDriver()) { createTasksDriver(...) }

val tasksDatabaseModule = module {
    includes(tasksPlatformModule)
    single<TasksDatabase> { createTasksDatabase(driver = get(tasksDriver())) }
    single<ITransactionFactory>(tasksTransaction()) { get<TasksDatabase>() }
}
```

Qualify the `ITransactionFactory`: an app with two databases has two factories, and an unqualified
binding would resolve to whichever was registered last. Keep it a `single`: each
`TransactionFactory` owns a pool of threads.

## Data sources

A data source wraps exactly one backend and is the only place where exceptions are caught.

```kotlin
package com.example.tasks.data.datasource.local

import app.cash.sqldelight.async.coroutines.awaitAsList
import app.cash.sqldelight.async.coroutines.awaitAsOne
import app.cash.sqldelight.async.coroutines.awaitAsOneOrNull
import com.example.tasks.data.database.TaskEntityQueries
import com.example.tasks.data.datasource.local.model.TaskDb.toDomain
import com.example.tasks.domain.error.TaskNotFound
import com.example.tasks.domain.model.Task
import dev.dexkot.mobile.core.database.transaction.ITransactionFactory
import dev.dexkot.mobile.core.foundation.errors.deviceError
import dev.dexkot.mobile.core.foundation.result.ResultOf

internal interface ITaskLocalDataSource {
    suspend fun getTask(id: Long): ResultOf<Task>
    suspend fun getTasks(projectId: Long): ResultOf<List<Task>>
    suspend fun addTask(projectId: Long, title: String): ResultOf<Task>
}

internal class TaskLocalDataSource(
    private val transactionFactory: ITransactionFactory,
    private val taskQueries: TaskEntityQueries
) : ITaskLocalDataSource {

    override suspend fun getTask(id: Long): ResultOf<Task> {
        return transactionFactory.withTransaction {
            try {
                val task = taskQueries.get(id).awaitAsOneOrNull()?.toDomain()
                if (task == null) ResultOf.Failure(TaskNotFound(id = id)) else ResultOf.Success(task)
            } catch (exception: Exception) {
                ResultOf.Failure(deviceError(exception))
            }
        }
    }

    override suspend fun getTasks(projectId: Long): ResultOf<List<Task>> {
        return transactionFactory.withTransaction {
            try {
                ResultOf.Success(taskQueries.getByProject(projectId).awaitAsList().map { it.toDomain() })
            } catch (exception: Exception) {
                ResultOf.Failure(deviceError(exception))
            }
        }
    }

    override suspend fun addTask(projectId: Long, title: String): ResultOf<Task> {
        return transactionFactory.withTransaction {
            try {
                taskQueries.insert(project_id = projectId, title = title, done = 0L)
                val id = taskQueries.lastInsertedId().awaitAsOne()
                ResultOf.Success(Task(id = id, projectId = projectId, title = title, done = false))
            } catch (exception: Exception) {
                rollback(ResultOf.Failure<Task>(deviceError(exception)))
            }
        }
    }
}
```

The choices in this code:

- **Every call is wrapped in `withTransaction`, reads included.** Alone, it is a short transaction.
  Called from a use case that already opened a transaction, it joins it. The same method therefore
  works standalone and inside an atomic operation.
- **Writes fail with `rollback(...)`.** Returning a `Failure` commits whatever was written before
  the error; `rollback` reverts it and still returns your failure.
- **Reading the new id inside the same transaction.** `last_insert_rowid()` is per connection; the
  transaction keeps both statements on one.
- **No `withContext(PlatformDispatchers.io)`** around pure DB work — `withTransaction` already runs
  the block on its own thread. Use `PlatformDispatchers.io` for file and network work.
- **Domain models in, domain models out.** Row classes never leave the data source.

A remote data source has the same shape without the transaction:

```kotlin
internal class ProjectRemoteDataSource(
    private val api: ProjectApi,
    private val errorMapper: ErrorMapper
) : IProjectRemoteDataSource {

    override suspend fun fetchProjects(): ResultOf<List<Project>> {
        return withContext(PlatformDispatchers.io) {
            try {
                ResultOf.Success(api.getProjects().map { it.toDomain() })
            } catch (exception: CancellationException) {
                throw exception
            } catch (exception: Exception) {
                errorMapper.handle(exception)
            }
        }
    }
}
```

## DB mappers

```kotlin
package com.example.tasks.data.datasource.local.model

import com.example.tasks.data.database.TaskEntity
import com.example.tasks.domain.model.Task

internal object TaskDb {
    fun TaskEntity.toDomain(): Task = Task(id = id, projectId = project_id, title = title, done = done == 1L)
}
```

One `object` per entity with `toDomain()` extensions keeps column names, integer booleans and
string-encoded enums in one place, so a schema change touches one file. Store enums through
explicit string constants rather than `enum.name`, so renaming a Kotlin constant cannot break stored
rows.

## Repositories

A repository implements a domain interface. With a single data source it simply delegates:

```kotlin
internal interface ITaskRepository {
    suspend fun getTask(id: Long): ResultOf<Task>
    suspend fun getTasks(projectId: Long): ResultOf<List<Task>>
    suspend fun addTask(projectId: Long, title: String): ResultOf<Task>
}

internal class TaskRepository(
    private val localDataSource: ITaskLocalDataSource
) : ITaskRepository {
    override suspend fun getTask(id: Long) = localDataSource.getTask(id)
    override suspend fun getTasks(projectId: Long) = localDataSource.getTasks(projectId)
    override suspend fun addTask(projectId: Long, title: String) = localDataSource.addTask(projectId, title)
}
```

The thin layer is still worth having: the domain depends on `ITaskRepository`, not on SQLite, and a
second source (a server) can be added later without touching any use case.

Repositories do not hold business rules. "A project may have at most 50 tasks" goes in a use case,
where it can be tested once and reused.

## Caching repositories

With a local cache and a remote source, the repository decides which to use, driven by
`CacheFetchPolicy`:

```kotlin
internal class ProjectRepository(
    private val local: IProjectLocalDataSource,
    private val remote: IProjectRemoteDataSource
) : IProjectRepository {

    override suspend fun getProjects(policy: CacheFetchPolicy): ResultOf<List<Project>> = when (policy) {
        CacheFetchPolicy.CACHE_ONLY -> local.getProjects()
        CacheFetchPolicy.FETCH_CURRENT -> fetchAndStore()
        CacheFetchPolicy.CACHED_OR_FETCHED -> when (val cached = local.getProjects()) {
            is ResultOf.Success -> cached
            is ResultOf.Failure -> fetchAndStore()
        }
        CacheFetchPolicy.FETCHED_OR_CACHED -> when (val fresh = fetchAndStore()) {
            is ResultOf.Success -> fresh
            is ResultOf.Failure -> local.getProjects()
        }
    }

    private suspend fun fetchAndStore(): ResultOf<List<Project>> {
        val fresh = remote.fetchProjects()
        if (fresh is ResultOf.Success) local.replaceAll(fresh.data)   // fresh data stays valid if caching fails
        return fresh
    }
}
```

For `CACHED_OR_FETCHED` to fetch on a cold start, the local source must *fail* when the cache has
never been filled. An empty table returns an empty list, which is a valid cached answer. A common
fix is a "populated" flag in a key-value store: return `ErrorEntity.NoData()` until the first
successful `replaceAll`, and set the flag only after that transaction commits.

## Use cases

```kotlin
interface IGetProjectTasksUseCase {
    suspend operator fun invoke(projectId: Long): ResultOf<List<Task>>
}

internal class GetProjectTasksUseCase(
    private val taskRepository: ITaskRepository
) : IGetProjectTasksUseCase {
    override suspend operator fun invoke(projectId: Long): ResultOf<List<Task>> =
        taskRepository.getTasks(projectId)
}
```

```kotlin
factory<IGetProjectTasksUseCase> { GetProjectTasksUseCase(taskRepository = get()) }
```

- Name them `I{Verb}{Entity}UseCase`: `IGetTaskUseCase`, `ICreateProjectUseCase`,
  `IDuplicateProjectUseCase`, `IObserveTasksUseCase`.
- One-shot operations are `suspend operator fun invoke(...)` returning `ResultOf`; observations
  return a `Flow`.
- Use cases may call other use cases.
- Register as `factory` — they are stateless.
- ViewModels depend on use case interfaces only, never on repositories.

For automatic logging of every use case call, annotate the interface with `@AutoLogUseCase` and
register the implementation wrapped with the generated `.withLogging()`; see the logging guide.

## Transactions

`ITransactionFactory.withTransaction { }` runs a block in a database transaction and returns its
`ResultOf`. Inside the block you have an `ITransaction` receiver with `rollback(failure)` and
`rollBackIfFailure(result)`.

| What happens in the block | Outcome |
|---|---|
| returns `Success` | committed; `Success` returned |
| returns `Failure` | **committed**; `Failure` returned |
| calls `rollback(failure)` | rolled back; `failure` returned |
| calls `rollBackIfFailure(result)` with a failure | rolled back; that failure returned |
| throws anything (including `CancellationException`) | rolled back; `Failure(UnexpectedError(e))` returned |
| calls `withTransaction` again | joins the outer transaction through a `SAVEPOINT` |
| nested block rolls back or throws | only the savepoint is reverted; the outer block receives a `Failure` and continues |

How it works, and why it matters for your code:

- Every top-level transaction runs on its own single-thread dispatcher taken from a pool. SQLite
  binds a transaction to one connection, and the drivers bind connections to threads; confining the
  whole block to one thread keeps every query in the transaction. Do not `withContext` to another
  dispatcher inside the block — those queries would run outside the transaction and can deadlock
  waiting for its lock.
- Nesting is detected from the coroutine context, not from the factory. A nested call with *any*
  factory joins the current transaction. Do not nest transactions of two different databases; run
  them one after the other.
- `rollback` returns `Nothing`. When it is the only exit from the block, Kotlin cannot infer the
  result type: write `withTransaction<Task> { ... }`.
- `withTransaction` never throws for problems inside the block, cancellation included: a
  cancellation raised inside becomes a `Failure`. Nested callers should treat that failure like any
  other and pass it to `rollBackIfFailure`.
- Keep transactions short. A network call inside the block holds the write lock for its whole duration.

## Atomic operations across repositories

A use case makes several repository calls atomic by opening the outer transaction itself. It gets
the database's qualified factory from DI (`get(tasksTransaction())`).

```kotlin
internal class DuplicateProjectUseCase(
    private val transactionFactory: ITransactionFactory,
    private val projectRepository: IProjectRepository,
    private val taskRepository: ITaskRepository
) : IDuplicateProjectUseCase {

    override suspend operator fun invoke(projectId: Long): ResultOf<Project> {
        return transactionFactory.withTransaction {
            val original = rollBackIfFailure(projectRepository.getProject(projectId))
            val copy = rollBackIfFailure(projectRepository.addProject(title = "${original.title} (copy)"))
            val tasks = rollBackIfFailure(taskRepository.getTasks(projectId))
            tasks.forEach { task ->
                rollBackIfFailure(taskRepository.addTask(projectId = copy.id, title = task.title))
            }
            ResultOf.Success(copy)
        }
    }
}
```

Each repository call reaches a data source that calls `withTransaction`, which becomes a savepoint
inside this transaction. `rollBackIfFailure` unwraps successes and, on the first failure, rolls
everything back and returns that failure from `withTransaction`. Forgetting it on one call means a
failed step is silently skipped while the rest commits.

Put irreversible side effects after the commit:

```kotlin
val result = transactionFactory.withTransaction { rollBackIfFailure(projectRepository.deleteProject(id)); successOf() }
if (result is ResultOf.Success) attachmentRepository.deleteFiles(projectId = id)
```

A rollback restores rows, not deleted files or sent requests.

## Key-value stores

`core-preferences` provides `settingsFactoryModule`, which binds a multiplatform
`Settings.Factory` (SharedPreferences on Android, NSUserDefaults on iOS). Each feature creates its
own named store:

```kotlin
import com.russhwolf.settings.Settings
import dev.dexkot.mobile.core.preferences.di.settingsFactoryModule
import org.koin.core.qualifier.named
import org.koin.dsl.module

private const val PROJECT_STORE = "com.example.projects"

val projectPreferencesModule = module {
    includes(settingsFactoryModule)

    single<IProjectPreferencesDataSource> {
        ProjectPreferencesDataSource(settings = get<Settings.Factory>().create(PROJECT_STORE))
    }
    // or expose the store itself under a qualifier:
    single<Settings>(named("projects_settings")) { get<Settings.Factory>().create(PROJECT_STORE) }
}
```

```kotlin
internal interface IProjectPreferencesDataSource {
    suspend fun getLastOpenedProjectId(): ResultOf<Long>
    suspend fun setLastOpenedProjectId(id: Long): ResultOf<Unit>
}

internal class ProjectPreferencesDataSource(
    private val settings: Settings
) : IProjectPreferencesDataSource {

    override suspend fun getLastOpenedProjectId(): ResultOf<Long> = try {
        settings.getLongOrNull(LAST_OPENED_KEY)?.let { ResultOf.Success(it) }
            ?: ResultOf.Failure(ErrorEntity.NoData())
    } catch (exception: Exception) {
        ResultOf.Failure(deviceError(exception))
    }

    override suspend fun setLastOpenedProjectId(id: Long): ResultOf<Unit> = try {
        settings.putLong(LAST_OPENED_KEY, id)
        successOf()
    } catch (exception: Exception) {
        ResultOf.Failure(deviceError(exception))
    }

    private companion object {
        const val LAST_OPENED_KEY = "last_opened_project_id"
    }
}
```

- Including `settingsFactoryModule` from many modules is safe: it is a single `val` and Koin loads it once.
- `create(name)` returns a new `Settings` each time — call it once, inside a `single`.
- The name is the SharedPreferences file / NSUserDefaults suite. Changing it later orphans existing
  data, so treat it as a stable identifier, and use one store per module so keys never collide.
- On Android, Koin needs `androidContext(...)`.

## Testing

Because domain code depends on `ITransactionFactory`, use cases can be tested with a fake that runs
blocks inline:

```kotlin
import dev.dexkot.mobile.core.database.transaction.ITransaction
import dev.dexkot.mobile.core.database.transaction.ITransactionFactory
import dev.dexkot.mobile.core.foundation.result.ResultOf
import dev.dexkot.mobile.core.foundation.result.map

class FakeTransactionFactory : ITransactionFactory {

    var rolledBack = 0
        private set

    override suspend fun <Out> withTransaction(
        block: suspend ITransaction<*>.() -> ResultOf<Out>
    ): ResultOf<Out> {
        val transaction = object : ITransaction<Out> {
            override suspend fun <O> execute(block: suspend ITransaction<*>.() -> ResultOf<O>): ResultOf<O> = block()
            override suspend fun <O> innerTransaction(block: suspend ITransaction<*>.() -> ResultOf<O>): ResultOf<O> = block()
            override suspend fun rollback(failure: ResultOf.Failure<*>): Nothing {
                rolledBack++
                throw FakeRollback(failure)
            }
        }
        return try {
            transaction.block()
        } catch (rollback: FakeRollback) {
            rollback.failure.map()
        }
    }

    private class FakeRollback(val failure: ResultOf.Failure<*>) : Exception()
}
```

It verifies control flow (which failure is returned, whether a rollback was requested), not SQL.
To test real commit/rollback, run data sources against a temporary file database
(`JdbcSqliteDriver` from `app.cash.sqldelight:sqlite-driver` on the JVM) with a real
`TransactionFactory`. Prefer a file to `jdbc:sqlite:` in-memory, which is private to a single
connection while transactions run on their own thread.

Remember that results have no `equals`: assert with `is`.

## Pitfalls

- **Returning `Failure` commits.** Use `rollback` / `rollBackIfFailure` when writes must be undone.
- **Inner failures stay inner** unless you pass them to `rollBackIfFailure`.
- **`rollback` needs a type** when it is the only exit: `withTransaction<Task> { ... }`.
- **Switching dispatchers inside a transaction** takes queries out of it and can deadlock.
- **Nesting across databases** joins the wrong transaction.
- **Network calls inside transactions** hold the write lock.
- **Cancellation raised inside a transaction is returned as a `Failure`**, not thrown from `withTransaction`.
- **Forgetting `generateAsync = true`** → `TransactionFactory` does not accept the database; forgetting
  `.synchronous()` → the driver does not accept the schema.
- **Creating `Settings` outside a `single`** or renaming a store name → multiple instances or lost data.
- **Empty cache counted as a hit** in `CACHED_OR_FETCHED` → return `NoData` until populated.

## API reference

```kotlin
// core-database — dev.dexkot.mobile.core.database.transaction
interface ITransactionFactory {
    suspend fun <Out> withTransaction(block: suspend ITransaction<*>.() -> ResultOf<Out>): ResultOf<Out>
}

interface ITransaction<Output> {
    suspend fun <Out> execute(block: suspend ITransaction<*>.() -> ResultOf<Out>): ResultOf<Out>
    suspend fun rollback(failure: ResultOf.Failure<*>): Nothing
    suspend fun <Data> rollBackIfFailure(result: ResultOf<Data>): Data
    suspend fun <Out> innerTransaction(block: suspend ITransaction<*>.() -> ResultOf<Out>): ResultOf<Out>
}

// core-database-sqldelight — dev.dexkot.mobile.core.database.sqldelight
class TransactionFactory(
    sqlDriver: app.cash.sqldelight.db.SqlDriver,
    transacter: app.cash.sqldelight.SuspendingTransacter
) : ITransactionFactory

// core-preferences — dev.dexkot.mobile.core.preferences.di
expect val settingsFactoryModule: org.koin.core.module.Module   // binds com.russhwolf.settings.Settings.Factory
```

`execute` and `innerTransaction` are used by `TransactionFactory` internally; application code calls
`withTransaction` (including for nesting), `rollback` and `rollBackIfFailure`.
