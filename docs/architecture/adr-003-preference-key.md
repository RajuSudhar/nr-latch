# ADR-003: Use `SubscriptionInfo.cardId` (Int) as Preference Key, Not ICCID

- **Status**: ACCEPTED
- **Date**: 2026-04-25
- **Supersedes**: —
- **Superseded by**: —

## Context

NRLatch persists a per-SIM mode preference (`NR_LTE` or `LTE_ONLY`) so that
when the user swaps the active data SIM, the saved choice for the *other*
SIM is re-applied automatically by `DataSubscriptionChangedReceiver`. The
key for that persistence must satisfy:

1. **Stable** across SIM-slot reorderings and reboots. The Android subId is
   ephemeral on Samsung — moving a SIM from slot 1 to slot 2 may produce a
   different subId for the same physical/eSIM card.
2. **Accessible** from a non-privileged third-party app. NRLatch holds
   `WRITE_SECURE_SETTINGS` (granted via ADB) and `READ_PHONE_STATE` (runtime).
   It does **not** hold any signature\|privileged permission.
3. **Per-card** rather than per-slot. The user's mental model is "this SIM
   gets NR_LTE, that SIM gets LTE_ONLY", not "slot 1 gets NR_LTE".

The original architect brief proposed `SubscriptionInfo.getIccId()` as the
key. ICCID is the globally-unique 19–20-digit identifier printed on the
physical SIM and embedded in eSIM profiles. It satisfies (1) and (3)
trivially. However, on API 29+ Android restricts `getIccId()` to apps
holding `READ_PRIVILEGED_PHONE_STATE`, which is `signature|privileged` —
not ADB-grantable to a user-installed app. For NRLatch, `info.iccId`
returns an empty string. ICCID therefore fails (2).

## Options considered

### Option A — ICCID

`SubscriptionInfo.getIccId(): String`.

| Criterion | Status |
| --- | --- |
| (1) Stable | ✓ |
| (2) Accessible | ✗ (returns empty string on API 29+ for non-privileged apps) |
| (3) Per-card | ✓ |

**Rejected** because (2) fails categorically.

### Option B — `cardId` (CHOSEN)

`SubscriptionInfo.getCardId(): Int`. Returns a stable identifier for the
UICC (physical SIM card or eSIM) currently associated with this
subscription. Available API 29+ with `READ_PHONE_STATE` granted at runtime.
Returns `INVALID_CARD_ID = -1` if not supported.

| Criterion | Status |
| --- | --- |
| (1) Stable | ✓ per Android API contract; **empirically validated by spike Step 2** |
| (2) Accessible | ✓ with `READ_PHONE_STATE` |
| (3) Per-card | ✓ — `cardId` identifies the UICC, not the slot |

The integer is opaque (effectively a hash of the eUICC's identity / a
slot-bound but stable handle to the physical UICC). It is **not**
human-readable — the user-facing label uses `SubscriptionInfo.displayName`,
while `cardId` is internal-only.

### Option C — `subscriptionId` alone

`SubscriptionInfo.getSubscriptionId(): Int`.

| Criterion | Status |
| --- | --- |
| (1) Stable | ✗ — non-stable on Samsung when SIM is moved between slots, per architect brief and confirmed by community telemetry on dontkillmyapp + AOSP source. |
| (2) Accessible | ✓ |
| (3) Per-card | ✗ — bound to slot context, not card. |

**Rejected** because (1) and (3) fail.

### Option D — Composite `subId + carrierId`

A composite key combining `getSubscriptionId()` and
`getCarrierId()` (the MCC/MNC-derived numeric carrier identifier).

| Criterion | Status |
| --- | --- |
| (1) Stable | Partial — `carrierId` is stable per carrier, but two physical SIMs from the same carrier (e.g. two Airtel postpaid lines) collide. |
| (2) Accessible | ✓ |
| (3) Per-card | ✗ — collisions on same-carrier SIMs. |

**Rejected** as a primary scheme. Reserved as a **fallback** if the
spike's Step 2 invalidates Option B (i.e. cardId returns `-1` or proves
unstable on the target device).

## Decision

Use **Option B**: key `PreferenceStore` by `cardId: Int`.

### Interface signature

```kotlin
interface PreferenceStore {
    fun getPreferredMode(cardId: Int): Flow<NetworkMode?>
    suspend fun setPreferredMode(cardId: Int, mode: NetworkMode)
    suspend fun clearPreferredMode(cardId: Int)
}
```

`SubscriptionProvider` exposes the active card:

```kotlin
interface SubscriptionProvider {
    suspend fun getActiveDataSubscriptionInfo(): SimIdentity?
    fun activeDataSubscriptionFlow(): Flow<SimIdentity?>
}

data class SimIdentity(
    val subscriptionId: Int,
    val cardId: Int,
    val displayName: String
)
```

`PolicyEngine.resolve()` reads `getActiveDataSubscriptionInfo().cardId` and
queries `PreferenceStore.getPreferredMode(cardId)`.

## Consequences

### One non-obvious behavior

`cardId` is a hash-like Int. Logs (e.g. the diagnostic screen, debug
logcat) may surface the value verbatim because there is no privacy
implication: it is not user-derivable to an ICCID, IMSI, or phone number
without privileged data already on the device. User-facing UI labels SIMs
by `displayName` only.

### Cross-device portability is moot

If a user moves a SIM between two NRLatch-installed devices (a hypothetical
future scenario, out of scope for v1.0), preferences do not roam. This is
already true regardless of key choice because `allowBackup="false"` and
`fullBackupContent="false"` (see `04-security-hardening.md` §1) prevent
DataStore contents from leaving the device.

### Empirically validated by the spike

ADR-001's validation spike (`06-validation-spike.md` Step 2) explicitly
tests:

- That `cardId` returns a non-`-1` integer on the S24+ target device.
- That two consecutive calls return the same integer.
- (Stretch) Whether `iccId` is non-empty — a happy surprise that would
  unlock a future ADR amendment to migrate to ICCID. Expected: empty.

If Step 2 fails, fall back to Option D (composite) with documented
instability — and amend or supersede this ADR.

### Stretch path — future migration to ICCID

If a future Android version relaxes the `getIccId()` restriction for apps
holding `WRITE_SECURE_SETTINGS`, or if Samsung ships a non-standard accessor,
the architecture can migrate to ICCID:

1. Add a one-shot migration in `PreferenceStoreImpl` that, at first launch
   under the new scheme, reads each existing `cardId`-keyed entry, resolves
   the corresponding ICCID from `SubscriptionManager`, and rewrites the
   entry under the ICCID key.
2. Bump the DataStore preference key namespace.
3. Update this ADR's Status to SUPERSEDED.

This migration path is **not** in v1.0 scope. It is documented here so that
the ADR records the option without requiring a new ADR if the situation
changes.

### Lint / CI surface

None. The `cardId` choice is purely an interface decision and does not
affect the Single-Privileged-Caller invariant or any lint rule.

## Cross-references

- Spike that validates this choice empirically: [`06-validation-spike.md`](06-validation-spike.md) Step 2 + ADR-001.
- `PreferenceStore` interface signature: `02-public-interfaces.md`.
- DataStore-backed implementation specifics: `03-component-designs.md`.
