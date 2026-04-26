# NRLatch — Public Interfaces

This document specifies every public interface and value type that crosses a module boundary in v1.0. The signatures here are the **stable v1.0 contract**: any change requires a major version bump.

For the modules these types live in and the dependencies between them, see [`01-module-graph.md`](./01-module-graph.md). For the implementation classes that satisfy these interfaces, see [`03-component-designs.md`](./03-component-designs.md).

---

## Value types — `:core:model`

```kotlin
package net.nrlatch.core.model

/**
 * The set of preferred-network-mode values NRLatch knows how to write to
 * [android.provider.Settings.Global] under the key
 * `preferred_network_mode<subId>`.
 *
 * The integer in [settingsValue] is the underlying framework constant.
 *  - [NR_LTE]   = 33 — `TelephonyManager.NETWORK_MODE_NR_LTE`. NR (5G) + LTE fallback.
 *  - [LTE_ONLY] =  9 — `TelephonyManager.NETWORK_MODE_LTE_ONLY`.
 *  - [UNKNOWN]  = -1 — sentinel returned when the underlying value is absent
 *                       or does not match a known constant. Never written.
 */
enum class NetworkMode(val settingsValue: Int) {
    NR_LTE(33),
    LTE_ONLY(9),
    UNKNOWN(-1);

    companion object {
        /**
         * Maps a framework integer (typically read from `Settings.Global`) to a [NetworkMode].
         * Returns [UNKNOWN] for unrecognized values rather than throwing, because
         * other apps and OS components may write values we do not enumerate.
         */
        fun fromSettingsValue(value: Int): NetworkMode =
            entries.firstOrNull { it.settingsValue == value } ?: UNKNOWN
    }
}

/**
 * Identity of a SIM subscription as observed by NRLatch.
 *
 * @property subscriptionId framework subId — **unstable across SIM re-insertions**.
 *                          Used only for the immediate `Settings.Global` key suffix.
 * @property cardId         stable per physical SIM card / eUICC profile.
 *                          NRLatch keys preferences on this value. See ADR-003.
 * @property displayName    user-facing carrier or slot label, e.g. "Vi" or "SIM 1".
 */
data class SimIdentity(
    val subscriptionId: Int,
    val cardId: Int,
    val displayName: String,
)

/**
 * The decision returned by a [PolicyEngine] for a given subscription.
 *
 * @property targetMode the mode the engine recommends.
 * @property source     why this decision was produced.
 * @property reason     human-readable diagnostic; surfaced in logs and the
 *                      MainActivity diagnostic panel.
 */
data class PolicyDecision(
    val targetMode: NetworkMode,
    val source: PolicySource,
    val reason: String,
)

/**
 * Provenance of a [PolicyDecision].
 *  - [MANUAL]  user pressed the QS tile.
 *  - [POLICY]  a registered [ModePolicy] returned the decision (unused in v1.0).
 *  - [DEFAULT] no manual override and no policy fired; default applied.
 */
enum class PolicySource { MANUAL, POLICY, DEFAULT }

/**
 * A persisted user preference for one physical SIM card.
 *
 * @property cardId from [SimIdentity.cardId].
 * @property mode   the preferred [NetworkMode] for this card.
 */
data class PerSimPreference(
    val cardId: Int,
    val mode: NetworkMode,
)
```

**Stability**: every type above is part of the v1.0 stable API. Adding new enum constants is permitted; removing or renumbering is not.

---

## `NetworkModeController` — `:core:settings`

**Module**: `:core:settings`. **Threading**: all methods are `suspend` and the implementation switches to `Dispatchers.IO`. Callers may invoke from the main dispatcher freely; the call will not block.

```kotlin
package net.nrlatch.core.settings

import net.nrlatch.core.model.NetworkMode

/**
 * Reads and writes the device's preferred network mode for a specific subscription.
 *
 * The implementation wraps `Settings.Global.getInt`/`putInt` against the key
 * `preferred_network_mode<subscriptionId>`. The write requires
 * `WRITE_SECURE_SETTINGS`, granted once via ADB.
 *
 * **Stability**: v1.0 stable.
 */
interface NetworkModeController {

    /**
     * Returns the current [NetworkMode] reported by `Settings.Global` for the
     * given [subscriptionId]. Returns [NetworkMode.UNKNOWN] if the stored
     * value is not one this app recognizes.
     *
     * Threading: safe to call from any dispatcher; impl uses `Dispatchers.IO`.
     */
    suspend fun getCurrentMode(subscriptionId: Int): NetworkMode

    /**
     * Writes [mode] to `Settings.Global` for the given [subscriptionId].
     *
     * Failure modes (the [Result.failure] case wraps these):
     *  - `SecurityException`     `WRITE_SECURE_SETTINGS` not granted.
     *  - `IllegalArgumentException` if [mode] is [NetworkMode.UNKNOWN].
     *  - `IllegalStateException` if `Settings.Global.putInt` returns `false`
     *    (vendor-side rejection — observed on some One UI builds when the
     *    radio is mid-handover; callers may retry).
     *
     * Threading: safe to call from any dispatcher; impl uses `Dispatchers.IO`.
     * The radio mode change is asynchronous and typically observable within
     * 2–5 s on the validated device class (see ADR-001 / 07-validation-spike).
     */
    suspend fun setMode(subscriptionId: Int, mode: NetworkMode): Result<Unit>

    /**
     * Reads the current mode and writes the opposite of {[NetworkMode.NR_LTE],
     * [NetworkMode.LTE_ONLY]}. If the current value is anything else
     * (including [NetworkMode.UNKNOWN]), the target defaults to
     * [NetworkMode.NR_LTE].
     *
     * On success, [Result] wraps the new mode actually written.
     * Failure modes: same as [setMode].
     *
     * Threading: safe to call from any dispatcher; impl uses `Dispatchers.IO`.
     */
    suspend fun toggle(subscriptionId: Int): Result<NetworkMode>
}
```

---

## `SubscriptionProvider` — `:core:telephony`

**Module**: `:core:telephony`. **Threading**: synchronous accessors are safe to call from the main dispatcher (`SubscriptionManager` is in-process). `observeActiveSubscription()` returns a cold `Flow` whose collection should occur on a non-main dispatcher when the consumer additionally fans out work; the impl emits on the framework callback's binder thread before bridging into a coroutine via `callbackFlow`.

```kotlin
package net.nrlatch.core.telephony

import kotlinx.coroutines.flow.Flow
import net.nrlatch.core.model.SimIdentity

/**
 * Reads SIM subscription state from the framework.
 *
 * Requires `READ_PHONE_STATE` to be held by the application. Calls made
 * without the permission throw `SecurityException` from the framework;
 * callers should treat that as a programming error (the manifest declares
 * the permission and `MainActivity` checks it on startup).
 *
 * **Stability**: v1.0 stable.
 */
interface SubscriptionProvider {

    /**
     * Returns the [SimIdentity] of the **default data subscription**
     * (`SubscriptionManager.getDefaultDataSubscriptionId()`), or `null`
     * if there is no active data SIM (airplane mode, no SIM inserted,
     * SIM disabled).
     *
     * Threading: main-safe; in-process binder call.
     */
    fun getActiveSubscription(): SimIdentity?

    /**
     * Returns all currently active subscriptions
     * (`SubscriptionManager.getActiveSubscriptionInfoList()`).
     * Empty list if none are active.
     *
     * Threading: main-safe.
     */
    fun getAllSubscriptions(): List<SimIdentity>

    /**
     * Cold flow that emits the current active data subscription whenever
     * `OnSubscriptionsChangedListener` fires, plus an initial value at
     * collection time. Emits `null` when no active data SIM is present.
     *
     * Threading: emissions arrive on the listener's binder thread, then
     * are forwarded through `callbackFlow`; consumers should `flowOn`
     * a context appropriate to their work.
     */
    fun observeActiveSubscription(): Flow<SimIdentity?>
}
```

---

## `PreferenceStore` — `:core:preferences`

**Module**: `:core:preferences`. **Threading**: read flows are cold; the writer is `suspend` and runs on `Dispatchers.IO` internally. `cardId` from `SubscriptionInfo.getCardId()` is the only key shape supported.

```kotlin
package net.nrlatch.core.preferences

import kotlinx.coroutines.flow.Flow
import net.nrlatch.core.model.NetworkMode
import net.nrlatch.core.model.PerSimPreference

/**
 * Persists per-SIM preferred [NetworkMode] keyed by `cardId`.
 *
 * Backed by Jetpack DataStore (Preferences). The DataStore file is
 * `nrlatch_prefs` in the app's internal storage; backup is disabled via
 * `allowBackup="false"` on the application manifest.
 *
 * Default behaviour: if no preference has been written for a `cardId`,
 * [getPreferredMode] emits `null`. Higher layers (`PolicyEngine`) translate
 * `null` into [NetworkMode.LTE_ONLY] as the documented application default.
 *
 * **Stability**: v1.0 stable.
 */
interface PreferenceStore {

    /**
     * Cold flow of the preferred [NetworkMode] for the given [cardId].
     * Emits `null` when no preference has ever been written for this card.
     *
     * Threading: cold; collection is dispatcher-agnostic (DataStore handles
     * its own IO).
     */
    fun getPreferredMode(cardId: Int): Flow<NetworkMode?>

    /**
     * Writes [mode] as the preference for [cardId]. Overwrites any existing
     * value. Suspends until DataStore has flushed the write.
     *
     * Threading: safe from any dispatcher; impl uses `Dispatchers.IO`.
     */
    suspend fun setPreferredMode(cardId: Int, mode: NetworkMode)

    /**
     * Cold flow of all stored preferences. Re-emits on every write.
     * Used by `MainActivity`'s diagnostic panel.
     *
     * Threading: cold.
     */
    fun getAllPreferences(): Flow<List<PerSimPreference>>

    /**
     * Removes any stored preference for [cardId]. After this returns,
     * [getPreferredMode] will emit `null` for that card.
     *
     * Threading: safe from any dispatcher; impl uses `Dispatchers.IO`.
     */
    suspend fun clearPreference(cardId: Int)
}
```

---

## `ModePolicy` — `:core:policy`

**Module**: `:core:policy`. **Threading**: `evaluate()` is `suspend` and impl-defined; `observeChanges()` is a cold flow. v1.0 has no concrete `ModePolicy` instance — the interface exists as the extensibility seam for future policies (battery, time-of-day, geofence). `ManualOnlyPolicyEngine` ignores all registered policies in v1.0.

```kotlin
package net.nrlatch.core.policy

import kotlinx.coroutines.flow.Flow
import net.nrlatch.core.model.PolicyDecision

/**
 * One pluggable policy. Registered policies are consulted by [PolicyEngine.resolve]
 * in priority order (highest [priority] wins). v1.0 ships no implementations;
 * the interface is the extensibility seam.
 *
 * **Stability**: v1.0 stable.
 */
interface ModePolicy {

    /**
     * Stable, unique identifier; used by [PolicyEngine.unregisterPolicy].
     * Convention: reverse-DNS, e.g. `net.nrlatch.policy.battery`.
     */
    val id: String

    /**
     * Higher value wins. The manual override held by the engine itself uses
     * `Int.MAX_VALUE`; concrete policies should use values well below that
     * to remain overridable.
     */
    val priority: Int

    /**
     * Returns this policy's decision for [subscriptionId], or `null` if the
     * policy declines to decide right now (e.g. its inputs are unavailable).
     *
     * Threading: suspend; impl-defined dispatcher.
     */
    suspend fun evaluate(subscriptionId: Int): PolicyDecision?

    /**
     * Cold signal that fires `Unit` whenever this policy's inputs change in
     * a way that may alter [evaluate]'s output. The engine listens on this
     * to push updates to [PolicyEngine.observeEffectiveDecision].
     */
    fun observeChanges(): Flow<Unit>
}
```

---

## `PolicyEngine` — `:core:policy`

**Module**: `:core:policy`. **Threading**: `suspend` methods run on the impl's injected dispatcher (`Dispatchers.Default`); flow methods are cold. The v1.0 implementation is `ManualOnlyPolicyEngine`.

```kotlin
package net.nrlatch.core.policy

import kotlinx.coroutines.flow.Flow
import net.nrlatch.core.model.NetworkMode
import net.nrlatch.core.model.PolicyDecision

/**
 * Resolves the effective [PolicyDecision] for a subscription by combining
 * the manual override (highest priority) with any registered [ModePolicy]s.
 *
 * v1.0 implementation: [ManualOnlyPolicyEngine] — manual override only,
 * registered policies are no-ops.
 *
 * **Stability**: v1.0 stable.
 */
interface PolicyEngine {

    /**
     * Returns the current effective decision for [subscriptionId].
     *  - If a manual override exists for the subscription's cardId, returns
     *    `PolicyDecision(mode, MANUAL, "User toggle")`.
     *  - Else if any registered policy produces a non-null decision, returns
     *    the highest-priority one (v2.0+).
     *  - Else returns `PolicyDecision(LTE_ONLY, DEFAULT, "No preference set")`.
     *
     * Threading: suspend; impl uses `Dispatchers.IO` for DataStore reads.
     */
    suspend fun resolve(subscriptionId: Int): PolicyDecision

    /**
     * Records a manual override for the cardId backing [subscriptionId] and
     * triggers `NetworkModeController.setMode()` to apply it immediately.
     * Sticky: persists until [clearManualOverride] is called.
     *
     * Threading: suspend; impl uses `Dispatchers.IO`.
     */
    suspend fun applyManualOverride(subscriptionId: Int, mode: NetworkMode)

    /**
     * Removes the manual override for the cardId backing [subscriptionId].
     * After this, [resolve] falls back to registered policies, then default.
     *
     * Threading: suspend; impl uses `Dispatchers.IO`.
     */
    suspend fun clearManualOverride(subscriptionId: Int)

    /**
     * Registers [policy] for future evaluation. v1.0 is a no-op.
     *
     * Threading: main-safe.
     */
    fun registerPolicy(policy: ModePolicy)

    /**
     * Unregisters by [policyId]. v1.0 is a no-op.
     *
     * Threading: main-safe.
     */
    fun unregisterPolicy(policyId: String)

    /**
     * Cold flow of the effective decision for [subscriptionId], re-emitting
     * whenever an override changes or any registered policy signals
     * `observeChanges()`.
     *
     * Threading: cold; safe to collect on any dispatcher.
     */
    fun observeEffectiveDecision(subscriptionId: Int): Flow<PolicyDecision>
}
```

### `ManualOnlyPolicyEngine` v1.0 contract

The v1.0 implementation behaves as follows. This is the documented runtime contract; deviations are bugs.

- **`resolve(subscriptionId)`**:
  1. Resolves the active `cardId` via `SubscriptionProvider.getAllSubscriptions().firstOrNull { it.subscriptionId == subscriptionId }?.cardId`.
  2. If the cardId is not found, returns `PolicyDecision(LTE_ONLY, DEFAULT, "No active subscription")`.
  3. Reads `PreferenceStore.getPreferredMode(cardId).first()`.
  4. If non-null, returns `PolicyDecision(mode, MANUAL, "User toggle")`.
  5. Else returns `PolicyDecision(LTE_ONLY, DEFAULT, "No preference set")`.
- **`applyManualOverride(subscriptionId, mode)`**:
  1. Resolves cardId as above.
  2. Calls `PreferenceStore.setPreferredMode(cardId, mode)`.
  3. Calls `NetworkModeController.setMode(subscriptionId, mode)`.
  4. Surfaces failures by completing normally; the QS tile observes failure via the controller's `Result` and updates state.
- **`clearManualOverride(subscriptionId)`**: resolves cardId, calls `PreferenceStore.clearPreference(cardId)`. Does **not** revert the radio mode — the user can press the tile again to re-toggle.
- **Manual override priority**: conceptually `Int.MAX_VALUE`. Sticky until `clearManualOverride` is invoked.
- **`registerPolicy` / `unregisterPolicy`**: no-ops in v1.0. Implementations should accept the call and return without throwing.
- **`observeEffectiveDecision(subscriptionId)`**: emits the result of `resolve(subscriptionId)` on first collection and re-emits whenever `PreferenceStore.getPreferredMode(cardId)` emits.

For the wiring of `ManualOnlyPolicyEngine` into Hilt, the interaction with `NetworkModeController`, and the threading model in detail, see [`03-component-designs.md`](./03-component-designs.md).
