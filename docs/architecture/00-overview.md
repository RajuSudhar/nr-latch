# NRLatch — Phase 1 Architecture Overview

## Purpose

NRLatch is a single-user Android utility for the Samsung Galaxy S24+ that toggles the device's preferred network mode between value `33` (NR + LTE, i.e. 5G enabled) and value `9` (LTE-only) by writing to `Settings.Global` under the key `preferred_network_mode<subId>`. The write requires the `WRITE_SECURE_SETTINGS` permission, which is granted to the application a single time over ADB. The user-facing surface is a Quick Settings tile; there are no in-app controls beyond ADB-grant instructions and diagnostics. The application is fully client-side: zero network calls post-install, no telemetry, no backends.

## Phase 1 Scope

### In scope

- Eight Gradle modules: `:app`, `:core:model`, `:core:settings`, `:core:telephony`, `:core:preferences`, `:core:policy`, `:feature:tile`, `:feature:simswitch`.
- A `TileService` (`NrLatchTileService`) bound to the system Quick Settings panel.
- Two manifest-declared `BroadcastReceiver`s: `DataSubscriptionChangedReceiver` and `BootCompletedReceiver`.
- A `MainActivity` whose sole responsibilities are: showing the ADB-grant command, displaying permission status, surfacing diagnostics, and providing CTAs for battery exemption and Samsung Sleeping Apps settings. It performs no toggling.
- Per-SIM preference persistence keyed by `SubscriptionInfo.getCardId()` (Int, stable per physical SIM card, available on API 29+ with `READ_PHONE_STATE`).
- Auto-restore of the saved preference on default-data-SIM switch via `android.telephony.action.DEFAULT_DATA_SUBSCRIPTION_CHANGED`.
- Best-effort re-application of the saved preference on `BOOT_COMPLETED`.
- A validation spike (ADR-001) that proves the `Settings.Global` write triggers an actual radio mode change on the target device class.
- Security hardening: `INTERNET` removed from the merged manifest, all impl classes `internal`, `allowBackup="false"`, `filterTouchesWhenObscured="true"` on the activity root, no exported component capable of triggering a privileged write from intent data.

### Out of scope

- Any battery-aware policy beyond user-toggled overrides.
- Hotspot, Wi-Fi extender, Tasker integration.
- Wear OS, DeX, Android Auto.
- Material You dynamic theming.
- Jetpack Compose.
- A formal domain layer (use-cases / interactors).
- The `Repository` + `DataSource` split.
- `:api` / `:impl` module splits.
- Play Store distribution.

## Non-negotiable Constraints

1. **Fully client-side.** Zero network calls post-install. No telemetry. No Firebase, no Crashlytics, no analytics SDKs, no first-party HTTP usage.
2. **`INTERNET` permission must be removed from the merged manifest** via `<uses-permission android:name="android.permission.INTERNET" tools:node="remove"/>` in `:app`'s manifest. Any transitively-merged permission is excised at build time.
3. **Toolchain pinned**:
   - Kotlin `2.3.20`
   - AGP `9.1.0`
   - Gradle `9.4.1`
   - JDK `21` (build), JVM target `17` (bytecode)
   - KSP `2.3.20-x` (no kapt)
   - Hilt `2.56.2`
   - DataStore `1.2.1`
   - `minSdk` `29`, `targetSdk` `35`, `compileSdk` `35`, SDK Build Tools `36.0.0`
   - R8 full mode
4. **Architectural invariant (CI-enforced).** No exported component performs a privileged `Settings.Global` write based on intent data. `TileService.onClick` is the only entry point that may invoke `NetworkModeController.setMode()` directly. Receivers may only re-apply the *already-stored* preference. `MainActivity` may not invoke `setMode()` at all. See `ci-lint-spec.md`.

## Success Definition

Phase 1 is successful when both of the following are true:

1. A competent Android engineer, given only the documents in this directory, can implement v1.0 end-to-end without asking design questions. Module boundaries, public interface signatures, threading contracts, manifest declarations, and security gates are all specified to the level required for direct implementation.
2. The validation spike (ADR-001) has empirically demonstrated, on a Samsung Galaxy S24+ running One UI on the latest stable Android version, that writing `33` and `9` to `Settings.Global["preferred_network_mode<subId>"]` produces an observable radio mode change (NR-RSRP visible / not visible in `*#*#4636#*#*` or equivalent diagnostics) within five seconds.

## Document Map

1. [`00-overview.md`](./00-overview.md) — this document.
2. [`01-module-graph.md`](./01-module-graph.md) — the eight-module structure, dependency direction, and per-module Gradle wiring.
3. [`02-public-interfaces.md`](./02-public-interfaces.md) — the five public interface contracts (`NetworkModeController`, `SubscriptionProvider`, `PreferenceStore`, `ModePolicy`, `PolicyEngine`) plus model types.
4. [`03-component-designs.md`](./03-component-designs.md) — `TileService`, both `BroadcastReceiver`s, `MainActivity`, and the four DI implementation classes.
5. [`04-security-hardening.md`](./04-security-hardening.md) — manifest hardening, build config, signing keystore policy, FLAG_IMMUTABLE mandate, defense-in-depth, "deliberately not done" list.
6. [`05-samsung-reliability.md`](./05-samsung-reliability.md) — Samsung 5/5 dontkillmyapp realities, first-launch UX flow, diagnostic screen, anti-patterns, acceptance criteria.
7. [`06-validation-spike.md`](./06-validation-spike.md) — BLOCKING pre-implementation 2-hour spike spec; exit criteria for ADR-001.
8. [`07-testing-strategy.md`](./07-testing-strategy.md) — JVM ≥80% coverage targets, `TileToggleHandler`/`SimSwitchHandler` extraction, no Robolectric/Espresso/Pitest.
9. [`08-ci-lint-rules.md`](./08-ci-lint-rules.md) — custom `NrLatchSetModeDetector` lint rule enforcing the architectural invariant; dependency-analysis config; manifest-grep gate.
10. [`adr-001-validation-spike.md`](./adr-001-validation-spike.md) — PROPOSED; result template to be filled post-spike.
11. [`adr-002-boot-completed.md`](./adr-002-boot-completed.md) — ACCEPTED; BOOT_COMPLETED in-scope, best-effort.
12. [`adr-003-preference-key.md`](./adr-003-preference-key.md) — ACCEPTED; `cardId` as preference key, ICCID rejected (privileged-permission gated on API 29+).
13. [`threat-model.md`](./threat-model.md) — 10-row threat table mapped to specific hardening items in `04-security-hardening.md` and `08-ci-lint-rules.md`; risks-accepted-without-mitigation section.
14. [`checklist.md`](./checklist.md) — phased trackable checklist from BLOCKING spike through release tagging.

## Glossary

| Term | Definition |
|------|------------|
| **NR** | New Radio. The 3GPP physical-layer designation for 5G. In `Settings.Global` the constant `33` (`NETWORK_MODE_NR_LTE`) selects NR + LTE. |
| **LTE** | Long-Term Evolution. The 4G radio mode. Constant `9` (`NETWORK_MODE_LTE_ONLY`) selects LTE-only. |
| **QS** | Quick Settings. The notification-shade tile area where `TileService`-backed tiles appear. |
| **subId** | Subscription ID. An `Int` assigned by the framework to each active `SubscriptionInfo`; not stable across SIM re-insertions. The `Settings.Global` key is suffixed with this value: `preferred_network_mode<subId>`. |
| **cardId** | The integer identifier of the SIM card slot/eUICC profile assigned by the framework. Stable across SIM re-insertions for the same physical card. NRLatch keys preferences by cardId, not subId. See ADR-003. |
| **ICCID** | Integrated Circuit Card Identifier. The 19-20 digit serial of the SIM. Returned by `SubscriptionInfo.getIccId()` only with `READ_PRIVILEGED_PHONE_STATE` (signature\|privileged) on API 29+; **not used** by NRLatch. |
| **DSDS** | Dual SIM Dual Standby. The S24+ supports DSDS; NRLatch persists a per-cardId preference and applies the one matching the active data subscription. |
| **WRITE_SECURE_SETTINGS** | The `signature\|privileged\|development` permission required to write `Settings.Global`. Granted via `adb shell pm grant net.nrlatch android.permission.WRITE_SECURE_SETTINGS`. |
