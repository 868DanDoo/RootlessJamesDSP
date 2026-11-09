# Baseline Build Record (Phase 0)

**Date:** 8 November 2026
**Result: the baseline build was NOT verified.** No APK was produced and no test was run. The blockers are environmental and are recorded below exactly as observed. Nothing here should be read as "the project builds" or "the project does not build".

---

## 1. What was pinned

| Item | Value |
|---|---|
| App repo | `868DanDoo/RootlessJamesDSP` @ `60d25ae8a53c6f4691c090673df290a73c6b6357` (2026-07-25) |
| Native submodule | `ThePBone/JamesDSPManager` @ `1b9f89dbd4b332a07724beb41d58bc2ffcb05395` (`heads/extensions`) |
| Gradle wrapper | 8.10.2 (`gradle/wrapper/gradle-wrapper.properties`) |
| AGP / Kotlin / KSP | 8.7.3 / 2.1.0 / 2.1.0-1.0.29 |
| compileSdk / targetSdk / minSdk | 35 / 34 / 29 |
| NDK/CMake | CMake 3.22.1 required by `app/build.gradle.kts`; NDK version not pinned in the script |

The submodule was fetched successfully with `git submodule update --init --depth 1 app/src/main/cpp/libjamesdsp`, which checked out the pinned commit.

## 2. Commands run and exact results

All commands were run from the repository root in the audit workspace.

| # | Command | Result |
|---|---|---|
| 1 | `git submodule update --init --depth 1 app/src/main/cpp/libjamesdsp` | **Success.** Checked out `1b9f89db…`. |
| 2 | `./gradlew tasks --all` | **Failed (exit 1).** The wrapper tried to download `https://services.gradle.org/distributions/gradle-8.10.2-bin.zip` and died with `Unable to tunnel through proxy. Proxy returns "HTTP/1.1 403 Forbidden"`. |
| 3 | `gradle tasks --all --no-daemon` (system Gradle **8.14.3**, not the wrapper version) | **Failed (exit 1).** `buildSrc/build.gradle.kts` line 1: `Plugin [id: 'org.gradle.kotlin.kotlin-dsl', version: '5.2.0'] was not found` (Gradle Plugin Portal unreachable). |
| 4 | `curl` reachability probes | `services.gradle.org`, `plugins.gradle.org`, `maven.google.com`, `dl.google.com`, `repo.maven.apache.org`, `jitpack.io` all returned no HTTP response (`000`). The session proxy's own status listed a `connect_rejected` / 403 for `services.gradle.org:443`. |

Not attempted, because steps 2–4 show the dependencies cannot be downloaded here: `assembleRootless*` builds, unit tests, lint.

## 3. Toolchain found in the workspace

- JDK: OpenJDK **21** (the project compiles to Java 17 bytecode; use JDK 17 for parity with CI-style builds).
- Gradle: system 8.14.3 (not the pinned 8.10.2).
- **No Android SDK** (`ANDROID_HOME` / `ANDROID_SDK_ROOT` empty), no NDK, no CMake.

## 4. Why this blocks, and what it does not tell us

The network policy of the audit workspace only allows a small set of hosts. The Gradle distribution, Gradle plugin portal, Google Maven (AGP, AndroidX, Firebase), Maven Central and JitPack (used for `unifile`, `Treessence`, `libsu`) are all required and none were reachable. The Android SDK and NDK are also absent. The results above are therefore about the workspace, not the project.

## 5. Variant names (inferred, not confirmed)

From `app/build.gradle.kts` (flavors `rootless|root|plugin` × `fdroid|full`, build types `debug|release|preview`). The script itself references these literal task names in `afterEvaluate`: `assembleRootlessFullRelease`, `assembleRootFullRelease`, `assembleRootlessFullPreview`, `assembleRootFullPreview`.

Recommended first baseline build for this project (privacy-friendly, no Firebase):

```
./gradlew assembleRootlessFdroidDebug
```

Debug builds append `.debug` to the application ID (`me.timschneeberger.rootlessjamesdsp.debug`), so they install side-by-side with a release build.

Confirm names with `./gradlew tasks --all` before scripting anything.

## 6. How to produce the baseline record on your PC

Prerequisites: Android Studio (recent), JDK 17, Android SDK Platform 35, NDK (side-by-side) and CMake 3.22.1 from the SDK Manager, `git` on PATH.

```
git clone --recursive https://github.com/868DanDoo/RootlessJamesDSP.git
cd RootlessJamesDSP
git rev-parse HEAD
git submodule status
./gradlew tasks --all > docs/gradle-tasks.txt
./gradlew assembleRootlessFdroidDebug
```

Use a **full** clone (not `--depth`), because the build runs `git rev-list --count HEAD`.

Record in this file after the first successful build: the exact command, Gradle/JDK/SDK/NDK versions, the APK path(s) under `app/build/outputs/apk/`, and `aapt dump badging <apk>` output (package, version, minSdk, targetSdk).

If the build fails, copy the first error and the Gradle task that raised it. Do not assume the cause is the fork: a known pre-existing hazard is the JitPack dependencies (`unifile`, `Treessence`), which depend on JitPack being reachable.

## 7. Build-script hazards noticed (not changed)

- `release` is signed with the debug key.
- `google-services.json` is committed; `full` release builds enable Crashlytics.
- `getCommitCount()` is wrong on shallow clones.
- Test coverage is minimal (2 JVM tests), so a green build says little about behaviour.

## 8. Status

| Exit criterion (roadmap Phase 0) | Status |
|---|---|
| Clean baseline build succeeds or a precise pre-existing blocker is documented | **Documented blocker (environment), build not verified** |
| Exact source and submodule commits pinned | **Done** |
