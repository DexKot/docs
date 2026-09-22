# core-permission reference (0.27.0, Android only)

Contents
1. Model
2. API
3. Wiring (Koin + Compose root)
4. Requesting from a screen
5. Checking from ViewModels and background code
6. Custom permissions
7. How status and results are computed
8. Logging
9. Gotchas

## 1. Model

- **`Permission`**: a manifest permission string plus the SDK level where it exists.
- **`IPermissionChecker`**: "is it granted?" Needs only a `Context`, so it is safe in ViewModels, use
  cases and workers.
- **`IPermissionController`**: state (with rationale / first-time info) and requests. Needs an `Activity`
  and activity-result launchers, so it lives in the UI.

Why the split: most code only needs a yes/no, and giving it an Activity-bound object would leak the
Activity. Only UI code ever asks the user.

## 2. API

```kotlin
// dev.dexkot.mobile.core.permission
open class Permission(val key: String, val minSdk: Int? = null) {
    val isRelevantForCurrentSdk: Boolean              // minSdk == null || SDK_INT >= minSdk
    // equals / hashCode use `key` only
    companion object {
        val RecordAudio; val Camera; val FineLocation; val CoarseLocation
        val BackgroundLocation /* Q */; val ActivityRecognition /* Q */
        val BluetoothScan /* S */; val BluetoothConnect /* S */
        val Notifications /* TIRAMISU */; val ReadMediaImages /* TIRAMISU */
        val ReadMediaVideo /* TIRAMISU */; val ReadMediaAudio /* TIRAMISU */
    }
}
enum class PermissionStatus { Granted, Denied, PermanentlyDenied }
sealed interface PermissionResult { data object Granted; data object Denied; data object AutoDenied }
data class PermissionState(
    val status: PermissionStatus,
    val requiresRationale: Boolean,
    val isFirstTimeEver: Boolean,
    val isFirstTimeThisSession: Boolean,
) {
    val isGranted: Boolean              // status == Granted
    val canRequest: Boolean             // status != PermanentlyDenied
    val shouldShowDisclosure: Boolean   // !isGranted && (isFirstTimeEver || requiresRationale)
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
class PermissionController(
    activity: Activity,
    preferences: SharedPreferences,
    singlePermissionLauncher: ActivityResultLauncher<String>,
    multiplePermissionsLauncher: ActivityResultLauncher<Array<String>>,
) : IPermissionController {
    fun onSinglePermissionResult(granted: Boolean)
    fun onMultiplePermissionsResult(grantedMap: Map<String, Boolean>)
}
class DummyPermissionController(permissionStates: Map<Permission, PermissionState> = emptyMap()) : IPermissionController
// previews / tests: unknown permissions report PermissionState.Granted

// core-permission-di-koin: dev.dexkot.mobile.core.permission.di.koin
fun permissionPreferences(): StringQualifier            // named("permission_preferences")
fun permissionModule(preferencesName: String = "dev.dexkot.permissions.preferences"): Module

// core-ui: dev.dexkot.mobile.core.ui.compose.permission
@Composable fun rememberPermissionController(activity: Activity, preferences: SharedPreferences): IPermissionController
val LocalPermissionController: ProvidableCompositionLocal<IPermissionController>    // default DummyPermissionController()
@Composable fun PermissionController(permissionController: IPermissionController, content: @Composable () -> Unit)
@Composable fun rememberPermissionState(permission: Permission): State<PermissionState>
@Composable fun rememberPermissionsState(vararg permissions: Permission): State<Map<Permission, PermissionState>>
fun IPermissionController.withLogging(logger: dev.dexkot.mobile.core.logger.interactors.ui.ILoggerInteractor): IPermissionController
```

`permissionModule()` registers `single<IPermissionChecker>`, the qualified `SharedPreferences` used for
request history, and a `factory` for `PermissionController` taking `(Activity, launcher, launcher)` as
parameters (for non-Compose hosts).

## 3. Wiring

```kotlin
// DI
val appUiModule = module {
    includes(permissionModule(preferencesName = "com.example.notes.permissions"))
}
```

```kotlin
// MainActivity : FragmentActivity
setContent {
    val permissionController = rememberPermissionController(
        activity = this,
        preferences = koinInject(qualifier = permissionPreferences()),
    )
    AppTheme {
        Navigation(
            graphs = graphs,
            navigator = navigator,
            dispatcher = koinInject(),
            permissionController = permissionController,
            /* remoteConfigProvider, loggerFactory, decorators ... */
        )
    }
}
```

Keep this call at the very top of `setContent`, before anything that reads changing state (see
Gotcha 1). Without core-ui's `Navigation`, provide the controller yourself with
`PermissionController(permissionController) { ... }`.

## 4. Requesting from a screen

```kotlin
import android.content.Intent
import android.net.Uri
import android.provider.Settings
import androidx.compose.material3.Button
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.compose.ui.platform.LocalContext
import dev.dexkot.mobile.core.permission.Permission
import dev.dexkot.mobile.core.permission.PermissionResult
import dev.dexkot.mobile.core.ui.compose.permission.LocalPermissionController
import dev.dexkot.mobile.core.ui.compose.permission.rememberPermissionState

@Composable
fun AttachPhotoButton(onAttach: () -> Unit) {
    val context = LocalContext.current
    val controller = LocalPermissionController.current
    val camera by rememberPermissionState(Permission.Camera)

    val openSettings = {
        context.startActivity(Intent(Settings.ACTION_APPLICATION_DETAILS_SETTINGS, Uri.fromParts("package", context.packageName, null)))
    }

    Button(onClick = {
        when {
            camera.isGranted -> onAttach()
            !camera.canRequest -> openSettings()
            else -> controller.requestPermission(Permission.Camera) { result ->
                when (result) {
                    PermissionResult.Granted -> onAttach()
                    PermissionResult.AutoDenied -> openSettings()     // the OS did not show a dialog
                    PermissionResult.Denied -> Unit                   // user said no; you may explain and retry later
                }
            }
        }
    }) { Text("Attach photo") }
}
```

Use `camera.shouldShowDisclosure` to show your own explanation before the first request or when the OS
recommends a rationale. `rememberPermissionState` refreshes on every `ON_RESUME`, which covers returning
from the OS dialog and from the settings screen.

For permissions with `minSdk` (e.g. `Permission.Notifications`), check `isRelevantForCurrentSdk` first and
treat irrelevant ones as granted.

## 5. Checking from ViewModels and background code

```kotlin
class RecorderViewModel(
    private val permissionChecker: IPermissionChecker,
) : AndroidViewModel<RecorderModel, RecorderIntention>() {
    private fun canRecord(): Boolean = permissionChecker.isGranted(Permission.RecordAudio)
}
```

core-work's `GenericWorkRunner` uses `IPermissionChecker` to skip notifications when
`POST_NOTIFICATIONS` is missing, which is why the WorkManager backend needs `permissionModule()`.

## 6. Custom permissions

```kotlin
object AppPermissions {
    val ReadContacts = Permission(Manifest.permission.READ_CONTACTS)
    val NearbyWifi = Permission(Manifest.permission.NEARBY_WIFI_DEVICES, minSdk = Build.VERSION_CODES.TIRAMISU)
}
```

`Permission` is `open`, so you can also subclass it to attach metadata (a rationale string resource, for
example). Equality is by `key`, so two instances with the same key are the same permission everywhere
(maps, request history).

## 7. How status and results are computed

`getState`:
- `Granted` if the OS reports it granted;
- `PermanentlyDenied` if no rationale should be shown **and** it was already requested in this session;
- otherwise `Denied`.

Request results:
- `Granted`;
- `AutoDenied` when rationale was false both before and after the request (the OS answered without a
  dialog, typically "don't ask again");
- `Denied` otherwise.

History: each request stores `permission_requested_<key>` = true in the permission SharedPreferences (for
`isFirstTimeEver`) and adds the permission to an in-memory set (for `isFirstTimeThisSession`).

## 8. Logging

core-ui's screens wrap the controller with `withLogging(LocalLogger.current)`. Each request logs a
`PermissionEvent` with `Status.Request`, and each result logs `Granted`, `Denied` or `RequestIgnored`
(for `AutoDenied`). See **dexkot-logging**.

## 9. Gotchas

1. **`rememberPermissionController` is not memoised.** Each recomposition of its caller creates a new
   controller; the launchers route results to the newest one. A request in flight on an older instance
   then never calls back, and the per-session history is lost. Call it at the root of `setContent`, in a
   scope that does not recompose, or wrap your own `remember`-based host.
2. **Requests always show the OS dialog unless already granted.** There is no short-circuit for
   permanently denied permissions; you get `AutoDenied` after the round trip. Check
   `getState(p).canRequest` first to go straight to settings.
3. **`PermanentlyDenied` needs a request in this session.** After a cold start, a permanently denied
   permission reports `Denied` until requested once, because the only signal is "no rationale after a
   request".
4. **`requestPermissions` skips the dialog only when all are granted.**
5. **Request history lives in SharedPreferences.** Keep the same `preferencesName` across releases, or
   every user looks like a first-time user again.
6. **Pending requests do not survive Activity recreation.** Launchers are re-registered, but the callback
   you passed is gone; rely on `rememberPermissionState` (refreshed on resume) for the final state.
7. **Android only.** There is no iOS counterpart in 0.27.0.
