# core-remoteconfig reference (0.27.0)

Contents
1. Layers: handler and provider
2. API
3. Typed values with kotlinx.serialization
4. Choosing and wiring a backend
5. Firebase backend semantics
6. PostHog backend semantics
7. Compose helpers (core-ui)
8. Testing
9. Gotchas

## 1. Layers

- **`IRemoteConfigHandler`**: talks to one backend. Returns `ResultOf` for reads, exposes cached hot
  `StateFlow`s per key, and `sync()` to fetch. Stateful, so there is one instance per backend.
- **`IRemoteConfigProvider`**: what features inject. Convenience reads with defaults or nullables,
  delegating to the handler. Cheap `factory`.
- **Typed extensions** you write per feature on `IRemoteConfigProvider`, hiding key names and defaults.

Why two layers: features never see `ResultOf` plumbing or backend choice, and swapping Firebase for
PostHog is a one-line DI change.

## 2. API

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

class FirebaseRemoteConfigHandler(syncTimeOut: Long, serializer: Json, expirationTime: Long = 60L)   // core-remoteconfig-firebase
class PostHogRemoteConfigHandler(serializer: Json)                                                     // core-remoteconfig-posthog

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
class RemoteConfigProvider(override val remoteConfigHandler: IRemoteConfigHandler) : IRemoteConfigProvider
inline fun <reified T> IRemoteConfigProvider.get(key: String): T?
inline fun <reified T> IRemoteConfigProvider.get(key: String, defaultValue: T): T
inline fun <reified T> IRemoteConfigProvider.getFlow(key: String, defaultValue: T): StateFlow<T>

// dev.dexkot.mobile.core.remoteconfig.error
class ConfigNotFound(val configKey: String) : ErrorEntity.BusinessError      // severity INFO

// dev.dexkot.mobile.core.remoteconfig.flow
fun <T, R> StateFlow<T>.mapState(transform: (T) -> R): StateFlow<R>             // no scope needed; stays a StateFlow

// dev.dexkot.mobile.core.remoteconfig.di.koin
fun firebaseRemoteConfig(): StringQualifier     // named("firebase_remote_config")
fun postHogRemoteConfig(): StringQualifier      // named("posthog_remote_config")
fun remoteConfigModule(dispatcher: CoroutineDispatcher = Dispatchers.Default): Module
```

`firebaseRemoteConfig`'s `syncTimeOut` is in milliseconds; `expirationTime` (minimum fetch interval) is
in seconds.

Read results:
- key missing on the backend: `Failure(ConfigNotFound)`;
- value that cannot be parsed (`getLong` on `"abc"`): `Failure(UnexpectedError)`, logged;
- `get<T>`: empty string is `Failure(ErrorEntity.NoData)`; invalid JSON or wrong shape is
  `Failure(UnexpectedError)` with the serialization exception; handler failures pass through.

`IRemoteConfigProvider.get(key)` turns any failure into `null`; `get(key, default)` into `default`.

## 3. Typed values

```kotlin
package com.example.sync.remoteconfig

import dev.dexkot.mobile.core.remoteconfig.flow.mapState
import dev.dexkot.mobile.core.remoteconfig.provider.IRemoteConfigProvider
import dev.dexkot.mobile.core.remoteconfig.provider.get
import dev.dexkot.mobile.core.remoteconfig.provider.getFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.serialization.Serializable

private const val SYNC_ENABLED = "sync_enabled"
private const val SYNC_INTERVAL_MINUTES = "sync_interval_minutes"
private const val SYNC_POLICY = "sync_policy"

@Serializable
data class SyncPolicy(val maxBatch: Int = 50, val wifiOnly: Boolean = true)

fun IRemoteConfigProvider.syncEnabled(): StateFlow<Boolean> = getBooleanFlow(SYNC_ENABLED, defaultValue = true)

fun IRemoteConfigProvider.syncIntervalMinutes(): Int = getLong(SYNC_INTERVAL_MINUTES, defaultValue = 15L).toInt()

fun IRemoteConfigProvider.syncPolicy(): SyncPolicy = get(SYNC_POLICY, defaultValue = SyncPolicy())

fun IRemoteConfigProvider.syncPolicyFlow(): StateFlow<SyncPolicy> = getFlow(SYNC_POLICY, defaultValue = SyncPolicy())

fun IRemoteConfigProvider.maxBatchFlow(): StateFlow<Int> = syncPolicyFlow().mapState { it.maxBatch }
```

Store JSON values in the backend as objects (`{"maxBatch": 20}`). Give DTO properties defaults and
configure the `Json` with `ignoreUnknownKeys = true`, so adding a field on the backend does not break old
app versions. The consumer module needs the `kotlinx-serialization` compiler plugin for its DTOs.

`getFlow<T>` decodes on every read of `.value` and every emission; decoding errors return the default and
are logged via `logger { }`. For expensive types, derive once with `mapState` in a long-lived owner.

## 4. Choosing and wiring a backend

`remoteConfigModule()` registers both handlers as qualified `single`s created from Koin parameters, plus
`factory<IRemoteConfigProvider> { RemoteConfigProvider(get()) }`, which resolves an **unqualified**
`IRemoteConfigHandler` that you bind:

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
        get(qualifier = firebaseRemoteConfig(), parameters = { parametersOf(15_000L /* syncTimeOut ms */, get<Json>()) })
    }
}
```

PostHog variant: `get(qualifier = postHogRemoteConfig(), parameters = { parametersOf(get<Json>()) })`.
Parameters are positional: `(Long, Json)` for Firebase, `(Json)` for PostHog.

Without Koin, construct directly: `RemoteConfigProvider(FirebaseRemoteConfigHandler(syncTimeOut = 15_000L, serializer = json))`.

Initialise the SDK (Firebase app / PostHog) before the handler is created: its constructor touches the SDK.

When to call `sync()`:
- Firebase: optional. The handler registers a real-time update listener when constructed and serves
  values activated in previous sessions. Call `sync()` at startup if you need fresh values in this session.
- PostHog: flags load with the SDK; call `sync()` to reload and push new values to existing flows.

```kotlin
appScope.launch { get<IRemoteConfigHandler>().sync() }   // ResultOf<Boolean>: Firebase true if something was activated
```

## 5. Firebase semantics

- Only values whose source is **remote** (fetched and activated, now or in an earlier session) are
  returned. In-app defaults registered on the Firebase SDK are ignored and read as `ConfigNotFound`, so
  your own default applies. Keep defaults in code, next to the key.
- `sync()` = fetch with `expirationTime` (default 60 s) + activate, bounded by `syncTimeOut`. Flows are
  updated only if activation changed something.
- The real-time listener re-activates and updates every flow created so far.
- `getBoolean` uses `String.toBoolean()`: only `"true"` (any case) is true; anything else is false, not an error.
- `getLong` fails on non-integer strings.
- iOS: the app links `FirebaseRemoteConfig` itself (pods or SPM).

## 6. PostHog semantics

- Values are **feature flag payloads** (`getFeatureFlagPayload`), not flag on/off state or variant names.
  A flag with no payload reads as `ConfigNotFound`. To drive a boolean, set the payload to `true`.
- Object / array payloads are re-serialised to JSON strings, ready for `get<T>`. Plain string payloads are
  returned as-is by `getString`; `get<String>` on them fails because they are not JSON.
- `sync()` reloads flags with a fixed 15 s timeout and always returns `Success(true)` on completion. There
  is no real-time listener; flows change only on `sync()`.
- `getLong` accepts decimals and truncates.

## 7. Compose helpers (core-ui, Android)

```kotlin
// dev.dexkot.mobile.core.ui.remoteconfig
@Composable fun IRemoteConfigProvider.getBooleanState(key: String, defaultValue: Boolean): State<Boolean>
@Composable fun IRemoteConfigProvider.getLongState(key: String, defaultValue: Long): State<Long>
@Composable fun IRemoteConfigProvider.getStringState(key: String, defaultValue: String): State<String>

// dev.dexkot.mobile.core.ui.compose.remoteconfig
val LocalRemoteConfig: ProvidableCompositionLocal<IRemoteConfigProvider>     // default DummyRemoteConfigProvider
@Composable fun RemoteConfig(remoteConfigProvider: IRemoteConfigProvider, content: @Composable () -> Unit)
object DummyRemoteConfigHandler : IRemoteConfigHandler
object DummyRemoteConfigProvider : IRemoteConfigProvider                       // previews
```

Pass `remoteConfigProvider = koinInject()` to core-ui's `Navigation(...)`; every screen then reads
`LocalRemoteConfig.current`:

```kotlin
val showBanner by LocalRemoteConfig.current.getBooleanState("show_sync_banner", defaultValue = false)
```

Prefer your typed extensions from ViewModels, and the Compose helpers only for purely visual flags.
The helpers `remember` the flow without keys, so pass a constant key: changing `key` on recomposition
keeps observing the first one.

## 8. Testing

The real handlers start native SDKs in their constructors. In unit tests, implement
`IRemoteConfigHandler` with maps and `MutableStateFlow`s and wrap it in `RemoteConfigProvider`, or fake
`IRemoteConfigProvider` directly. Typed extensions are then tested through the fake.

## 9. Gotchas

1. **Bind an unqualified `IRemoteConfigHandler`.** Otherwise injecting `IRemoteConfigProvider` fails on
   first resolution (it is a `factory`, so startup does not catch it).
2. **Handler parameters apply once.** The handlers are `single`; resolving again with different parameters
   returns the first instance.
3. **Flows are cached by key.** The first `defaultValue` requested for a key is the one used. After a sync,
   a key that disappeared keeps its last value instead of resetting to the default.
4. **`remoteConfigModule(dispatcher)` has no visible effect in 0.27.0**: the dispatcher builds an internal
   scope that the handlers do not use.
5. **PostHog booleans need a payload** (section 6).
6. **Firebase SDK defaults are ignored** (section 5).
