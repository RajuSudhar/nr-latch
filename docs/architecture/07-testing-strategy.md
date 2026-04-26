# 07 — Testing Strategy

NRLatch's testing posture is deliberately **lean and asymmetric**: heavy
JVM-only unit tests against well-isolated logic, thin instrumented smoke
tests against framework boundaries, and a manual on-device checklist for
release. There is no Robolectric, no Espresso, no Pitest.

The goal: a CI run completes in well under one minute, never flakes, and
catches every regression that can be caught without a real radio.

## 1. Test pyramid

```
                ┌─────────────────────────────────────────┐
                │  Manual on-device checklist (release)   │
                │  — toggling, SIM swap, reboot           │
                └─────────────────────────────────────────┘
              ┌───────────────────────────────────────────┐
              │  Instrumented smoke tests (NOT in CI)     │
              │  — DataStore I/O, SubscriptionManager,    │
              │    TileService bind lifecycle             │
              └───────────────────────────────────────────┘
        ┌─────────────────────────────────────────────────────┐
        │  JVM unit tests (CI gate, ≥80% logic coverage)      │
        │  — :core:model, :core:policy, :core:preferences,    │
        │    TileToggleHandler, SimSwitchHandler              │
        └─────────────────────────────────────────────────────┘
```

## 2. JVM unit tests — the workhorse layer

**Target: ≥ 80% line coverage of all "logic" code via Kover.** Logic =
anything not in `*Impl` classes that touch `android.*` framework types.

### 2.1 `:core:model` — 100% coverage required

`NetworkMode`, `SimIdentity`, `PolicyDecision`, `PerSimPreference` are pure
data. They get exhaustive coverage:

```kotlin
class NetworkModeTest {
    @Test fun `fromSettingsValue maps 9 to LTE_ONLY`() {
        assertThat(NetworkMode.fromSettingsValue(9)).isEqualTo(NetworkMode.LTE_ONLY)
    }

    @Test fun `fromSettingsValue maps NR_LTE settings value`() {
        assertThat(NetworkMode.fromSettingsValue(NetworkMode.NR_LTE.settingsValue))
            .isEqualTo(NetworkMode.NR_LTE)
    }

    @Test fun `fromSettingsValue returns null for unknown ints`() {
        assertThat(NetworkMode.fromSettingsValue(999)).isNull()
    }

    @Test fun `enum has exactly two entries`() {
        assertThat(NetworkMode.values().toList())
            .containsExactly(NetworkMode.NR_LTE, NetworkMode.LTE_ONLY)
    }
}
```

Equality of `SimIdentity`, `PolicyDecision`, `PerSimPreference` must be
exercised (data class `equals`/`hashCode` are auto-generated but the test
catches an accidental `class` → `data class` regression).

### 2.2 `:core:policy` — engine logic

`ManualOnlyPolicyEngine.resolve()` is tested against fakes of
`PreferenceStore` and `SubscriptionProvider`. Required cases:

- No saved preference for the active card → returns
  `PolicyDecision(target = LTE_ONLY, reason = DEFAULT)`.
- Saved preference NR_LTE for the active card → returns
  `PolicyDecision(target = NR_LTE, reason = USER_PREFERENCE)`.
- Saved preference LTE_ONLY → returns LTE_ONLY with USER_PREFERENCE reason.
- Active card changes mid-test (collected via SubscriptionProvider Flow) →
  next resolve uses the new card's preference.
- Multiple registered `ModePolicy` impls (use fakes of higher and lower
  priority) → the engine selects the highest-priority non-null decision.
  Even though v1.0 ships only `ManualOverridePolicy`, the priority chain is
  exercised so future additions don't silently break.

Tools:

```kotlin
val store = FakePreferenceStore()
val subs = FakeSubscriptionProvider(activeCardId = 42)
val engine = PolicyEngineImpl(setOf(ManualOverridePolicy(store)), subs)

runTest {
    store.setPreferredMode(42, NetworkMode.NR_LTE)
    val decision = engine.resolve()
    assertThat(decision.target).isEqualTo(NetworkMode.NR_LTE)
    assertThat(decision.reason).isEqualTo(PolicyReason.USER_PREFERENCE)
}
```

### 2.3 `TileToggleHandler` — extracted plain class

`NrLatchTileService` is a thin Android shell. The tile-toggle logic lives
in a plain `TileToggleHandler` class (POJO Kotlin) injected into the
service:

```kotlin
class TileToggleHandler @Inject constructor(
    private val controller: NetworkModeController,
    private val subscriptions: SubscriptionProvider,
    private val store: PreferenceStore
) {
    suspend fun toggle(): ToggleResult { /* ... */ }
}
```

JVM tests cover:

- Toggling NR_LTE → LTE_ONLY when current mode is NR_LTE.
- Toggling LTE_ONLY → NR_LTE when current mode is LTE_ONLY.
- Persisting the new mode to `PreferenceStore` keyed by `cardId`.
- Returning `ToggleResult.NoActiveSim` when `getActiveCardId()` returns null.
- Returning `ToggleResult.PermissionDenied` when controller throws
  `SecurityException`.
- Idempotency: two toggles back-to-back leave state consistent.

The Android `TileService` framework class is **not** unit-tested directly;
its only logic delegates to `TileToggleHandler`.

### 2.4 `SimSwitchHandler` — extracted plain class

Similarly, both `DataSubscriptionChangedReceiver` and `BootCompletedReceiver`
are thin wrappers that call `SimSwitchHandler.reapplySavedPreference()`. JVM
tests cover:

- Saved NR_LTE preference for the (now-active) card → `setMode(NR_LTE)`
  invoked exactly once.
- No saved preference → no `setMode` invoked (the receiver is silent).
- `setMode` throws `SecurityException` → caller is **not** retried; error
  observable via the diagnostic timestamp pattern.
- Idempotency: handler invoked twice for the same card with the same saved
  preference → only one effective state transition (controller is allowed to
  be called twice; the underlying putInt is naturally idempotent).
- `SubscriptionProvider.getActiveCardId()` returns null → handler returns
  silently without calling `setMode`.

### 2.5 `:core:preferences` — in-memory + DataStore

Two layers:

- **In-memory logic** (the `PreferenceStore` interface contract): tested
  against `FakePreferenceStore` to verify the keying scheme (cardId → mode)
  and Flow emission semantics.
- **`PreferenceStoreImpl` over DataStore**: tested in instrumented tests
  (see §3) because DataStore's IO is real-disk.

Useful Turbine pattern:

```kotlin
@Test fun `flow emits when preference changes`() = runTest {
    val store = FakePreferenceStore()
    store.getPreferredMode(cardId = 7).test {
        assertThat(awaitItem()).isNull()
        store.setPreferredMode(7, NetworkMode.NR_LTE)
        assertThat(awaitItem()).isEqualTo(NetworkMode.NR_LTE)
        cancelAndIgnoreRemainingEvents()
    }
}
```

### 2.6 Tools (versions per `libs.versions.toml`)

| Tool | Use |
| --- | --- |
| JUnit 4 | Test runner. (JUnit 5 is gratuitous here.) |
| MockK 1.13.x | Kotlin-friendly mocks of any interface; minimize use — prefer hand-rolled fakes. |
| kotlinx-coroutines-test | `runTest`, `TestDispatcher`, `StandardTestDispatcher` for deterministic Flow timing. |
| Turbine | Flow assertions: `flow.test { awaitItem() }`. |
| Truth (com.google.truth) | Fluent assertions: `assertThat(x).isEqualTo(...)`. |
| Kover | Coverage reporting (Kotlin-aware, KSP-compatible). |

## 3. Instrumented tests — thin & manual-only

Run on a connected device; **excluded from CI**.

| Test class | What it verifies |
| --- | --- |
| `NetworkModeControllerImplTest` | `Settings.Global.putInt` round-trip with `WRITE_SECURE_SETTINGS` granted via ADB. Skipped on emulator. Asserts readback equals written value; does NOT assert radio actually switched (that's the manual checklist). |
| `PreferenceStoreImplTest` | Real DataStore writes & reads. Verify cardId keying, Flow emission, restart persistence (write, kill process, reopen, assert value). |
| `SubscriptionProviderImplTest` | `SubscriptionManager` listener emits a Flow update when the default data subId changes. Single-SIM emulator suffices for one update; for two-SIM behavior, see manual checklist. |
| `NrLatchTileServiceTest` | Smoke: bind the service, send `onClick`, assert no exception, assert the handler was invoked. Uses Hilt test runner. |

Excluded from CI because:

- The CI runner has no `WRITE_SECURE_SETTINGS` grant on the default emulator
  image (would require a device-prep script outside the CI scope).
- They take 30 s+ even with a hot emulator.
- They are smoke checks, not regression-prevention; the manual checklist
  catches the real bugs.

These tests are run **manually before each release** as part of
[`checklist.md`](checklist.md) — "Instrumented smoke tests" item.

## 4. No integration tests against real telephony in CI

The radio behavior cannot be reliably emulated. There is no AOSP emulator
image with NR + Samsung modem behavior. CI therefore makes no attempt to
exercise the radio path; the **manual on-device checklist** in
`checklist.md` is the canonical pre-release verification.

## 5. CI strategy

### 5.1 Pipeline (GitHub Actions or local Gradle equivalent)

```bash
./gradlew \
    testDebugUnitTest \
    dependencyAnalysis \
    lint \
    koverHtmlReport \
    assembleRelease
```

Each task **must** pass:

- `testDebugUnitTest`: zero failing tests, zero ignored tests (Kotlin
  `@Ignore` is not allowed in committed code).
- `dependencyAnalysis`: zero violations of the dependency rules in
  `01-module-graph.md` (no `:core:*` → `:feature:*` edges, no unused deps,
  no undeclared transitives).
- `lint`: zero issues from the custom rule
  `NrLatchSetModeFromExportedComponent` (see `08-ci-lint-rules.md`); zero
  errors of severity ≥ `WARNING` from baseline Android Lint with
  `lintConfig` tightening.
- `koverHtmlReport`: line coverage of all `:core:*` modules ≥ 80%.
- `assembleRelease`: builds, R8 succeeds. The release APK is then re-fed
  through a manifest grep (see `08-ci-lint-rules.md` "Manifest verification")
  that asserts `INTERNET` is absent.

### 5.2 Coverage threshold enforcement

Kover configuration (in `build-logic` or root `build.gradle.kts`):

```kts
koverReport {
    verify {
        rule {
            isEnabled = true
            entity = kotlinx.kover.gradle.plugin.dsl.GroupingEntityType.APPLICATION
            bound {
                minValue = 80
                metric = kotlinx.kover.gradle.plugin.dsl.MetricType.LINE
            }
        }
    }
}
```

Build fails if coverage drops below 80%.

### 5.3 No flaky-test tolerance

A test that fails intermittently is **deleted or fixed**, never `@Ignore`d
or rerun-in-CI. Coroutine-timing flakes are addressed with
`TestDispatcher` and explicit `advanceUntilIdle()` / `runCurrent()`, never
with `delay()` or retry shells.

## 6. Things we explicitly DO NOT test

| Excluded | Reason |
| --- | --- |
| Robolectric | +30 s per test class, masks real framework bugs, and the framework-touching code in NRLatch is already isolated to `*Impl` classes that don't deserve fake-Android coverage. |
| Espresso UI tests for MainActivity | MainActivity is a single-shot static screen with deep-link buttons. Manual testing of the first-launch flow is sufficient and faster. |
| Performance benchmarks (Macrobenchmark) | NRLatch does nothing 99% of the time. There is no perf budget. |
| Mutation testing (Pitest) | Overkill for ~500 LOC of logic. The logic is so simple that hand-written tests achieve effectively-100% mutation kill on the lines that matter. |
| Property-based testing (Kotest, jqwik) | Overkill; the input space (NetworkMode is two values, cardId is an Int) doesn't justify the framework. |
| Snapshot tests for Compose UI | Compose is **excluded from v1.0 entirely** (decided in `00-overview.md`). |
| Network mock tests | NRLatch makes no network calls. |

## 7. Cross-references

- Module graph and per-module dependencies: `01-module-graph.md`.
- Public interfaces under test: `02-public-interfaces.md`.
- Custom lint rule run in CI: [`08-ci-lint-rules.md`](08-ci-lint-rules.md).
- Pre-release on-device checklist: [`checklist.md`](checklist.md).
