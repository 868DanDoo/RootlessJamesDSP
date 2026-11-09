# Upstream Audit — RootlessJamesDSP (Phase 0)

**Audit date:** 8 November 2026
**Method:** static source review of the checked-out tree. **Nothing in this document was verified by running the app or the build** (see `BASELINE_BUILD.md` for why). Findings marked *static* are read from code; they are leads to test on a device, not confirmed runtime bugs.
**Scope rule:** no application behaviour, native code, dependency versions, package ID or submodule pin was changed.

---

## 1. Provenance and pins

| Item | Value |
|---|---|
| Audited repo | `868DanDoo/RootlessJamesDSP` (user's fork of `timschneeb/RootlessJamesDSP`) |
| Audited commit | `60d25ae8a53c6f4691c090673df290a73c6b6357` (2026-07-25, "Merge pull request #337 … codex/pr-multirate-convolution") |
| App version | `versionName 1.6.14`, `versionCode 51` (`buildSrc/.../AndroidConfig.kt`) |
| History available | Shallow clone (depth 1). Older history, blame and `git rev-list --count` are **not** available. |
| Native submodule | `app/src/main/cpp/libjamesdsp` → `https://github.com/ThePBone/JamesDSPManager`, branch `extensions` |
| Submodule pinned commit | `1b9f89dbd4b332a07724beb41d58bc2ffcb05395` (`git submodule status` → `(heads/extensions)`) |
| Gradle wrapper | 8.10.2 |
| AGP / Kotlin / KSP | 8.7.3 / 2.1.0 / 2.1.0-1.0.29 |
| compileSdk / targetSdk / minSdk | 35 / **34** / 29 (rootless) |
| Upstream activity | Last commit is ~3.5 months before the audit date. The project is quiet, not abandoned. |

Note: `.gitmodules` also contains a stray second entry `[submodule "cpp/libjamesdsp"]` with only a `branch` line and no path/url. It is harmless to the build but should be cleaned up in the fork.

Reproducibility rule going forward: record app SHA + `1b9f89db…` + variant + toolchain for every build, and use a **full** clone for builds (the build script uses `git rev-list --count HEAD`, which returns `1` on a shallow clone and feeds `versionNameSuffix` and `BuildConfig.COMMIT_COUNT`).

---

## 2. Gradle modules and build variants

`settings.gradle` includes: `:app`, `:codeview`, `:hidden-api-stubs`, `:hidden-api-impl`, `:hidden-api-refined`. Repositories: Google, Maven Central, Gradle Plugin Portal, **JitPack**.

`app/build.gradle.kts` defines (read from the script; **not confirmed by running `./gradlew tasks`**):

- **Build types:** `debug` (adds `.debug` applicationId suffix), `release`, `preview` (copies release).
- **Flavor dimension "version":** `rootless`, `root`, `plugin`.
- **Flavor dimension "dependencies":** `fdroid` (`FOSS_ONLY=true`, builds native with `-DNO_CRASHLYTICS=1`), `full` (Firebase).
- **Application IDs:** rootless `me.timschneeberger.rootlessjamesdsp`; root `james.dsp`; plugin inherits the default.
- **ABIs:** armeabi-v7a, arm64-v8a, x86, x86_64, plus a universal APK.
- **Native:** CMake 3.22.1, `-DANDROID_ARM_NEON=ON`.

Likely rootless task names (derived from the flavor/build-type names and from the literal names used in the script's own `afterEvaluate` hooks): `assembleRootlessFdroidDebug`, `assembleRootlessFullRelease`. **Run `./gradlew tasks --all` on a machine with the SDK to confirm before relying on these.**

Build-configuration observations:

- `release` is signed with the **debug** signing config (`signingConfig = signingConfigs.getByName("debug")`). Fine for personal use; not acceptable for a public release.
- `isMinifyEnabled = false` for release.
- `google-services.json` is committed. The `full` flavor applies the Google Services and Crashlytics plugins; release sets `crashlyticsCollectionEnabled = true`. A `full` release build from this fork would therefore report crashes to the **upstream author's Firebase project**. The `fdroid` flavor excludes Firebase.
- `AndroidConfig.minSdk` is a mutable `var` assigned inside flavor blocks (order-dependent; works today, fragile).

---

## 3. Manifest, permissions and exported surface

Permissions (`app/src/main/AndroidManifest.xml`): `INTERNET`, `RECEIVE_BOOT_COMPLETED`, `POST_NOTIFICATIONS`, `WAKE_LOCK`, `FOREGROUND_SERVICE`, `FOREGROUND_SERVICE_MEDIA_PROJECTION`, `FOREGROUND_SERVICE_DATA_SYNC`, `SYSTEM_ALERT_WINDOW`, `MODIFY_AUDIO_SETTINGS`, `RECORD_AUDIO`, `DUMP`. `queries` lists Shizuku.

Services: `RootlessAudioProcessorService` (`foregroundServiceType="mediaProjection"`), `RootAudioProcessorService` (`mediaPlayback`, root-only), `NotificationListenerService`, `QuickTileService`, `BackupRestoreService` (`dataSync`).

Receivers: `SessionReceiver` (exported; `OPEN/CLOSE_AUDIO_EFFECT_CONTROL_SESSION`), `BootCompletedReceiver`, `PowerStateReceiver` (**exported**, action `…SET_POWER_STATE`, `tools:ignore="ExportedReceiver"`).

Also: `ShizukuProvider` (exported), backup rules include all shared prefs except `variable.xml`, `allowBackup="true"`.

Implications for the roadmap:

- The roadmap's "no telemetry / no cloud" rule conflicts with the `full` flavor (Firebase Analytics + Crashlytics + NDK). Use `fdroid` as the product baseline or remove Firebase.
- The exported `PowerStateReceiver` (verified in `receiver/PowerStateReceiver.kt`) calls `toggleEnginePower(intent.getBooleanExtra("rootlessjamesdsp.enabled", false))` for **any** incoming broadcast. It declares no permission and checks no sender, so any app on the device can switch the DSP on or off. This is the existing external-automation hook; roadmap §4.6 work should either protect it (signature permission or user-set token) or remove it. Note that a missing extra means `false`, i.e. "turn off".
- Root, plugin and Shizuku code paths are present in the same module. The roadmap's "no root artifact" acceptance test (Phase 5) requires removing the `root` and `plugin` flavors, `RootAudioProcessorService`, `libsu`, and the root manifest overlay — not just hiding them.

---

## 4. Source layout and UI framework

Package: `me.timschneeberger.rootlessjamesdsp`. Subpackages: `activity`, `adapter`, `api`, `backup`, `contract`, `delegates`, `editor`, `fragment`, `interop`, `liveprog`, `model`, `preference`, `receiver`, `service`, `session`, `utils`, `view`.

- **UI:** Android Views + ViewBinding, Navigation component, AndroidX Preference screens, Material 1.9. **No Compose.**
- **DI:** Koin 3.3.3.
- **Persistence/async:** SharedPreferences, kotlinx-serialization-json 1.5.0, Gson + Retrofit, Room 2.6.1 (+KSP), coroutines.
- **Source size:** ~20.9k lines of Kotlin under `app/src`.
- **Native:** `app/src/main/cpp` contains `libjamesdsp` (submodule), `libjamesdsp-wrapper` (JNI, tracked in this repo), `libjdspimptoolbox`, `libcrashlytics-connector`, plus `EELStdOutExtension`.

Relevant correction to the roadmap's constraint "do not add Room": **Room is already a dependency** (app blocklist database). The constraint should read "do not add *new* persistence libraries without justification".

---

## 5. Capture service and MediaProjection lifecycle (static review)

File: `service/RootlessAudioProcessorService.kt` (707 lines). Flow:

1. `onCreate()` builds the session manager, `JamesDspLocalEngine`, registers receivers/preference listeners, and calls `startForeground(..., FOREGROUND_SERVICE_TYPE_MEDIA_PROJECTION)` immediately.
2. `onStartCommand()` reads the consent `Intent` from `EXTRA_MEDIA_PROJECTION_DATA`, calls `MediaProjectionManager.getMediaProjection(RESULT_OK, intent)`, registers a `MediaProjection.Callback`, then `startRecording()`.
3. `startRecording()` builds an `AudioRecord` (with `AudioPlaybackCaptureConfiguration`, usages MEDIA/GAME/UNKNOWN, UID exclusions) and an `AudioTrack`, and runs a blocking read → `engine.process*` → blocking write loop on a plain `Thread`.
4. The output `AudioTrack` uses `setAllowedCapturePolicy(ALLOW_CAPTURE_BY_NONE)` so the app does not capture its own output.
5. `onDestroy()` stops the thread, closes the engine, removes the notification, unregisters the callback and nulls the projection.

### Findings

Severity is my judgement of risk to the roadmap's Phase 1 goals. All *static*.

| # | Finding | Evidence | Why it matters | Severity |
|---|---|---|---|---|
| L1 | `MediaProjection.stop()` is **never called** anywhere in the app. `onDestroy()` only unregisters the callback and sets the field to `null`. | `grep` for `.stop()` on the projection returns nothing; `onDestroy()` at lines 222–252 | The roadmap lists "MediaProjection remaining active after DSP is disabled" (upstream issue #358) as a priority. This is the most direct code-level candidate. Needs device confirmation (Android 14–17, with/without the system capture indicator). | **High** |
| L2 | The consent `Intent` is stored (`mediaProjectionStartIntent`) and re-sent in session-loss and app-problem notifications; the service also returns `START_REDELIVER_INTENT`. | lines 194, 219, 318, 363; `ServiceNotificationHelper.kt` 138–172, 243–272 | On Android 14+ a projection consent token must not be reused. Redelivery after process death and notification-driven restarts both hand the same token back to `getMediaProjection()`. Failure is caught and triggers `ACTION_DISCARD_AUTHORIZATION`, so it degrades to "ask again" rather than crashing — but the behaviour is unverified. `targetSdk` is 34, so these rules apply. | **High** |
| L3 | `MediaProjection.Callback.onStop()` returns early when `key_is_activity_active` is true ("toast too disruptive"). | lines 261–282 | If the system stops the projection while the app UI is in the foreground, the service neither stops nor releases the recorder, and UI/service state can disagree. Roadmap Phase 1 task 4 specifically requires correct `onStop` handling. | **Medium–High** |
| L4 | Cross-thread control flags are plain `var`s: `recreateRecorderRequested`, `isProcessorDisposing`, `isProcessorIdle`, `suspendOnIdle`. Only `notificationSessions` is `@Volatile`. | lines 79, 92–99 | The recorder thread polls these without synchronization; visibility is not guaranteed by the Java memory model. Likely works in practice; a correctness hazard for pause/resume and stop paths. | **Medium** |
| L5 | `stopSelf()` is called from many places, including from the recorder thread and from `onDestroy()` itself. | lines 180, 216, 280, 321, 367, 421, 454, 490, 545, 250 | Cleanup is spread across paths rather than centralized; roadmap Phase 1 task 5 asks for centralization. | Medium |
| L6 | `stopRecording()` waits `join(500)` and then drops the reference even if the thread has not exited. | lines 563–570 | Possible leak of the `AudioRecord`/`AudioTrack` or a still-running loop after rapid enable/disable. | Medium |
| L7 | The idle-suspend path stops the recorder/track and sleeps 50 ms in a loop; resume is driven by `isProcessorIdle`. | lines 499–525 | Relevant to upstream reports "audio stops when another track starts" / "after repeated pause/resume" (#356). Candidate area, not a diagnosis. | Medium |
| L8 | Session-loss retry is limited to one attempt (`SESSION_LOSS_MAX_RETRIES = 1`), then the service self-terminates with a notification. | lines 297–324, 681 | Reasonable design; documents why users see the service stop. | Info |

### What I could not verify

- Whether L1/L2/L3 reproduce on any real device or Android version.
- Upstream issue numbers and wording (#335, #356, #357, #358). The roadmap cites them; I could not read GitHub issues from this workspace.

---

## 6. DSP boundary

- JNI wrapper: `interop/JamesDspWrapper.kt` + `app/src/main/cpp/libjamesdsp-wrapper/JamesDspWrapper.cpp`.
- Engine abstraction: `JamesDspBaseEngine` (abstract) with `JamesDspLocalEngine` (rootless, in-process) and `JamesDspRemoteEngine` (root). A single `syncWithPreferences()` pushes the whole preference set to the native engine.
- Settings flow from SharedPreferences XML files named `dsp_*.xml` into the engine via `PreferenceCache`. Changes are broadcast with `ACTION_PREFERENCES_UPDATED`.

This is already close to the roadmap's "single adapter" idea: `JamesDspBaseEngine` is the adapter. New features should add settings to it rather than call JNI directly.

---

## 7. Persistence, presets and backup

- **Live settings:** `shared_prefs/dsp_*.xml` (one file per effect group, e.g. `dsp_parametriceq`, `dsp_graphiceq`, `dsp_bass`).
- **Presets:** `model/preset/Preset.kt`. A preset is a **tar** containing the `dsp_*.xml` files, plus metadata (`META_VERSION`, `META_APP_VERSION`, `META_APP_FLAVOR`, `META_MIN_VERSION_CODE`) and optionally the Liveprog script. Stored in `getExternalFilesDir(null)/Presets`. This is a versioned format with a minimum-version field — good basis for roadmap §5.3.
- **Backups:** `backup/BackupManager.kt` writes `jamesdsp_backup_*.tar.gz` (okio gzip + the same `Tar` helper), with scheduled auto-backup and a keep-N policy.
- **Device profiles:** `utils/ProfileManager.kt` stores each profile in `files/profiles/<id>/` as `profile.json` + `profile.tar` (a regular preset). See §8.
- **App blocklist:** Room database (`model/room`).

---

## 8. What already exists versus what the roadmap assumes

This is the most important section. Several roadmap phases assume features are missing that upstream already has.

| Roadmap item | Upstream state (verified in source) | Gap |
|---|---|---|
| **Device profiles** (§4.1) | **Exists.** `ProfileManager` + `RoutingObserver` + `DeviceProfilesCardFragment`; setting `key_device_profiles_enable`. Automatically stores the current settings into the outgoing device's profile and loads the incoming one. | Policies `Ask` / `Off` per device, a default profile, explicit precedence, manual temporary selection, friendly names, ambiguity handling, debounce. |
| **Device events** (§4.6) | **Exists, but via `MediaRouter` callbacks**, not `AudioManager.registerAudioDeviceCallback()` (`RoutingObserver.kt`). | Roadmap requires `AudioDeviceCallback`, debounce and a stable snapshot. |
| Device identity (§4.1) | Profile id = device group + sanitized product name; **for Bluetooth the id uses `AudioDeviceInfo.address`** (a MAC-like value). Analog/speaker/other share a single profile per group. USB uses the product name, which for generic DACs is often not useful. | Roadmap says do not depend on Bluetooth addresses. Needs a migration plan, because existing profile directories are keyed by these ids. |
| **Graphic EQ** (§4.2) | **Exists** (`GraphicEqualizerFragment`, GraphicEQ import/export, interpolation modes). | Regression tests, validation. |
| **Parametric EQ** (§4.2) | **Exists** (`ParametricEqualizerFragment`, `ParametricEqSurface`, APO-style import). Filter types: **Peaking, Low shelf, High shelf only** (`ParametricEqFilterType`). Applied through `setGraphicEqCombined(...)`, i.e. combined with the graphic EQ path, with a preamp value. | No LP/HP/notch/all-pass; no per-band bypass; response is magnitude-based rather than a biquad cascade (consistent with the reference fork's description — not independently measured). |
| Preamp/headroom (§4.2) | A PEQ preamp value exists (`key_peq_preamp`). | No automatic compensation or clip warning (not found in the code I read). |
| **Bass** (§4.3) | **Exists:** `setBassBoost(enable, maxGain)` and a `dsp_bass` settings card. | Roadmap's "turnover frequency / headroom" controls are not exposed. Whether the effect is reachable and audible in rootless mode needs a device test. |
| **AutoEQ** (§4.4) | **Exists:** `AeqSelectorActivity` + `AutoEqClient` (Retrofit) fetching from a configurable backend, default `https://aeq.timschneeberger.me/` (the author's hosted `AutoEqApi`). Returns data used by the Graphic EQ. | No local import cache, no PEQ fitting, no fit-error/preamp display, no source/licence metadata. Core feature depends on one maintainer-hosted server. |
| **Diagnostics** (§4.5) | Partial: `session/dump/DumpManager` (debug dump), session managers, `AppCompatibilityActivity`, notifications for session loss / app problem. | No explicit state machine, no dashboard status model, no redacted export. |
| Automation (§4.6) | Device profile switching only; `PowerStateReceiver` for external on/off. | Rule engine, ask/off policies. |
| Tests (§8) | 2 JVM tests (`AudioSampleRateDetectorTest`, `ConvolverSampleRateFilesTest`) + 1 template unit test + 1 template instrumented test. | Almost no regression coverage. |

### Specific issues in the existing profile code

- `ProfileManager.rotate()` stores the *current* settings into the outgoing profile **before** loading the new one. A rapid connect/disconnect or a bad route event can therefore write the wrong device's settings into a profile.
- No debounce. `RoutingObserver.onRouteChanged` re-evaluates the selected route on every change.
- `ProfileManager.finalize()` is used to unregister receivers; finalizers are not guaranteed to run.
- `RoutingObserver.onRouteRemoved` calls `router.unselect(...)`.

These are the right targets for roadmap Phase 2, rather than writing a new profile system.

---

## 9. Privacy and "rootless-only" surface

| Concern | Finding |
|---|---|
| Telemetry | Firebase Analytics + Crashlytics (+NDK) in the `full` flavor; `google-services.json` committed. `fdroid` flavor has none. |
| Network | `INTERNET` permission; Retrofit client to the AutoEQ backend; GitHub-based updater in `root`/`plugin` flavors only. |
| Debug tooling | LeakCanary and Pluto in debug builds only (no-op for preview/release). |
| Root | `root` + `plugin` flavors, `RootAudioProcessorService`, `libsu` (`rootImplementation`), `BUILD_ROOT.md`. |
| Privileged helpers | Shizuku API/provider dependency and manifest provider present. |
| Overlays | `SYSTEM_ALERT_WINDOW` declared. |
| Diagnostics | `DUMP` permission and `DebugDumpFileProvider`. |
| Hidden APIs | `hiddenapibypass`, Rikka Refine; `:hidden-api-*` modules. |

---

## 10. Corrections to the roadmap

1. **§2.1 / §5:** correct as written (modules, AGP 8.7.3, Kotlin 2.1.0, submodule URL/branch). Add the pinned submodule SHA `1b9f89db…`.
2. **§4.1/§7 Phase 2:** profiles are not greenfield; extend `ProfileManager`.
3. **§4.2/§7 Phase 3:** a PEQ editor and Graphic EQ already exist; the work is validation, more filter types, bypass, headroom and tests — not "add a PEQ".
4. **§4.3:** a bass effect already exists; reuse is the plan, with a device test first.
5. **§4.4/§7 Phase 4:** AutoEQ search/apply already exists (online only); the work is offline import/cache, PEQ fitting and metadata.
6. **§4.6:** device events use `MediaRouter` today; migrating to `AudioDeviceCallback` is a real change, not a first implementation.
7. **§5.3 / Codex brief:** "do not add Room" — Room is already present.
8. **Privacy policy:** the product baseline should be the `fdroid` flavor, or Firebase must be removed.
9. **Phase 5 "no root artifact":** requires deleting `root`/`plugin` flavors and root code, not hiding settings.
10. **Phase 1 must start with L1–L3** above; they match the roadmap's own priority list.

---

## 11. Items that could not be verified in this audit

- Any build output, APK metadata or test result (build blocked; see `BASELINE_BUILD.md`).
- Behaviour on a real device or on Android 14–17.
- Upstream issue tracker contents.
- Contents of the native submodule beyond confirming it checks out at `1b9f89db…` (I did not read `libjamesdsp` DSP code).
- That the `bass` effect is audible in the rootless capture path.
- Task names from `./gradlew tasks`.
