# Remote config

`core-remoteconfig` gives features typed, default-safe access to remote values (feature flags, limits,
JSON settings) without knowing which service serves them. Two backends ship with it: Firebase Remote
Config and PostHog feature flag payloads. Both are KMP.

Version: **0.27.0**.

- [Install](#install)
- [How it is layered](#how-it-is-layered)
- [1. Wire a backend](#1-wire-a-backend)
- [2. Write typed accessors](#2-write-typed-accessors)
- [3. Read values](#3-read-values)
- [4. Refresh](#4-refresh)
- [Firebase vs PostHog](#firebase-vs-posthog)
- [Pitfalls](#pitfalls)
- [API reference](#api-reference)

## Install

```kotlin
// feature modules
implementation("dev.dexkot.mobile:core-remoteconfig:0.27.0")

// app / composition root (pulls both backends)
implementation("dev.dexkot.mobile:core-remoteconfig-di-koin:0.27.0")
```

Add `maven("https://dexkot.github.io/maven")` to your repositories. Features that store JSON values also
need the `org.jetbrains.kotlin.plugin.serialization` plugin. On iOS, the backends use CocoaPods cinterop
and your app links `FirebaseRemoteConfig` or `PostHog` itself. The iOS klibs are built from source
(`publishToMavenLocal`).

## How it is layered

- **`IRemoteConfigHandler`**: one per backend. Reads return `ResultOf`, and each key has a cached hot
  `StateFlow`. `sync()` fetches fresh values.
- **`IRemoteConfigProvider`**: what features inject. Reads take a default or return `null`.
- **Your extensions** on `IRemoteConfigProvider`: one function per setting. They keep the key name and the
  default in one private place.

With this split, switching backends is a one-line DI change, and no feature code sees `ResultOf`
plumbing for configuration.

## 1. Wire a backend

```kotlin
import dev.dexkot.mobile.core.remoteconfig.di.koin.firebaseRemoteConfig
import dev.dexkot.mobile.core.remoteconfig.di.koin.remoteConfigModule
import dev.dexkot.mobile.core.remoteconfig.handler.IRemoteConfigHandler
import kotlinx.serialization.json.Json
import org.koin.core.parameter.parametersOf
import org.koin.dsl.module

val remoteConfigAppModule = module {
    single { Json { ignoreUnknownKeys = true } }
    includes(remoteConfigModule())
    single<IRemoteConfigHandler> {
        get(qualifier = firebaseRemoteConfig(), parameters = { parametersOf(15_000L, get<Json>()) })
    }
}
```

`remoteConfigModule()` defines both handlers under qualifiers and an `IRemoteConfigProvider` factory.
The provider needs an *unqualified* handler, and that binding is how you pick a backend. For PostHog:

```kotlin
single<IRemoteConfigHandler> {
    get(qualifier = postHogRemoteConfig(), parameters = { parametersOf(get<Json>()) })
}
```

The parameters are positional:
- Firebase takes the sync timeout in milliseconds, then the `Json`.
- PostHog takes only the `Json`.

Initialise the Firebase or PostHog SDK before the handler is first created.

## 2. Write typed accessors

```kotlin
private const val SYNC_ENABLED = "sync_enabled"
private const val SYNC_INTERVAL_MINUTES = "sync_interval_minutes"
private const val SYNC_POLICY = "sync_policy"

@Serializable
data class SyncPolicy(val maxBatch: Int = 50, val wifiOnly: Boolean = true)

fun IRemoteConfigProvider.syncEnabled(): StateFlow<Boolean> = getBooleanFlow(SYNC_ENABLED, defaultValue = true)
fun IRemoteConfigProvider.syncIntervalMinutes(): Int = getLong(SYNC_INTERVAL_MINUTES, defaultValue = 15L).toInt()
fun IRemoteConfigProvider.syncPolicy(): SyncPolicy = get(SYNC_POLICY, defaultValue = SyncPolicy())
fun IRemoteConfigProvider.maxBatchFlow(): StateFlow<Int> = getFlow(SYNC_POLICY, SyncPolicy()).mapState { it.maxBatch }
```

Why every accessor has a default: the app must behave correctly offline, before the first fetch, and
when someone deletes a key on the backend.

For JSON values, give DTO properties defaults and use `ignoreUnknownKeys = true`. Then new fields on the
backend don't break older app versions.

## 3. Read values

From a ViewModel or use case, inject `IRemoteConfigProvider` and call your extensions.

In Compose (Android, core-ui), pass `remoteConfigProvider = koinInject()` to `Navigation(...)`, then read
`LocalRemoteConfig.current`:

```kotlin
val showBanner by LocalRemoteConfig.current.getBooleanState("show_sync_banner", defaultValue = false)
```

For previews, `DummyRemoteConfigProvider` is the default value of `LocalRemoteConfig`.

## 4. Refresh

```kotlin
appScope.launch { remoteConfigHandler.sync() }
```

**Firebase.**
- `sync()` fetches (at most once per 60 seconds, the handler's default `expirationTime`) and activates.
- The handler also listens for real-time updates from the moment it is created.
- Values activated in earlier sessions are served immediately.

**PostHog.**
- `sync()` reloads feature flags, with a fixed 15-second timeout.
- Flows only change when you call `sync()`.

After either one, every flow created so far is updated.

## Firebase vs PostHog

| | Firebase | PostHog |
|---|---|---|
| Source of values | Remote Config parameters (activated remote values only) | Feature flag **payloads** |
| Real-time updates | yes | no, only `sync()` |
| `sync()` timeout | your `syncTimeOut` | 15 s, fixed |
| `getBoolean` | `"true"` (any case) is true, anything else false | same |
| `getLong` | integers only | decimals truncated |
| In-app defaults on the SDK | ignored (use code defaults) | n/a |

A PostHog flag without a payload reads as "not found". To drive a boolean from PostHog, set the payload
to `true`.

## Pitfalls

- **No unqualified `IRemoteConfigHandler`**: injecting `IRemoteConfigProvider` fails at first use, not at
  startup.
- **Handler parameters apply once.** The handlers are singletons; later resolutions ignore new
  parameters.
- **Flows are cached per key.** The first default requested for a key wins everywhere.
- **Missing keys after a sync** keep their previous value in flows; they don't reset to the default.
- **`get<T>` on an empty value** returns `Failure(ErrorEntity.NoData)` (fixed in 0.27.0), so provider
  reads fall back to your default.
- **`remoteConfigModule(dispatcher)`** does not change any behaviour in 0.27.0.
- **Tests:** the real handlers start native SDKs. Fake `IRemoteConfigHandler` (maps plus
  `MutableStateFlow`s) and wrap it in `RemoteConfigProvider`.

## API reference

```kotlin
// dev.dexkot.mobile.core.remoteconfig.handler
interface IRemoteConfigHandler {
    val serializer: Json
    suspend fun sync(): ResultOf<Boolean>
    fun getBoolean(key: String): ResultOf<Boolean>
    fun getLong(key: String): ResultOf<Long>
    fun getString(key: String): ResultOf<String>
    fun getBooleanFlow(key: String, defaultValue: Boolean): StateFlow<Boolean>
    fun getLongFlow(key: String, defaultValue: Long): StateFlow<Long>
    fun getStringFlow(key: String, defaultValue: String): StateFlow<String>
}
inline fun <reified T> IRemoteConfigHandler.get(key: String): ResultOf<T>
inline fun <reified T> IRemoteConfigHandler.getFlow(key: String, defaultValue: T): StateFlow<T>
class FirebaseRemoteConfigHandler(syncTimeOut: Long, serializer: Json, expirationTime: Long = 60L)
class PostHogRemoteConfigHandler(serializer: Json)

// dev.dexkot.mobile.core.remoteconfig.provider
interface IRemoteConfigProvider {
    val remoteConfigHandler: IRemoteConfigHandler
    fun getBoolean(key: String): Boolean?
    fun getString(key: String): String?
    fun getLong(key: String): Long?
    fun getBoolean(key: String, defaultValue: Boolean = false): Boolean
    fun getLong(key: String, defaultValue: Long = 0L): Long
    fun getString(key: String, defaultValue: String = ""): String
    fun getBooleanFlow(key: String, defaultValue: Boolean): StateFlow<Boolean>
    fun getLongFlow(key: String, defaultValue: Long): StateFlow<Long>
    fun getStringFlow(key: String, defaultValue: String): StateFlow<String>
}
class RemoteConfigProvider(remoteConfigHandler: IRemoteConfigHandler) : IRemoteConfigProvider
inline fun <reified T> IRemoteConfigProvider.get(key: String): T?
inline fun <reified T> IRemoteConfigProvider.get(key: String, defaultValue: T): T
inline fun <reified T> IRemoteConfigProvider.getFlow(key: String, defaultValue: T): StateFlow<T>

// dev.dexkot.mobile.core.remoteconfig.error / .flow
class ConfigNotFound(val configKey: String) : ErrorEntity.BusinessError
fun <T, R> StateFlow<T>.mapState(transform: (T) -> R): StateFlow<R>

// dev.dexkot.mobile.core.remoteconfig.di.koin
fun firebaseRemoteConfig(): StringQualifier
fun postHogRemoteConfig(): StringQualifier
fun remoteConfigModule(dispatcher: CoroutineDispatcher = Dispatchers.Default): Module

// core-ui (Android)
@Composable fun IRemoteConfigProvider.getBooleanState(key: String, defaultValue: Boolean): State<Boolean>   // + getLongState, getStringState
val LocalRemoteConfig: ProvidableCompositionLocal<IRemoteConfigProvider>
@Composable fun RemoteConfig(remoteConfigProvider: IRemoteConfigProvider, content: @Composable () -> Unit)
object DummyRemoteConfigProvider : IRemoteConfigProvider
```
