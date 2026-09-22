# DexKot mobile-core

Kotlin libraries for building Android and Kotlin Multiplatform apps with a consistent, layered
architecture: typed results and errors, transactional data access, MVI-style ViewModels, Compose
screens that survive process death, deep-link based navigation, AutoLog (KSP-generated logging),
background work, remote config and runtime permissions.

This repository holds the **public documentation** and the **Claude Code skills** for the libraries.
The artifacts are published to a public Maven repository.

**Current version: `0.27.0`**

## Install

```kotlin
// settings.gradle.kts
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
        maven("https://dexkot.github.io/maven")
    }
}

// build.gradle.kts
dependencies {
    implementation("dev.dexkot.mobile:core-foundation:0.27.0")
    implementation("dev.dexkot.mobile:core-ui:0.27.0")
    // ...the modules you need, see below
}
```

Toolchain: Kotlin 2.3.10, KSP 2.3.10, JDK 21, Android `minSdk` 28 / `compileSdk` 36, Koin 4.1.1.

Kotlin Multiplatform modules publish Android artifacts to the Maven repository. iOS artifacts are not
published yet; build them from source with `publishToMavenLocal`.

## Modules

| Area | Artifact | Platforms | What it gives you |
|------|----------|-----------|-------------------|
| Foundation | `core-foundation` | KMP | `ResultOf`, `ErrorEntity`, `ErrorMapper`, `CacheFetchPolicy`, `CommonParcelize`, `PlatformDispatchers` |
| Data | `core-database` | KMP | Transaction API: `ITransactionFactory`, `ITransaction` |
| | `core-database-sqldelight` | KMP | SQLDelight implementation: `TransactionFactory` with savepoint-based nesting |
| | `core-preferences` | KMP | Multiplatform key-value stores (`settingsFactoryModule`) |
| Presentation | `core-ui` | Android | `AndroidViewModel`, `Resource`, `fieldMutableStateOf`, Compose locals, navigation (`Route`, `Graph`, `INavAction`, `INavigator`), transitions |
| Logging | `core-logger` | KMP | `ILogger`, logger composition with `+`, logic and UI interactors, `LoggerContextElement` |
| | `core-logger-firebase`, `core-logger-posthog` | KMP | Firebase Analytics/Crashlytics and PostHog backends |
| | `core-annotation`, `core-ksp-processor` | KMP / JVM | `@AutoLog*` annotations and the KSP processor that generates logging decorators |
| Infrastructure | `core-work` | KMP | Background work: WorkManager backend (Android) and in-process coroutine backend |
| | `core-remoteconfig`, `-firebase`, `-posthog` | KMP | Typed remote config with Firebase or PostHog |
| | `core-permission` | Android | Runtime permissions with state tracking |
| DI (Koin) | `core-logger-di-koin`, `core-remoteconfig-di-koin`, `core-work-di-koin` | KMP | Koin modules for each infrastructure |
| | `core-permission-di-koin`, `core-ui-di-koin` | Android | |

The core modules don't depend on Koin. Each infrastructure ships a separate `-di-koin` module and
your app composes the ones it uses. See [Dependency injection](docs/dependency-injection.md).

## Architecture at a glance

```
UI (Compose)        Screen(screenState, onIntention)   ← ScreenState (@Parcelize, fieldMutableStateOf)
                         │ Intention         ▲ Model
Presentation        AndroidViewModel<Model, Intention> ── emit(INavAction) ──► NavActionDispatcher ─► INavigator
                         │ ResultOf ► Resource
Domain              I{Verb}{Entity}UseCase  (@AutoLogUseCase)
                         │ ResultOf
Data                Repository ─► DataSource  (withTransaction, never throws)
```

- **`ResultOf` versus `Resource`.** `ResultOf<T>` flows from data sources up to the ViewModel. The
  ViewModel turns it into `Resource<T>`, which is the async state the UI renders.
- **ViewModels express intent; the navigation layer carries it out.** A ViewModel only emits an
  `INavAction`. It never calls the `NavController` directly.
- **Logging is generated.** KSP decorators log intentions, use cases and navigation, so business code
  never calls the logger.

## Guides

1. [Getting started](docs/getting-started.md): from an empty app to one screen, end to end.
2. [Results and errors](docs/results-and-errors.md)
3. [Data layer](docs/data-layer.md): data sources, repositories, use cases, transactions, preferences
4. [Presentation](docs/presentation.md): ViewModels, `Resource`, screens and UI state
5. [Navigation](docs/navigation.md)
6. [Dependency injection](docs/dependency-injection.md)
7. [Logging and AutoLog](docs/logging.md)
8. [Background work](docs/background-work.md)
9. [Remote config](docs/remote-config.md)
10. [Permissions](docs/permissions.md)

## Claude Code skills

This repository is also a Claude Code plugin marketplace. Install the skills so Claude knows how to
write code with DexKot:

```
/plugin marketplace add DexKot/docs
/plugin install dexkot-mobile-core@dexkot
```

| Skill | Use it for |
|-------|------------|
| `dexkot-results-errors` | `ResultOf`, `ErrorEntity`, custom business errors, error mappers, cache policies |
| `dexkot-data-layer` | Data sources, repositories, use cases, SQLDelight transactions, preferences |
| `dexkot-viewmodel` | ViewModels with the Model/Intention pattern, `Resource`, saved state |
| `dexkot-screen` | Compose screens, `ScreenState`, `fieldMutableStateOf`, composition locals, permissions in UI |
| `dexkot-navigation` | Routes, graphs, the NavAction triple, arguments and results, deep links |
| `dexkot-di-koin` | Koin wiring for DexKot modules, screens and the application |
| `dexkot-logging` | Loggers, AutoLog annotations and KSP setup |
| `dexkot-infra` | Background work, remote config and permissions |

Each skill is triggered by the task it covers. Claude only loads a skill's detailed references when it
needs them.
