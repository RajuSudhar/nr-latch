# 05 — Samsung Reliability Strategy

This document describes how NRLatch achieves acceptable reliability on
Samsung Galaxy S24+ running One UI 6.1+, given Samsung's documented hostility
to background app execution.

The decisions here are coupled tightly to:

- ADR-002 ([`adr-002-boot-completed.md`](adr-002-boot-completed.md)) — why we
  ship `BootCompletedReceiver` despite the kill behavior.
- Threat-model row 3 ([`threat-model.md`](threat-model.md)) — Samsung
  background-killing as a functional-failure threat.
- The diagnostic-screen UI requirements that flow into
  [`03-component-designs.md`](03-component-designs.md).

## 1. Background-killing reality

Samsung's Galaxy devices have a **5/5 hostility score** on
[dontkillmyapp.com](https://dontkillmyapp.com/samsung) — the worst tier. The
relevant subsystems are:

- **Sleeping Apps**: an app placed here is killed shortly after entering the
  background and is **not** allowed to receive *any* manifest-declared
  broadcasts, including `BOOT_COMPLETED` and `DEFAULT_DATA_SUBSCRIPTION_CHANGED`.
- **Deep Sleeping Apps**: a stricter tier; the app is also denied push
  notifications and is never woken.
- **ChimeraPolicyHandler**: One UI's internal policy daemon that reclassifies
  apps over time. An app the user has used recently is considered "active" and
  spared. After ~3 days of non-use, ChimeraPolicyHandler may move the app to
  Sleeping Apps **automatically**, regardless of what the user previously
  configured, unless the app is explicitly on the **Never sleeping apps**
  allowlist.

**Critical fact**: on Samsung, **manifest-declared broadcast receivers are
NOT exempt from Sleeping Apps**. This contradicts the AOSP guarantee that
manifest-declared receivers wake the app. Without user intervention, NRLatch's
`DataSubscriptionChangedReceiver` and `BootCompletedReceiver` **will fail
intermittently** after a few days of inactivity.

The architecture therefore does not pretend reliability is automatic. Instead,
v1.0:

1. Surfaces the requirement to the user explicitly during first-launch.
2. Provides deep-links to the right Settings screens.
3. Provides a diagnostic screen the user can open at any time to verify
   broadcast-receiver health.

## 2. First-launch UX flow

`MainActivity` opens with a sequenced checklist. Each step is rendered as a
status row with a status indicator (`✓` granted / `!` action required) and an
action button. The flow proceeds top-to-bottom; the user is not blocked from
exiting at any step, but the diagnostic surface displays which steps remain.

### Step 1 — `WRITE_SECURE_SETTINGS` granted?

Check via `PackageManager.checkPermission("android.permission.WRITE_SECURE_SETTINGS", BuildConfig.APPLICATION_ID)`.

If `PERMISSION_DENIED`:

- Show the exact ADB command in a copyable text view:

  ```bash
  adb shell pm grant com.nrlatch android.permission.WRITE_SECURE_SETTINGS
  ```

- Provide a "Copy" button that puts that string on the clipboard.
- Provide a "How to enable USB debugging" expandable help block.

If `PERMISSION_GRANTED`: show `✓ Granted`.

### Step 2 — `READ_PHONE_STATE` granted?

Check via `ContextCompat.checkSelfPermission(this, Manifest.permission.READ_PHONE_STATE)`.

If `PERMISSION_DENIED`: render a "Grant" button that calls
`ActivityCompat.requestPermissions(...)`. On result, refresh the row.

If `PERMISSION_GRANTED`: show `✓ Granted`.

### Step 3 — Battery exemption

Check via `PowerManager.isIgnoringBatteryOptimizations(packageName)`.

If `false`: render an "Open battery settings" button that fires:

```kotlin
val intent = Intent(Settings.ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS).apply {
    data = Uri.parse("package:${'$'}packageName")
}
startActivity(intent)
```

Inline help text: "Tap **Allow** so Android keeps NRLatch's tiny SIM-watcher
alive after the screen goes off."

### Step 4 — Sleeping Apps allowlist (Samsung-specific)

This step has no programmatic check (Samsung exposes no public API for the
Sleeping Apps list). Show it as **always-actionable** with the instruction
text:

> **Add NRLatch to "Never sleeping apps".**
>
> 1. Tap the button below to open Samsung's battery settings.
> 2. Tap **Background usage limits**.
> 3. Tap **Never sleeping apps**.
> 4. Tap the **+** button.
> 5. Find and add **NRLatch**.
>
> Without this step, NRLatch's SIM-switch and boot receivers may stop firing
> after a few days.

The button preference list:

```kotlin
val candidates = listOf(
    Intent("com.samsung.android.sm.ACTION_BATTERY_USAGE"),
    Intent("com.samsung.android.lool.ACTION_BATTERY_ACTIVITY"),
    Intent(Settings.ACTION_APPLICATION_DETAILS_SETTINGS).apply {
        data = Uri.parse("package:${'$'}packageName")
    }
)
val resolved = candidates.first { it.resolveActivity(packageManager) != null }
startActivity(resolved)
```

### Step 5 — Verification

Static instructional copy:

> **Verify the install:**
> 1. Pull down the QS shade and add the **NRLatch** tile if you haven't.
> 2. Tap the tile — the radio should switch (5G ↔ LTE).
> 3. Swap the active data SIM in **Settings → Connections → SIM manager**.
> 4. Return here. The **"Last SIM-change broadcast"** timestamp below should
>    be within a few seconds of step 3.

A "Refresh" button re-reads the diagnostic state.

## 3. Diagnostic screen requirements

Rendered below the first-launch checklist on `MainActivity`. **All values are
read at `onResume()` only — there is no automatic polling, animation, or
periodic refresh.** A user-tap "Refresh" button forces a re-read.

### 3.1 Required fields

| Field | Source | Display format |
| --- | --- | --- |
| Last SIM-change broadcast | DataStore key `last_sim_change_at_millis`, written from `DataSubscriptionChangedReceiver.onReceive()`. | `2026-04-25 14:32:18  (3 minutes ago)` or `Never` |
| Last `BOOT_COMPLETED` | DataStore key `last_boot_completed_at_millis`, written from `BootCompletedReceiver.onReceive()`. | `2026-04-23 08:11:44  (2 days ago)` or `Never` |
| Active data SIM | `SubscriptionManager.getActiveSubscriptionInfo(SubscriptionManager.getDefaultDataSubscriptionId())`. | `Airtel Postpaid (cardId: 1726488811, subId: 4)` or `No active SIM` |
| Saved preference for this card | `PreferenceStore.getPreferredMode(cardId).first()`. | `NR_LTE` / `LTE_ONLY` / `(none)` |
| `WRITE_SECURE_SETTINGS` permission | `PackageManager.checkPermission(...)`. | `Y` or `N` |
| `READ_PHONE_STATE` permission | `ContextCompat.checkSelfPermission(...)`. | `Y` or `N` |

### 3.2 The "ago" formatter

Use `android.text.format.DateUtils.getRelativeTimeSpanString` for the relative
component (e.g. "3 minutes ago", "2 hours ago"). Concatenate with a fixed
ISO-8601-ish absolute timestamp for unambiguous reading.

If the underlying millis value is `0L` (never written), display `Never`.

### 3.3 No-network constraint

These rendering paths must not perform any I/O outside reading DataStore and
the `SubscriptionManager`/`PackageManager` queries listed above. There is no
network call, no analytics ping, no toast, no notification.

## 4. What we explicitly DO NOT do

| Anti-pattern | Reason for rejection |
| --- | --- |
| Persistent foreground service | Violates goal G3 ("zero noise" — no notifications). The user installed NRLatch precisely to *not* see another notification. |
| `JobScheduler` periodic checks | Battery cost is non-trivial for a utility that does nothing 99% of the time. The OS broadcast for `DEFAULT_DATA_SUBSCRIPTION_CHANGED` is already the right signal. |
| `WorkManager` wrapping the receivers | `WorkManager` does not bypass Sleeping Apps on Samsung — it relies on the same `JobScheduler` underneath. Adds complexity, dependency surface, and Hilt wiring for zero reliability gain. |
| Retry with exponential backoff | The receivers' work is **idempotent** (re-applying the same `Settings.Global.putInt` is a no-op). The next event (next SIM swap, next boot) will re-attempt naturally. Backoff machinery would just outlive the process under Samsung. |
| AlarmManager wake-up pings | Same reason as `JobScheduler`. Battery cost > value. |
| In-app analytics on receiver fires | No `INTERNET` permission, no backend. The diagnostic screen is the only telemetry surface and it's user-driven. |

## 5. Reliability acceptance criteria

The architecture is considered acceptable if all of the following hold,
**measured manually via the diagnostic screen**, with the user having
completed the first-launch flow including step 4:

1. **SIM-switch receiver**: after seven (7) consecutive days of mixed normal
   phone use (including periods of idle / sleep), the user can swap the data
   SIM via `Settings → Connections → SIM manager` and the diagnostic field
   "Last SIM-change broadcast" reflects the swap within 60 seconds, on at
   least 6 of 7 attempts.

2. **Boot receiver**: across 5 reboots performed within a 3-day window of
   active app use (i.e. user opened MainActivity at least once in the last 72h),
   the diagnostic field "Last BOOT_COMPLETED" reflects the boot in at least
   4 of 5 attempts (≥80%).

3. **Manual override always works**: tapping the QS tile from the lock screen
   or notification shade flips the mode within 5 seconds, regardless of any
   background-receiver state. This is the **fallback guarantee** when
   automatic behavior fails.

These thresholds are deliberately **manual / pen-and-paper**. There is no
telemetry pipeline. If the criteria are not met after the first-launch flow
is complete, the user (or developer) opens the Sleeping Apps list and
re-confirms NRLatch is on the allowlist; that is the only supported remedy.

## 6. Cross-references

- ADR-002 boot-receiver decision: [`adr-002-boot-completed.md`](adr-002-boot-completed.md).
- Diagnostic-screen component design: `03-component-designs.md`.
- DataStore keys consumed by the diagnostic screen: `02-public-interfaces.md` (`PreferenceStore`).
- Threat-model row 3: [`threat-model.md`](threat-model.md).
