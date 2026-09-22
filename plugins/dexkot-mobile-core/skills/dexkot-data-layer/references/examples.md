# Data layer examples

Longer, end-to-end examples for the `dexkot-data-layer` skill. Domain: `Project` synced from a
server and cached in SQLite.

## Contents

1. [Domain](#1-domain)
2. [Remote data source with ErrorMapper](#2-remote-data-source-with-errormapper)
3. [Local cache data source with NoData](#3-local-cache-data-source-with-nodata)
4. [Orchestrating repository with CacheFetchPolicy](#4-orchestrating-repository-with-cachefetchpolicy)
5. [Use cases and Koin module](#5-use-cases-and-koin-module)
6. [Testing with a fake ITransactionFactory](#6-testing-with-a-fake-itransactionfactory)

## 1. Domain

```kotlin
package com.example.projects.domain.model

import dev.dexkot.mobile.core.foundation.CommonParcelable
import dev.dexkot.mobile.core.foundation.CommonParcelize

@CommonParcelize
data class Project(val id: Long, val title: String, val archived: Boolean) : CommonParcelable
```

```kotlin
package com.example.projects.domain.repository

internal interface IProjectRepository {
    suspend fun getProjects(policy: CacheFetchPolicy = CacheFetchPolicy.FETCHED_OR_CACHED): ResultOf<List<Project>>
}
```

## 2. Remote data source with ErrorMapper

```kotlin
package com.example.projects.data.datasource.remote

import dev.dexkot.mobile.core.foundation.coroutines.PlatformDispatchers
import dev.dexkot.mobile.core.foundation.errors.mapper.ErrorMapper
import dev.dexkot.mobile.core.foundation.result.ResultOf
import kotlinx.coroutines.withContext
import kotlin.coroutines.cancellation.CancellationException

/** Your HTTP client wrapper; throws on failure. */
interface ProjectApi {
    suspend fun getProjects(): List<ProjectDto>
}

data class ProjectDto(val id: Long, val title: String, val archived: Boolean) {
    fun toDomain() = Project(id = id, title = title, archived = archived)
}

internal interface IProjectRemoteDataSource {
    suspend fun fetchProjects(): ResultOf<List<Project>>
}

internal class ProjectRemoteDataSource(
    private val api: ProjectApi,
    private val errorMapper: ErrorMapper
) : IProjectRemoteDataSource {

    override suspend fun fetchProjects(): ResultOf<List<Project>> {
        return withContext(PlatformDispatchers.io) {
            try {
                ResultOf.Success(api.getProjects().map { it.toDomain() })
            } catch (exception: CancellationException) {
                throw exception                       // keep structured cancellation working
            } catch (exception: Exception) {
                errorMapper.handle(exception)         // mapped error, or UnexpectedError
            }
        }
    }
}
```

The mapper is injected as `ProjectApiErrorMapper() + TransportErrorMapper()` (specific first); see
the `dexkot-results-errors` skill.

## 3. Local cache data source with NoData

A cache that was never filled and a cache that is legitimately empty look the same in SQL. A flag
in a key-value store tells them apart, so `CACHED_OR_FETCHED` knows when to fetch.

```kotlin
package com.example.projects.data.datasource.local

import app.cash.sqldelight.async.coroutines.awaitAsList
import com.russhwolf.settings.Settings
import dev.dexkot.mobile.core.database.transaction.ITransactionFactory
import dev.dexkot.mobile.core.foundation.errors.ErrorEntity
import dev.dexkot.mobile.core.foundation.errors.deviceError
import dev.dexkot.mobile.core.foundation.result.ResultOf
import com.example.projects.data.datasource.local.model.ProjectDb.toDomain

internal interface IProjectLocalDataSource {
    suspend fun getProjects(): ResultOf<List<Project>>
    suspend fun replaceAll(projects: List<Project>): ResultOf<Unit>
}

internal class ProjectLocalDataSource(
    private val transactionFactory: ITransactionFactory,
    private val projectQueries: ProjectEntityQueries,
    private val settings: Settings
) : IProjectLocalDataSource {

    override suspend fun getProjects(): ResultOf<List<Project>> {
        if (!settings.getBoolean(POPULATED_KEY, false)) return ResultOf.Failure(ErrorEntity.NoData())
        return transactionFactory.withTransaction {
            try {
                ResultOf.Success(projectQueries.getAll().awaitAsList().map { it.toDomain() })
            } catch (exception: Exception) {
                ResultOf.Failure(deviceError(exception))
            }
        }
    }

    override suspend fun replaceAll(projects: List<Project>): ResultOf<Unit> {
        val result = transactionFactory.withTransaction {
            try {
                projectQueries.deleteAll()
                projects.forEach { projectQueries.insert(id = it.id, title = it.title, archived = if (it.archived) 1L else 0L) }
                ResultOf.Success(Unit)
            } catch (exception: Exception) {
                rollback(ResultOf.Failure<Unit>(deviceError(exception)))   // never leave a half-replaced table
            }
        }
        // Only mark the cache as populated once the rows are committed.
        if (result is ResultOf.Success) settings.putBoolean(POPULATED_KEY, true)
        return result
    }

    private companion object {
        const val POPULATED_KEY = "projects_populated"
    }
}
```

```kotlin
package com.example.projects.data.datasource.local.model

internal object ProjectDb {
    fun ProjectEntity.toDomain(): Project = Project(id = id, title = title, archived = archived == 1L)
}
```

## 4. Orchestrating repository with CacheFetchPolicy

```kotlin
package com.example.projects.data.repository

import dev.dexkot.mobile.core.foundation.data.policies.CacheFetchPolicy
import dev.dexkot.mobile.core.foundation.result.ResultOf

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
        if (fresh is ResultOf.Success) {
            // The fetched data is still valid if caching fails; the local source already reported it.
            local.replaceAll(fresh.data)
        }
        return fresh
    }
}
```

This is orchestration, not business logic: which source to ask and when to refresh the cache.
Filtering archived projects or enforcing limits belongs in a use case.

## 5. Use cases and Koin module

```kotlin
package com.example.projects.usecase

import dev.dexkot.mobile.core.foundation.data.policies.CacheFetchPolicy
import dev.dexkot.mobile.core.foundation.result.ResultOf
import dev.dexkot.mobile.core.foundation.result.map

interface IGetActiveProjectsUseCase {
    suspend operator fun invoke(forceRefresh: Boolean = false): ResultOf<List<Project>>
}

internal class GetActiveProjectsUseCase(
    private val projectRepository: IProjectRepository
) : IGetActiveProjectsUseCase {

    override suspend operator fun invoke(forceRefresh: Boolean): ResultOf<List<Project>> {
        val policy = if (forceRefresh) CacheFetchPolicy.FETCH_CURRENT else CacheFetchPolicy.CACHED_OR_FETCHED
        return projectRepository.getProjects(policy).map { projects -> projects.filterNot { it.archived } }
    }
}
```

```kotlin
package com.example.projects.di

import com.russhwolf.settings.Settings
import dev.dexkot.mobile.core.foundation.errors.mapper.ErrorMapper
import dev.dexkot.mobile.core.preferences.di.settingsFactoryModule
import org.koin.dsl.module

private const val PROJECTS_STORE = "com.example.projects"

val projectsModule = module {
    includes(settingsFactoryModule, projectsDatabaseModule)

    single<ErrorMapper>(projectsErrorMapper()) { ProjectApiErrorMapper() + TransportErrorMapper() }

    single<IProjectRemoteDataSource> {
        ProjectRemoteDataSource(api = get(), errorMapper = get(projectsErrorMapper()))
    }
    single<IProjectLocalDataSource> {
        val database: ProjectsDatabase = get()
        ProjectLocalDataSource(
            transactionFactory = get(projectsTransaction()),
            projectQueries = database.projectEntityQueries,
            settings = get<Settings.Factory>().create(PROJECTS_STORE)   // created once, inside a single
        )
    }
    factory<IProjectRepository> { ProjectRepository(local = get(), remote = get()) }

    factory<IGetActiveProjectsUseCase> { GetActiveProjectsUseCase(projectRepository = get()) }
    // With @AutoLogUseCase on the interface: GetActiveProjectsUseCase(get()).withLogging()
}

internal fun projectsErrorMapper() = org.koin.core.qualifier.named("projects_errorMapper")
```

Data sources are `single` (they may hold connections, stores or caches); repositories and use cases
are stateless `factory`. The ViewModel receives `IGetActiveProjectsUseCase` and converts its
`ResultOf` to `Resource` (see `dexkot-viewmodel`).

## 6. Testing with a fake ITransactionFactory

Use cases depend on `ITransactionFactory`, so their control flow can be tested without SQLite:

```kotlin
import dev.dexkot.mobile.core.database.transaction.ITransaction
import dev.dexkot.mobile.core.database.transaction.ITransactionFactory
import dev.dexkot.mobile.core.foundation.result.ResultOf
import dev.dexkot.mobile.core.foundation.result.map

/** Runs blocks inline and turns rollback() into a returned failure. Does not undo anything. */
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

Assert with `is` (results have no `equals`): `assertTrue(result is ResultOf.Failure && result.error is TaskNotFound)`
and `assertEquals(1, fakeTransactions.rolledBack)`. For the real commit/rollback behaviour, test data
sources against a temp-file database (`JdbcSqliteDriver("jdbc:sqlite:/path/to/tmp.db")` from
`app.cash.sqldelight:sqlite-driver` on the JVM) with a real `TransactionFactory`. Prefer a file over
`jdbc:sqlite:` in-memory: the transaction runs on its own thread, and a plain in-memory database is
private to one connection, so concurrent or cross-thread tests would not see the same data.
