# 06 — Validation Spike

This document specifies a **blocking, pre-implementation spike**. Its purpose
is to prove — on the actual target device, before a single line of NRLatch
module code is written — that the architecture's central mechanism actually
works.

The result is captured in [`adr-001-validation-spike.md`](adr-001-validation-spike.md).
Until that ADR is filled in with PASS results, no module skeleton, no Gradle
plumbing beyond what already exists, and no Hilt wiring is to be authored.

## 1. Goal

Prove the following on a real Samsung Galaxy S24+ running stock One UI 6.1+:

1. `Settings.Global.putInt(contentResolver, "preferred_network_mode${'$'}subId", value)`
   succeeds with `WRITE_SECURE_SETTINGS` granted via ADB.
2. The radio actually switches mode (status bar 5G ↔ LTE) in response.
3. `SubscriptionInfo.getCardId()` returns a stable non-`-1` integer with
   `READ_PHONE_STATE` granted at runtime.
4. Whether `SubscriptionInfo.getIccId()` returns a non-empty string (stretch
   goal; expected empty on API 29+ for non-privileged apps).
5. The exact integer value that activates each NetworkMode on this Samsung
   modem (specifically: does `33` activate NR+LTE, or must we use `27` or a
   different value?).

If any of (1), (2), (3) fail, **the architecture is invalidated** and the
project halts pending escalation.

## 2. Setup

### 2.1 Device

- Samsung Galaxy S24+ (model `SM-S926B` or local-region SKU).
- One UI 6.1 or later.
- Active SIM with mobile data enabled. NR (5G) coverage available at the
  test location, verified by the SIM normally showing "5G" in the status bar.
- USB cable; PC with `adb` from a current Android SDK platform-tools.

### 2.2 Device prep

1. Settings → About phone → Software information → tap **Build number** 7×
   to enable Developer Options.
2. Settings → Developer options → enable **USB debugging**.
3. Connect via USB. Authorize the debugging fingerprint when prompted.
4. Verify with `adb devices` — exactly one device listed as `device` (not
   `unauthorized`).

### 2.3 Throwaway spike app

Create a fresh project **outside** the NRLatch repo (or in
`/tmp/nrlatch-spike/`). It must NOT be committed to the NRLatch repo.

- Single Activity, no Hilt, no DataStore, no Compose, no modules.
- < 50 lines of Kotlin in `MainActivity`.
- `applicationId = "com.nrlatch.spike"` (distinct from production app).
- `minSdk = 29`, `targetSdk = 35`.
- Manifest declares `WRITE_SECURE_SETTINGS` and `READ_PHONE_STATE`.

```kotlin
// Spike MainActivity — paste, modify, throw away.
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        // Run each step in a button onClick; results go to logcat tag "Spike".
        // Implementation kept inline to avoid any architectural inference.
    }
}
```

### 2.4 Install & grant

```bash
./gradlew :app:assembleDebug
adb install -r app/build/outputs/apk/debug/app-debug.apk
adb shell pm grant com.nrlatch.spike android.permission.WRITE_SECURE_SETTINGS
# Also grant READ_PHONE_STATE at runtime from the app, OR via:
adb shell pm grant com.nrlatch.spike android.permission.READ_PHONE_STATE
```

Verify:

```bash
adb shell dumpsys package com.nrlatch.spike | grep -E "WRITE_SECURE_SETTINGS|READ_PHONE_STATE"
# Expect both lines to show "granted=true".
```

## 3. Test plan

Each step is an independent **PASS / FAIL** test. Record the result, the raw
logcat output, and any observed status-bar/UI change in ADR-001.

### Step 1 — Read current mode

```kotlin
val subId = SubscriptionManager.getDefaultDataSubscriptionId()
val current = Settings.Global.getInt(
    contentResolver,
    "preferred_network_mode${'$'}subId",
    -1
)
Log.i("Spike", "subId=${'$'}subId current=${'$'}current")
```

- **PASS**: `subId` is a positive integer; `current` is a non-`-1` integer.
  Record the exact value (likely `33`, `27`, `9`, or `11`).
- **FAIL**: `subId == INVALID_SUBSCRIPTION_ID (-1)` or `current == -1`.

### Step 2 — Get cardId and ICCID

```kotlin
val sm = getSystemService(SubscriptionManager::class.java)!!
// Requires READ_PHONE_STATE granted.
val info = sm.getActiveSubscriptionInfo(subId)
Log.i("Spike", "cardId=${'$'}{info.cardId} iccId='${'$'}{info.iccId}'")
```

- **PASS (cardId)**: `info.cardId` returns a stable integer **other than**
  `SubscriptionManager.INVALID_SIM_SLOT_INDEX` (-1) **and** other than
  `INVALID_CARD_ID` (-1; same constant). Record the value.
- **PASS-stretch (iccId)**: `info.iccId` is a non-empty string. **Expected
  result: empty string** on API 29+ for non-privileged apps. If it is
  non-empty, that is a happy surprise — flag it for ADR-003 follow-up.
- **FAIL (cardId)**: returns `-1` or differs across two consecutive reads.

### Step 3 — Write LTE_ONLY (value 9) and observe

```kotlin
Settings.Global.putInt(
    contentResolver,
    "preferred_network_mode${'$'}subId",
    9
)
// Wait 5 seconds.
val readback = Settings.Global.getInt(
    contentResolver,
    "preferred_network_mode${'$'}subId",
    -1
)
Log.i("Spike", "after write 9, readback=${'$'}readback")
```

In parallel from the host:

```bash
adb shell dumpsys telephony.registry \
    | grep -iE "voice|data|nr|preferred"
```

- **PASS-storage**: `readback == 9`.
- **PASS-radio**: within 30 seconds, the status-bar indicator flips from "5G"
  to "LTE" / "4G", AND the `dumpsys telephony.registry` output shows the
  preferred network type or radio capability switching to an LTE-only mode.
- **FAIL-storage**: `Settings.Global.putInt` throws `SecurityException` (the
  privilege was not effective) OR `readback != 9`.
- **FAIL-radio**: storage write succeeds but the modem is still on NR after
  60 seconds. This is the architecture-killer case (see §4).

### Step 4 — Write NR_LTE (value 33) and observe reversion

```kotlin
Settings.Global.putInt(
    contentResolver,
    "preferred_network_mode${'$'}subId",
    33
)
// Wait 5 seconds.
val readback = Settings.Global.getInt(
    contentResolver,
    "preferred_network_mode${'$'}subId",
    -1
)
Log.i("Spike", "after write 33, readback=${'$'}readback")
```

- **PASS**: `readback == 33` AND status bar returns to "5G" within 30s AND
  `dumpsys` confirms NR.
- **PARTIAL**: `readback == 33` but the modem stays on LTE — try Step 5.
- **FAIL-storage**: `readback` returns a different value than 33 (the modem
  silently coerced; see Step 5).

### Step 5 — Fallback values if Step 4 didn't restore NR

If Samsung's modem doesn't accept `33`, try the documented alternatives in
order. After each write, repeat the same readback + 30s observation:

| Value | Meaning (per AOSP `RILConstants`) |
| --- | --- |
| 33 | NR + LTE |
| 27 | NR + LTE + WCDMA + GSM |
| 26 | NR + LTE + GSM |
| 22 | NR + LTE (variant) |

If a value ≥ 22 with NR is accepted, record that value as the **canonical
NR_LTE settings value** for the architecture. Update
`NetworkMode.NR_LTE.settingsValue` accordingly before any module code is
written.

If **none** of the listed values restores NR, that is FAIL-radio at Step 3
and the architecture is invalidated regardless of which write was attempted.

## 4. Failure modes and escalation

| Symptom | Root cause | Action |
| --- | --- | --- |
| Step 3 `putInt` throws `SecurityException` | `WRITE_SECURE_SETTINGS` not effective on this device (e.g. enterprise policy, Knox, locked-down firmware variant). | **HALT PROJECT.** Escalate. Possible remediations are out of v1.0 scope. |
| Step 3 PASS-storage but FAIL-radio | `Settings.Global.putInt` is insufficient on Samsung; the AOSP Settings app calls `TelephonyManager.setPreferredNetworkTypeBitmask()`, which requires `MODIFY_PHONE_STATE` (signature\|privileged, **not** ADB-grantable to user-installed apps). | **HALT PROJECT.** Architecture invalidated. Escalate. The only paths from here are: (a) ship as a Magisk/Shizuku-mediated app (out of scope), (b) abandon. |
| Step 4 FAIL — value 33 not recognized; modem coerces or ignores | Samsung's modem uses a different bitmask. | Run Step 5. Document the working value. Update `NetworkMode.NR_LTE.settingsValue` and `08-ci-lint-rules.md` references. |
| Step 5 — no value works | Same as Step-3 FAIL-radio | **HALT PROJECT.** |
| Step 2 — `cardId == -1` or unstable across reads | `getCardId()` not honored on this device, or the SIM lacks a card identifier (rare for physical SIM, more plausible for eSIM profile in some firmware variants). | Re-evaluate ADR-003. Fall back to `subId + carrierId` composite. Document instability as a known v1.0 limitation. Project may continue but ADR-003 must be amended or replaced. |
| Step 2 stretch — `iccId` returns non-empty | Surprising but useful. | Flag for a future ADR amendment to ADR-003; v1.0 still uses cardId. |

## 5. Deliverable

A completed [`adr-001-validation-spike.md`](adr-001-validation-spike.md) with:

- **Status** flipped from `PROPOSED` to `ACCEPTED` (PASS) or `REJECTED` (FAIL).
- **Date** set to the date the spike ran.
- Device firmware string from `Settings → About phone → Software information`.
- Per-step result table (Steps 1–5) with raw logcat snippets and screenshots
  (or a short Markdown description) of the status-bar transitions.
- Final confirmed value of `NetworkMode.NR_LTE.settingsValue` (33 / 27 / 26 / 22).
- Final confirmed value of `NetworkMode.LTE_ONLY.settingsValue` (almost
  certainly 9, but verified by Step 3).
- A one-paragraph "lessons learned" addendum if anything surprised the spike
  runner.

The throwaway spike project is then **deleted**. Do not commit it to the
NRLatch repo. Logcat dumps may be attached to the ADR (in
`docs/architecture/spike-evidence/` if needed).

## 6. Time-box

**2 hours wall-clock.**

If the spike has not produced a definitive PASS or FAIL result within 2 hours,
**stop**. A spike taking longer than 2 hours indicates a structural problem
(unexpected device behavior, ADB grant not sticking, modem in odd state) that
deserves escalation rather than further fiddling.

## 7. Cross-references

- ADR-001 (the result document): [`adr-001-validation-spike.md`](adr-001-validation-spike.md).
- ADR-003 (preference-key decision, partially validated by Step 2): [`adr-003-preference-key.md`](adr-003-preference-key.md).
- `NetworkMode` enum definition: `02-public-interfaces.md`.
- Architectural overview: `00-overview.md`.
