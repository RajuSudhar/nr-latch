# NRLatch — Component Designs

This document specifies every concrete component in the v1.0 build: the three Android components (`TileService`, two `BroadcastReceiver`s), the activity, and the four DI implementation classes that satisfy the public interfaces defined in [`02-public-interfaces.md`](./02-public-interfaces.md). Module locations are from [`01-module-graph.md`](./01-module-graph.md).

For every component the spec states: purpose, manifest declaration, lifecycle, threading model, error handling, security gating, and KPI (the empirical verification that the component works).

---

## 1. `NrLatchTileService` — `:feature:tile`

### Purpose

Subclass of `android.service.quicksettings.TileService`. The single user-facing control surface. On click, toggles the active data SIM's preferred network mode between `NR_LTE` and `LTE_ONLY` and persists the choice as a manual override.

### Manifest declaration

```xml
<service
    android:name=".NrLatchTileService"
    android:label="@string/tile_label"
    android:icon="@drawable/ic_tile"
    android:permission="android.permission.BIND_QUICK_SETTINGS_TILE"
    android:exported="true">
    <intent-filter>
        <action android:name="android.service.quicksettings.action.QS_TILE"/>
    </intent-filter>
    <meta-data
        android:name="android.service.quicksettings.ACTIVE_TILE"
        android:value="false"/>
</service>
```

`exported="true"` is mandatory for `TileService`; the `BIND_QUICK_SETTINGS_TILE` permission ensures only `system_server` may bind. `ACTIVE_TILE=false` ⇒ listening (non-active) tile: state is refreshed on every QS panel open via `onStartListening()` rather than pushed proactively.

### Lifecycle diagram

```
[QS panel opens]
        │
        ▼
onCreate (one-time per process)
        │
        ▼
onStartListening ──► launch coroutine ──► refreshState()
        │                                       │
        │                                       └─► qsTile.updateTile()
        │
[user taps]
        │
        ▼
onClick ──► (if isSecure) unlockAndRun { performToggle() }
        │   else                          performToggle()
        │
        ▼
[QS panel closes]
        │
        ▼
onStopListening ──► cancel scope
        │
        ▼
onDestroy (when system reclaims)
```

### Threading model

```kotlin
@AndroidEntryPoint
internal class NrLatchTileService : TileService() {

    @Inject lateinit var controller: NetworkModeController
    @Inject lateinit var policyEngine: PolicyEngine
    @Inject lateinit var subscriptions: SubscriptionProvider

    private var scope: CoroutineScope? = null

    override fun onStartListening() {
        super.onStartListening()
        scope = CoroutineScope(Dispatchers.Main + SupervisorJob())
        scope!!.launch { refreshState() }
    }

    override fun onStopListening() {
        scope?.cancel()
        scope = null
        super.onStopListening()
    }

    override fun onClick() {
        if (isSecure) unlockAndRun { trigger() } else trigger()
    }

    private fun trigger() {
        scope?.launch { performToggle() } ?: run {
            // QS already gone; nothing to do.
        }
    }
}
```

`scope` is `Dispatchers.Main + SupervisorJob()`. The interface implementations switch to `Dispatchers.IO` themselves; the tile coroutine stays on Main so it can mutate `qsTile` without re-dispatching.

### Click handling — `performToggle`

```kotlin
private suspend fun performToggle() {
    val sub = subscriptions.getActiveSubscription() ?: run {
        markUnavailable("No active SIM"); return
    }
    val current = controller.getCurrentMode(sub.subscriptionId)
    val target = if (current == NetworkMode.NR_LTE) NetworkMode.LTE_ONLY else NetworkMode.NR_LTE
    policyEngine.applyManualOverride(sub.subscriptionId, target)
    val verify = controller.getCurrentMode(sub.subscriptionId)
    if (verify == target) markActive(target) else markFailure()
}
```

The tile must call `qsTile.updateTile()` within ~1 s of the click to satisfy the QS UX contract. The chain `getCurrentMode → applyManualOverride → getCurrentMode` is two `Settings.Global` reads + one write, which on the validated device class completes well inside 1 s.

### State mapping

| Mode after toggle      | `qsTile.state`        | `qsTile.label`         | `qsTile.subtitle` (API 29+) |
|------------------------|-----------------------|------------------------|------------------------------|
| `NR_LTE`               | `Tile.STATE_ACTIVE`   | `5G`                   | active SIM displayName       |
| `LTE_ONLY`             | `Tile.STATE_INACTIVE` | `LTE`                  | active SIM displayName       |
| Failure / no permission| `Tile.STATE_UNAVAILABLE` | `Permission missing` | (empty)                      |

### Error handling

- `NetworkModeController.setMode()` returning `Result.failure(SecurityException)` ⇒ set tile to `STATE_UNAVAILABLE`, label `Permission missing`, surface a diagnostic toast `"Run: adb shell pm grant net.nrlatch android.permission.WRITE_SECURE_SETTINGS"`.
- Any other failure ⇒ keep prior label, set state `STATE_UNAVAILABLE` for one render, and immediately schedule another `refreshState()` to recover on the next listen cycle.
- All exceptions in the coroutine are caught within `performToggle`; uncaught exceptions are logged at `Log.e` with tag `NRLatch.Tile` and otherwise swallowed — `TileService` cannot crash the QS panel.

### Security gating

- `BIND_QUICK_SETTINGS_TILE` ensures only `system_server` may bind the service.
- `isSecure` + `unlockAndRun` ensures a locked-screen tap goes through device unlock before any privileged write.
- The service performs the toggle by going through `PolicyEngine.applyManualOverride`, which is the single entry point invariant required by [`00-overview.md`](./00-overview.md). It never branches on intent extras.

### KPI

Tap the tile ten times in a row on a Galaxy S24+ with WRITE_SECURE_SETTINGS held: state alternates between `5G`/`LTE`, the radio mode reported by `*#*#4636#*#*` follows within five seconds each time, and `dumpsys settings global` reflects the written value at `preferred_network_mode<subId>`.

---

## 2. `DataSubscriptionChangedReceiver` — `:feature:simswitch`

### Purpose

Re-applies the saved per-cardId preference when the user changes the **default data subscription**, defeating One UI's tendency to reset the preferred network mode to vendor default during the SIM switch.

### Manifest declaration

```xml
<receiver
    android:name="net.nrlatch.feature.simswitch.DataSubscriptionChangedReceiver"
    android:exported="false">
    <intent-filter>
        <action android:name="android.telephony.action.DEFAULT_DATA_SUBSCRIPTION_CHANGED"/>
    </intent-filter>
</receiver>
```

`DEFAULT_DATA_SUBSCRIPTION_CHANGED` is an AOSP **protected broadcast**: only the system can send it, so `exported="false"` is sufficient and no signature permission is required.

### Lifecycle

A manifest receiver is instantiated per delivery; `onReceive` runs on the main thread. We extend lifetime via `goAsync()` so the 1500 ms re-apply delay can complete.

```
[system sends ACTION_DEFAULT_DATA_SUBSCRIPTION_CHANGED]
        │
        ▼
onReceive (main)
        │
        ├─ validate intent.action
        ├─ pendingResult = goAsync()
        ▼
launch coroutine on Dispatchers.IO
        │
        ├─ delay(1500)                         ← settling window
        ├─ sub = SubscriptionProvider.getActiveSubscription()
        ├─ decision = PolicyEngine.resolve(sub.subscriptionId)
        ├─ current = NetworkModeController.getCurrentMode(sub.subscriptionId)
        ├─ if (decision.targetMode != current) NetworkModeController.setMode(...)
        ▼
pendingResult.finish()
```

The 1500 ms delay is empirically necessary: One UI's own SIM-switch handlers re-write the preferred network mode after the broadcast, so an immediate re-apply gets clobbered. 1500 ms gives the framework time to settle before our write lands.

### Threading model

`onReceive` returns immediately after `goAsync()` and `launch`. Work happens on a Hilt-injected `CoroutineScope` bound to `SupervisorJob() + Dispatchers.IO`. The receiver instance and its `pendingResult` are captured in the launched lambda; `pendingResult.finish()` is in a `finally` block.

```kotlin
@AndroidEntryPoint
internal class DataSubscriptionChangedReceiver : BroadcastReceiver() {

    @Inject lateinit var controller: NetworkModeController
    @Inject lateinit var policyEngine: PolicyEngine
    @Inject lateinit var subscriptions: SubscriptionProvider
    @Inject @AppScope lateinit var scope: CoroutineScope

    override fun onReceive(context: Context, intent: Intent) {
        if (intent.action != "android.telephony.action.DEFAULT_DATA_SUBSCRIPTION_CHANGED") return
        val pending = goAsync()
        scope.launch(Dispatchers.IO) {
            try {
                delay(1500)
                val sub = subscriptions.getActiveSubscription() ?: return@launch
                val decision = policyEngine.resolve(sub.subscriptionId)
                val current = controller.getCurrentMode(sub.subscriptionId)
                if (decision.targetMode != current && decision.targetMode != NetworkMode.UNKNOWN) {
                    controller.setMode(sub.subscriptionId, decision.targetMode)
                }
            } finally {
                pending.finish()
            }
        }
    }
}
```

### Idempotency

If the resolved mode already matches the current mode, the receiver is a no-op. The DataStore last-broadcast timestamp is updated regardless (so `MainActivity` can show "last seen SIM-switch broadcast" diagnostics).

### Error handling

- Defensive action-string check at entry; if mismatched, return without `goAsync()`.
- Any exception inside the coroutine is caught at the outer `try/finally`; logged at `Log.w` with tag `NRLatch.SimSwitch`. `pendingResult.finish()` always runs.
- A `setMode` failure surfaces only in logs; the next QS-tile open will rediscover and reflect the inconsistency.

### Security gating

- `exported="false"`: no third-party app can deliver this intent.
- The action is a protected broadcast: only `system_server` sends it.
- The receiver does **not** read intent extras to decide what to write. It re-reads state from `SubscriptionProvider` and `PolicyEngine`. This satisfies the architectural invariant.

### KPI

With two active SIMs in the S24+, switch the default data SIM in the system Settings UI ten times. Each switch results in a log line `NRLatch.SimSwitch reapplied <mode> for cardId=<x>`, and `dumpsys settings global | grep preferred_network_mode` shows the saved per-cardId preference for the new active SIM within 3 seconds of the switch.

---

## 3. `BootCompletedReceiver` — `:feature:simswitch`

### Purpose

Best-effort re-application of the saved preference for the active data SIM at device boot.

### Manifest declaration

In `:feature:simswitch`'s `AndroidManifest.xml`:

```xml
<uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED"/>

<receiver
    android:name="net.nrlatch.feature.simswitch.BootCompletedReceiver"
    android:exported="false">
    <intent-filter>
        <action android:name="android.intent.action.BOOT_COMPLETED"/>
    </intent-filter>
</receiver>
```

### Lifecycle and threading

Identical mechanism to `DataSubscriptionChangedReceiver`: validate action, `goAsync()`, launch on `Dispatchers.IO`, resolve active SIM, resolve decision, compare, write if different, `pendingResult.finish()`. No 1500 ms delay required at boot because there is no concurrent vendor handler racing us — but a small 500 ms delay is included to allow `SubscriptionManager` to populate.

### Reliability caveat (must be in the doc)

> **Samsung Sleeping Apps** may prevent `BootCompletedReceiver` from firing if NRLatch has not been opened recently. This is a One UI behaviour outside our control. The application falls back to the user opening the QS panel and tapping the tile, which always re-applies via `PolicyEngine.applyManualOverride` even on a freshly-booted device. `MainActivity` provides a CTA that deep-links to Samsung's Sleeping Apps settings so the user can exempt NRLatch.

### Security gating

- `exported="false"`.
- `RECEIVE_BOOT_COMPLETED` requested.
- No intent-extras-based dispatch; same invariant as the SIM-switch receiver.

### KPI

After granting WRITE_SECURE_SETTINGS and writing a non-default preference, reboot the device. Within 30 seconds of reaching the home screen with the tile not opened and Sleeping Apps disabled for NRLatch, the radio mode matches the saved preference and a log line `NRLatch.Boot reapplied <mode>` is present.

---

## 4. `MainActivity` — `:app`

### Purpose

The single launcher activity. Its responsibilities are strictly:

1. Show the application title and a short description.
2. Indicate whether `WRITE_SECURE_SETTINGS` is currently granted (`PackageManager.PERMISSION_GRANTED`).
3. Display the ADB grant command in a monospace `TextView` and provide a Copy button.
4. Show diagnostics: last received SIM-switch broadcast timestamp (read from a DataStore key updated by the receiver), last toggle outcome, current per-SIM stored preferences.
5. Provide a CTA to request battery-optimization exemption (`Settings.ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS`).
6. Provide a CTA that deep-links to Samsung's Sleeping Apps settings if the device exposes the relevant intent component.

**MainActivity has no toggle controls.** All toggling happens in the Quick Settings tile.

### Layout

XML, `androidx.constraintlayout.widget.ConstraintLayout` root. No Compose anywhere in the project. Scrollable single screen. The root view sets `android:filterTouchesWhenObscured="true"` to prevent overlay-based tap hijacking on a screen that displays clipboard-copyable strings.

```xml
<androidx.constraintlayout.widget.ConstraintLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:filterTouchesWhenObscured="true"
    android:padding="16dp">

    <!-- Title, status row, ADB command box, Copy button,
         diagnostics box, battery CTA, Sleeping Apps CTA -->
</androidx.constraintlayout.widget.ConstraintLayout>
```

### Manifest declaration

```xml
<activity
    android:name=".MainActivity"
    android:exported="true"
    android:taskAffinity="">
    <intent-filter>
        <action android:name="android.intent.action.MAIN"/>
        <category android:name="android.intent.category.LAUNCHER"/>
    </intent-filter>
</activity>
```

`taskAffinity=""` puts the activity in its own task to limit task-hijack vectors.

### Threading

Permission status check on `onResume` via `checkSelfPermission(Manifest.permission.WRITE_SECURE_SETTINGS)`. Diagnostics flows collected in `lifecycleScope.launch(Dispatchers.Main.immediate)` with `repeatOnLifecycle(STARTED)`.

### Architectural rule (must be in the doc)

> **MainActivity must NEVER call `NetworkModeController.setMode()` or `PolicyEngine.applyManualOverride()`** — directly or indirectly — based on intent data. It only reads state, displays instructions, and copies strings. This rule is enforced by the custom CI lint rule documented in `ci-lint-spec.md` (the rule scans `:app` for any direct or transitive call from `MainActivity` into `NetworkModeController.setMode` or `PolicyEngine.applyManualOverride`).

### Error handling

- If the Sleeping Apps deep-link intent fails to resolve, the CTA is hidden.
- Clipboard copy failure (rare) shows a `Snackbar` and falls back to displaying the command for manual copy.
- All other UI state is purely declarative and cannot fail.

### Security gating

- `exported="true"` is required for the launcher entry; the activity has no other intent filters and accepts no extras.
- The activity does not register dynamic receivers; it observes state via injected interfaces backed by DataStore and `SubscriptionManager`.

### KPI

Launching the app shows the correct permission state (matching `dumpsys package net.nrlatch | grep WRITE_SECURE_SETTINGS`), the Copy button places the exact ADB command on the clipboard, and the diagnostics panel updates within 2 s of toggling the data SIM in system Settings.

---

## 5. `PreferenceStoreImpl` — `:core:preferences`

### Purpose

The DataStore-Preferences-backed implementation of [`PreferenceStore`](./02-public-interfaces.md#preferencestore--corepreferences).

### Storage

- Backend: `androidx.datastore.preferences` (NOT Proto — minimal schema, no migration concerns).
- File: `nrlatch_prefs` in the app's internal storage (default DataStore location).
- Backup: disabled at the app level via `android:allowBackup="false"` on `<application>` in `:app`'s manifest.

### Key shape

```kotlin
private fun keyFor(cardId: Int) = intPreferencesKey("mode_card_$cardId")
```

Stored value is the `Int` from `NetworkMode.settingsValue`. Reads map back via `NetworkMode.fromSettingsValue`.

### Threading

DataStore handles its own IO; the impl decorates writes with `withContext(Dispatchers.IO)` purely so callers retain dispatcher discipline.

### Read implementation sketch

```kotlin
override fun getPreferredMode(cardId: Int): Flow<NetworkMode?> =
    dataStore.data.map { prefs ->
        prefs[keyFor(cardId)]?.let(NetworkMode::fromSettingsValue)
    }
```

Default contract: `null` when no key exists. `NetworkMode.UNKNOWN` is mapped through faithfully if a stale value somehow ends up in storage (it should not).

### `getAllPreferences()`

```kotlin
override fun getAllPreferences(): Flow<List<PerSimPreference>> =
    dataStore.data.map { prefs ->
        prefs.asMap()
            .mapNotNull { (k, v) ->
                val name = k.name
                if (!name.startsWith("mode_card_") || v !is Int) return@mapNotNull null
                val cardId = name.removePrefix("mode_card_").toIntOrNull() ?: return@mapNotNull null
                PerSimPreference(cardId, NetworkMode.fromSettingsValue(v))
            }
    }
```

### KPI

Unit tests with an in-memory DataStore: write, read, observe-then-mutate, and clear all behave per the KDoc contract.

---

## 6. `NetworkModeControllerImpl` — `:core:settings`

### Purpose

The privileged-write surface. Only one of these in the app.

### Implementation sketch

```kotlin
internal class NetworkModeControllerImpl @Inject constructor(
    @ApplicationContext private val context: Context,
) : NetworkModeController {

    private val cr get() = context.contentResolver

    override suspend fun getCurrentMode(subscriptionId: Int): NetworkMode =
        withContext(Dispatchers.IO) {
            val raw = Settings.Global.getInt(
                cr,
                "preferred_network_mode$subscriptionId",
                NetworkMode.NR_LTE.settingsValue,
            )
            NetworkMode.fromSettingsValue(raw)
        }

    override suspend fun setMode(subscriptionId: Int, mode: NetworkMode): Result<Unit> =
        withContext(Dispatchers.IO) {
            if (mode == NetworkMode.UNKNOWN) {
                return@withContext Result.failure(IllegalArgumentException("Cannot write UNKNOWN"))
            }
            runCatching {
                val ok = Settings.Global.putInt(
                    cr,
                    "preferred_network_mode$subscriptionId",
                    mode.settingsValue,
                )
                if (!ok) error("Settings.Global.putInt returned false")
            }
        }

    override suspend fun toggle(subscriptionId: Int): Result<NetworkMode> {
        val current = getCurrentMode(subscriptionId)
        val target = if (current == NetworkMode.NR_LTE) NetworkMode.LTE_ONLY else NetworkMode.NR_LTE
        return setMode(subscriptionId, target).map { target }
    }
}
```

### Threading

All public methods `withContext(Dispatchers.IO)`. The provider context is the singleton `@ApplicationContext` from Hilt.

### Spike-validated assumption

> On Samsung Galaxy S24+ with `WRITE_SECURE_SETTINGS` granted, `Settings.Global.putInt(contentResolver, "preferred_network_mode<subId>", value)` triggers a radio mode switch within ~2–5 seconds. This is the assumption underlying the entire architecture as of freeze. If the validation spike (ADR-001 / `07-validation-spike.md`) shows this insufficient on the target device class — for example if a `TelephonyManager.setPreferredNetworkType` call gated by `MODIFY_PHONE_STATE` is additionally required — this implementation must be revised. Such a revision changes the permission model and is therefore a Phase 1 architecture change, not a v1.0 bug.

### Error handling

- `SecurityException` ⇒ `Result.failure`. Tile maps to `STATE_UNAVAILABLE`.
- `putInt` returning `false` ⇒ `Result.failure(IllegalStateException(...))`. Caller may retry; the receiver does not.
- `IllegalArgumentException` for `UNKNOWN` mode is a programming error; included for defence.

### Security gating

- This is the only class allowed to call `Settings.Global.putInt` for `preferred_network_mode*`. Enforced by lint (see `ci-lint-spec.md`).
- The class itself is `internal`; consumers go through the `NetworkModeController` interface.

### KPI

Manual test: with the permission held, calls succeed and the radio mode follows. Without the permission, calls return `Result.failure(SecurityException)` and `dumpsys settings global` shows no change.

---

## 7. `SubscriptionProviderImpl` — `:core:telephony`

### Purpose

Wraps `SubscriptionManager` for the rest of the app. Maps framework `SubscriptionInfo` to NRLatch's `SimIdentity`.

### Implementation sketch

```kotlin
internal class SubscriptionProviderImpl @Inject constructor(
    @ApplicationContext private val context: Context,
) : SubscriptionProvider {

    private val sm: SubscriptionManager
        get() = context.getSystemService(SubscriptionManager::class.java)

    override fun getActiveSubscription(): SimIdentity? {
        val defaultDataId = SubscriptionManager.getDefaultDataSubscriptionId()
        if (defaultDataId == SubscriptionManager.INVALID_SUBSCRIPTION_ID) return null
        return sm.getActiveSubscriptionInfoList()
            ?.firstOrNull { it.subscriptionId == defaultDataId }
            ?.toSimIdentity()
    }

    override fun getAllSubscriptions(): List<SimIdentity> =
        sm.getActiveSubscriptionInfoList().orEmpty().map { it.toSimIdentity() }

    override fun observeActiveSubscription(): Flow<SimIdentity?> = callbackFlow {
        val listener = object : SubscriptionManager.OnSubscriptionsChangedListener() {
            override fun onSubscriptionsChanged() { trySend(getActiveSubscription()) }
        }
        trySend(getActiveSubscription())
        sm.addOnSubscriptionsChangedListener(context.mainExecutor, listener)
        awaitClose { sm.removeOnSubscriptionsChangedListener(listener) }
    }

    private fun SubscriptionInfo.toSimIdentity() = SimIdentity(
        subscriptionId = subscriptionId,
        cardId = cardId, // SubscriptionInfo.getCardId() — ADR-003.
        displayName = displayName?.toString() ?: "SIM $simSlotIndex",
    )
}
```

### Permissions

- `READ_PHONE_STATE` is requested in `:app`'s merged manifest. `MainActivity` checks at runtime and prompts.
- `getCardId()` returns a stable card identifier with `READ_PHONE_STATE` (`UNINITIALIZED_CARD_ID` ≡ `-2` if unavailable). No `READ_PRIVILEGED_PHONE_STATE` is required.

### Threading

Synchronous accessors are main-safe (in-process binder calls). `observeActiveSubscription` registers on `context.mainExecutor`; `callbackFlow` bridges to coroutine context.

### Security gating

- Class is `internal`.
- No exported entry point.

### KPI

Unit tests with `mockk`-mocked `SubscriptionManager` cover: zero SIMs returns null, two SIMs returns the active-data one, listener callbacks emit through the flow. Instrumentation test on the S24+ verifies `cardId` is non-`-2` for both physical SIM slots.

---

## DI module sketch

Each `:core:*` module ships an `internal @Module @InstallIn(SingletonComponent::class)` with `@Binds` mappings:

```kotlin
@Module
@InstallIn(SingletonComponent::class)
internal abstract class SettingsModule {
    @Binds @Singleton
    abstract fun bindController(impl: NetworkModeControllerImpl): NetworkModeController
}
```

`:core:policy` additionally binds the `PolicyEngine` to `ManualOnlyPolicyEngine`. `:app` binds the application-wide `CoroutineScope` qualified `@AppScope` for receiver use. There are no `@Provides` for stateful singletons that hold `Context` — Hilt's `@ApplicationContext` is injected directly into impl constructors.
