# ADR-001: Validation Spike for `Settings.Global` Network-Mode Write

- **Status**: PROPOSED — pending spike execution
- **Date**: TBD (set on completion of the spike)
- **Supersedes**: —
- **Superseded by**: —

## Context

NRLatch's central mechanism is a single privileged write:

```kotlin
Settings.Global.putInt(
    contentResolver,
    "preferred_network_mode${'$'}subId",
    NetworkMode.NR_LTE.settingsValue   // or LTE_ONLY.settingsValue
)
```

This relies on three premises that are not jointly guaranteed by AOSP
documentation alone on a Samsung device:

1. The `WRITE_SECURE_SETTINGS` permission, granted via `adb shell pm grant`,
   is *honored at runtime* on Samsung Galaxy S24+ / One UI 6.1+. (Some
   privileged settings are gated additionally by Knox / enterprise policy on
   Samsung devices.)
2. A successful `putInt` to `preferred_network_mode${'$'}subId` actually causes
   the modem to switch network mode. The Telephony codebase research
   surfaced a known caveat: on AOSP the user-facing Settings app calls
   `TelephonyManager.setPreferredNetworkTypeBitmask()`, which requires
   `MODIFY_PHONE_STATE` (signature\|privileged, **not** ADB-grantable).
   `Settings.Global.putInt` may be insufficient on some Samsung firmware.
3. The integer value `33` (NR + LTE per `RILConstants`) is the value the
   Samsung modem accepts to engage NR. On some Samsung firmware the accepted
   value is `27` (NR + LTE + WCDMA + GSM) or `26` instead.

If any premise fails, the entire NRLatch architecture is invalidated — no
amount of module skeleton, Hilt wiring, or DataStore plumbing recovers from
a non-functional radio write.

This ADR therefore records the decision to **spike before designing**.

## Decision

Run the throwaway validation spike specified in
[`06-validation-spike.md`](06-validation-spike.md) before authoring any
NRLatch module skeleton, Hilt graph, or feature-module code. The spike is a
**hard gate** on Phase 2 (implementation).

The spike is time-boxed at 2 hours. If it cannot reach a definitive PASS or
FAIL within 2 hours, halt and escalate.

## Consequences (PROPOSED — to be confirmed by spike)

### If the spike PASSES

- The architecture proceeds as documented in `00-overview.md` through
  `08-ci-lint-rules.md`.
- The exact integer values for `NetworkMode.NR_LTE.settingsValue` and
  `NetworkMode.LTE_ONLY.settingsValue` are confirmed (and updated in
  `02-public-interfaces.md` if Step 5 of the spike chose a fallback like
  `27` over `33`).
- ADR-001 status flips to **ACCEPTED**.

### If the spike FAILS at storage (Step 3 storage)

- `WRITE_SECURE_SETTINGS` is not effective on this device.
- **HALT PROJECT.** No remediation paths fit v1.0's "tiny utility" goal.
- Possible future remedies (out of v1.0 scope): Shizuku-mediated app, Magisk
  module, signed system-app build.
- ADR-001 status flips to **REJECTED**.

### If the spike FAILS at the radio (Step 3 radio with storage PASS)

- `Settings.Global.putInt` is insufficient on Samsung's modem; the OS-level
  Settings app uses a different (privileged) API path.
- **HALT PROJECT.** Architecture invalidated.
- ADR-001 status flips to **REJECTED**.

### If the spike FAILS at value 33 only (Step 4)

- Re-run Step 5's fallback values. Update `NetworkMode.NR_LTE.settingsValue`
  to the working integer.
- Architecture proceeds; this is **PARTIAL PASS** documented in the result
  section.

### If the spike FAILS at cardId stability (Step 2)

- Re-evaluate ADR-003. Likely outcome: amend ADR-003 to use a `subId +
  carrierId` composite key with documented instability caveats, or fall back
  to a per-slot key with an explicit user-visible warning that preferences
  do not survive SIM-slot swaps.

## Result (TO BE FILLED on completion of spike)

| Field | Value |
| --- | --- |
| Date run | _e.g._ 2026-04-26 |
| Device | Samsung Galaxy S24+ (`SM-S926B`) |
| Firmware build | _from Settings → About phone → Software information_ |
| Carrier | _e.g._ Airtel India, Postpaid |
| One UI version | _e.g._ 6.1.1 |
| Android version | _e.g._ 14 |

### Per-step outcomes

| Step | Outcome | Notes |
| --- | --- | --- |
| 1 — Read current mode | _PASS / FAIL_ | `subId=__`, initial `current=__` |
| 2 — `cardId` | _PASS / FAIL_ | `cardId=__`, stable across N reads: yes/no |
| 2 — `iccId` (stretch) | _PASS / EMPTY_ | _expected: empty string_ |
| 3 — Write 9 (LTE_ONLY) | _PASS / FAIL_ | readback=__, status-bar transition observed: yes/no, dumpsys snippet: ____ |
| 4 — Write 33 (NR_LTE) | _PASS / PARTIAL / FAIL_ | readback=__, NR re-engaged: yes/no |
| 5 — Fallback value used | _N/A or 27/26/22_ | Final accepted value: __ |

### Final canonical values (to be propagated into `02-public-interfaces.md`)

```kotlin
enum class NetworkMode(val settingsValue: Int) {
    NR_LTE(/* 33 — or fallback from Step 5 */),
    LTE_ONLY(9);
    // ...
}
```

### Lessons-learned addendum

_One paragraph describing anything surprising — radio took longer than 30s to
flip, cardId differed from a previous reading on a different SIM, dumpsys
output schema differed, etc._

### Evidence

- Logcat dump: `docs/architecture/spike-evidence/logcat-YYYY-MM-DD.txt`
  (committed only if file is < 200 KB; otherwise summarize inline).
- Status-bar before/after screenshots: optional, summarize in Notes column.
- `dumpsys telephony.registry` snippet pre/post each write: included inline
  above.
