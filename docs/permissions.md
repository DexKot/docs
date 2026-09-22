# Runtime permissions (Android)

`core-permission` wraps Android runtime permissions in two small interfaces:

- **`IPermissionChecker`**: "is this granted?" It only needs a `Context`, so ViewModels, use cases and
  workers can use it safely.
- **`IPermissionController`**: the full state (rationale, first request) plus requests. It needs an
  `Activity`, so it lives in the UI.

Keeping them apart means most code never holds an Activity, and only UI code ever interrupts the user.
core-ui adds Compose helpers and logs every request and result automatically.

Version: **0.27.0**. Android only.

- [Install](#install)
- [1. Register the module](#1-register-the-module)
- [2. Create the controller at the root](#2-create-the-controller-at-the-root)
- [3. Request from a screen](#3-request-from-a-screen)
- [4. Check from anywhere else](#4-check-from-anywhere-else)
- [5. Custom permissions](#5-custom-permissions)
- [How states and results are decided](#how-states-and-results-are-decided)
- [Pitfalls](#pitfalls)
- [API reference](#api-reference)

## Install

```kotlin
implementation("dev.dexkot.mobile:core-permission:0.27.0")
implementation("dev.dexkot.mobile:core-permission-di-koin:0.27.0")
implementation("dev.dexkot.mobile:core-ui:0.27.0")          // Compose helpers
```

Remember to declare the permissions in your `AndroidManifest.xml` as usual.

## 1. Register the module

```kotlin
val appUiModule = module {
    includes(permissionModule(preferencesName = "com.example.notes.permissions"))
}
```

This binds `IPermissionChecker` and a `SharedPreferences` (qualifier `permissionPreferences()`) that
remembers which permissions were ever requested. Keep `preferencesName` stable across releases.

## 2. Create the controller at the root

```kotlin
class MainActivity : FragmentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            val permissionController = rememberPermissionController(
                activity = this,
                preferences = koinInject(qualifier = permissionPreferences()),
            )
            Navigation(
                graphs = graphs,
                navigator = navigator,
                dispatcher = koinInject(),
                permissionController = permissionController,
            )
        }
    }
}
```

`rememberPermissionController` registers the activity-result launchers the controller needs. core-ui's
`Navigation` gives every screen the controller as `LocalPermissionController`, wrapped so that requests
and results are logged under that screen.

Call it first thing in `setContent`, before reading any state that changes. The function does not
`remember` the controller it builds, so a recomposition creates a new one and a request that is in
flight can lose its callback.

## 3. Request from a screen

```kotlin
@Composable
fun AttachPhotoButton(onAttach: () -> Unit, openAppSettings: () -> Unit) {
    val controller = LocalPermissionController.current
    val camera by rememberPermissionState(Permission.Camera)

    Button(onClick = {
        when {
            camera.isGranted -> onAttach()
            !camera.canRequest -> openAppSettings()
            else -> controller.requestPermission(Permission.Camera) { result ->
                when (result) {
                    PermissionResult.Granted -> onAttach()
                    PermissionResult.AutoDenied -> openAppSettings()
                    PermissionResult.Denied -> Unit
                }
            }
        }
    }) { Text("Attach photo") }
}
```

- `rememberPermissionState` refreshes every time the screen resumes, including after the system dialog
  and after the user returns from settings.
- `camera.shouldShowDisclosure` is true when you should explain first: before the very first request, or
  when Android recommends a rationale.
- `PermissionResult.AutoDenied` means the system answered without showing a dialog (the user chose
  "don't ask again"). The only way forward is the app settings screen.
- For several permissions at once, use `rememberPermissionsState(...)` and `requestPermissions(listOf(...))`.

## 4. Check from anywhere else

```kotlin
class RecorderViewModel(
    private val permissionChecker: IPermissionChecker,
) : AndroidViewModel<RecorderModel, RecorderIntention>() {
    private fun canRecord() = permissionChecker.isGranted(Permission.RecordAudio)
}
```

The background work module uses the same checker to decide whether it may post notifications.

## 5. Custom permissions

```kotlin
object AppPermissions {
    val ReadContacts = Permission(Manifest.permission.READ_CONTACTS)
    val NearbyWifi = Permission(Manifest.permission.NEARBY_WIFI_DEVICES, minSdk = Build.VERSION_CODES.TIRAMISU)
}
```

- `minSdk` records where the permission exists. Check `isRelevantForCurrentSdk` before requesting it, and
  treat it as granted on older versions.
- `Permission` is an open class, so you can subclass it to attach extra data.
- Equality uses the permission string only.

Built in: `RecordAudio`, `Camera`, `FineLocation`, `CoarseLocation`, `BackgroundLocation`,
`ActivityRecognition`, `BluetoothScan`, `BluetoothConnect`, `Notifications`, `ReadMediaImages`,
`ReadMediaVideo`, `ReadMediaAudio`.

## How states and results are decided

Android does not tell apps directly that a permission is permanently denied, so the controller infers it:

| State | When |
|---|---|
| `Granted` | the system reports it granted |
| `PermanentlyDenied` | no rationale recommended **and** already requested in this session |
| `Denied` | otherwise |

| Result | When |
|---|---|
| `Granted` | granted |
| `AutoDenied` | no rationale before **and** after the request (no dialog was shown) |
| `Denied` | the user declined in the dialog |

## Pitfalls

- **`PermanentlyDenied` shows up only after a request in the current session.** After a cold start, the
  same permission reports `Denied`.
- **A request always goes to the system unless the permission is already granted.** If you want to skip
  straight to settings, check `canRequest` yourself, as in the example.
- **Pending callbacks don't survive Activity recreation.** Rely on `rememberPermissionState` for the final
  state.
- **Previews and tests:** `DummyPermissionController` (the default of `LocalPermissionController`) reports
  everything as granted unless you pass explicit states.

## API reference

```kotlin
// dev.dexkot.mobile.core.permission
open class Permission(val key: String, val minSdk: Int? = null) { val isRelevantForCurrentSdk: Boolean }
enum class PermissionStatus { Granted, Denied, PermanentlyDenied }
sealed interface PermissionResult { Granted; Denied; AutoDenied }
data class PermissionState(status: PermissionStatus, requiresRationale: Boolean, isFirstTimeEver: Boolean, isFirstTimeThisSession: Boolean) {
    val isGranted: Boolean; val canRequest: Boolean; val shouldShowDisclosure: Boolean
    companion object { val Granted: PermissionState }
}

// .checker
interface IPermissionChecker { fun isGranted(permission: Permission): Boolean }
class PermissionChecker(context: Context) : IPermissionChecker

// .controller
interface IPermissionController {
    fun getState(permission: Permission): PermissionState
    fun requestPermission(permission: Permission, onResult: (PermissionResult) -> Unit)
    fun requestPermissions(permissions: List<Permission>, onResult: (Map<Permission, PermissionResult>) -> Unit)
}
class PermissionController(activity: Activity, preferences: SharedPreferences,
    singlePermissionLauncher: ActivityResultLauncher<String>, multiplePermissionsLauncher: ActivityResultLauncher<Array<String>>)
class DummyPermissionController(permissionStates: Map<Permission, PermissionState> = emptyMap())

// dev.dexkot.mobile.core.permission.di.koin
fun permissionPreferences(): StringQualifier
fun permissionModule(preferencesName: String = "dev.dexkot.permissions.preferences"): Module

// dev.dexkot.mobile.core.ui.compose.permission (core-ui)
@Composable fun rememberPermissionController(activity: Activity, preferences: SharedPreferences): IPermissionController
val LocalPermissionController: ProvidableCompositionLocal<IPermissionController>
@Composable fun PermissionController(permissionController: IPermissionController, content: @Composable () -> Unit)
@Composable fun rememberPermissionState(permission: Permission): State<PermissionState>
@Composable fun rememberPermissionsState(vararg permissions: Permission): State<Map<Permission, PermissionState>>
fun IPermissionController.withLogging(logger: ILoggerInteractor): IPermissionController
```
