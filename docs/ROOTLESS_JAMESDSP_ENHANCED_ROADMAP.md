# JamesDSP Enhanced — Product Vision & Codex Implementation Roadmap

**Prepared:** 8 November 2026  
**Upstream:** https://github.com/timschneeb/RootlessJamesDSP  
**Product direction:** Enhanced  
**Distribution intent:** Personal use first; public release possible later  
**Root policy:** Rootless only. No Magisk module, no root requirement, no root-only feature path in the shipped product.

---

## 1. Executive summary

Build **JamesDSP Enhanced**, a polished and reliable rootless Android audio enhancement fork based on RootlessJamesDSP.

The product should make it easier to:

1. Create, select and automatically apply audio profiles for headphones, USB DACs, Bluetooth devices, wired headsets and the phone speaker.
2. Shape sound with a useful Graphic EQ and a precise Parametric EQ.
3. Diagnose whether capture, DSP initialization and audio routing are actually working.
4. Automate profile selection when audio devices connect or disconnect.
5. Find headphone tuning data and convert it into a safe, editable EQ preset (AutoEQ).
6. Improve bass without uncontrolled clipping, distortion or unnecessary DSP complexity.

**Core product rule:** preserve the existing rootless capture/service path and `libjamesdsp` foundation wherever possible. Put new behaviour in well-defined application-level modules and communicate with the DSP through a small adapter. Modify native DSP only when the required effect cannot be achieved safely with existing engine functionality.

This is an enhancement fork, not a ground-up rewrite.

## 2. Research findings and baseline realities

### 2.1 Upstream structure

The upstream repository currently shows these Gradle modules in `settings.gradle`:

- `:app`
- `:codeview`
- `:hidden-api-stubs`
- `:hidden-api-impl`
- `:hidden-api-refined`

The root Gradle build declares Android Gradle Plugin 8.7.3 and Kotlin 2.1.0. The `.gitmodules` file places the native `libjamesdsp` dependency at `app/src/main/cpp/libjamesdsp`, sourced from `ThePBone/JamesDSPManager` on its `extensions` branch. Pin and record the actual submodule commit used by every build; do not assume that a branch name is a reproducible version.

The repository README says the rootless implementation uses Android internal audio capture and `libjamesdsp`. It also documents the fundamental limitations: source apps may block capture, some Android audio-effect applications conflict, and the capture loop adds latency.

### 2.2 Android platform constraints

Rootless capture is not equivalent to unrestricted system-level audio interception.

Android playback capture is subject to source-app policy and audio-usage restrictions. Android's documentation says eligible captured playback uses `USAGE_UNKNOWN`, `USAGE_GAME` or `USAGE_MEDIA`, and the source app's capture policy and manifest must permit capture. The capture app and playback app must also be in the same Android user profile. A source app that prohibits capture cannot be made universally processable by adding an EQ screen or changing the DSP engine.

MediaProjection lifecycle handling is critical:

- Ask the user for consent for each capture session as required by the Android version.
- For apps targeting Android 14+, declare and use the `mediaProjection` foreground-service type and required permission.
- Register `MediaProjection.Callback.onStop()` and release associated capture resources.
- On Android 14+, do not reuse a MediaProjection consent token or call `createVirtualDisplay()` more than once on the same projection instance.
- When the user disables DSP, stop the projection and all dependent resources; do not leave the system capture indicator active.

Audio device connection events can be observed through `AudioManager.registerAudioDeviceCallback()`. Use that supported API for profile triggers. Do not depend on privileged/root APIs or assume a Bluetooth MAC address is a stable, accessible identifier.

### 2.3 Current upstream issues that influence the plan

The upstream issue tracker in late October 2026 includes reports about:

- MediaProjection remaining active after JamesDSP is disabled on Android 17 / OxygenOS 17.
- Work-profile applications not appearing in the exclusion picker.
- Audio stopping when another track starts.
- Audio stopping after repeated pause/resume operations.
- A request for manual profile/preset selection because USB DACs and headphones may not report a useful model identity.

Treat lifecycle stability and manual profile selection as product requirements, not nice-to-have polish.

### 2.4 A related enhanced fork already exists

A public fork, `evoeram/Loundess-PEQ-RootlessJamesDSP`, is a useful reference implementation. Its README describes device-linked presets, a PEQ biquad cascade, AutoEQ fitting, Squig Live measurement search/cache, GraphicEQ conversion, loudness correction and other features. Its release page currently shows v1.6.29 (3 November 2026).

However, that fork is substantially larger than upstream. Its README reports more than 150 additional commits, 370+ changed files and roughly 73,000 lines of changes. It also contains optional root builds and several features outside this project's requested scope. It documents that rootless Standard and Low-Latency modes retain more effects but add capture-loop latency, while its Direct Mode is EQ-only and does not provide bass boost or several other DSP effects.

**Recommendation:** use upstream as the clean product baseline, and use the enhanced fork as a source of ideas and selectively ported, reviewed components. Do not wholesale merge its entire change set. Before porting any feature, audit its source, rootless build path, native dependencies, tests, licence notices and actual behaviour on a non-rooted device.

---

## 3. Product definition

### 3.1 Product promise

> Better sound, smarter profiles and reliable processing — without root.

### 3.2 Target users

- Personal user who wants predictable audio tuning across different headphones and DACs.
- Enthusiast who wants control over EQ and bass without manually rebuilding settings each time.
- Public users who need clear setup, compatibility diagnostics, safe defaults and transferable presets.

### 3.3 Non-goals

Do not build these into the first releases:

- A new Android audio routing or capture framework.
- Guaranteed processing of every Android app.
- Root, Magisk, KernelSU or AudioFlinger injection support.
- A replacement media player.
- Full acoustic room-correction measurement using an arbitrary phone microphone.
- Neural voice isolation, NOS/R2R simulation, tape emulation, volume-key interception or other unrelated effect suites.
- A cloud account, user tracking, telemetry upload or always-online dependency.
- Automatic patching of third-party applications to defeat their audio-capture restrictions.

---

## 4. Feature specification

### 4.1 Device profiles

**Purpose:** Give each listening route its own saved DSP configuration, with manual selection always available.

MVP behaviour:

- Create, rename, duplicate, delete and select a profile.
- Save the selected EQ, bass settings and supported JamesDSP effect settings in the profile.
- Assign a profile to a connected device, or choose `Ask`, `Automatic` or `Off` behaviour.
- Show the active device and active profile on the dashboard.
- Provide a default profile for unmatched devices.
- Provide import/export and preserve compatibility with existing preset/backup formats where practical.

Matching strategy:

- Use device type and publicly available descriptive fields (for example product/port name) as match hints.
- Let the user give a connected device a friendly name and explicitly bind a profile to it.
- Do not treat transient audio device IDs as durable identity.
- Do not require Bluetooth MAC addresses or privileged permissions.
- If device identity is ambiguous (common with generic USB DACs), ask the user instead of silently selecting an uncertain profile.

Precedence:

1. Explicit temporary manual selection.
2. Exact user-confirmed device-to-profile binding.
3. User-defined device-type fallback.
4. Default profile.

A profile switch must be idempotent: repeated connect events must not repeatedly reload or reset the DSP.

### 4.2 Equalizer

**Purpose:** Make EQ useful for beginners without taking away precise control from enthusiasts.

MVP:

- Retain and test existing Graphic EQ functionality.
- Add or selectively port a Parametric EQ implementation if the baseline engine does not already provide an acceptable PEQ path.
- Support only filter types confirmed by the actual native implementation.
- Provide per-band enable/bypass, frequency, gain and Q controls where supported.
- Provide preamp/headroom control and a clear warning when requested boosts could clip.
- Provide a frequency-response preview calculated from the same filter model used by the DSP.
- Retain GraphicEQ import/export and existing preset compatibility.

Engineering safeguards:

- Validate all numeric input before applying it to native DSP.
- Reject NaN, infinity, invalid frequency ranges, invalid Q and unstable coefficients.
- Clamp frequency relative to the current sample rate/Nyquist limit.
- Keep DSP updates thread-safe; do not mutate data structures while the real-time audio thread is reading them.
- Avoid allocations, file I/O, network access and UI work on the audio callback thread.
- Smooth parameter changes or crossfade filter states to avoid clicks when changing profiles or EQ.
- Calculate conservative preamp compensation for positive EQ peaks; do not promise a hard no-clipping guarantee unless the pipeline includes a verified limiter.

### 4.3 Bass enhancement

**Purpose:** Add bass in a controlled way while keeping voices, kick drums and the rest of the spectrum clean.

MVP:

- First reuse the existing JamesDSP bass enhancement effect if the rootless Standard processing path exposes it reliably.
- If no suitable effect is present, implement a simple low-shelf/low-frequency EQ using the same validated filter layer as PEQ.
- Expose a small number of understandable controls: amount, turnover frequency, enable/bypass and output headroom.
- Keep bass enhancement off by default in a new profile.
- Store bass settings in the profile and include them in A/B comparisons.

Later, after objective tests, consider dynamic bass or harmonic/sub-bass synthesis. These are not V1 requirements because they increase DSP complexity, CPU use and distortion risk.

### 4.4 AutoEQ

**Purpose:** Turn a headphone measurement/tuning profile into an editable PEQ preset.

MVP path:

1. Let the user search a supported headphone tuning catalogue or import a compatible local GraphicEQ/AutoEQ text file.
2. Show the exact headphone model/variant and measurement source before applying.
3. Let the user choose a target curve where the data source supports it.
4. Convert the target frequency response into PEQ bands using a tested fitting algorithm.
5. Calculate and show the resulting preamp/headroom requirement.
6. Preview the fitted curve and show the number of filters and fit-error metrics.
7. Apply as a new profile or merge into an existing profile only after explicit user action.
8. Keep the result editable and undoable.

Algorithm requirements:

- Use logarithmic-frequency interpolation.
- Fit a bounded number of filters (start with 8–10; allow a configurable higher limit after performance testing).
- Use a tested biquad model (for example RBJ cookbook filters) that matches the runtime processor.
- Penalize excessive Q and avoid attempting to correct narrow, deep cancellation nulls.
- Enforce bounds on gain, Q and center frequency.
- Compute preamp from the peak of the final cascade response.
- Report fit error (RMS and maximum dB deviation) against the chosen target.
- Test with synthetic curves and known reference data before exposing an `Apply` button.

Data-source policy:

- Prefer an offline-capable catalogue/cache and documented data sources.
- Record source, model variant, target curve, measurement rig/date and licence/attribution metadata when supplied.
- Do not scrape an undocumented website endpoint as a core dependency.
- Make online lookup optional; already imported profiles must continue to work offline.
- Do not describe database-based AutoEQ as measurement of the user's individual physical headphone. In-situ acoustic measurement is a separate advanced feature and is out of MVP scope.

### 4.5 Diagnostics

**Purpose:** Replace guesswork with an honest, useful explanation of audio-processing state.

Dashboard indicators:

- DSP enabled/disabled.
- Permission/MediaProjection state.
- Capture service state.
- DSP engine initialized/not initialized.
- Current audio route/device type.
- Input sample rate and channel count when observable.
- Last successful processing/route event.
- Clipping/overload indicator if it can be measured at a valid point in the pipeline.
- Capture interruption or service failure state.

Suggested state model:

`DISABLED -> WAITING_FOR_CONSENT -> STARTING -> CAPTURE_ACTIVE -> DSP_READY -> PROCESSING`

Error/edge states:

`NO_SIGNAL`, `CAPTURE_STOPPED`, `DSP_INIT_FAILED`, `UNSUPPORTED_ROUTE`, `RECOVERY_REQUIRED`.

Important truthfulness rule: because playback capture is mixed and source apps can disallow it, the application may not always be able to identify which third-party app is responsible for missing audio. Use labels such as `Capture active; no audio signal observed` or `Capture may be restricted by the source app`. Do not claim a source app is blocking capture unless the platform exposes evidence.

Privacy:

- Never record or save user audio for diagnostics.
- Use a bounded in-memory event ring buffer.
- Do not collect track names, message content or unrelated app activity.
- Diagnostic export is user initiated and redacts device identifiers and package names by default.
- Explain what is included before export.

### 4.6 Automation

**Purpose:** Make profile switching reliable and predictable.

MVP triggers:

- Supported audio device connected.
- Supported audio device disconnected.

MVP actions:

- Apply a selected profile.
- Ask the user which profile to use.
- Restore the default profile.
- Optionally enable/disable DSP only where the existing service lifecycle supports this safely and the user has explicitly configured it.

Rules:

- Use `AudioManager.registerAudioDeviceCallback()` and unregister it correctly.
- Debounce transient route changes and process a stable device snapshot.
- Define deterministic rule precedence and prevent recursive/repeated triggers.
- Do not attempt to bypass MediaProjection user-consent or foreground-service restrictions.
- Do not add AccessibilityService or `WRITE_SECURE_SETTINGS` dependencies for basic profile automation.
- Defer Tasker/MacroDroid external actions to a later phase. If added, use a documented, narrowly scoped integration contract and validate all incoming actions.

---

## 5. Recommended architecture

Use package-level separation inside the existing application module initially. Do not create several new Gradle modules until the current source layout, dependencies and testability have been audited.

### 5.1 Logical layers

```text
Existing Android UI (Activities / Fragments / Views)
                    |
                    v
             Use cases / state
                    |
       +------------+-------------+
       |            |             |
       v            v             v
   Profiles       EQ/AutoEQ    Diagnostics
       |            |             |
       +------------+-------------+
                    |
                    v
          Audio / DSP adapters
                    |
          Existing rootless service
                    |
                    v
             libjamesdsp
```

Automation is an event source and coordinator that calls the profile use cases. It must not call native DSP functions directly.

### 5.2 Suggested package boundaries

These are logical suggestions, not claims about the current repository's package names. Codex must inspect the real source tree and adapt to its established conventions.

```text
app/src/main/java/<existing-package>/
  enhanced/
    profile/
      model/           Profile, DeviceMatcher, ProfileBinding
      repository/      ProfileRepository, ProfileImportExport
      usecase/         SaveProfile, ApplyProfile, ResolveProfile
      runtime/         ProfileCoordinator
    equalizer/
      model/           EqBand, EqPreset, EqValidationResult
      repository/      EqPresetRepository
      usecase/         ValidateEq, ApplyEq, ImportGraphicEq
      adapter/         ExistingDspEqAdapter
    bass/
      model/           BassConfig
      usecase/         ApplyBassConfig
      adapter/         ExistingDspBassAdapter
    autoeq/
      catalog/         HeadphoneCatalog, MeasurementSource
      fitting/         AutoEqFitter, FitResult, TargetCurve
      import/          AutoEqImporter
      cache/           MeasurementCache
    diagnostics/
      model/           DiagnosticEvent, DiagnosticSnapshot
      collector/       CaptureCollector, DspCollector, RouteCollector
      export/          RedactedDiagnosticReport
    automation/
      model/           AutomationRule, Trigger, Action
      event/           AudioDeviceEventSource
      runtime/         AutomationCoordinator
    audio/
      state/           AudioSessionState
      adapter/         ExistingCaptureServiceAdapter
```

### 5.3 Persistence

First inspect the existing preference, preset-file and backup implementations. Reuse them if they can support safe, versioned storage. Avoid adding Room or a new serialization dependency without a clear need.

Any new profile document should include a schema version, stable application-generated profile ID, human-readable name, supported effect settings, optional device matcher, creation/update metadata and optional source/licence metadata for AutoEQ-derived content. Validate the document before applying it. Migrate older versions explicitly; never silently discard unknown or unsupported effect settings.

A conceptual profile document:

```json
{
  "schemaVersion": 1,
  "id": "generated-uuid",
  "name": "USB DAC - Neutral",
  "deviceMatcher": {
    "deviceType": "USB_DEVICE",
    "displayNameHint": "USB Audio"
  },
  "equalizer": {
    "preampDb": -3.0,
    "bands": []
  },
  "bass": {
    "enabled": false,
    "amountDb": 0.0,
    "frequencyHz": 90
  }
}
```

This is an illustrative schema only. Align field names and data formats with the existing app and native engine.

### 5.4 DSP boundary

Create or identify one application-side adapter that translates validated profile state into existing engine settings. The rest of the application should depend on a stable interface, not on `libjamesdsp` implementation details.

Requirements:

- Keep real-time audio processing code isolated from Android UI and storage.
- Apply updates as immutable/snapshot state where possible.
- Serialize control-thread updates.
- Ensure safe state replacement and filter coefficient validation.
- Smooth audible transitions when settings change.
- Maintain a simple bypass path for every new effect.

Do not change the `libjamesdsp` submodule in the first phase. If a required feature needs native changes, isolate that work in a dedicated commit and document why the app-side adapter cannot solve it.

---

## 6. Priority and feasibility matrix

| Feature | Feasibility | Effort | Priority | Main risk |
|---|---|---:|---|---|
| Manual profiles and import/export | High | Medium | P0 | Existing preset format / migration |
| Device-linked profiles | High | Medium | P0 | Ambiguous USB/Bluetooth identity |
| Diagnostics and lifecycle health | High | Medium–High | P0 | OEM/Android capture lifecycle differences |
| Basic bass enhancement | High if engine effect exists | Low–Medium | P1 | Headroom and clipping |
| Parametric EQ | Medium–High | Medium–High | P1 | Native engine integration / filter stability |
| Device-triggered automation | Medium–High | Medium | P1 | Background lifecycle and duplicate events |
| Database-backed AutoEQ | Medium–High | High | P1/P2 | Dataset licensing/API reliability and fit quality |
| In-situ microphone measurement | Low–Medium | Very High | Later | Calibration, latency, device mic response |
| Dynamic bass / harmonic synthesis | Medium | High | Later | Distortion, CPU and tuning complexity |

**Priority rationale:** establish reliability and predictable profiles first; then add sound-shaping controls; then add fitting/automation convenience. A sophisticated EQ is not useful if the capture service randomly stops or a device switch applies the wrong profile.

---

## 7. Roadmap and deliverables

### Phase 0 — Repository audit and reproducible baseline

**Goal:** know exactly what we are modifying before changing code.

Tasks:

1. Clone upstream with submodules and record upstream commit SHA and `git submodule status`.
2. Inspect `settings.gradle`, root/app Gradle files, manifest, services, media projection controller, DSP wrapper, settings/preset persistence, backup/import/export and tests.
3. Run `./gradlew tasks`; identify the actual rootless debug/release build variants. Do not guess task names.
4. Build the unmodified project and save the build output and APK metadata.
5. Review open issues affecting capture, session cleanup, work profiles, app exclusions and audio route changes.
6. Compare the enhanced fork at a pinned tag/commit. Generate a focused diff report for: device presets, PEQ, AutoEQ, bass effects, diagnostics/latency instrumentation and automation.
7. Test the enhanced fork's `rootlessFullRelease` artifact separately if possible. Verify that selected features operate in rootless Standard mode, not only root builds or EQ-only Direct Mode.
8. Review the licences and notices of any code or measurement data proposed for reuse.

Deliverables:

- `docs/UPSTREAM_AUDIT.md`
- `docs/ENHANCED_FORK_COMPARISON.md`
- `docs/BASELINE_BUILD.md`
- `docs/FEATURE_PORTING_DECISIONS.md`

Exit criteria:

- Clean baseline build succeeds or a precise pre-existing build blocker is documented.
- Exact source and submodule commits are pinned.
- Every planned feature has a decision: reuse existing implementation, selectively port, or implement new.
- No feature is merged solely because it exists in the other fork.

### Phase 1 — Reliability and diagnostics foundation

**Goal:** make the rootless processing lifecycle observable and recoverable.

Tasks:

1. Map the existing MediaProjection and foreground-service lifecycle.
2. Model explicit session states and errors.
3. Verify consent-token handling against the Android target versions.
4. Register and handle `MediaProjection.Callback.onStop()`.
5. Centralize cleanup of projection, capture surface, AudioRecord/virtual display, queues, native engine and foreground service resources.
6. Ensure disabling DSP stops capture promptly and does not leave an active system capture indicator.
7. Add a bounded in-memory diagnostic event buffer and status snapshot.
8. Add a user-initiated, redacted diagnostic export with no audio samples.
9. Add regression tests around start, stop, repeated enable/disable, pause/resume, route changes and failed initialization.

Deliverables:

- Reliable capture state machine.
- Dashboard status source.
- Diagnostics screen (basic).
- `docs/DIAGNOSTICS.md`.

Acceptance criteria:

- Toggling DSP off releases all owned resources.
- A MediaProjection stop callback transitions UI/service state correctly.
- Repeated start/stop does not leak services, threads or native handles.
- Diagnostics never stores audio content.

### Phase 2 — Profiles and device matching

**Goal:** make settings portable and manually controllable.

Tasks:

1. Implement profile model and versioned persistence through the existing storage convention.
2. Add create, duplicate, rename, delete, export and import.
3. Add explicit manual profile selection in the main UI and quick controls where consistent with the existing app.
4. Implement device matcher using supported `AudioDeviceInfo` fields.
5. Add user-confirmed device-to-profile binding.
6. Add `Off`, `Ask` and `Automatic` policies.
7. Add default profile and deterministic matching precedence.
8. Preserve existing preset formats and backup/restore behaviour.
9. Test ambiguous and generic USB DACs; never silently pick a wrong device profile.

Deliverables:

- Profiles screen.
- Device profile bindings.
- Versioned profile import/export.
- `docs/PROFILE_FORMAT.md`.

Acceptance criteria:

- A user can manually apply a profile even when a device reports no model name.
- An exact user-confirmed binding takes precedence over a generic device-type rule.
- Disconnect/reconnect and duplicate system events do not cause repeated or unstable switching.
- Existing presets remain importable or a documented one-time migration is provided.

### Phase 3 — EQ and bass controls

**Goal:** make sound shaping consistent, safe and profile-aware.

Tasks:

1. Inventory existing Graphic EQ and bass effects before adding anything.
2. Keep current Graphic EQ behaviour and preset compatibility.
3. Add/port PEQ domain model, coefficient validation and response calculator.
4. Add PEQ editing UI that follows the existing UI framework and design patterns.
5. Add preamp/headroom handling.
6. Add a simple bass screen using an existing bass effect or a low-shelf fallback.
7. Make EQ and bass part of profile save/apply/import/export.
8. Add A/B and bypass controls.
9. Add unit tests for filter response, coefficient stability, boundaries and serialization.
10. Add native/DSP integration tests and device listening checks.

Deliverables:

- Graphic EQ regression coverage.
- Parametric EQ editor and validated DSP adapter.
- Bass controls.
- `docs/EQ_AND_BASS.md`.

Acceptance criteria:

- Invalid values cannot reach native DSP.
- EQ and bass changes apply without audible clicks or service restart where the engine supports live updates.
- Bypass restores the expected unprocessed path.
- Positive boosts trigger headroom compensation or a clear clipping warning.
- No measurable regression in playback stability versus the Phase 1 baseline.

### Phase 4 — AutoEQ

**Goal:** create useful headphone EQ from known measurement data.

Tasks:

1. Select an initial supported data source and document its licence/attribution requirements.
2. Implement local import and offline cache first.
3. Add headphone search and model/variant disambiguation.
4. Implement or port bounded AutoEQ fitting with a known response calculator.
5. Validate the fit algorithm on synthetic curves and a fixed set of reference measurements.
6. Display target curve, predicted corrected response, filter count, fit error and preamp.
7. Apply results as a new profile; never overwrite a user's current profile without confirmation.
8. Keep a record of measurement source and target curve metadata.

Deliverables:

- AutoEQ catalogue/import.
- Fit engine and tests.
- Preview/apply flow.
- `docs/AUTOEQ_DATA_SOURCES.md`.

Acceptance criteria:

- Offline-imported AutoEQ profiles remain usable without network access.
- Fitter output is deterministic for a fixed input.
- Every generated filter passes runtime validation.
- The preview and runtime use the same filter response model.
- The app explains that a database tuning is not a measurement of the user's individual headphone.

### Phase 5 — Automation polish and public beta

**Goal:** make device switching dependable for everyday use and prepare public distribution.

Tasks:

1. Add lifecycle-safe `AudioDeviceCallback` registration.
2. Add debounce, event deduplication and a stable device snapshot.
3. Add configurable automatic/ask/off policy and user feedback when a profile is switched.
4. Test Bluetooth A2DP, BLE audio if available, wired headset, USB DAC and speaker routes.
5. Add external Tasker/MacroDroid integration only if the internal automation contract is stable; document actions and validate every request.
6. Add public README, setup guide, supported-device matrix, known limitations, changelog and licence/attribution notices.
7. Use a distinct fork identity/package ID for side-by-side development testing; verify backup/restore and signing implications before public release.
8. Create reproducible CI builds and a release checklist.

Deliverables:

- Device-triggered automation.
- Public beta APK (rootless build only).
- Compatibility matrix and troubleshooting guide.
- Release workflow and source/licence notices.

Acceptance criteria:

- Device events are deterministic and do not repeatedly reapply the same profile.
- The user can always disable automation and choose a profile manually.
- No root/Magisk artifact is produced or required by the public product.
- Clean install, update and backup/restore are tested.

---

## 8. Test strategy

### 8.1 Test layers

- **Unit tests:** profile matcher, precedence, profile schema migration, EQ validation, filter response, AutoEQ fitting, rule evaluation and diagnostic redaction.
- **Integration tests:** profile-to-DSP adapter, settings persistence, import/export, capture lifecycle and device event coordinator.
- **Device tests:** at least one Samsung/One UI device and one Pixel/AOSP-like device if available; add other OEMs as the project grows.
- **Audio tests:** known sample/test signal, bypass comparison, clipping/overload detection where observable, playback transition, pause/resume and route change.
- **Release tests:** install over prior fork build, clean install, no-root permissions, backup/restore and app upgrade.

### 8.2 Minimum device/OS matrix

Test the app's declared minimum Android version (upstream rootless capture relies on Android 10/API 29 or later) and at least the newest Android versions available on the team's actual hardware. Specifically include Android 14+ MediaProjection consent rules and recent OEM builds that have reported capture-stop or route-change issues.

### 8.3 Regression scenarios

- Enable DSP, start playback, stop DSP, verify the OS capture indicator disappears.
- Start/stop quickly several times.
- Pause/resume repeatedly.
- Switch between two audio playback apps.
- Connect/disconnect two Bluetooth devices in succession.
- Connect a generic USB DAC with no useful model identity.
- Apply a profile during active playback.
- Import malformed profile and malformed EQ files.
- Apply high positive EQ/bass boosts and verify headroom/clip warning behaviour.
- Disable network and use a previously imported AutoEQ profile.
- Kill/restart the app process and recover the correct state without reusing stale MediaProjection tokens.

---

## 9. Public release, security and maintenance policy

- Upstream is GPL-3.0. Preserve the applicable licence, upstream copyright notices and attribution. Review every additional code and data source before reuse. For a public derivative, publish the corresponding source and required notices under the applicable licence terms.
- Keep the rootless product build separate from any root-enabled artifacts in development. Prefer removing unused root-only build outputs and user-facing settings from the product rather than merely hiding a toggle.
- Do not commit signing keys, API tokens or private user logs.
- Do not transmit diagnostic information without explicit user action and disclosure.
- Keep third-party AutoEQ data provenance and licence metadata with imported profiles.
- Maintain a `CHANGELOG.md`, compatibility table and a reproducible build record containing app commit, submodule commit, build variant and toolchain versions.

---

## 10. Codex-ready execution brief

Paste the following into Codex with the repository open.

```text
You are working on a rootless-only enhancement fork of:
https://github.com/timschneeb/RootlessJamesDSP

Read docs/ROOTLESS_JAMESDSP_ENHANCED_ROADMAP.md first. The product is JamesDSP Enhanced. Its chosen features are device profiles, Graphic/Parametric EQ, diagnostics, device-triggered automation, headphone AutoEQ and controlled bass enhancement. It must remain rootless. Do not add Magisk, KernelSU, AudioFlinger injection or any root-only requirement.

IMPORTANT: Do not start by implementing all features. First complete Phase 0 (repository audit and reproducible baseline).

PHASE 0 REQUIREMENTS
1. Inspect the actual source tree, Gradle modules, app manifest, application ID, build variants, capture service, MediaProjection lifecycle, DSP wrapper, native submodule, preferences/preset format, backup/import/export, UI framework and existing tests.
2. Run `./gradlew tasks` and discover the real build tasks. Do not assume task/flavor names from another fork.
3. Record the exact upstream commit SHA and `git submodule status`; make the build reproducible.
4. Attempt a clean unmodified baseline build and report the exact command/result. Do not claim a build passed unless it did.
5. Compare the upstream repository to `https://github.com/evoeram/Loundess-PEQ-RootlessJamesDSP`, especially its device presets, PEQ, AutoEQ, bass path and rootless capture path. The other fork is a reference, not an instruction to merge its entire change set. Its README reports a very large divergence and includes optional rooted and out-of-scope functionality.
6. Identify code that is safe to reuse/port, dependencies that would be required, and anything that is root-only, Direct-Mode-only, unsupported or untested in rootless Standard mode.
7. Review upstream GPL-3.0 and all additional code/data licences before recommending reuse.
8. Produce:
   - docs/UPSTREAM_AUDIT.md
   - docs/ENHANCED_FORK_COMPARISON.md
   - docs/BASELINE_BUILD.md
   - docs/FEATURE_PORTING_DECISIONS.md
9. Do not modify application behaviour, native DSP code, dependency versions, package ID or submodules during this audit phase.

AFTER PHASE 0
Return the audit findings and a proposed implementation sequence. Then implement only Phase 1 (reliability and diagnostics foundation) in a separate, reviewable set of commits. Do not proceed to later phases until Phase 1 tests and acceptance criteria pass.

ENGINEERING RULES
- Preserve the upstream rootless capture design; do not claim that every Android app can be captured.
- Follow official Android MediaProjection consent, foreground-service and onStop cleanup rules for the app's target SDK.
- Do not reuse MediaProjection consent tokens on Android 14+.
- Centralize capture/session resource cleanup and keep UI state synchronized with actual service state.
- No audio samples or personal media content in diagnostic logs.
- Do not add cloud analytics or background uploads.
- Keep Android UI/storage/network work off the real-time DSP thread.
- Validate EQ and bass parameters before passing them to native code.
- Do not add Room, Compose, new serialization libraries or other dependencies without first showing why existing project patterns are insufficient.
- Do not modify the libjamesdsp submodule unless a documented feature requirement cannot be handled through the existing engine interface. Isolate and test any native change.
- Keep all root-only build flavours and modules out of the distributed product. Do not introduce root requirements.
- Preserve existing preset and backup compatibility or provide explicit migration tests.
- Every feature must have unit/integration tests and a manual device test plan.
- Make small changes, explain the files changed, run targeted tests, then run the appropriate rootless build.

REPORTING
At the end of each phase, report: what changed, files touched, exact commands run, test/build results, unresolved issues, and the next phase recommendation. Never report a build or test as successful if it was not actually run.
```

---

## 11. Key research links

### Source repositories

- Upstream RootlessJamesDSP: https://github.com/timschneeb/RootlessJamesDSP
- Upstream README: https://github.com/timschneeb/RootlessJamesDSP/blob/master/README.md
- Upstream releases: https://github.com/timschneeb/RootlessJamesDSP/releases
- Upstream issue tracker: https://github.com/timschneeb/RootlessJamesDSP/issues
- Manual profile request (#335): https://github.com/timschneeb/RootlessJamesDSP/issues/335
- MediaProjection cleanup report (#358): https://github.com/timschneeb/RootlessJamesDSP/issues/358
- Work profile exclusion picker (#357): https://github.com/timschneeb/RootlessJamesDSP/issues/357
- Playback transition issue (#356): https://github.com/timschneeb/RootlessJamesDSP/issues/356
- Enhanced reference fork: https://github.com/evoeram/Loundess-PEQ-RootlessJamesDSP
- Enhanced reference fork releases: https://github.com/evoeram/Loundess-PEQ-RootlessJamesDSP/releases

### Android platform documentation

- Audio playback capture: https://developer.android.com/media/platform/av-capture
- AudioPlaybackCaptureConfiguration: https://developer.android.com/reference/android/media/AudioPlaybackCaptureConfiguration
- MediaProjection: https://developer.android.com/media/grow/media-projection
- Android 14 MediaProjection behaviour: https://developer.android.com/about/versions/14/behavior-changes-14
- AudioManager and AudioDeviceCallback: https://developer.android.com/reference/android/media/AudioManager

---

## 12. Final recommendation

Do not begin with a complete UI redesign or a large native DSP rewrite.

Start with a reproducible upstream baseline, compare the existing enhanced fork carefully, and make capture lifecycle diagnostics reliable. Next build manual profiles and device-linked switching, followed by EQ/bass controls, then AutoEQ and automation polish.

The strongest differentiator should be **reliability and clarity**: users should know exactly which route and profile are active, whether processing is functioning, what AutoEQ has changed, and how to recover when Android interrupts capture.
