# NRLatch — Module Graph

This document specifies the eight-module Gradle layout, the dependency direction rules between them, and the per-module Gradle wiring expressed via the version catalog (`libs.versions.toml`).

## Final `settings.gradle.kts` Layout

```kotlin
rootProject.name = "NRLatch"
include(":app")
include(":core:model")
include(":core:settings")
include(":core:telephony")
include(":core:preferences")
include(":core:policy")
include(":feature:tile")
include(":feature:simswitch")
```

## Per-module Justification

| Module | Justification (one line) |
|--------|--------------------------|
| `:app` | Application module. Hosts `MainActivity`, the merged manifest, the `Application` class with Hilt root, and stitches DI bindings together. |
| `:core:model` | Pure-Kotlin module for value types (`NetworkMode`, `SimIdentity`, `PolicyDecision`, `PerSimPreference`). No Android dependencies — JVM only — so it can be consumed by any other module without contamination. |
| `:core:settings` | Owns the `NetworkModeController` interface and its `Settings.Global` write impl. Isolated so the privileged write surface has exactly one home. |
| `:core:telephony` | Owns the `SubscriptionProvider` interface and the `SubscriptionManager` impl. Isolated so `READ_PHONE_STATE`-gated APIs have exactly one consumer. |
| `:core:preferences` | Owns the `PreferenceStore` interface and the DataStore Preferences impl. Isolated so persistence is replaceable without touching policy or features. |
| `:core:policy` | Owns the `ModePolicy` and `PolicyEngine` interfaces and the v1.0 `ManualOnlyPolicyEngine`. Extensibility seam for future battery-aware or context-aware policies. |
| `:feature:tile` | Hosts `NrLatchTileService`. Depends on `:core:settings` and `:core:policy`; never reaches into `:core:preferences` directly. |
| `:feature:simswitch` | Hosts `DataSubscriptionChangedReceiver` and `BootCompletedReceiver`. Depends on `:core:telephony`, `:core:policy`, `:core:settings`. |

## Module Type Matrix

| Module | Plugin | Has manifest | Exposes interface | Has impl |
|--------|--------|--------------|-------------------|----------|
| `:app` | `com.android.application` | yes | no | yes |
| `:core:model` | `org.jetbrains.kotlin.jvm` | no | n/a (value types) | n/a |
| `:core:settings` | `com.android.library` | yes (minimal, no components) | yes (`NetworkModeController`) | yes (`NetworkModeControllerImpl`) |
| `:core:telephony` | `com.android.library` | yes (minimal, no components) | yes (`SubscriptionProvider`) | yes (`SubscriptionProviderImpl`) |
| `:core:preferences` | `com.android.library` | yes (minimal, no components) | yes (`PreferenceStore`) | yes (`PreferenceStoreImpl`) |
| `:core:policy` | `com.android.library` | yes (minimal, no components) | yes (`ModePolicy`, `PolicyEngine`) | yes (`ManualOnlyPolicyEngine`) |
| `:feature:tile` | `com.android.library` | yes (declares `NrLatchTileService`) | no | yes |
| `:feature:simswitch` | `com.android.library` | yes (declares both receivers) | no | yes |

`:core:model` uses the JVM Kotlin plugin rather than the Android library plugin because it carries no Android types. This keeps it dependable from any module without dragging in an Android target.

## Dependency Direction Rules

1. `:feature:*` modules may depend on any `:core:*` module they need. **`:core:*` modules MUST NOT depend on `:feature:*` modules.**
2. `:core:*` modules may depend on `:core:model` and on each other where strictly necessary (`:core:policy` → `:core:preferences` is the only inter-core edge in v1.0).
3. `:app` may depend on every module.
4. `api()` is used **only** when an interface return type or parameter type from another module appears in a public API surface. Examples:
   - `:core:settings` declares `api(project(":core:model"))` because `NetworkMode` is a return type of `NetworkModeController.getCurrentMode`.
   - `:core:telephony` declares `api(project(":core:model"))` because `SimIdentity` is a return type of `SubscriptionProvider.getActiveSubscription`.
   - `:core:preferences` declares `api(project(":core:model"))` because `NetworkMode` and `PerSimPreference` cross the `PreferenceStore` boundary.
   - `:core:policy` declares `api(project(":core:model"))` because `PolicyDecision`/`NetworkMode` cross the `PolicyEngine` boundary.
5. All other inter-module dependencies use `implementation()` to keep the ABI surface small and the build incremental.
6. Feature → core dependencies are `implementation()` because feature modules do not re-export core types.

The dependency-direction rule is enforced by the `dependency-analysis-gradle-plugin` (see snippet below).

## Explicit Dependency Graph

```
:app ──→ :feature:tile ──→ :core:settings ──→ :core:model
     │                 ──→ :core:policy   ──→ :core:model
     │                                    ──→ :core:preferences ──→ :core:model
     │
     ──→ :feature:simswitch ──→ :core:telephony ──→ :core:model
     │                     ──→ :core:settings  ──→ :core:model
     │                     ──→ :core:policy    ──→ :core:model
     │                                         ──→ :core:preferences ──→ :core:model
     │
     ──→ :core:settings, :core:telephony, :core:preferences,
         :core:policy, :core:model   (for DI module wiring in :app)
```

## Visibility Rules

- All implementation classes (`NetworkModeControllerImpl`, `SubscriptionProviderImpl`, `PreferenceStoreImpl`, `ManualOnlyPolicyEngine`) are declared `internal` in their host module.
- Only the interfaces and value types are `public`.
- Hilt bindings live in an `internal` `@Module` inside each `:core:*` module and use `@Binds` to map the `internal` impl to the `public` interface. The `@Module` itself is `internal`; Hilt's annotation processor still picks it up because Hilt runs at compile time within the same module.
- `:app` depends on every `:core:*` module *only* so that the Hilt component graph in the application can resolve all bindings; it never touches impl classes directly.

## Why no Convention Plugins

Eight modules with three plugin types (`com.android.application`, `com.android.library`, `org.jetbrains.kotlin.jvm`) and a shared version catalog do not justify a `build-logic/` convention-plugin tree. Convention plugins are valuable when the same fifteen-line `android { … }` block is repeated across twenty modules. Here the duplication is bounded, the build files are short, and the version catalog already centralizes versions. Introducing convention plugins would add a precompiled-script-plugin classpath, slow first-time configuration, and obscure the per-module config for an engineer reading the project for the first time. **YAGNI at this scale.**

## Dependency-analysis Plugin Configuration

In the root `build.gradle.kts`:

```kotlin
plugins {
    alias(libs.plugins.dependency.analysis)
}

dependencyAnalysis {
    structure {
        // Disallow :core:* depending on :feature:*
        ignoreKtx(true)
    }
    issues {
        all {
            onIncorrectConfiguration { severity("fail") }
            onUnusedDependencies     { severity("fail") }
            onUsedTransitiveDependencies { severity("fail") }
        }
        // Custom rule: any :core:* module that declares a dependency on a :feature:* module fails the build.
        project(":core:model")       { onAny { severity("fail") } }
        project(":core:settings")    { onAny { severity("fail") } }
        project(":core:telephony")   { onAny { severity("fail") } }
        project(":core:preferences") { onAny { severity("fail") } }
        project(":core:policy")      { onAny { severity("fail") } }
    }
}

// Project-evaluation guard: hard-fail if any :core:* project depends on a :feature:* project.
gradle.projectsEvaluated {
    subprojects
        .filter { it.path.startsWith(":core:") }
        .forEach { core ->
            core.configurations.all { cfg ->
                cfg.dependencies.matching { dep ->
                    dep is ProjectDependency && dep.dependencyProject.path.startsWith(":feature:")
                }.all {
                    throw GradleException(
                        "Forbidden dependency: ${core.path} -> ${(it as ProjectDependency).dependencyProject.path}"
                    )
                }
            }
        }
}
```

The hand-rolled `projectsEvaluated` guard is the load-bearing enforcement; the analysis plugin's reports are advisory by comparison.

## Per-module Gradle Dependencies (libs.versions.toml aliases)

### `:core:model` (`org.jetbrains.kotlin.jvm`)

- `kotlinx.coroutines.core` *(only the Flow type from the public API of `:core:telephony`/`:core:preferences`/`:core:policy` — `:core:model` itself imports no coroutines symbols and may omit this if the value types remain Flow-free; current model types are pure data classes / enums, so no coroutines dep is required here)*

In v1.0 `:core:model` has **no** runtime dependencies.

### `:core:settings` (`com.android.library`)

- `api(project(":core:model"))`
- `androidx.core.ktx`
- `kotlinx.coroutines.android`
- `hilt.android`
- `ksp(hilt.compiler)`

### `:core:telephony` (`com.android.library`)

- `api(project(":core:model"))`
- `androidx.core.ktx`
- `kotlinx.coroutines.android`
- `hilt.android`
- `ksp(hilt.compiler)`

### `:core:preferences` (`com.android.library`)

- `api(project(":core:model"))`
- `androidx.core.ktx`
- `androidx.datastore.preferences`
- `kotlinx.coroutines.android`
- `hilt.android`
- `ksp(hilt.compiler)`

### `:core:policy` (`com.android.library`)

- `api(project(":core:model"))`
- `implementation(project(":core:preferences"))`
- `implementation(project(":core:settings"))`
- `kotlinx.coroutines.android`
- `hilt.android`
- `ksp(hilt.compiler)`

### `:feature:tile` (`com.android.library`)

- `implementation(project(":core:model"))`
- `implementation(project(":core:settings"))`
- `implementation(project(":core:policy"))`
- `implementation(project(":core:telephony"))`
- `androidx.core.ktx`
- `kotlinx.coroutines.android`
- `hilt.android`
- `ksp(hilt.compiler)`

### `:feature:simswitch` (`com.android.library`)

- `implementation(project(":core:model"))`
- `implementation(project(":core:settings"))`
- `implementation(project(":core:policy"))`
- `implementation(project(":core:telephony"))`
- `implementation(project(":core:preferences"))`
- `androidx.core.ktx`
- `kotlinx.coroutines.android`
- `hilt.android`
- `hilt.androidx.compiler` *(for `BroadcastReceiver` injection)*
- `ksp(hilt.compiler)`
- `ksp(hilt.androidx.compiler)`

### `:app` (`com.android.application`)

- `implementation(project(":core:model"))`
- `implementation(project(":core:settings"))`
- `implementation(project(":core:telephony"))`
- `implementation(project(":core:preferences"))`
- `implementation(project(":core:policy"))`
- `implementation(project(":feature:tile"))`
- `implementation(project(":feature:simswitch"))`
- `androidx.core.ktx`
- `androidx.appcompat`
- `androidx.constraintlayout`
- `androidx.material`
- `androidx.lifecycle.runtime.ktx`
- `kotlinx.coroutines.android`
- `hilt.android`
- `ksp(hilt.compiler)`
- Test: `bundles.test.jvm`

Cross-references:

- For the version-catalog aliases used above, see [`libs.versions.toml`](./libs.versions.toml).
- For the public interfaces referenced from these modules, see [`02-public-interfaces.md`](./02-public-interfaces.md).
- For the components hosted in `:feature:*` and `:app`, see [`03-component-designs.md`](./03-component-designs.md).
