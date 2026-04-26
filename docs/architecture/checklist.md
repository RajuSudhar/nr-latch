# NRLatch Implementation Checklist

A trackable checklist that takes NRLatch from "Phase 1 architecture
documents complete" to "v1.0 release tagged and on-device verified".

Items are organized in the order they must be executed. Some sections
**block** subsequent sections (notably the validation spike). Do not skip
the gating sequence.

Cross-reference each section back to the corresponding architecture
document for full specification.

---

## 1. Pre-implementation (BLOCKING)

These items gate all subsequent work. If the spike fails, halt and
escalate; do not begin module skeleton until the spike passes.

- [ ] Provision a Samsung Galaxy S24+ test device with Developer Options
      and USB debugging enabled. (See `06-validation-spike.md` §2.2.)
- [ ] Run the validation spike per [`06-validation-spike.md`](06-validation-spike.md)
      Steps 1–5 within the 2-hour time-box.
- [ ] Capture logcat output, status-bar transitions, and `dumpsys
      telephony.registry` snippets as evidence.
- [ ] Fill in [`adr-001-validation-spike.md`](adr-001-validation-spike.md)
      Result section with per-step PASS / FAIL.
- [ ] Set ADR-001 Status to ACCEPTED (PASS) or REJECTED (FAIL).
- [ ] If PASS: confirm the canonical integer values for
      `NetworkMode.NR_LTE.settingsValue` and
      `NetworkMode.LTE_ONLY.settingsValue`; update `02-public-interfaces.md`
      if Step 5 chose a fallback like `27` over `33`.
- [ ] If FAIL at storage or radio: HALT. Do not proceed. Escalate per
      `06-validation-spike.md` §4.
- [ ] Delete the throwaway spike project; do not commit it to the NRLatch
      repo.

## 2. Module skeleton

Begin only after section 1's PASS gate.

- [ ] Create `settings.gradle.kts` listing all modules per `01-module-graph.md`.
- [ ] Confirm `gradle/libs.versions.toml` is in place (the sibling agent
      drafted `docs/architecture/libs.versions.toml`; promote it to
      `gradle/libs.versions.toml` as the actual version catalog).
- [ ] Author per-module `build.gradle.kts` files for: `:app`,
      `:feature:tile`, `:feature:simswitch`, `:feature:diagnostic`,
      `:core:model`, `:core:policy`, `:core:controller`,
      `:core:preferences`, `:core:subscription`, `:lint`.
- [ ] Apply Kotlin 2.3.20, AGP 9.1.0, Gradle 9.4.1, JDK 21 (toolchain),
      JVM target 17 across all modules.
- [ ] Configure KSP 2.3.20-x and Hilt 2.56.2 in modules that use Hilt.
- [ ] `minSdk = 29`, `targetSdk = 35`, `compileSdk = 35` in all Android
      modules.
- [ ] Implement public interfaces from `02-public-interfaces.md` with full
      KDoc:
      `NetworkModeController`, `PolicyEngine`, `ModePolicy`,
      `PreferenceStore`, `SubscriptionProvider`, model types
      (`NetworkMode`, `SimIdentity`, `PolicyDecision`, `PerSimPreference`).
- [ ] Implement components per `03-component-designs.md`:
      `NetworkModeControllerImpl`, `PolicyEngineImpl`,
      `ManualOverridePolicy`, `PreferenceStoreImpl`,
      `SubscriptionProviderImpl`, `TileToggleHandler`,
      `SimSwitchHandler`, `NrLatchTileService`,
      `DataSubscriptionChangedReceiver`, `BootCompletedReceiver`,
      `MainActivity` (diagnostic screen).
- [ ] Wire Hilt graph: `NrLatchApplication` annotated `@HiltAndroidApp`,
      modules in `:app/di/`, `@AndroidEntryPoint` on each Android
      framework class.

## 3. Manifest hardening

- [ ] Apply every attribute and `<uses-permission>` from
      [`04-security-hardening.md`](04-security-hardening.md) §1.
- [ ] Add `tools:node="remove"` for `INTERNET` per §1.1.
- [ ] Declare custom permission `com.nrlatch.TOGGLE` with
      `protectionLevel="signature"` per §1.3.
- [ ] Create `app/src/main/res/xml/data_extraction_rules.xml` per §2.1.
- [ ] Create `app/src/main/res/xml/network_security_config.xml` per §2.2.
- [ ] Confirm `<application>` has all of: `allowBackup="false"`,
      `dataExtractionRules`, `fullBackupContent="false"`, `taskAffinity=""`,
      `usesCleartextTraffic="false"`, `networkSecurityConfig`.
- [ ] Build a release APK and verify INTERNET absent:
      `aapt dump permissions app/build/outputs/apk/release/app-release.apk
      | grep -i internet` returns nothing.
- [ ] Verify the four required permissions are present (see
      `08-ci-lint-rules.md` §3.2).
- [ ] Add `filterTouchesWhenObscured="true"` on `MainActivity`'s root view
      per §5.1.

## 4. Build configuration hardening

- [ ] R8 full mode enabled (`android.enableR8.fullMode=true` in
      `gradle.properties`).
- [ ] Release build type: `isMinifyEnabled = true`,
      `isShrinkResources = true` (see `04-security-hardening.md` §3).
- [ ] `app/proguard-rules.pro` contains all `-keep` rules from §3.1.
- [ ] CI ProGuard sanity grep wired up per `08-ci-lint-rules.md` §3.3.

## 5. Samsung reliability UX

- [ ] Implement first-launch flow steps 1–5 in `MainActivity` per
      [`05-samsung-reliability.md`](05-samsung-reliability.md) §2.
- [ ] Implement diagnostic screen with the six fields listed in §3.1.
- [ ] DataStore keys `last_sim_change_at_millis` and
      `last_boot_completed_at_millis` written from the receivers and read
      on `onResume()`.
- [ ] Step 4's deep-link tries the Samsung-specific intent first, falls
      back to `ACTION_APPLICATION_DETAILS_SETTINGS`.
- [ ] User-facing copy for step 4 instructs adding NRLatch to "Never
      sleeping apps".

## 6. CI gates

- [ ] Implement custom lint module `:lint` per
      [`08-ci-lint-rules.md`](08-ci-lint-rules.md) §1, including
      `NrLatchSetModeDetector` and `NrLatchIssueRegistry`.
- [ ] Wire `lintChecks(project(":lint"))` into `:app`, `:feature:tile`,
      `:feature:simswitch`.
- [ ] Implement and run the `NrLatchSetModeDetectorTest` (positive and
      negative cases pass).
- [ ] Implement `PendingIntentImmutability` lint check (severity ERROR)
      per §2.
- [ ] Configure `dependency-analysis-gradle-plugin` per §3.1; build fails
      on unused / undeclared / layering violations.
- [ ] Add `scripts/verify-manifest.sh` per §3.2 and invoke it as a CI
      step after `assembleRelease`.
- [ ] Add ProGuard sanity-grep CI step per §3.3.
- [ ] CI pipeline runs:
      `./gradlew testDebugUnitTest dependencyAnalysis lint koverHtmlReport
      assembleRelease` plus `verify-manifest.sh`.
- [ ] CI fails on any violation; no `@Ignore`d tests permitted.

## 7. Testing

Per [`07-testing-strategy.md`](07-testing-strategy.md).

### 7.1 JVM unit tests (CI gate)

- [ ] `:core:model`: `NetworkModeTest`, equality tests for `SimIdentity`,
      `PolicyDecision`, `PerSimPreference`. 100% line coverage.
- [ ] `:core:policy`: `ManualOnlyPolicyEngineTest` with fakes (default
      LTE_ONLY, sticky behavior, multi-policy priority).
- [ ] `:core:preferences`: in-memory `FakePreferenceStore` exercise; Flow
      emission via Turbine.
- [ ] `TileToggleHandlerTest` covering toggle direction, persistence,
      `NoActiveSim`, `PermissionDenied`, idempotency.
- [ ] `SimSwitchHandlerTest` covering re-apply, idempotency, error
      handling, no-active-card silent return.
- [ ] Kover line coverage ≥ 80% across `:core:*`; CI fails below threshold.

### 7.2 Instrumented tests (manual, not in CI)

- [ ] `NetworkModeControllerImplTest`: round-trip `Settings.Global.putInt`
      with `WRITE_SECURE_SETTINGS` granted.
- [ ] `PreferenceStoreImplTest`: real DataStore I/O including persistence
      across process restart.
- [ ] `SubscriptionProviderImplTest`: listener-Flow smoke.
- [ ] `NrLatchTileServiceTest`: bind + onClick smoke.

### 7.3 Negative tests

- [ ] No Robolectric, no Espresso, no Pitest committed to the repo.
- [ ] No flaky tests; any intermittent failure is fixed or deleted, never
      `@Ignore`d.

## 8. Release preparation

- [ ] Generate the release keystore: RSA 4096, 100-year validity.
- [ ] Encrypt and back up the keystore to ≥ 2 offline locations
      (encrypted USB, encrypted password-manager attachment, etc.).
- [ ] Document the recovery runbook in `docs/runbook/keystore-loss.md`
      (outside this architecture set).
- [ ] Configure release `signingConfig` in `app/build.gradle.kts`
      referencing the keystore via env variables (never check the
      keystore into git).
- [ ] Run the full CI pipeline against `main`: all green.
- [ ] Run the manual instrumented tests (section 7.2) against the S24+
      device.

## 9. On-device manual verification (pre-release)

For each verification, record PASS/FAIL with date and firmware build in a
release notes addendum.

- [ ] Install the release APK on the S24+ device:
      `adb install -r app-release.apk`.
- [ ] Grant `WRITE_SECURE_SETTINGS`:
      `adb shell pm grant com.nrlatch android.permission.WRITE_SECURE_SETTINGS`.
- [ ] Open the app; complete the first-launch flow (steps 1–5).
- [ ] Add NRLatch's tile to the QS shade.
- [ ] **Toggle test**: tap the QS tile from an unlocked device. Status bar
      switches NR ↔ LTE within 5 seconds. PASS / FAIL.
- [ ] **Lock-screen toggle test**: tap the QS tile from the lock screen.
      The device prompts for unlock; after unlock the radio switches.
      PASS / FAIL.
- [ ] **SIM-swap test**: with two SIMs, swap the active data SIM via
      Settings → Connections → SIM manager. Diagnostic screen's "Last
      SIM-change broadcast" timestamp updates within 60 seconds, AND the
      saved preference for the new card is auto-applied. PASS / FAIL.
- [ ] **Reboot test**: reboot the device. Diagnostic screen's "Last
      BOOT_COMPLETED" timestamp updates AND the saved preference for the
      active card is auto-applied. PASS / FAIL. (Best-effort; ≥ 80% target
      across 5 reboots per `05-samsung-reliability.md` §5.)
- [ ] **Manifest decompile audit**:
      `apktool d app-release.apk -o release-decoded`, then grep
      `<uses-permission` lines. Confirm exactly the four expected
      permissions, no INTERNET. PASS / FAIL.

## 10. Tagging and release

- [ ] All previous sections marked complete.
- [ ] Tag the commit: `git tag -s v1.0.0 -m "NRLatch v1.0.0"`.
- [ ] Archive the verified release APK alongside the tag.
- [ ] Append the manual verification record (section 9 PASS/FAIL log) to
      `docs/release-notes/v1.0.0.md` (outside this architecture set).

---

## Cross-references

- Module graph & build wiring: `01-module-graph.md`.
- Public interfaces: `02-public-interfaces.md`.
- Component designs: `03-component-designs.md`.
- Security hardening: [`04-security-hardening.md`](04-security-hardening.md).
- Samsung reliability: [`05-samsung-reliability.md`](05-samsung-reliability.md).
- Validation spike: [`06-validation-spike.md`](06-validation-spike.md).
- Testing strategy: [`07-testing-strategy.md`](07-testing-strategy.md).
- CI lint rules: [`08-ci-lint-rules.md`](08-ci-lint-rules.md).
- ADRs: [`adr-001-validation-spike.md`](adr-001-validation-spike.md),
  [`adr-002-boot-completed.md`](adr-002-boot-completed.md),
  [`adr-003-preference-key.md`](adr-003-preference-key.md).
- Threats: [`threat-model.md`](threat-model.md).
