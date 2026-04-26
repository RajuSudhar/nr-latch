# 08 — CI Lint Rules

This document defines the build-time gates that enforce NRLatch's
architectural invariants and security posture. Two custom checks plus a set
of off-the-shelf gates compose the pipeline.

The principal custom rule is `NrLatchSetModeFromExportedComponent`, which
enforces the Single-Privileged-Caller invariant from
[`04-security-hardening.md`](04-security-hardening.md) §4.1.

## 1. Custom Android Lint rule — `NrLatchSetModeFromExportedComponent`

### 1.1 Identity

| Field | Value |
| --- | --- |
| Issue id | `NrLatchSetModeFromExportedComponent` |
| Severity | `Severity.ERROR` (fails the build) |
| Category | `Category.SECURITY` |
| Priority | 10 |
| Scope | `Scope.JAVA_FILE_SCOPE` (Kotlin sources are included) |
| Implementation | `Implementation(NrLatchSetModeDetector::class.java, Scope.JAVA_FILE_SCOPE)` |

### 1.2 Brief description

> A Settings.Global write reachable from an exported component is a privilege
> hand-off to any app that can craft an `Intent`.

### 1.3 Long explanation

```
NRLatch holds android.permission.WRITE_SECURE_SETTINGS, granted via ADB.
Any code path that takes Intent data from an exported component and feeds
it (directly or transitively) into NetworkModeController.setMode() is a
confused-deputy vulnerability: an attacker app can flip the radio mode by
sending an Intent.

Allowed call sites for NetworkModeController.setMode(),
NetworkModeController.toggle(), and PolicyEngine.applyManualOverride() are:
  - NrLatchTileService.onClick
  - DataSubscriptionChangedReceiver.onReceive
  - BootCompletedReceiver.onReceive

If you genuinely need a new caller, add it to the allowlist in
NrLatchSetModeDetector AND add an ADR justifying the change.
```

### 1.4 Scope of analysis

The rule scans Kotlin sources in:

- `:app`
- `:feature:tile`
- `:feature:simswitch`
- Any future `:feature:*` module added to the project.

It does **not** scan `:core:*` because `:core` modules cannot have
manifest-declared exported entry points (they are pure-Kotlin / library
modules with no Application or Activity classes).

### 1.5 Allowlist

Hard-coded in the detector:

```kotlin
private val ALLOWLIST = setOf(
    "com.nrlatch.tile.NrLatchTileService#onClick",
    "com.nrlatch.simswitch.DataSubscriptionChangedReceiver#onReceive",
    "com.nrlatch.boot.BootCompletedReceiver#onReceive",
)
```

`NrLatchTileService` is technically `exported="true"` because the QS-tile
binding requires it. However, the only caller that can actually trigger
`onClick` is SystemUI, which holds `BIND_QUICK_SETTINGS_TILE` (signature). No
third-party app can invoke it. The allowlist therefore deliberately
includes that method.

The two receivers are `exported="false"` and listen only to OS-protected
broadcasts (`DEFAULT_DATA_SUBSCRIPTION_CHANGED`, `BOOT_COMPLETED`); their
allowlist entries are uncontroversial.

### 1.6 Detection strategy

The detector extends `SourceCodeScanner` and implements:

1. **`getApplicableMethodNames()` returns**:
   `["setMode", "toggle", "applyManualOverride"]`.
2. For each call to one of those methods:
   1. Resolve the receiver type.
   2. Filter to only calls where the receiver type is
      `com.nrlatch.core.controller.NetworkModeController` or
      `com.nrlatch.core.policy.PolicyEngine`.
   3. Walk up the AST to find the **enclosing method** (`UMethod`).
   4. Compute the fully-qualified method id `Class#method`.
   5. If that id is on the allowlist, skip.
   6. Otherwise, look at the enclosing class. If the class extends
      `Activity`, `Service`, or `BroadcastReceiver`, **or** is annotated
      `@AndroidEntryPoint` and the manifest declares it `exported="true"`
      (this requires a manifest cross-reference), report the issue.
   7. For non-entry-point classes, perform a shallow call-graph walk: any
      method invoked from an entry-point handler that itself calls the
      tracked methods is also flagged. Depth bounded at 5 to keep lint
      cheap.

For v1.0 the simpler heuristic suffices: the rule flags **any** call to the
three tracked methods unless the enclosing `Class#method` is on the
allowlist. Because `:core` modules cannot import the controllers (they are
the *defining* modules) and `:feature` modules contain only exported
components, this approximation has zero false positives in the present
codebase.

### 1.7 Skeleton — `NrLatchSetModeDetector.kt`

```kotlin
package com.nrlatch.lint

import com.android.tools.lint.client.api.UElementHandler
import com.android.tools.lint.detector.api.*
import org.jetbrains.uast.UCallExpression
import org.jetbrains.uast.UElement
import org.jetbrains.uast.UMethod
import org.jetbrains.uast.getContainingUClass
import org.jetbrains.uast.getContainingUMethod

class NrLatchSetModeDetector : Detector(), Detector.UastScanner {

    override fun getApplicableMethodNames(): List<String> =
        listOf("setMode", "toggle", "applyManualOverride")

    override fun visitMethodCall(
        context: JavaContext,
        node: UCallExpression,
        method: com.intellij.psi.PsiMethod
    ) {
        val receiverFqn = method.containingClass?.qualifiedName ?: return
        if (receiverFqn !in TRACKED_RECEIVER_TYPES) return

        val enclosingMethod: UMethod = node.getContainingUMethod() ?: return
        val enclosingClass = enclosingMethod.getContainingUClass() ?: return
        val id = "${'$'}{enclosingClass.qualifiedName}#${'$'}{enclosingMethod.name}"

        if (id in ALLOWLIST) return

        context.report(
            ISSUE,
            node,
            context.getLocation(node),
            "Call to ${'$'}{method.name}() from ${'$'}id is not on the " +
                "Single-Privileged-Caller allowlist. See " +
                "docs/architecture/04-security-hardening.md §4.1."
        )
    }

    companion object {
        private val TRACKED_RECEIVER_TYPES = setOf(
            "com.nrlatch.core.controller.NetworkModeController",
            "com.nrlatch.core.policy.PolicyEngine"
        )

        private val ALLOWLIST = setOf(
            "com.nrlatch.tile.NrLatchTileService#onClick",
            "com.nrlatch.simswitch.DataSubscriptionChangedReceiver#onReceive",
            "com.nrlatch.boot.BootCompletedReceiver#onReceive",
        )

        val ISSUE: Issue = Issue.create(
            id = "NrLatchSetModeFromExportedComponent",
            briefDescription = "Privileged setMode call reachable from an exported component",
            explanation = """
                NRLatch holds android.permission.WRITE_SECURE_SETTINGS. Calls
                to NetworkModeController.setMode() / toggle() and
                PolicyEngine.applyManualOverride() are restricted to:
                  - NrLatchTileService.onClick
                  - DataSubscriptionChangedReceiver.onReceive
                  - BootCompletedReceiver.onReceive
                See docs/architecture/04-security-hardening.md §4.1.
            """.trimIndent(),
            category = Category.SECURITY,
            priority = 10,
            severity = Severity.ERROR,
            implementation = Implementation(
                NrLatchSetModeDetector::class.java,
                Scope.JAVA_FILE_SCOPE
            )
        )
    }
}
```

### 1.8 Issue registry

```kotlin
package com.nrlatch.lint

import com.android.tools.lint.client.api.IssueRegistry
import com.android.tools.lint.client.api.Vendor
import com.android.tools.lint.detector.api.CURRENT_API
import com.android.tools.lint.detector.api.Issue

class NrLatchIssueRegistry : IssueRegistry() {
    override val issues: List<Issue> = listOf(NrLatchSetModeDetector.ISSUE)
    override val api: Int = CURRENT_API
    override val vendor: Vendor = Vendor(
        vendorName = "NRLatch",
        identifier = "com.nrlatch.lint",
        feedbackUrl = "https://github.com/sudharsan/nrlatch/issues"
    )
}
```

Register via `META-INF/services/com.android.tools.lint.client.api.IssueRegistry`
in the `lint/` module's resources.

### 1.9 Implementation location

Gradle module `:lint` of plugin type `com.android.lint`:

```
lint/
├── build.gradle.kts          // applies com.android.lint
├── src/main/kotlin/com/nrlatch/lint/
│   ├── NrLatchSetModeDetector.kt
│   └── NrLatchIssueRegistry.kt
└── src/test/kotlin/com/nrlatch/lint/
    └── NrLatchSetModeDetectorTest.kt
```

`:lint`'s `build.gradle.kts`:

```kts
plugins {
    id("com.android.lint")
    id("org.jetbrains.kotlin.jvm")
}

dependencies {
    compileOnly("com.android.tools.lint:lint-api:31.9.0")
    compileOnly("com.android.tools.lint:lint-checks:31.9.0")
    testImplementation("com.android.tools.lint:lint:31.9.0")
    testImplementation("com.android.tools.lint:lint-tests:31.9.0")
    testImplementation("junit:junit:4.13.2")
}
```

(Pin lint-api version to AGP 9.1.0's matching lint version per `libs.versions.toml`.)

Other modules wire the rule in via:

```kts
// In :app, :feature:tile, :feature:simswitch build.gradle.kts:
dependencies {
    lintChecks(project(":lint"))
}
```

### 1.10 Test cases

`NrLatchSetModeDetectorTest.kt` using `lint-test-junit`:

```kotlin
class NrLatchSetModeDetectorTest : LintDetectorTest() {

    override fun getDetector() = NrLatchSetModeDetector()
    override fun getIssues() = listOf(NrLatchSetModeDetector.ISSUE)

    @Test
    fun `tile onClick is allowlisted`() {
        lint().files(
            kotlin("""
                package com.nrlatch.tile
                import com.nrlatch.core.controller.NetworkModeController
                import com.nrlatch.core.model.NetworkMode

                class NrLatchTileService(
                    private val controller: NetworkModeController
                ) {
                    fun onClick() {
                        controller.setMode(NetworkMode.NR_LTE)
                    }
                }
            """).indented(),
            controllerStub
        ).run().expectClean()
    }

    @Test
    fun `MainActivity calling setMode is flagged`() {
        lint().files(
            kotlin("""
                package com.nrlatch.ui
                import android.app.Activity
                import com.nrlatch.core.controller.NetworkModeController
                import com.nrlatch.core.model.NetworkMode

                class MainActivity(
                    private val controller: NetworkModeController
                ) : Activity() {
                    fun onSomeButtonClick() {
                        controller.setMode(NetworkMode.LTE_ONLY) // BAD
                    }
                }
            """).indented(),
            controllerStub
        ).run().expectErrorCount(1)
    }

    private val controllerStub = kotlin("""
        package com.nrlatch.core.controller
        import com.nrlatch.core.model.NetworkMode
        interface NetworkModeController {
            fun setMode(mode: NetworkMode)
            fun toggle()
        }
    """).indented()
}
```

## 2. `PendingIntentImmutability` — secondary lint check

A second custom check (small enough to live in the same `:lint` module).
Severity ERROR.

**Rule**: every `PendingIntent.getActivity / getBroadcast / getService /
getForegroundService` invocation must include `FLAG_IMMUTABLE` in the flags
argument.

This is a strict superset of the platform's bundled
`UnspecifiedImmutableFlag` lint check; we keep our own only because we want
the failure to be ERROR even on minSdk < 31.

v1.0 has no `PendingIntent` use sites. The rule is a future-proofing tripwire.

## 3. Off-the-shelf CI gates

### 3.1 `dependency-analysis-gradle-plugin`

Configured in the root `build.gradle.kts`:

```kts
plugins {
    id("com.autonomousapps.dependency-analysis") version "2.1.0"
}

dependencyAnalysis {
    issues {
        all {
            onAny { severity("fail") }
        }
    }
    structure {
        // Forbid :core:* from depending on :feature:* — the architectural rule.
        // This is enforced as a custom rule in the plugin's `structure` DSL
        // (or via a graph-assertion test in :build-logic).
    }
}
```

Run via `./gradlew dependencyAnalysis`. CI fails if:

- Any module declares an unused dependency.
- Any module uses an undeclared transitive dependency.
- Any `:core:*` module depends on a `:feature:*` module (the layering rule).
- Any `:core:*` module depends on `:app`.

### 3.2 Manifest permission verification

A CI step that runs after `assembleRelease`:

```bash
APK=app/build/outputs/apk/release/app-release.apk
echo "Inspecting permissions in ${'$'}APK..."
PERMS=${'$'}(aapt dump permissions "${'$'}APK")

echo "${'$'}PERMS"

# Required absent:
if echo "${'$'}PERMS" | grep -qi 'android.permission.INTERNET'; then
    echo "FAIL: INTERNET permission present in release APK."
    exit 1
fi

# Required present:
for required in WRITE_SECURE_SETTINGS READ_PHONE_STATE RECEIVE_BOOT_COMPLETED; do
    if ! echo "${'$'}PERMS" | grep -q "${'$'}required"; then
        echo "FAIL: required permission ${'$'}required missing."
        exit 1
    fi
done

echo "PASS: manifest permission set is correct."
```

This script lives in `scripts/verify-manifest.sh` and is invoked as a CI step
after `assembleRelease`.

### 3.3 ProGuard rules sanity check

A trivial CI step:

```bash
for keep in \
    "com.nrlatch.core.controller.NetworkModeController" \
    "com.nrlatch.core.policy.PolicyEngine" \
    "com.nrlatch.core.preferences.PreferenceStore"; do
    if ! grep -q "${'$'}keep" app/proguard-rules.pro; then
        echo "FAIL: -keep for ${'$'}keep missing in proguard-rules.pro"
        exit 1
    fi
done
```

Catches the regression where a maintainer reformats the file and drops a rule.

## 4. Manual verification before each release

In addition to the automated CI gates, before tagging a release:

1. Run the on-device manual checklist in [`checklist.md`](checklist.md).
2. Decompile the release APK with [`apktool`](https://apktool.org):

   ```bash
   apktool d app-release.apk -o release-decoded
   grep -E 'permission|exported' release-decoded/AndroidManifest.xml
   ```

   Expected: only the four declared permissions (`WRITE_SECURE_SETTINGS`,
   `READ_PHONE_STATE`, `RECEIVE_BOOT_COMPLETED`, custom `com.nrlatch.TOGGLE`)
   appear. No `INTERNET`. No `<uses-permission>` block from a transitive dep.

3. Spot-check the `<application>` block in the decoded manifest still has
   `allowBackup="false"`, `taskAffinity=""`, `usesCleartextTraffic="false"`,
   `networkSecurityConfig="@xml/network_security_config"`.

## 5. Cross-references

- The invariant being enforced: [`04-security-hardening.md`](04-security-hardening.md) §4.1.
- The threat row driving these gates: [`threat-model.md`](threat-model.md) row 1.
- Module layering rule enforced in §3.1: `01-module-graph.md`.
- Pre-release manual checklist: [`checklist.md`](checklist.md).
