# SQLDelight + TransactionFactory setup

End-to-end wiring of a SQLDelight database for DexKot's `TransactionFactory`, for a KMP module
`tasks` with package `com.example.tasks`. Android-only modules use the same steps minus the iOS parts.

## 1. Gradle

```kotlin
// tasks/build.gradle.kts
import org.jetbrains.kotlin.gradle.plugin.mpp.KotlinNativeTarget

plugins {
    // your KMP + Android library plugins
    id("app.cash.sqldelight") version "2.2.1"
}

sqldelight {
    databases {
        create("TasksSqlDatabase") {
            packageName.set("com.example.tasks.data.database")
            generateAsync.set(true)          // required: TransactionFactory needs a SuspendingTransacter
        }
    }
}

kotlin {
    // NativeSqliteDriver calls the system SQLite but does not link it. Linking libsqlite3 here keeps
    // test binaries (and frameworks that do not link it themselves) from failing with
    // "Undefined symbols: _sqlite3_open_v2 ...".
    targets.withType<KotlinNativeTarget>().configureEach {
        binaries.all { linkerOpts("-lsqlite3") }
    }

    sourceSets {
        commonMain.dependencies {
            implementation("dev.dexkot.mobile:core-database-sqldelight:0.27.0")
            implementation("app.cash.sqldelight:runtime:2.2.1")
            implementation("app.cash.sqldelight:async-extensions:2.2.1")    // awaitAsList(), synchronous()
            implementation("app.cash.sqldelight:coroutines-extensions:2.2.1") // asFlow(), mapToList() (optional)
            implementation("io.insert-koin:koin-core:<version>")
        }
        androidMain.dependencies {
            implementation("app.cash.sqldelight:android-driver:2.2.1")
        }
        iosMain.dependencies {
            implementation("app.cash.sqldelight:native-driver:2.2.1")
        }
    }
}
```

Why `generateAsync`: with it, the generated `TasksSqlDatabase` implements `SuspendingTransacter`
and every generated query is a `suspend` function or an async `Query`. `TransactionFactory`'s
constructor takes a `SuspendingTransacter`, so without async generation it does not compile.

## 2. Schema

`src/commonMain/sqldelight/com/example/tasks/data/database/TaskEntity.sq`:

```sql
CREATE TABLE TaskEntity (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    project_id INTEGER NOT NULL REFERENCES ProjectEntity(id) ON DELETE CASCADE,
    title TEXT NOT NULL,
    done INTEGER NOT NULL DEFAULT 0
);

get:
SELECT * FROM TaskEntity WHERE id = ?;

getByProject:
SELECT * FROM TaskEntity WHERE project_id = ? ORDER BY id;

insert:
INSERT INTO TaskEntity(project_id, title, done) VALUES (?, ?, ?);

setDone:
UPDATE TaskEntity SET done = ? WHERE id = ?;

lastInsertedId:
SELECT last_insert_rowid();
```

Naming the table `TaskEntity` generates a `TaskEntity` row class and `TaskEntityQueries`, which
avoids clashing with the domain `Task`. Read with `awaitAsOne()`, `awaitAsOneOrNull()`,
`awaitAsList()` from `app.cash.sqldelight.async.coroutines`; mutators such as `insert(...)` are
`suspend` functions.

## 3. Database wrapper and factory (commonMain)

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

The same `driver` must back both the generated database and the `TransactionFactory`: savepoints
are issued on `sqlDriver`, and the transaction on `transacter`.

## 4. Drivers

Drivers are platform-specific; the schema must be wrapped with `.synchronous()` because the
Android and native drivers expect a synchronous `SqlSchema` while async generation produces an
async one.

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
    onConfiguration = { configuration ->
        configuration.copy(
            extendedConfig = configuration.extendedConfig.copy(foreignKeyConstraints = true)
        )
    }
)
```

Why WAL: `TransactionFactory` runs each top-level transaction on its own thread, so several may be
in flight. In WAL mode readers do not block on the single writer, and concurrent writers queue on
the write lock instead of failing. SQLite enforces foreign keys only when asked, per connection —
hence the explicit setting. The native driver (SQLiter) uses WAL by default; raise
`maxReaderConnections` if you run many concurrent reads.

## 5. Koin wiring

```kotlin
// commonMain
package com.example.tasks.di

import app.cash.sqldelight.db.SqlDriver
import com.example.tasks.data.database.TasksDatabase
import com.example.tasks.data.database.createTasksDatabase
import dev.dexkot.mobile.core.database.transaction.ITransactionFactory
import org.koin.core.module.Module
import org.koin.core.qualifier.named
import org.koin.dsl.module

/** Public: other modules resolve this database's transactions with `get(tasksTransaction())`. */
fun tasksTransaction() = named("tasks_transactionFactory")

internal fun tasksDriver() = named("tasks_sqlDriver")

internal expect val tasksPlatformModule: Module   // binds SqlDriver under tasksDriver()

val tasksDatabaseModule = module {
    includes(tasksPlatformModule)
    single<TasksDatabase> { createTasksDatabase(driver = get<SqlDriver>(tasksDriver())) }
    single<ITransactionFactory>(tasksTransaction()) { get<TasksDatabase>() }
}
```

```kotlin
// androidMain
import org.koin.android.ext.koin.androidContext

internal actual val tasksPlatformModule: Module = module {
    single<SqlDriver>(tasksDriver()) { createTasksDriver(androidContext()) }
}

// iosMain
internal actual val tasksPlatformModule: Module = module {
    single<SqlDriver>(tasksDriver()) { createTasksDriver() }
}
```

Data sources get the queries and the qualified factory:

```kotlin
single<ITaskLocalDataSource> {
    val database: TasksDatabase = get()
    TaskLocalDataSource(
        transactionFactory = get(tasksTransaction()),
        taskQueries = database.taskEntityQueries
    )
}
```

Why the qualifier: an app often has several databases, each with its own `ITransactionFactory`.
An unqualified binding would silently resolve to whichever was registered last. The qualifier
function is public so a use case in another module can join this database's transactions.
Why a `single`: each `TransactionFactory` owns a pool of transaction threads that are created on
demand and never closed; extra instances only multiply those threads.
More on module structure in the `dexkot-di-koin` skill.

## 6. Observing queries

```kotlin
import app.cash.sqldelight.coroutines.asFlow
import app.cash.sqldelight.coroutines.mapToList
import dev.dexkot.mobile.core.foundation.coroutines.PlatformDispatchers

fun observeTasks(projectId: Long): Flow<List<Task>> =
    taskQueries.getByProject(projectId)
        .asFlow()
        .mapToList(PlatformDispatchers.io)
        .map { rows -> rows.map { it.toDomain() } }
```

Flows re-emit after every committed write to the table. Wrap them in `ResultOf` or `.catch { }`
in the data source if the observation must never throw into the collector.

## 7. Migrations

Schema migrations are SQLDelight's (`.sqm` files, `Schema.version`), not DexKot's. Nothing in
`TransactionFactory` changes for them.
