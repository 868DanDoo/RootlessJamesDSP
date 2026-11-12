# Phase 0 Audit — code map for safe changes

Read-only audit at commit `47905ea` (code identical to upstream `60d25ae`). Nothing was built or run, so every finding is from reading code. "Unverified" marks anything the code alone cannot prove.

Path shorthand: `K/` = `app/src/main/java/me/timschneeberger/rootlessjamesdsp/`, `N/` = `app/src/main/cpp/`, `SVC` = `K/service/RootlessAudioProcessorService.kt`, `JNI` = `N/libjamesdsp-wrapper/JamesDspWrapper.cpp`.

## 1. Audio path

| Step | What happens | Where |
|---|---|---|
| Consent | Activity asks for screen-capture consent and passes the result Intent to the service | `K/activity/MainActivity.kt:311-317`, `K/activity/EngineLauncherActivity.kt:47-61` |
| Service start | Foreground service of type mediaProjection; gets the projection from the Intent | `SVC:159-163`, `SVC:196-200` |
| Capture | `AudioRecord` with playback-capture config, usages MEDIA/GAME/UNKNOWN, excluded UIDs, stereo | `SVC:636-663`, `SVC:633` |
| Mute original | Each captured app session gets a muting effect (DynamicsProcessing at -200 dB, or a hidden Volume effect) so audio isn't heard twice | `K/utils/MutedAudioEffectFactory.kt:41-42`, `:51-59`; `K/session/rootless/RootlessSessionDatabase.kt:40-48` |
| DSP | Blocking read → `engine.processFloat/Int16` → JNI → `processFloatMultiplexd` | `SVC:528-537`, `K/interop/JamesDspLocalEngine.kt:73-87`, `JNI:304-323` |
| Output | Blocking write to an `AudioTrack` that forbids being captured | `SVC:536`, `SVC:592` |

**Threads**

- Audio thread: one plain `Thread`, no audio priority set (`SVC:464`; no `setThreadPriority` anywhere in `K/service` or `K/interop`).
- Main thread: service lifecycle, broadcasts, projection callback (`SVC:209`), session callbacks (`K/session/shared/BaseSessionManager.kt:61`, `:85`).
- IO coroutines: all effect settings are pushed to native from `Dispatchers.IO` (`K/interop/JamesDspBaseEngine.kt:33`, `:43-47`).
- UI → service: local broadcast `ACTION_PREFERENCES_UPDATED` (`K/fragment/PreferenceGroupFragment.kt:54` → `SVC:289`).

**Latency sources**

- Block size: default 8192 samples (`app/src/main/res/values/defaults.xml:35`), interleaved stereo, so 4096 frames (`JNI:319` halves the length). That is about 85 ms at 48 kHz per blocking read (`SVC:534`). Arithmetic; unverified on a device.
- Record and track buffers are each one more block (`SVC:661`, `SVC:619`).
- JNI array access may copy each buffer both ways (`JNI:316-322`).
- Sample rate is clamped to 44.1–48 kHz (`SVC:438`); other device rates get resampled by Android (unverified).
- Idle suspend sleeps in 50 ms steps (`SVC:500-515`).

## 2. Native boundary

- Kotlin side: `K/interop/JamesDspWrapper.kt:10-48` (all `external` functions), library loaded at `:62`.
- C++ side: `JNI:84-757`. Second native library for impulse responses: `K/interop/JdspImpResToolbox.kt` (called at `K/interop/JamesDspBaseEngine.kt:248`).
- Only caller for rootless: `K/interop/JamesDspLocalEngine.kt`.

**Shared state**

| State | Detail | Where |
|---|---|---|
| Handle | Raw C++ pointer held as a Kotlin `Long`, plain `var`, read by the audio thread | `K/interop/JamesDspLocalEngine.kt:13`, `:43`; `JNI:27-33` |
| Engine struct | One `JamesDSPLib`, written by IO-thread setters and read by the audio thread. Locking is inside the vendored library (unverified); the wrapper only checks a mutex exists | `JNI:118-141`, `JNI:131` |
| Direct struct writes | Wrapper writes engine fields itself, outside library calls | `JNI:533-535`, `JNI:707`, `JNI:725` |
| Callbacks | Global ref to a Kotlin callback object plus a saved `JNIEnv*` | `JNI:88-89`, `N/libjamesdsp-wrapper/JamesDspWrapper.h:9-14` |
| Liveprog output | A process-wide static function pointer | `N/EELStdOutExtension.c:4-5`, `JNI:605` |
| Settings | SharedPreferences `dsp_*` files, diffed by a cache and applied per group | `K/interop/PreferenceCache.kt:26-42`, `K/interop/JamesDspBaseEngine.kt:125-154` |

PEQ note: the parametric EQ is turned into a 512-point curve in Kotlin and sent through the graphic-EQ call (`K/interop/JamesDspBaseEngine.kt:292-327`). There is no separate native PEQ.

## 3. Safe vs fragile

**Safe (look and wording)**

- Themes and colors: `app/src/main/res/values/themes.xml:3-237`, `values/colors*.xml`, `values-night/`; theme picker `K/delegates/ThemingDelegate.kt:40-43`.
- Layouts, drawables, menus: `app/src/main/res/layout/` (53 files), `drawable/`, `menu/`.
- Strings: `app/src/main/res/values/strings.xml`. Leave the `values-xx` translations alone (Crowdin, `crowdin.yml`).
- Custom views and adapters: `K/view/`, `K/adapter/`.
- App settings screens: `K/fragment/settings/`, `res/xml/app_*_preferences.xml`.

**Change with care (UI that feeds the engine)**

- `res/xml/dsp_*_preferences.xml` and `res/values/keys.xml`: keys and value formats are read by `K/interop/JamesDspBaseEngine.kt:63-123` and stored in presets (`K/model/preset/Preset.kt:65-78`). Renaming a key or changing a type breaks saved presets and profiles.
- `K/fragment/PreferenceGroupFragment.kt:54`: this broadcast is what applies settings.
- `K/activity/MainActivity.kt:311-317`, `:617-625`: consent flow.
- `K/utils/ProfileManager.kt:84-95`, `K/utils/RoutingObserver.kt:126-136`: device profile switching.

**Fragile (avoid until tests exist)**

- `SVC` (whole file), `K/service/BaseAudioProcessorService.kt`.
- `K/interop/*`, everything under `N/`.
- `K/session/**` (session detection and muting), `K/utils/MutedAudioEffectFactory.kt`.
- `app/src/main/AndroidManifest.xml:5-18`, `:218-296` (permissions, services, receivers).
- `K/utils/notifications/ServiceNotificationHelper.kt:186-272` (intents that restart the service).
- `hidden-api-*` modules, `app/build.gradle.kts`, `N/CMakeLists.txt`.

## 4. Risks

**Crashes and lifetime**

- Engine freed on a 100 ms timer after `close()` (`K/interop/JamesDspLocalEngine.kt:29-38`). The audio thread is only waited on for 500 ms (`SVC:567`) and IO setters are never cancelled because `close()` does not call the base class (`K/interop/JamesDspBaseEngine.kt:37-41`). A slow call can use freed memory.
- `alloc` returns `1` or `2` on failure (`JNI:125`, `JNI:138`), but Kotlin only treats `0` as invalid (`K/interop/JamesDspLocalEngine.kt:43`). `memset` runs before the null check (`JNI:118-121`).
- Saved `JNIEnv*` from the creating thread is reused in the Liveprog output callback (`JNI:89`, `JNI:747`). A `JNIEnv` is only valid on its own thread. Which thread fires it is inside the vendored library (unverified).
- Dangling pointer: `std::to_string(...).c_str()` used after the temporary dies (`JNI:671-673`).
- Array length read before the null check (`JNI:348` vs `:355`; `JNI:411` vs `:418`).
- Settings coroutines have no exception handler (`K/interop/JamesDspBaseEngine.kt:33`). A malformed stored value throws at `.toInt()` (`:72`, `:81`) or overruns the fixed arrays (`:168-177`, `:185-194`) and would crash the app (unverified at runtime).

**Races**

- Audio-thread flags are plain `var`s (`SVC:79`, `SVC:92-99`), read in the loop (`SVC:477-500`).
- `PreferenceCache` uses unsynchronized collections (`K/interop/PreferenceCache.kt:10-11`); `clearCache()` bypasses the mutex (`K/interop/JamesDspBaseEngine.kt:49-51`).
- Many `stopSelf()` paths, one from the audio thread (`SVC:490`, `SVC:545`).

**Android version**

- Projection is never stopped, only dropped (`SVC:240-241`).
- `onStop` is ignored while the app is open (`SVC:268-271`).
- Token reuse: MainActivity blocks reuse only on API 35+ (`K/activity/MainActivity.kt:619`), the launcher on API 34+ (`K/activity/EngineLauncherActivity.kt:38`). The service also asks for intent redelivery (`SVC:219`) and puts the token in a retry button (`K/utils/notifications/ServiceNotificationHelper.kt:218-226`).
- Known unfixed Android 15 issue noted by the author (`K/receiver/BootCompletedReceiver.kt:50-60`, root path).
- `targetSdk` is 34 (`buildSrc/src/main/kotlin/AndroidConfig.kt:5`).

**Device dependent**

- Session detection parses system dump text with per-version regexes (`K/session/dump/provider/AudioServiceDumpProvider.kt:24-28`, `K/session/dump/provider/AudioPolicyServiceDumpProvider.kt:25-29`, `K/session/dump/utils/AudioFlingerServiceDumpUtils.kt:26-30`) and needs the `DUMP` permission (`K/session/dump/utils/DumpUtils.kt:18`).
- Muting uses hidden APIs and can be taken over by other effect apps, which stops the service (`K/utils/MutedAudioEffectFactory.kt:100-104`, `SVC:299-322`).
- Hard-coded list of problem apps (`K/session/rootless/RootlessSessionDatabase.kt:18-28`).
- Hidden-API bypass at startup (`K/MainApplication.kt:58-61`).

**Security and privacy**

- Exported receiver toggles the DSP for any sender (`app/src/main/AndroidManifest.xml:287-290`, `K/receiver/PowerStateReceiver.kt:16`).
- `full` flavor sends every DSP setting to Crashlytics as custom keys (`K/interop/PreferenceCache.kt:39`).

## 5. Build

- Gradle 8.10.2 (`gradle/wrapper/gradle-wrapper.properties:3`), AGP 8.7.3 and Kotlin 2.1.0 (`build.gradle.kts:10-13`), Java 17 (`app/build.gradle.kts:141-146`), CMake 3.22.1 (`:160`), CI uses JDK 17 (`.github/workflows/build.yml:47`).
- compileSdk 35, targetSdk 34, minSdk 29 for rootless (`buildSrc/src/main/kotlin/AndroidConfig.kt:4-5`, `app/build.gradle.kts:93`).
- Variants: `rootless|root|plugin` × `fdroid|full` × `debug|release|preview` (`app/build.gradle.kts:46-113`). Task names unverified (build not run).
- Native submodule required (`.gitmodules:1-4`); a stray second entry exists (`.gitmodules:5-6`).

**Toolchain problems**

- No `ndkVersion` pinned (only an ABI block at `app/build.gradle.kts:42`), so the NDK depends on the machine.
- Native flags `-Ofast -DDEBUG` for all builds (`N/CMakeLists.txt:8-9`), plus suppressed C warnings (`app/build.gradle.kts:38`).
- Release is debug-signed and unminified (`app/build.gradle.kts:61-63`).
- Firebase plugins applied to every variant (`app/build.gradle.kts:7-8`); `app/google-services.json` is committed.
- JitPack dependencies (`settings.gradle:13`, `app/build.gradle.kts:219`, `:239`).
- Version name uses `git rev-list --count`, wrong on shallow clones (`buildSrc/src/main/kotlin/Commands.kt:10`); `project.exec` (`:27`) and `rootProject.buildDir` (`build.gradle.kts:24`) are deprecated Gradle APIs (removal version unverified).
- `kotlin-kapt` applied but unused (`app/build.gradle.kts:6`).

**Older dependencies** (`app/build.gradle.kts`): Material 1.9.0 (`:198`), Koin 3.3.3 (`:201`), kotlinx-serialization 1.5.0 (`:182`), coroutines-android 1.7.1 beside core 1.9.0 (`:180-181`), kotlin-reflect 2.0.20 with Kotlin 2.1.0 (`:179`), LeakCanary 2.10 (`:251`), retrofit converter-scalars 2.9.0 beside 2.11.0 (`:213-215`). `codeview` module compiles against SDK 33 (`codeview/build.gradle:5`). "Outdated" is relative to my knowledge, not checked online.

## Top 10 risks (ranked)

1. Use-after-free on shutdown: engine freed by timer while audio/IO threads may still be in native code (`K/interop/JamesDspLocalEngine.kt:29-38`).
2. Invalid native handle: `alloc` failure returns `1`/`2`, treated as valid (`JNI:125`, `JNI:138`).
3. `JNIEnv*` reused across threads in the Liveprog callback (`JNI:89`, `JNI:747`).
4. MediaProjection never stopped, and `onStop` ignored while the UI is open (`SVC:240-241`, `SVC:268-271`).
5. Consent token reuse on Android 14 and via redelivery/retry (`K/activity/MainActivity.kt:619`, `SVC:219`).
6. Unsynchronized flags and scattered shutdown paths in the audio loop (`SVC:79`, `SVC:92-99`, `SVC:563-570`).
7. Session detection depends on dump-text regexes and the `DUMP` permission; new Android or OEM builds can break it (`K/session/dump/provider/AudioServiceDumpProvider.kt:24-28`).
8. Uncaught exceptions in the settings coroutine from malformed stored values (`K/interop/JamesDspBaseEngine.kt:33`, `:168-177`).
9. Exported `PowerStateReceiver` with no permission (`app/src/main/AndroidManifest.xml:287-290`).
10. Unreproducible build: unpinned NDK, `-Ofast -DDEBUG`, debug-signed release, Firebase on by default in `full` (`app/build.gradle.kts:42`, `:63`; `N/CMakeLists.txt:8-9`).
