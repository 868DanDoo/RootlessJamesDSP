# Feature Porting Decisions (Phase 0)

**Date:** 8 November 2026
**Inputs:** `UPSTREAM_AUDIT.md`, `ENHANCED_FORK_COMPARISON.md`, `BASELINE_BUILD.md`.
**Rule from the roadmap:** every feature gets one decision — **Reuse** existing upstream code, **Port** selectively from the reference fork, or **New** implementation. Nothing is merged only because the other fork has it.

**Caveat that applies to every row:** no build or device test could be run in the audit workspace, so each decision is based on source reading. Each "needs device test" item below is a gate, not a formality.

---

## 1. Decisions by feature

| # | Feature | Decision | Reasoning | Gate before starting |
|---|---|---|---|---|
| 1 | **Capture lifecycle / MediaProjection** (Phase 1) | **New (fix in place)** | Upstream owns the capture design. Findings L1–L3 are in upstream code (`MediaProjection.stop()` never called; consent token redelivered; `onStop` ignored while UI is foreground). The reference fork shares the same loop, so there is nothing to port. | Reproduce L1 on a device: enable DSP, disable it, check the system capture indicator. |
| 2 | **Diagnostics model + dashboard + redacted export** | **New**, reusing `session/dump` and session managers as data sources | Neither project has the roadmap's state machine or redaction rules. The fork's `LatencyTracer`/CrashGuard are debug/crash tools, not a status model. | Define states and events in a doc first; confirm what is observable (sample rate, channel count, signal presence). |
| 3 | **Manual profile selection** | **New on top of `ProfileManager`** | Roadmap §4.1 asks for manual selection even for unidentifiable DACs; upstream has only automatic rotation by route. | None beyond Phase 1. |
| 4 | **Device-linked profiles (policies, precedence, default)** | **Reuse + extend** `ProfileManager`/`RoutingObserver` | Profiles already exist, with a profile directory per device and `.tar` presets. Add `Off/Ask/Automatic`, default fallback, friendly names and deterministic precedence. **Port ideas only** from the fork's `DevicePresetManager` / ask-dialog UX; do not import its classes (overlay + more permissions). | Decide a migration for existing profile ids (see risk R3). |
| 5 | **Fix profile rotate safety** (store-before-load race, no debounce) | **New (fix in place)** | In `ProfileManager.rotate()` the outgoing profile is overwritten with the live settings before the incoming one loads; a bad route event can corrupt the wrong profile. | Unit tests for rotate ordering before refactoring. |
| 6 | **Device event source** (`AudioDeviceCallback`) | **New**, replacing the `MediaRouter` route callbacks inside `RoutingObserver` | Roadmap §4.6 requires `AudioManager.registerAudioDeviceCallback()` with debounce. Both projects currently use `MediaRouter`. | Device test on A2DP, wired, USB DAC, speaker. |
| 7 | **Graphic EQ** | **Reuse** | Already works; add regression tests and input validation only. | — |
| 8 | **Parametric EQ — validation, per-band bypass, headroom warning, response preview** | **Reuse + extend** upstream PEQ | Upstream has a PEQ editor, APO import and a surface already. The roadmap's "add a PEQ" is already satisfied for PK/LS/HS; the gaps are safeguards (NaN/Nyquist/Q checks, preamp compensation, tests). | Add unit tests first (none exist upstream). |
| 9 | **Parametric EQ — true biquad cascade / more filter types / L-R routing** | **Defer; Port later, conditional** | Reference fork has a time-domain RBJ biquad cascade (`libjamesdsp-wrapper/biquad/*`, `ParametricEqResponseCalculator.kt`) with tests. The native code lives in the **app's** wrapper dir, not in the submodule, so a port does not require changing `libjamesdsp`. **Blocked by licence question H7** (headers say "GPLv2"; compatibility with GPLv3 unconfirmed) and by a real-time-safety review. | (a) Read the original EqualizerAPO licence grant and record it. (b) Review the processor for allocation/locks on the audio thread. (c) A/B listening test vs. upstream PEQ. |
| 10 | **Bass enhancement** | **Reuse** `setBassBoost(enable, maxGain)` and the `dsp_bass` card | Roadmap says reuse first. Default-off in new profiles, include in profile save. | Device test: audible and non-clipping in the rootless capture path; decide if "turnover/headroom" controls are possible with the existing API or need a low-shelf fallback. |
| 11 | Harmonic / sub-harmonic expanders (fork) | **Do not port (now)** | Roadmap says dynamic/harmonic bass is "later, after objective tests". | — |
| 12 | **AutoEQ — online search/apply** | **Reuse** (`AeqSelectorActivity`, `AutoEqClient`) | Works today (author-hosted API, configurable URL). | Treat the server as an external dependency; keep working offline (see 13). |
| 13 | **AutoEQ — local import, offline cache, metadata** | **New** | Roadmap requires offline-capable, with source/model/target/licence metadata. Neither project does this cleanly. | Choose and document the first data source and its licence. |
| 14 | **AutoEQ — PEQ fitting (greedy / Nelder-Mead) + fit-error report** | **Port, conditional** (reference `utils/GraphEqToPeqConverter.kt`, `measurement/AutoEqEngine.kt`, `squig/SquigAutoEqEngine.kt`, with `AutoEqEngineTest` among its 38 test classes) | Roadmap wants a bounded, deterministic, tested fitter. The fork has candidates **and tests**, which are worth porting even if the code is rewritten. | Licence/origin review of each file; confirm it uses the same filter model as the runtime (depends on decision 9). |
| 15 | Squig Live / SquigLink database | **Do not port until reviewed** | Endpoint documentation, rate limits and data licence unknown (not reviewed). Roadmap forbids depending on an undocumented endpoint. | Review `SquigLinkClient`, `SquigBatchDownloader`, terms of the data source. |
| 16 | Acoustic measurement / MEOW wizard | **Do not port** | Roadmap non-goal for MVP (arbitrary phone mic). | — |
| 17 | Loudness correction | **Do not port (now)** | Not in the chosen feature set. | — |
| 18 | Direct Mode (`DynamicsProcessing`) | **Defer; evaluate separately** | EQ-only (per the fork's README) but avoids capture latency and MediaProjection. Worth a separate decision after Phase 1; it is not required by the roadmap. | Product decision. |
| 19 | Low-Latency capture / QueueController | **Defer** | Could address latency but touches the capture loop that Phase 1 is stabilizing. | Phase 1 done first. |
| 20 | Shizuku privileged loopback | **Do not port** | Needs ADB/root helper; out of roadmap scope. | — |
| 21 | Voice isolation (ONNX Runtime) | **Do not port** | Roadmap non-goal; adds a large native dependency. | — |
| 22 | NOS R2R simulator | **Do not port** | Roadmap non-goal. | — |
| 23 | Smooth Volume (AccessibilityService, `WRITE_SECURE_SETTINGS`) | **Do not port** | Roadmap non-goal and explicitly rejects those permissions. | — |
| 24 | Magisk module / root flavors / KernelSU etc. | **Do not port; remove from product** | Roadmap root policy. Upstream's own `root`/`plugin` flavors also need removal before a public release. | Phase 5. |
| 25 | Fork's release keystore and signing config | **Never import** | Committed with a public password. | — |
| 26 | Fork's native submodule (`evoeram/JamesDSPManager`) | **Do not adopt** | Keep upstream's `ThePBone/JamesDSPManager @ 1b9f89db`. | — |
| 27 | Fork's unit tests | **Port selectively** | Tests for APO parsing, PEQ response and AutoEQ fitting are valuable regardless of decision 9. | Only tests for components we actually adopt. |
| 28 | Tasker/MacroDroid actions | **Defer**, and **secure the existing `PowerStateReceiver` first** | It is exported with no permission and no sender check (see audit §3). | Phase 5. |

---

## 2. Dependencies each port would add

| Port | New Gradle dependencies | New native code | New permissions |
|---|---|---|---|
| PEQ biquad cascade (#9) | none | yes (app wrapper dir) | none |
| AutoEQ fitter (#14) | none expected (Kotlin math) | none | none |
| Squig Live (#15) | none expected (already OkHttp/Retrofit) | none | none |
| Voice isolation / Smooth volume / Direct mode | `onnxruntime-android` for voice isolation | yes | AccessibilityService, `WRITE_SECURE_SETTINGS`, `FOREGROUND_SERVICE_MEDIA_PLAYBACK` |

Every adopted item in the table needs no new dependency and no new permission, which is the intended property of the plan.

---

## 3. Risk register

| ID | Risk | Mitigation |
|---|---|---|
| R1 | Capture lifecycle issues (L1–L3) are only suspected from code. | Reproduce on a device before changing code; add a state machine and tests. |
| R2 | GPLv2 vs GPLv3 for ported EqualizerAPO code (H7). | Read the original licence; record in a `NOTICE`; if "GPLv2 only", reimplement from the published algorithm (RBJ cookbook) instead of porting source. |
| R3 | Changing Bluetooth profile ids (currently MAC-like address) orphans stored profiles. | Write a one-time migration with tests; keep old directories until migration succeeds. |
| R4 | `full` flavor sends crash data to the upstream author's Firebase project. | Build `fdroid`; remove Firebase and `google-services.json` before any public release. |
| R5 | Single-maintainer AutoEQ backend (`aeq.timschneeberger.me`). | Offline import/cache (#13). |
| R6 | No automated tests upstream. | Add tests before refactoring any logic (#5, #8). |
| R7 | Build not verified anywhere yet. | First action on the user's PC: `BASELINE_BUILD.md` §6. |
| R8 | Release builds are debug-signed. | Create a new project keystore before any distribution; never commit it. |

---

## 4. Proposed implementation sequence

1. **Baseline on the PC** (`BASELINE_BUILD.md` §6); record the result in that file.
2. **Reproduce L1–L3** with the system capture indicator; write findings in `docs/DIAGNOSTICS.md` (draft).
3. **Phase 1** (fixes + state model + diagnostics screen), in small commits:
   - L1: stop the projection and release resources in one place.
   - L2: stop reusing consent tokens; don't redeliver the consent intent.
   - L3: make `onStop` always update state; show UI state from the service state.
   - L4/L6: make cross-thread flags volatile/atomic; make shutdown join robust.
4. **Phase 2**: profile tests → rotate-safety fix (#5) → manual selection (#3) → policies + default (#4) → `AudioDeviceCallback` (#6) → id migration (R3).
5. **Phase 3**: PEQ validation/tests (#8), bass reuse (#10), A/B/bypass.
6. **Phase 4**: local import + cache (#13) → fitter (#14, after licence check) → apply as a new profile.
7. **Phase 5**: secure/remove `PowerStateReceiver`, remove root/plugin flavors and Firebase, new signing key, release notes and notices.

Do not start Phase 1 code changes until the audit has been reviewed.

---

## 5. Roadmap exit criteria status (Phase 0)

| Criterion | Status |
|---|---|
| Clean baseline build succeeds or blocker documented | Blocker documented; **build not verified** |
| Source and submodule commits pinned | Done (`60d25ae8…`, `1b9f89db…`) |
| Every planned feature has a decision | Done (table above) |
| No feature merged only because it exists in the other fork | Nothing merged; no application code changed |

**Files touched in this phase:** only new files under `docs/` (the four documents plus a copy of the roadmap). No Kotlin, Gradle, manifest, native or submodule-pin change.
