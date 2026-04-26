# 04 — Security Hardening

This document specifies the **mandatory** security posture for NRLatch's `:app`
module. Every attribute, resource file, build flag, and architectural rule
listed here is **non-negotiable for v1.0** unless explicitly amended via a new
ADR.

The threat model that drives these mitigations is in
[`threat-model.md`](threat-model.md). The CI gates that enforce them are in
[`08-ci-lint-rules.md`](08-ci-lint-rules.md).

## 1. Mandatory manifest hardening

The merged `:app` `AndroidManifest.xml` MUST conform to the snippet below. The
`tools:` namespace is required to drive the manifest merger.

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    package="com.nrlatch">

    <!-- Strip INTERNET if any transitive dependency declares it. -->
    <uses-permission android:name="android.permission.INTERNET"
        tools:node="remove"/>

    <!-- Privileged: granted via ADB only. The whole app exists to use this. -->
    <uses-permission android:name="android.permission.WRITE_SECURE_SETTINGS"/>

    <!-- Runtime permission. Required for SubscriptionManager and getCardId(). -->
    <uses-permission android:name="android.permission.READ_PHONE_STATE"/>

    <!-- For BootCompletedReceiver. See ADR-002. -->
    <uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED"/>

    <!-- Reserved for hypothetical future internal IPC. Declared so it cannot
         be hijacked by a malicious app registering the same name. -->
    <permission
        android:name="com.nrlatch.TOGGLE"
        android:protectionLevel="signature"/>

    <application
        android:name=".NrLatchApplication"
        android:label="@string/app_name"
        android:icon="@mipmap/ic_launcher"
        android:theme="@style/Theme.NrLatch"
        android:allowBackup="false"
        android:dataExtractionRules="@xml/data_extraction_rules"
        android:fullBackupContent="false"
        android:debuggable="false"
        android:taskAffinity=""
        android:usesCleartextTraffic="false"
        android:networkSecurityConfig="@xml/network_security_config">

        <activity
            android:name=".ui.MainActivity"
            android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.MAIN"/>
                <category android:name="android.intent.category.LAUNCHER"/>
            </intent-filter>
        </activity>

        <service
            android:name=".tile.NrLatchTileService"
            android:exported="true"
            android:icon="@drawable/ic_tile"
            android:label="@string/tile_label"
            android:permission="android.permission.BIND_QUICK_SETTINGS_TILE">
            <intent-filter>
                <action android:name="android.service.quicksettings.action.QS_TILE"/>
            </intent-filter>
        </service>

        <receiver
            android:name=".simswitch.DataSubscriptionChangedReceiver"
            android:exported="false">
            <intent-filter>
                <action android:name="android.telephony.action.DEFAULT_DATA_SUBSCRIPTION_CHANGED"/>
            </intent-filter>
        </receiver>

        <receiver
            android:name=".boot.BootCompletedReceiver"
            android:exported="false">
            <intent-filter>
                <action android:name="android.intent.action.BOOT_COMPLETED"/>
            </intent-filter>
        </receiver>
    </application>
</manifest>
```

### 1.1 `INTERNET` removal — merger semantics & verification

The `tools:node="remove"` directive instructs the Android Gradle Plugin's
manifest merger to **drop** any `<uses-permission android:name=".INTERNET"/>`
contributed by `:app`'s direct or transitive library manifests. This is
defense-in-depth: if a future maintainer pulls in a library that silently
declares `INTERNET`, the merge step strips it.

To **verify** the release APK has no `INTERNET` permission:

```bash
# Required: aapt from Android SDK build-tools.
aapt dump permissions app/build/outputs/apk/release/app-release.apk \
    | grep -i internet
# Expected output: (empty)
# CI fails if grep returns a non-empty line.
```

The CI pipeline (see `08-ci-lint-rules.md` §"Manifest verification") runs this
exact grep on every `assembleRelease` and fails the build if `INTERNET`
appears.

### 1.2 Permission rationale

| Permission | Why | Granting mechanism |
| --- | --- | --- |
| `WRITE_SECURE_SETTINGS` | The single privileged write the app exists to perform: `Settings.Global.putInt(..., "preferred_network_mode${'$'}subId", value)`. | `adb shell pm grant com.nrlatch android.permission.WRITE_SECURE_SETTINGS`. Cannot be granted from app code. |
| `READ_PHONE_STATE` | Required for `SubscriptionManager.getActiveSubscriptionInfoList()` and `SubscriptionInfo.getCardId()` on API 29+. See ADR-003. | Runtime request via `MainActivity` first-launch flow. |
| `RECEIVE_BOOT_COMPLETED` | Allows `BootCompletedReceiver` to re-apply the saved preference for the active data SIM at boot. See ADR-002. | Normal install-time grant. |

### 1.3 Custom signature permission `com.nrlatch.TOGGLE`

Currently **unused**. Declared with `protectionLevel="signature"` so that:

1. No third-party app can claim the name first (a known technique against apps
   that introduce a permission only later).
2. Any future internal IPC (e.g. an in-app widget with its own broadcast
   receiver) can adopt this gate without an upgrade migration.

Removing the declaration is acceptable only via a new ADR.

## 2. Required XML resource files

### 2.1 `app/src/main/res/xml/data_extraction_rules.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<data-extraction-rules>
    <cloud-backup disableIfNoEncryptionCapabilities="true">
        <exclude domain="root"/>
    </cloud-backup>
    <device-transfer>
        <exclude domain="root"/>
    </device-transfer>
</data-extraction-rules>
```

Combined with `android:allowBackup="false"` and `android:fullBackupContent="false"`,
this guarantees that no DataStore preference, no shared-prefs file, and no
internal-storage artifact is ever included in:

- Auto Backup to Google Drive,
- Samsung Smart Switch device-to-device transfer,
- `adb backup` (further blocked by `allowBackup="false"`).

### 2.2 `app/src/main/res/xml/network_security_config.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <base-config cleartextTrafficPermitted="false">
        <trust-anchors>
            <certificates src="system"/>
        </trust-anchors>
    </base-config>
</network-security-config>
```

NRLatch makes **zero** network calls. The deny-all cleartext + system-only
trust anchors are belt-and-suspenders behind the `INTERNET`-removal: even if a
future maintainer accidentally introduces a network call **and** restores
`INTERNET`, plaintext HTTP is rejected and only the system CA chain is honored.

## 3. Build config hardening

In `app/build.gradle.kts`:

```kts
android {
    namespace = "com.nrlatch"
    compileSdk = 35

    defaultConfig {
        applicationId = "com.nrlatch"
        minSdk = 29
        targetSdk = 35
        versionCode = 1
        versionName = "1.0"
    }

    buildTypes {
        release {
            isMinifyEnabled = true
            isShrinkResources = true
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
            signingConfig = signingConfigs.getByName("release")
        }
        debug {
            isDebuggable = true
            isMinifyEnabled = false
            applicationIdSuffix = ".debug"
        }
    }

    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }
}

// R8 full mode is the default for AGP 9; pinned explicitly for clarity:
// android.enableR8.fullMode=true in gradle.properties.
```

### 3.1 `proguard-rules.pro` — minimum required keeps

```pro
# Public interfaces — Hilt binds at runtime via these names.
# Package names are canonical per 02-public-interfaces.md.
-keep interface net.nrlatch.core.settings.NetworkModeController { *; }
-keep interface net.nrlatch.core.policy.ModePolicy { *; }
-keep interface net.nrlatch.core.policy.PolicyEngine { *; }
-keep interface net.nrlatch.core.preferences.PreferenceStore { *; }
-keep interface net.nrlatch.core.telephony.SubscriptionProvider { *; }

# DataStore generated classes.
-keep class androidx.datastore.*.** { *; }

# Hilt generated classes — required at runtime by Hilt's reflective lookups.
-keep class dagger.hilt.** { *; }
-keep class * extends dagger.hilt.android.internal.managers.ApplicationComponentManager { *; }
-keepclasseswithmembers class * {
    @dagger.hilt.android.AndroidEntryPoint <init>(...);
}
```

A CI ProGuard-rules sanity check (see `08-ci-lint-rules.md`) ensures none of
these `-keep` lines are deleted accidentally.

### 3.2 Release signing

| Property | Value |
| --- | --- |
| Algorithm | RSA 4096 |
| Validity | 100 years (36500 days) |
| Storage | Local encrypted file + ≥2 encrypted offline backups (e.g. encrypted USB + 1Password vault attachment). |
| Recovery plan | Documented in `docs/runbook/keystore-loss.md` (out of architecture scope). Loss = user must uninstall and re-grant `WRITE_SECURE_SETTINGS` to a new APK with a new signing identity. |

**Why this matters**: `WRITE_SECURE_SETTINGS` granted via `pm grant` is bound
to the APK's signing identity. Re-signing with a different keystore is
indistinguishable from a different app from the OS's perspective; the previous
grant is voided.

### 3.3 `PendingIntent` — `FLAG_IMMUTABLE` mandate

Every `PendingIntent` constructed in NRLatch source code MUST include
`PendingIntent.FLAG_IMMUTABLE`. v1.0 has no `PendingIntent` use sites, but
this rule applies to all future code. It is enforced by a CI lint check (see
`08-ci-lint-rules.md`).

## 4. Architectural invariants

### 4.1 The Single-Privileged-Caller invariant

> `NetworkModeController.setMode()` may **only** be called from one of the
> following three call sites. No other call site is permitted.
>
> 1. `NrLatchTileService.onClick()` — the user explicitly tapped the QS tile.
> 2. `DataSubscriptionChangedReceiver.onReceive()` — the OS broadcast a
>    `DEFAULT_DATA_SUBSCRIPTION_CHANGED` (protected, system-only sender).
> 3. `BootCompletedReceiver.onReceive()` — the OS broadcast `BOOT_COMPLETED`
>    (protected, system-only sender).

By corollary:

- `MainActivity` MUST NOT call `NetworkModeController.setMode()`.
- No future deep-link handler, custom-scheme `Intent` handler, or third-party
  IPC entry point may call it.
- No `Service.onStartCommand()` based on user-supplied intent extras may call
  it.

The CI custom lint rule `NrLatchSetModeFromExportedComponent` (see
`08-ci-lint-rules.md`) enforces this at build time. The build fails on
violation.

### 4.2 Why this invariant exists

The threat model row 1 (see `threat-model.md`) is the strongest reason. If a
supply-chain compromise of the app inserts code that takes an `Intent`'s
extras and routes them to `setMode()`, **and** that path is reachable from an
exported component, an attacker app can flip the radio mode arbitrarily by
crafting an `Intent`. By restricting `setMode()` callers to the three sites
above — none of which trust intent data — the attack surface for
WRITE_SECURE_SETTINGS abuse is reduced to:

- Compromise of SystemUI (out of our threat model), or
- Compromise of the OS itself (out of our threat model).

## 5. Defense-in-depth UI hardening

### 5.1 Tapjacking prevention

`MainActivity`'s root view must include:

```xml
android:filterTouchesWhenObscured="true"
```

Even though `MainActivity` has **no toggle controls** that can flip the radio
mode (the toggle lives only in the QS tile, see §4.1), the diagnostic-screen
buttons (`Open Battery Settings`, `Open Sleeping Apps`) deserve protection
against an overlay redirecting a user tap to a different system intent.

### 5.2 Receiver action-string validation

Every `BroadcastReceiver.onReceive()` MUST validate the incoming action even
though the broadcasts are protected:

```kotlin
override fun onReceive(context: Context, intent: Intent) {
    if (intent.action != TelephonyManager.ACTION_DEFAULT_DATA_SUBSCRIPTION_CHANGED) {
        return
    }
    // ...proceed
}
```

This guards against future manifest typos and keeps the runtime guarantee
explicit in code.

### 5.3 Explicit `ComponentName` in `PendingIntent`s

Any future `PendingIntent` MUST be constructed with an `Intent` that has its
`component` explicitly set via `Intent.setComponent(ComponentName(...))`. This
prevents intent-redirection attacks where an implicit intent is resolved to a
different app.

## 6. Things we deliberately DO NOT do

| Excluded mitigation | Reasoning |
| --- | --- |
| Runtime signature self-verification | Trivially bypassable by an attacker who has already compromised the APK (they can patch the verification call). Adds complexity without raising the bar. |
| Play Integrity attestation | Requires a server backend to verify the token. NRLatch has zero server presence (no `INTERNET`). Useless without a verifier. |
| Root detection | False-positive prone (custom ROMs, dev devices). NRLatch is a personal utility, not a financial app; rooted-device users are explicitly in scope. |
| String encryption / class obfuscation beyond R8 default | R8 already renames identifiers in release. Encrypting strings only marginally raises decompilation effort and offers zero protection against runtime introspection (Frida, Xposed). |
| Custom DRM, anti-debug traps, anti-Frida hooks | Same reason. The threat model accepts a compromised attacker-controlled device as out of scope (single-user developer device). |
| SafetyNet (deprecated) | Replaced by Play Integrity, which we also reject above. |

These exclusions are **explicit and accepted**. Re-introducing any of them
requires a new ADR with measured justification.

## 7. Cross-references

- Threat-model rows mapping to specific items here: [`threat-model.md`](threat-model.md).
- CI rules enforcing the invariants: [`08-ci-lint-rules.md`](08-ci-lint-rules.md).
- Samsung-specific reliability hardening (separate concern): [`05-samsung-reliability.md`](05-samsung-reliability.md).
- Manifest module ownership and dependency rules: `01-module-graph.md`.
