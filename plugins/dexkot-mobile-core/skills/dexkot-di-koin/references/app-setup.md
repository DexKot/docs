# A complete app graph

An Android Notes app using every DexKot `-di-koin` module: logging, remote config, WorkManager
background sync, permissions, navigation and two screens. Package `com.example.notes`.

## Contents
- [Module tree](#module-tree)
- [Application](#application)
- [Core, logging, remote config](#core-logging-remote-config)
- [Background work (WorkManager)](#background-work)
- [Data and use cases](#data-and-use-cases)
- [UI: navigation and screens](#ui)
- [Manifest](#manifest)
- [iOS / shared code](#ios)

## Module tree

```
AppModule
├── CoreModule            settingsFactoryModule, Json
├── LoggingModule         loggerModule(...) + single<ILogger>
├── RemoteConfigModule    remoteConfigModule() + unqualified IRemoteConfigHandler
├── WorkModule            workManagerWorkModule + worker + WorkerClassProvider + providers
├── NotesModule           NoteModule (data source, repository), NotesUseCaseModule
└── UiModule              uiModule(...), permissionModule(...), CommonsNavigationModule, screen modules
```

## Application

```kotlin
package com.example.notes

import android.app.Application
import com.google.firebase.FirebaseApp
import org.koin.android.ext.koin.androidContext
import org.koin.androidx.workmanager.koin.workManagerFactory
import org.koin.core.context.startKoin
import org.koin.dsl.module

class NotesApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        FirebaseApp.initializeApp(this)
        // PostHog SDK setup goes here too, if you use PostHogLogger or PostHogRemoteConfigHandler.

        startKoin {
            androidContext(this@NotesApplication)
            workManagerFactory()
            modules(AppModule)
        }
    }
}

val AppModule = module {
    includes(CoreModule, LoggingModule, RemoteConfigModule, WorkModule, NotesModule, UiModule)
}
```

Order inside `includes` does not matter for resolution (definitions are lazy), except that the
`createdAtStart` logger interactor is created at the end of `startKoin`, once everything is registered.

## Core, logging, remote config

```kotlin
import dev.dexkot.mobile.core.logger.ConsoleLogger
import dev.dexkot.mobile.core.logger.FirebaseLogger
import dev.dexkot.mobile.core.logger.ILogger
import dev.dexkot.mobile.core.logger.composed.plus
import dev.dexkot.mobile.core.logger.di.koin.loggerModule
import dev.dexkot.mobile.core.preferences.di.settingsFactoryModule
import dev.dexkot.mobile.core.remoteconfig.di.koin.firebaseRemoteConfig
import dev.dexkot.mobile.core.remoteconfig.di.koin.remoteConfigModule
import dev.dexkot.mobile.core.remoteconfig.handler.IRemoteConfigHandler
import kotlinx.serialization.json.Json
import org.koin.core.parameter.parametersOf
import org.koin.dsl.module

val CoreModule = module {
    includes(settingsFactoryModule)
    single { Json { ignoreUnknownKeys = true } }
}

val LoggingModule = module {
    includes(loggerModule(consoleLoggerTag = "Notes", logicSourceComponent = "app"))
    single<ILogger> { get<ConsoleLogger>() + get<FirebaseLogger>() }
}

val RemoteConfigModule = module {
    includes(remoteConfigModule())
    single<IRemoteConfigHandler> {
        get<IRemoteConfigHandler>(firebaseRemoteConfig()) {
            parametersOf(15_000L, get<Json>())   // syncTimeOut in ms, serializer
        }
    }
}
```

## Background work

```kotlin
package com.example.notes.work

import android.content.Context
import androidx.work.CoroutineWorker
import androidx.work.WorkerParameters
import dev.dexkot.mobile.core.work.data.worker.GenericWorkRunner
import dev.dexkot.mobile.core.work.di.koin.workManagerWorkModule
import dev.dexkot.mobile.core.work.domain.config.IWorkTypeConfigProvider
import dev.dexkot.mobile.core.work.domain.config.model.WorkTypeConfig
import dev.dexkot.mobile.core.work.domain.config.model.network.NetworkRequirement
import dev.dexkot.mobile.core.work.domain.executor.IWorkExecutor
import dev.dexkot.mobile.core.work.domain.executor.IWorkExecutorProvider
import dev.dexkot.mobile.core.work.domain.worker.WorkerClassProvider
import org.koin.androidx.workmanager.dsl.worker
import org.koin.dsl.binds
import org.koin.dsl.module
import kotlin.uuid.Uuid

/** App-owned: WorkManager persists this class name, so keep it stable. */
class NotesWorker(
    context: Context,
    params: WorkerParameters,
    private val runner: GenericWorkRunner,
) : CoroutineWorker(context, params) {
    override suspend fun doWork(): Result = runner.run(this)
}

const val SYNC_NOTES_WORK = "sync_notes"

class SyncNotesConfigProvider : IWorkTypeConfigProvider {
    override val workType = SYNC_NOTES_WORK
    override fun getConfig() = WorkTypeConfig(networkRequirement = NetworkRequirement.CONNECTED)
}

class SyncNotesExecutorProvider(private val syncNotes: ISyncNotesUseCase) : IWorkExecutorProvider {
    override val workType = SYNC_NOTES_WORK
    override fun create(workId: Uuid): IWorkExecutor = SyncNotesExecutor(workId, syncNotes)
}

val WorkModule = module {
    includes(workManagerWorkModule)

    worker { NotesWorker(context = get(), params = get(), runner = get()) }
    single<WorkerClassProvider> { WorkerClassProvider { NotesWorker::class.java } }

    // One pair per work type; the workType strings must match.
    factory { SyncNotesConfigProvider() } binds arrayOf(IWorkTypeConfigProvider::class)
    factory { SyncNotesExecutorProvider(syncNotes = get()) } binds arrayOf(IWorkExecutorProvider::class)
    // Optional progress/completion notifications:
    // factory { SyncNotesNotificationProvider(androidContext()) } binds arrayOf(IWorkNotificationProvider::class)
}
```

`GenericWorkRunner` also needs `IPermissionChecker`, provided by `permissionModule()` in `UiModule`.
Scheduling (`IScheduleWorkUseCase(workType, workId)`) and executors are covered in dexkot-infra.

On iOS (or for in-process work on Android), replace `workManagerWorkModule` with
`coroutineWorkModule` and drop the worker, `WorkerClassProvider` and notification providers. Optionally
bind an app-lifetime scope: `single<CoroutineScope>(workCoroutineScope()) { appScope }`.

## Data and use cases

```kotlin
fun notesDatabase() = named("notes_database")   // resolved by other modules → public

val NotesModule = module {
    includes(NoteModule, NotesUseCaseModule)
}

internal val NoteModule = module {
    single(notesDatabase()) { createNotesDatabase(androidContext()) }                          // stateful
    single<INoteLocalDataSource> { NoteLocalDataSource(database = get(notesDatabase())) }     // cache / connection
    factory<INoteRepository> { NoteRepository(localDataSource = get()) }                     // stateless
}

internal val NotesUseCaseModule = module {
    factory<IGetNoteUseCase> { GetNoteUseCase(repository = get()).withLogging() }           // @AutoLogUseCase
    factory<ISyncNotesUseCase> { SyncNotesUseCase(repository = get()).withLogging() }
}
```

(`createNotesDatabase` stands for your database factory; see dexkot-data-layer.)

## UI

```kotlin
import dev.dexkot.mobile.core.permission.di.koin.permissionModule
import dev.dexkot.mobile.core.ui.di.koin.uiModule
import dev.dexkot.mobile.core.ui.navigation.action.INavActionFactory
import dev.dexkot.mobile.core.ui.navigation.action.INavActionLogDecorator
import dev.dexkot.mobile.core.ui.viewmodel.android.AndroidViewModel
import org.koin.core.module.dsl.viewModel
import org.koin.core.qualifier.named
import org.koin.dsl.binds
import org.koin.dsl.module

val UiModule = module {
    includes(
        uiModule(preferencesName = "com.example.notes.ui.preferences"),
        permissionModule(preferencesName = "com.example.notes.permissions"),
        CommonsNavigationModule,
        LauncherModule,
        NoteListModule,
        NoteDetailModule,
    )
}

// Screen module: nav factory + log decorator + ViewModel
val NoteListModule = module {
    factory { NoteListNavActionFactory() } binds arrayOf(INavActionFactory::class)
    factory { NoteListNavAction_LogDecorator() } binds arrayOf(INavActionLogDecorator::class)

    viewModel<AndroidViewModel<NoteListModel, NoteListIntention>>(noteListViewModel()) { parameters ->
        NoteListViewModel(observeNotes = get(), savedStateHandle = get())
            .withLogging(logger = get(), sourceComponent = parameters.get())
    }
}

internal fun noteListViewModel() = named("note_list_viewmodel")
```

The Route for this screen resolves it with
`koinViewModel<AndroidViewModel<NoteListModel, NoteListIntention>>(noteListViewModel()) { parametersOf(IDENTIFIER) }`
(see dexkot-navigation). `savedStateHandle = get()` works because Koin injects the ViewModel's
`SavedStateHandle` when it is created through `koinViewModel`.

The Navigation host resolves from this graph: `INavActionDispatcher`, the permission `SharedPreferences`
(`permissionPreferences()`), `IRemoteConfigProvider`, the UI `ILoggerInteractor` (with a parameter), and
`getAll<INavActionLogDecorator>()`; wrap it in `Preferences(uiPreferences = koinInject())`.

## Manifest

Required only with `workManagerFactory()`: stop WorkManager from initialising itself before Koin
provides its factory.

```xml
<provider
    android:name="androidx.startup.InitializationProvider"
    android:authorities="${applicationId}.androidx-startup"
    android:exported="false"
    tools:node="merge">
    <meta-data
        android:name="androidx.work.WorkManagerInitializer"
        android:value="androidx.startup"
        tools:node="remove" />
</provider>
```

## iOS

The KMP modules are plain `Module`s usable from shared code: put `LoggingModule`,
`RemoteConfigModule`, a `coroutineWorkModule`-based work module and your data modules in `commonMain`
and call `startKoin { modules(...) }` from the iOS entry point (after configuring the Firebase/PostHog
iOS SDKs). `uiModule`, `permissionModule` and `workManagerWorkModule` stay Android-only.
