# Threat Model

This document enumerates the threats considered for NRLatch v1.0 and maps
each to a specific mitigation in the codebase or build configuration. The
table is the canonical reference; row numbers match priority order.

The mitigations referenced here are defined in:

- [`04-security-hardening.md`](04-security-hardening.md) — manifest, build,
  and architectural invariants.
- [`05-samsung-reliability.md`](05-samsung-reliability.md) — Samsung-specific
  reliability mitigations.
- [`08-ci-lint-rules.md`](08-ci-lint-rules.md) — CI gates that enforce the
  invariants at build time.

## Scope

NRLatch is a single-user developer utility on a Samsung Galaxy S24+. The
threat model assumes:

- A non-rooted device under the user's physical control.
- The user installs only NRLatch (signed with the dedicated release
  keystore) plus normal third-party apps from Google Play.
- ADB access is restricted to the user's PC.
- The device is *not* enrolled in MDM / Knox enterprise management.
- The user is not a target of state-actor or sophisticated supply-chain
  attacks (i.e. Frida/Xposed/Magisk on the user's *own* device is out of
  scope; a compromised third-party app on the same device is in scope).

## Threat × Mitigation table

| # | Threat | Likelihood | Impact | Priority | Mitigation in NRLatch |
|---|---|---|---|---|---|
| 1 | App compromised via supply-chain dependency. Attacker uses NRLatch's `WRITE_SECURE_SETTINGS` to enable accessibility services / set a malicious DNS / enable wireless ADB. | Low | Critical | 1 | (a) Minimal dependency surface — only AndroidX core, Hilt, DataStore, Coroutines. No analytics, Firebase, network, or telemetry libs (see `01-module-graph.md`). (b) R8 full-mode minify+shrink reduces attack surface (see `04-security-hardening.md` §3). (c) `INTERNET` permission stripped via `tools:node="remove"` (see §1). (d) Deny-all `network_security_config.xml` (see §2.2). (e) `usesCleartextTraffic="false"` on `<application>` (see §1). (f) **Architectural invariant**: `NetworkModeController.setMode()` callable only from QS tile + protected receivers, never from intent-data paths (see §4.1). (g) Enforced at build time by custom lint rule `NrLatchSetModeFromExportedComponent` (see `08-ci-lint-rules.md` §1). |
| 2 | Physical attacker with ADB access (lost/stolen device with developer mode enabled) extracts DataStore preferences via `adb backup` or device-transfer flow. | Medium | Low (only mode preferences leak; no PII) | 2 | `android:allowBackup="false"`, `android:fullBackupContent="false"`, empty `data_extraction_rules.xml` (see `04-security-hardening.md` §1, §2.1). `android:debuggable="false"` for release builds (see §1). |
| 3 | Samsung's One UI / ChimeraPolicyHandler kills NRLatch in the background, preventing `DataSubscriptionChangedReceiver` from firing on SIM swap. | High | Medium (functional failure; no security implication) | 3 | First-launch UX flow guides user through Battery Unrestricted + "Never sleeping apps" allowlist (see `05-samsung-reliability.md` §2). Diagnostic screen surfaces "Last SIM-change broadcast" timestamp for self-verification (§3). The receiver's logic is idempotent — the next event re-applies the saved preference (§4). |
| 4 | Carrier configuration or `ro.telephony.default_network` resets `preferred_network_mode${'$'}subId` on reboot, silently dropping the user's saved preference. | Medium | Medium (functional failure) | 4 | `BootCompletedReceiver` (manifest, `exported="false"`) re-applies the saved preference for the active data SIM at boot. Best-effort (subject to row 3's killing). See ADR-002 ([`adr-002-boot-completed.md`](adr-002-boot-completed.md)). |
| 5 | Tapjacking on `MainActivity`: a malicious overlay app baits the user into tapping a Settings deep-link (battery optimizations / Sleeping Apps) for a different package. | Low | Low (would only redirect to a different Settings screen; no privileged write) | 5 | `android:filterTouchesWhenObscured="true"` on `MainActivity` root view (see `04-security-hardening.md` §5.1). MainActivity contains **no** controls that flip the radio mode (the toggle exists only in the QS tile, which SystemUI alone can invoke); a successful tapjack therefore cannot trigger a privileged write. |
| 6 | Task hijacking (StrandHogg / StrandHogg 2.0): a malicious app declares `taskAffinity="com.nrlatch"` and inserts an activity above NRLatch's in the back-stack. | Negligible | Low | 6 | `android:taskAffinity=""` on `<application>` (see `04-security-hardening.md` §1). Android 11+ already mitigates StrandHogg variants at the OS level. |
| 7 | TileService impersonation: a malicious app ships a QS tile with a similar icon and label, hoping the user adds it instead of NRLatch's. | Low | Low (social engineering; no NRLatch privilege transfer) | 7 | Out of NRLatch's controllable scope. SystemUI's QS-edit panel shows the source app for each tile, providing a UI cue. NRLatch's tile uses a distinctive icon and label per `:feature:tile` design (`03-component-designs.md`). |
| 8 | `WRITE_SECURE_SETTINGS` revoked because the APK signature changes (release keystore loss; user installs an unsigned debug build over release). | Medium (across upgrades) | Medium (functional failure) | 8 | Dedicated release keystore: RSA 4096, 100-year validity, encrypted backups in ≥ 2 secure offline locations (see `04-security-hardening.md` §3.2). Documented keystore-loss runbook (out of architecture scope). Diagnostic screen surfaces the revocation immediately ("WRITE_SECURE_SETTINGS=N") so the user sees it on next launch. |
| 9 | `SecurityException` thrown by `Settings.Global.putInt` because the privilege was never granted, was revoked, or the device firmware's Knox policy denies it. | Medium | Low (functional failure, not a security hole) | 9 | `NetworkModeControllerImpl` catches `SecurityException` and surfaces via tile state `STATE_UNAVAILABLE` and an `error` field on the diagnostic screen. MainActivity displays the ADB grant command for re-grant. See `03-component-designs.md`. |
| 10 | Lock-screen toggle abuse: a passer-by pulls down the QS shade on a locked device and taps the NRLatch tile to flip the radio. | Low | Low (worst case: user's data drops from 5G to LTE until the next manual toggle) | 10 | `NrLatchTileService.onClick()` checks `isSecure()` and wraps the privileged write in `unlockAndRun { … }`, requiring device unlock before the radio flip happens (see `03-component-designs.md`). |

## Risks accepted without mitigation

The following are **explicit non-goals** for v1.0. Re-introducing any of
them requires a new ADR with measured justification.

- **Frida / Xposed / Magisk runtime attacks**: assumes the device is
  attacker-controlled. NRLatch is a personal utility, not a financial /
  identity / DRM app; the threat model accepts that a user who roots their
  own device and runs Frida against NRLatch will succeed.
- **Play Integrity attestation**: requires a server backend to verify the
  attestation token. NRLatch has zero server presence. Useless without a
  verifier.
- **Root detection / SafetyNet** (deprecated): false-positive prone, no
  positive impact for a non-financial utility, and rooted-device users are
  in scope as primary users.
- **Runtime APK signature self-verification**: trivially bypassable (the
  attacker patches the verification call itself). Adds complexity, no
  defensive value.
- **String / class encryption beyond R8 default**: R8 already renames
  identifiers in release. Encrypting strings adds compile-time complexity
  and zero protection against runtime introspection.
- **Anti-debug / anti-emulator traps**: no DRM or licensing concern
  justifies them.
- **Network-traffic sandboxing / VPN-pinning**: NRLatch makes no network
  calls; the `INTERNET` permission is stripped. There is no traffic to
  pin.
- **Per-launch re-prompting for `WRITE_SECURE_SETTINGS` confirmation**: not
  technically possible from a non-privileged app; would also be UX hostile.

## Cross-references

- Mitigation specifics: [`04-security-hardening.md`](04-security-hardening.md), [`05-samsung-reliability.md`](05-samsung-reliability.md).
- CI enforcement: [`08-ci-lint-rules.md`](08-ci-lint-rules.md).
- Decisions referenced: [`adr-002-boot-completed.md`](adr-002-boot-completed.md), [`adr-003-preference-key.md`](adr-003-preference-key.md).
