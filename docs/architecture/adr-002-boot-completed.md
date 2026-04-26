# ADR-002: Include `BOOT_COMPLETED` Receiver in v1.0 (Best-Effort)

- **Status**: ACCEPTED
- **Date**: 2026-04-25
- **Supersedes**: —
- **Superseded by**: —

## Context

After a phone reboot, the value of the secure setting
`preferred_network_mode${'$'}subId` may be reset by carrier configuration or by
the device's `ro.telephony.default_network` build property. On Samsung
specifically, OEM modem-init scripts have been observed to overwrite this
value at boot, particularly after a firmware update or carrier-config push.
The user-visible symptom is: "I set NR_LTE last week, the phone rebooted
overnight, today my preference is silently gone."

The straightforward remedy is a manifest-declared `BOOT_COMPLETED` receiver
that re-applies the saved preference for the active data SIM. However:

- Samsung's One UI is rated 5/5 on dontkillmyapp; manifest-declared receivers
  are **not** exempt from Sleeping Apps.
- An app placed in Sleeping Apps will not receive `BOOT_COMPLETED`.
- ChimeraPolicyHandler may move NRLatch to Sleeping Apps automatically after
  ~3 days of inactivity unless the user has added it to "Never sleeping
  apps".
- Therefore, a `BootCompletedReceiver` is best-effort: it works when the user
  has correctly completed the first-launch reliability flow (see
  [`05-samsung-reliability.md`](05-samsung-reliability.md) §2), and silently
  fails to fire otherwise.

The PRD's success criterion #3 ("preferred mode survives reboot when the
app has recently been used") is achievable but probabilistic.

## Options considered

### Option A — In-scope, best-effort (CHOSEN)

Ship a manifest-declared `BootCompletedReceiver` with `exported="false"`
listening for `android.intent.action.BOOT_COMPLETED`. On fire:

1. Validate intent action.
2. Read active data subId / cardId.
3. Read saved preference for that cardId from `PreferenceStore`.
4. If a preference exists, call `NetworkModeController.setMode(savedMode)`.
5. Update DataStore key `last_boot_completed_at_millis` for the diagnostic
   screen.

The first-launch flow guides the user to add NRLatch to "Never sleeping
apps" so the receiver fires reliably. The diagnostic screen surfaces the
"Last BOOT_COMPLETED" timestamp so the user can self-verify.

### Option B — Out-of-scope; relax PRD success criterion #3

Drop the receiver entirely. Document that the user must tap the QS tile
manually after each reboot to restore their preference.

**Rejected** because:

- The whole point of a saved preference is "set it once". Forcing a manual
  tap after every reboot is a UX downgrade.
- ~20 LOC and a single manifest entry deliver substantial value when the
  receiver does fire.

### Option C — Foreground service to ensure delivery

Run a persistent foreground service that survives Sleeping Apps and applies
the preference on boot.

**Rejected** because:

- Violates goal G3 ("zero noise"). The user installed NRLatch to *avoid*
  another notification.
- A foreground service drains battery 24/7 for an event that occurs at most
  once per several days.
- Even foreground services on Samsung are subject to background-process
  policies after API 31 / 33 changes; the win is not as large as it appears.

### Option D — `WorkManager` periodic check on boot

Schedule a `OneTimeWorkRequest` from the receiver, trusting WorkManager to
deliver it.

**Rejected** because WorkManager wraps `JobScheduler`, which is *also*
subject to Samsung's background restrictions. It adds a dependency surface
and Hilt wiring without solving the underlying reliability problem.

## Decision

Adopt **Option A**. Ship `BootCompletedReceiver`:

- `android:exported="false"`
- Filters `android.intent.action.BOOT_COMPLETED`.
- Body is a thin wrapper that delegates to a plain Kotlin `SimSwitchHandler`
  class (shared with `DataSubscriptionChangedReceiver`) so the logic is
  unit-tested in `:core:policy`.
- Documented Samsung-kill caveat in `05-samsung-reliability.md` §1 and in
  the first-launch flow's step 4.
- No retry, no foreground promotion, no JobScheduler chaining. If the
  receiver doesn't fire, the next QS tile tap restores the preference
  manually — the fallback guarantee.

## Consequences

### Code surface

- ~20 lines of Kotlin (`BootCompletedReceiver.kt`).
- One `<receiver>` block in the manifest.
- One DataStore key (`last_boot_completed_at_millis`) consumed only by the
  diagnostic screen.
- One Hilt entry-point annotation (`@AndroidEntryPoint` on the receiver).
- One additional `<uses-permission>`: `android.permission.RECEIVE_BOOT_COMPLETED`.

### Reliability profile

- **Best-effort**: ≥ 80% of reboots fire the receiver when the user has
  used the app within the prior 3 days and is on the "Never sleeping apps"
  allowlist (acceptance criterion in `05-samsung-reliability.md` §5).
- **Falls back gracefully**: when the receiver doesn't fire, the user simply
  taps the QS tile once. No data loss, no inconsistent state, no error
  toast.

### Documentation surface

- `05-samsung-reliability.md` includes a step-4 first-launch instruction
  that explicitly mentions boot-time restore.
- The PRD's success criterion #3 stays as written. We do not weaken the
  goal; we accept that the goal is best-effort.

### Threat-model surface

- Threat-model row 4 ("Carrier config resets `preferred_network_mode` on
  reboot") cites this receiver as the mitigation.
- The receiver itself is `exported="false"` and only listens to a protected
  broadcast, so it does not increase attack surface beyond the manifest
  entry. Action validation in `onReceive` is mandatory per
  `04-security-hardening.md` §5.2.

### CI surface

- The custom lint rule `NrLatchSetModeFromExportedComponent` (see
  `08-ci-lint-rules.md`) explicitly allowlists
  `BootCompletedReceiver#onReceive` as a permitted caller of
  `NetworkModeController.setMode()`.

## Cross-references

- Reliability strategy and first-launch flow: [`05-samsung-reliability.md`](05-samsung-reliability.md).
- Security invariants and allowlist: [`04-security-hardening.md`](04-security-hardening.md) §4.1.
- Threat row 4: [`threat-model.md`](threat-model.md).
- CI rule allowlisting the receiver: [`08-ci-lint-rules.md`](08-ci-lint-rules.md) §1.5.
