# Enhanced Fork Comparison (Phase 0)

**Date:** 8 November 2026
**Reference fork:** `evoeram/Loundess-PEQ-RootlessJamesDSP`
**Reviewed at:** `0bf77cd9f95aa213d6a9df4fbb108fc63572ce3e` — "release: v1.6.29 (72)", 2026-11-04 (`master` HEAD).
**Compared against:** `868DanDoo/RootlessJamesDSP` @ `60d25ae8a53c6f4691c090673df290a73c6b6357` (upstream v1.6.14).
**Method:** shallow read-only clone, file/diff inventory, reading of selected source headers, manifest, Gradle files and README. **The reference fork was not built, installed or run.** Its native submodule (`evoeram/JamesDSPManager`) was **not** fetched or read. Statements about runtime behaviour come from the fork's own README and are labelled as such.

---

## 1. Size of the divergence (measured)

| Metric | Upstream fork | Reference fork |
|---|---|---|
| Kotlin lines under `app/src` | 20,858 | 55,769 |
| Entries only in the reference fork (`app/src`, excl. submodule; each is a file **or a whole new directory**, so the true file count is higher) | — | 124 |
| Files present in both but different | — | 76 |
| Unit-test classes (`*Test.kt`) | 2 (+1 template) | 38 |
| Version | 1.6.14 (51) | 1.6.29 (72); `targetSdk 35` (upstream 34) |

The fork's README claims "150+ commits, 370+ files, ~73,000 lines". My count of files and Kotlin lines is in the same order of magnitude; I did not verify the commit count (shallow clone). The conclusion in the roadmap holds: this is a different product, not a patch set.

Not a port target on size alone: a wholesale merge would mean owning ~35k additional Kotlin lines plus new native code.

---

## 2. Feature map

"Verified" = I found the code in the tree. "Claimed" = from the fork's README only.

| Area | Upstream | Reference fork | Status of evidence |
|---|---|---|---|
| Parametric EQ | PK / LS / HS bands, applied via the combined Graphic-EQ path | Time-domain RBJ biquad cascade (8 filter types + preamp), per-band L/R routing, up to 64 bands (README) | Native files **verified**: `libjamesdsp-wrapper/biquad/{BiQuad.h,BiQuadFilter.*,ParametricEqProcessor.*}`; Kotlin `ParametricEqResponseCalculator.kt`; tests `ParametricEqBandList*Test`. Audio quality **not tested**. |
| Device presets | Profiles keyed per device, auto-rotate (`ProfileManager`) | `DevicePresetManager`, `DeviceCardsFragment`, `PresetSelectionActivity`, `PresetOverlayManager`; modes none / ask / named preset | Files **verified**; behaviour claimed. Built on the same `RoutingObserver` (MediaRouter) as upstream. |
| AutoEQ | Online search via author-hosted API → Graphic EQ | Adds Squig Live (`SquigLink*`, `SquigMeasurementStore`, `squig/`), greedy PEQ fitter, Nelder–Mead, GraphEQ→PEQ converter | Files **verified**. Data source/licence **not reviewed** (see §5). |
| Bass | `setBassBoost(enable, maxGain)` | Same plus harmonic / sub-harmonic expanders (`ExpanderSurface`, `ViperExtrasMath`) | Expanders **verified present**; behaviour claimed. |
| Loudness correction | — | Fletcher–Munson classic + ISO 226:2023 | Verified present (`LoudnessCorrectionProcessor.h`, `LoudnessCurveCalculator.kt`); claimed behaviour. |
| Diagnostics | Session dump, compat screens | `latency/` (LatencyTracer), CrashGuard native crash recorder, `docs/TROUBLESHOOTING_ANDROID16.md` | Directory names verified; internals **not reviewed**. |
| Processing modes | One capture loop | Standard, Low-Latency (QueueController), **Direct Mode** (`DynamicsProcessing`) | Directory names verified (`androideq/`, `latency/`). README states Direct Mode is **EQ-only** (no reverb, convolver, bass boost). |
| Capture source | MediaProjection | Adds Shizuku privileged loopback (`loopback/`) | Present; requires Shizuku (ADB or root). |
| Measurement / MEOW | — | Farina sweep, deconvolution, mic calibration (`measurement/`, `MeasurementActivity`) | Present; matches roadmap's "advanced, later" classification. |
| Voice isolation | — | GTCRN / DeepFilterNet3 / DPDFNet via ONNX Runtime (`voiceisolation/`) | Present; adds `onnxruntime-android:1.20.0` (the only new Gradle dependency vs. upstream). |
| NOS R2R, Smooth Volume | — | `VolumeKeyAccessibilityService`, `VolumeOverlay` | Present. |
| Root | Root flavor + Magisk instructions | Enhanced Magisk module, 14 module zips committed | Present. |

---

## 3. Verified hazards in the reference fork

| # | Finding | Evidence | Consequence |
|---|---|---|---|
| H1 | A **release keystore is committed**: `keystore/release.jks`, and `app/build.gradle.kts` uses it with `storePassword = "android"`. | `git ls-files keystore`; gradle lines 48–65 | Anyone can sign an APK with the same key. Never reuse this key or this signing config; never ship anything signed by it. |
| H2 | `WRITE_SECURE_SETTINGS` is declared and used in `SettingsAudioFormatFragment`. | manifest line 15; source lines ~290–326 | Roadmap §4.6 explicitly rejects this permission for basic automation. |
| H3 | An **AccessibilityService** (`VolumeKeyAccessibilityService`) is declared. | manifest ~lines 307–314 | Roadmap non-goal (volume-key interception) and a sensitive permission for a public app. |
| H4 | `FOREGROUND_SERVICE_MEDIA_PLAYBACK` added; `targetSdk` raised to 35. | manifest; `AndroidConfig.kt` | Changes platform behaviour requirements; would need its own testing. |
| H5 | The native submodule points at **`evoeram/JamesDSPManager`**, not ThePBone's. | fork `.gitmodules` | Native engine provenance, patches and licensing would need a separate review; it changes what "reproducible build" means. |
| H6 | Binary artifacts and large non-source content are tracked: 14 Magisk module zips, a PDF (`next_step/`), `debug/hiby-r4/`, images. | `git ls-files` | Repository hygiene; do not import. |
| H7 | Ported EqualizerAPO code is labelled **"GPLv2"** in headers (e.g. `BiQuadFilter.cpp`: "Copyright (C) 2014 Jonas Thedering, GPLv2"). The fork's README calls this "GPLv2-compatible". | file header; README licence section | **Unresolved licence risk.** GPL-2.0-*only* code cannot be combined into a GPL-3.0 work; GPL-2.0-*or-later* can. I could not confirm which applies to the original EqualizerAPO files, and the header text does not say "or later". Do not port this code until the original licence grant is read and recorded. |
| H8 | Its own `NOTICE` covers only the Magisk sepolicy material (MIT, Rei Ryuki). | `NOTICE` | Attributions for ported DSP/ML code (EqualizerAPO, voice-isolation models) are not centrally recorded. |
| H9 | README states Direct Mode is EQ-only and rootless Standard/Low-Latency modes add capture-loop latency. | README "Processing Modes" | Matches the roadmap's concern. Features that exist only in Direct Mode or root must not be counted as rootless-Standard features. |

None of these were confirmed by running the app.

---

## 4. Rootless Standard path versus Direct/root

From the README (claimed, not run):

- **Standard** (legacy MediaProjection loop): full JamesDSP engine; ~150–170 ms latency.
- **Low-Latency:** optimized capture loop; ~20–80 ms; full engine.
- **Direct:** `DynamicsProcessing` attached per session; EQ only; no capture delay.
- **Root:** AudioFlinger integration via Magisk module.

For this project's rootless-only rule, the relevant paths are Standard and Low-Latency (both use the same capture model as upstream, so they inherit the same MediaProjection lifecycle issues listed in `UPSTREAM_AUDIT.md` §5) and possibly Direct Mode (needs separate evaluation; it is not a capture-loop feature).

---

## 5. Items not reviewed (must be done before any port)

- `SquigLinkClient` / `SquigBatchDownloader` / `SquigLinkParser`: what hosts and endpoints are used, whether they are documented APIs, and the licence/attribution terms of the measurement data. Roadmap §4.4 forbids depending on an undocumented endpoint.
- Native changes in `evoeram/JamesDSPManager` relative to ThePBone's `extensions` branch.
- `DevicePresetManager` / `PresetOverlayManager` internals (overlay uses `SYSTEM_ALERT_WINDOW`, which upstream also declares).
- `latency/` and CrashGuard (cost, risk, and whether anything uploads).
- The correctness and audio quality of the biquad cascade, and whether it is safe on the real-time thread.
- Whether any shipped feature talks to the network beyond AutoEQ/Squig.

---

## 6. Bottom line

- The fork is a useful **idea and test-vector source** (device-preset UX, biquad PEQ with a response calculator, GraphEQ→PEQ conversion, unit tests) but is unsuitable as a base: huge, multi-mode, root-enabled, with a committed signing key, a sensitive-permission surface and an unresolved GPLv2 question.
- Recommendation unchanged from the roadmap: keep upstream as the baseline, port selectively, one reviewed component at a time. See `FEATURE_PORTING_DECISIONS.md`.
