# Algorithm Comparison: mobgap vs SKDH

This document catalogs all gait analysis algorithms available across **mobgap** and **scikit-digital-health (SKDH)** for each step of the pipeline. The goal is to inform selection of the best-performing workflow for a given dataset by making the theoretical basis and practical trade-offs of each algorithm explicit.

> **Framework note**: mobgap is built on [tpcp](https://github.com/mad-lab-fau/tpcp) — algorithms are injectable objects with typed pipeline slots. SKDH uses a simpler `BaseProcess.predict(**kwargs)` interface with dict-based data flow. Integrating the two requires adapter classes; see the architecture discussion in project docs.

---

## Step 1 — Gait Sequence Detection (GSD)

### mobgap

| Algorithm | Core Method | Key Parameters | Notes |
|---|---|---|---|
| `GsdIluz` | Sinusoidal template convolution (2 Hz sine) on IS + PA axes; activity/posture thresholds; window-based | `window_length_s=3`, `sin_template_freq_hz=2`, `min_gsd_duration_s=5` | Validated on Mobilise-D healthy cohort; needs body-frame axes |
| `GsdIonescu` | CWT (Mexican Hat) on acceleration norm; fixed amplitude threshold; merges nearby step detections | `active_signal_threshold=0.1g`, `min_n_steps=5`, `max_gap_s=3.5` | Paraschiv-Ionescu 2014; accepts sensor-frame (uses norm) |
| `GsdAdaptiveIonescu` | CWT + **data-driven adaptive threshold** from Hilbert envelope of active periods | `active_signal_fallback_threshold=0.15g`, `min_n_steps=5` | Paraschiv-Ionescu 2019; designed for impaired gait (PD, MS, stroke) |

### SKDH — `context` module

| Algorithm | Core Method | Key Parameters | Notes |
|---|---|---|---|
| `Ambulation` | **LightGBM classifier** (27 features: spectral, statistical, entropy, wavelet) on non-overlapping 3-second windows | `prob_threshold=0.7`; resamples to 20 Hz | General ambulation detection; orientation-invariant; outputs per-epoch predictions + bout indices |
| `PredictGaitLumbarLgbm` | **LightGBM classifier** (18 features, band-pass 0.25–5 Hz) on non-overlapping 3-second windows; **lumbar-specific** | `prob_threshold=0.7`; models for 20 Hz and 50 Hz | Optimised for lumbar-mounted IMU; outputs `Gait Bout Start/Stop` indices directly |

> **Practical note**: SKDH context algorithms output binary per-window predictions or bout indices directly. They connect to `GaitLumbar` via `get_gait_bouts()` (which merges short-gap predictions, default `max_sep=0.5s`, `min_duration=8s`). They are **not** drop-in replacements for mobgap's `BaseGsDetector` without an adapter.

---

## Step 2 — Initial Contact Detection (ICD)

IC detection is the most critical step — errors here propagate into every downstream metric.

| Algorithm | Package | Core Method | Axis Used | Also Detects FC? | Theoretical Relation |
|---|---|---|---|---|---|
| `IcdIonescu` | mobgap | Integrate vertical accel → CWT (Ricker, width=9) → zero-crossing; negative peaks = IC | Vertical (IS) | No | ≈ SKDH `VerticalCwtGaitEvents` |
| `IcdShinImproved` | mobgap | CWT + zero-crossing on filtered accelerometer; linear interpolation for sub-sample timing | AP, ML, IS, or norm | No | ≈ SKDH `ApCwtGaitEvents` |
| `IcdHKLeeImproved` | mobgap | **Morphological filtering** (closing − opening > 0 → IC candidate) | AP, ML, IS, or norm | No | Unique; no SKDH equivalent |
| `VerticalCwtGaitEvents` | SKDH | CWT (gaus1) on vertical **velocity**; IC = minima; adaptive scale: `−10·sf + 56` | Vertical (auto-detected) | **Yes** | ≈ mobgap `IcdIonescu` |
| `ApCwtGaitEvents` | SKDH | CWT (gaus1) on AP accel (IC) and AP velocity (FC); IC at zero-crossing between peaks | AP (auto-detected) | **Yes** | ≈ mobgap `IcdShinImproved` |

**Key difference**: SKDH's two methods simultaneously detect **Final Contact (toe-off)**, which unlocks stance/swing time metrics. mobgap's three methods detect IC only.

---

## Step 3 — Laterality Classification (LRC)

| Algorithm | Package | Core Method | Sensor |
|---|---|---|---|
| `LrcMansour` | mobgap | Sign of ML acceleration **derivative** after 1 Hz Butterworth low-pass | Accelerometer (ML axis) |
| `LrcMcCamley` | mobgap | Sign of **gyroscope angular velocity** after 0.5–2 Hz bandpass | Gyroscope (yaw/roll) |
| `LrcUllrich` | mobgap | **SVM classifier** on 6D gyroscope feature vector (value + 1st + 2nd derivative at IC); MinMax-scaled; pre-trained on MS-Project dataset | Gyroscope (IS + PA axes) |
| — | SKDH | No explicit LRC step; left/right foot assignment is implicit through IC/FC pairing in `CreateStridesAndQc` | — |

---

## Step 4 — Cadence

| Algorithm | Package | Output Granularity | Method |
|---|---|---|---|
| `CadFromIc` | mobgap | Per second (steps/min) | IC intervals → Hampel filter (outlier removal) → linear interpolation → 1-second bin mean |
| `Cadence` endpoint | SKDH | Per step (steps/min) | `60 / step_time` per step; stored in `gait["cadence"]` |
| `PreprocessGaitBout` | SKDH | Per bout (Hz; for internal use) | Wavelet CWT on AP accel in 5-second windows → median step frequency |

**Theoretical basis is identical** across all three. The difference is granularity: mobgap gives one value per second; SKDH gives one value per step.

---

## Step 5 — Stride Length

| Algorithm | Package | Model | Formula | Height Reference | Requires FC? |
|---|---|---|---|---|---|
| `SlZijlstra` | mobgap | **Inverted pendulum** (Zijlstra & Hof 2003) | `2√(2·h·Δd − Δd²)` | Lower-back height (default 0.54 m) | No |
| `StepLengthModel1` | SKDH | **Inverted pendulum** (same model) | `2√(2·l_leg·Δh − Δh²)` | Leg length = 0.53 × height | No |
| `StepLengthModel2` | SKDH | **Double pendulum** (IDS + SS phases) | `L_IDS + L_SS` (separate phase modeling) | Leg length | **Yes** |
| `StrideLengthModel1/2` | SKDH | Sum of two consecutive step lengths | `L_step[i] + L_step[i+1]` | — | Depends on model |

`SlZijlstra` and `StepLengthModel1` implement **the same formula** — differences are only in the height reference point and a cohort-specific scaling factor (`~0.97` for MS, `~1.0` general). `StepLengthModel2` is a more complex double-pendulum approach that should theoretically be more accurate but requires FC detection.

---

## Step 6 — Walking Speed

| Algorithm | Package | Formula |
|---|---|---|
| `WsNaive` | mobgap | `stride_length × cadence / 120` |
| `GaitSpeedModel1` | SKDH | `stride_length_m1 / stride_time` |
| `GaitSpeedModel2` | SKDH | `stride_length_m2 / stride_time` |

All three are mathematically equivalent (`speed = distance / time`). Differences come from upstream stride length and cadence calculations.

---

## Step 7 — Turn Detection

| Algorithm | Package | Core Method | Sensor |
|---|---|---|---|
| `TdElGohary` | mobgap | Peak-based yaw angular velocity (LP filtered); thresholds on peak amplitude and duration; merges consecutive same-direction turns | Gyroscope (yaw) |
| `TurnDetection` | SKDH | Rotation matrix yaw tracking via Rodrigues' formula; sign changes in unwrapped yaw angle; hesitation filtering (<0.5s reversals) | Gyroscope (3-axis) |

Both use gyroscope-based yaw tracking but differ in implementation: mobgap uses threshold-based peak detection; SKDH uses full rotation matrix integration.

---

## Step 8 — Temporal Gait Parameters (SKDH only — requires FC)

These metrics are **not available in mobgap** because it does not detect Final Contact (toe-off).

| Metric | Endpoint Class | Formula | Level |
|---|---|---|---|
| Stance Time | `StanceTime` | `(FC[i] − IC[i]) / fs` | Per step |
| Swing Time | `SwingTime` | `(IC[i+2] − FC[i]) / fs` | Per step |
| Initial Double Support | `InitialDoubleSupport` | `(FC_oppfoot[i] − IC[i]) / fs` | Per step |
| Terminal Double Support | `TerminalDoubleSupport` | `(FC_oppfoot[i+1] − IC[i+1]) / fs` | Per step |
| Double Support | `DoubleSupport` | IDS + TDS | Per step |
| Single Support | `SingleSupport` | `(IC[i+1] − FC_oppfoot[i]) / fs` | Per step |

---

## Step 9 — Symmetry and Regularity Metrics (SKDH only)

mobgap's `MobilisedAggregator` computes summary statistics (mean, CV, 90th percentile) across walking bouts but does not implement any of the following signal-processing-based symmetry metrics.

### Event-level (per step; IC only)

| Metric | Endpoint Class | Method | Range |
|---|---|---|---|
| Intra-step Covariance | `IntraStepCovarianceV` | Autocorrelation of vertical accel at step-time lag | 0–1 |
| Intra-stride Covariance | `IntraStrideCovarianceV` | Autocorrelation of vertical accel at stride-time lag | 0–1 |
| Harmonic Ratio | `HarmonicRatioV` | `Σ F[even harmonics] / Σ F[odd harmonics]` (FFT, 20 harmonics) | >0; higher = more symmetric |
| Stride SPARC | `StrideSPARC` | Spectral Arc Length of `‖accel‖` (smoothness measure) | ≤0; closer to 0 = smoother |

### Bout-level (one value per gait bout; IC only)

| Metric | Endpoint Class | Method | Range | Reference |
|---|---|---|---|---|
| Step Regularity | `StepRegularityV` | Unbiased autocovariance of vertical accel at step-time lag | 0–1 | Moe-Nilssen & Helbostad 2004 |
| Stride Regularity | `StrideRegularityV` | Unbiased autocovariance of vertical accel at stride-time lag | 0–1 | Moe-Nilssen & Helbostad 2004 |
| Gait Symmetry Index | `GaitSymmetryIndex` | `√(autocovariance at 0.5·stride_lag across 3 axes) / √3` | 0–1 | Zhang 2018, Buckley 2020 |
| Phase Coordination Index | `PhaseCoordinationIndex` | `100 × (CV(step/stride ratio) + mean_abs_dev / 0.5)` | ≥0; lower = better | Plotnik 2007 |
| Autocovariance Symmetry | `AutocovarianceSymmetryV` | `abs(stride_regularity − step_regularity)` | 0–1 | — |
| Regularity Index | `RegularityIndexV` | `1 − 2·abs(R_stride − R_step) / (R_step + R_stride)` | 0–1 | Angelini 2020 |

---

## Summary: Algorithm Coverage by Step

| Pipeline Step | mobgap | SKDH | Shared Theory |
|---|---|---|---|
| **GSD** | 3 algorithms (CWT fixed, CWT adaptive, sinusoidal template) | 2 algorithms (LightGBM general, LightGBM lumbar) | Partially (CWT-based) |
| **ICD** | 3 algorithms (CWT vertical, CWT AP, morphological) | 2 algorithms (CWT vertical + FC, CWT AP + FC) | Yes — CWT methods overlap |
| **LRC** | 3 algorithms (accel derivative, gyro sign, SVM) | None (implicit via IC/FC pairing) | — |
| **Cadence** | 1 (per-second bins) | 1 endpoint (per-step) | Same formula |
| **Stride Length** | 1 (inverted pendulum) | 3 (inverted pendulum, double pendulum, stride sum) | Inverted pendulum = identical |
| **Walking Speed** | 1 (naive product) | 2 (from each stride length model) | Identical formula |
| **Turn Detection** | 1 (peak-based yaw) | 1 (rotation matrix yaw) | Same concept, different impl. |
| **Temporal params** (stance, swing, DS) | None | 6 endpoints (all require FC) | SKDH only |
| **Symmetry / Regularity** | None | 10 endpoints (IC only or IC+FC) | SKDH only |

---

## Decision Guide: Which Algorithms to Combine

```
Goal                                    Feasible path
──────────────────────────────────────────────────────────────────
Standard spatiotemporal metrics only    Pure mobgap pipeline
  (cadence, stride length, speed)

Add symmetry/regularity metrics         mobgap pipeline (GSD → ICD → LRC)
  without stance/swing time             + SKDH bout-level endpoints as
  (HarmonicRatio, GSI, PCI, etc.)       post-processing (IC only needed)

Add stance/swing/double-support time    Replace mobgap ICD with SKDH
                                        ApCwtGaitEvents or VerticalCwtGaitEvents
                                        (these also produce FC)

Add double-pendulum stride length       Same as above (Model2 needs FC)

Compare IC detection methods            Run all 5 ICD algorithms on same data;
  to minimise error on your dataset     evaluate against mocap ground truth

Compare GSD methods                     Run all 5 GSD algorithms;
  for free-living data                  evaluate on labelled segments
```

---

## Technical Challenges

### 1. Incompatible Framework Architectures

mobgap is built on **tpcp**: algorithms are typed objects injected into pipeline slots via constructor, results stored as instance attributes with trailing `_` (`ic_list_`, `gs_list_`), and every algorithm must subclass a domain-specific base class (`BaseGsDetector`, `BaseIcDetector`, etc.).

SKDH uses a flat **`BaseProcess.predict(**kwargs)`** interface: data flows through a sequential list of steps as a shared dict, and results are returned as dictionaries rather than stored on the object.

These two designs are incompatible without adapter classes. Every SKDH algorithm intended for use inside a mobgap pipeline requires a wrapper that:
- Subclasses the correct mobgap base class
- Translates the input DataFrame into SKDH's expected array/dict format
- Converts the output dict back into the DataFrame schema mobgap expects
- Follows tpcp's `clone()`-before-run convention

### 2. Final Contact (FC) as a Dependency Chain Blocker

mobgap's pipeline has no step for detecting toe-off (Final Contact). This is the single largest gap between the two frameworks. A large subset of SKDH's most clinically informative metrics — stance time, swing time, single/double support time, and the double-pendulum stride length model — all require FC as input.

To unlock these metrics, the ICD step must be replaced with a SKDH method (`ApCwtGaitEvents` or `VerticalCwtGaitEvents`), which means:
- Writing an ICD adapter that wraps a SKDH event detector and exposes FC alongside IC
- Deciding how to pass FC downstream through mobgap's pipeline (which has no FC slot)
- Handling the fact that FC availability becomes a conditional capability depending on which ICD algorithm is chosen

### 3. SKDH Endpoint Data Format Is Internal

SKDH's symmetry and regularity endpoints (`GaitSymmetryIndex`, `PhaseCoordinationIndex`, etc.) do not take standard DataFrames as input. They require:
- A `gait` dict pre-populated with `IC`, `FC`, `FC opp foot`, `Bout N`, and other internal keys
- A `gait_aux` dict containing per-bout raw acceleration arrays with explicit vertical and AP axis indices

This format is SKDH-internal and is populated by `GaitLumbar`'s own pipeline. Using these endpoints on data produced by mobgap requires a format converter that reconstructs `gait` and `gait_aux` from mobgap's `ic_list_`, `stride_list_`, `wb_list_`, and the original raw IMU data. The `gait_aux["accel"]` field in particular requires slicing the raw IMU into per-bout arrays aligned with the correct axis mapping — information that is not directly available in mobgap's output.

### 4. GSD Interface Mismatch

SKDH's context module (`Ambulation`, `PredictGaitLumbarLgbm`) outputs binary per-window predictions or `(N, 2)` bout index arrays. mobgap's GSD slot expects a `BaseGsDetector` subclass that produces a `gs_list_` DataFrame with `start` and `end` columns in samples.

The two representations are convertible, but the adapter must also handle:
- SKDH operating on 3-second non-overlapping windows (coarser resolution than mobgap's sample-level output)
- Probability thresholding (SKDH uses `prob_threshold=0.7` by default)
- Minimum bout duration and gap-merging logic (`get_gait_bouts()`) that may conflict with mobgap's own `GsIterator` assumptions

### 5. Axis Convention Differences

mobgap expects data in body-frame coordinates with explicit axes: `acc_is` (inferior-superior / vertical), `acc_ml` (mediolateral), `acc_pa` (posterior-anterior). Sensor-to-body alignment is handled in the preprocessing step and must be correct before any algorithm runs.

SKDH's `PreprocessGaitBout` **auto-detects** the vertical and AP axes from the data itself (vertical = axis with highest mean magnitude; AP = cross-correlation with vertical). This makes SKDH more robust to unknown sensor orientations but introduces a potential source of inconsistency: if SKDH's axis estimation disagrees with the preprocessing rotation applied for mobgap, the two frameworks will be operating on different axis assumptions for the same recording.

### 6. Endpoint Dependency Resolution

SKDH endpoints declare dependencies on other endpoints via `super().__init__(depends=[...])`. When called standalone (outside `GaitLumbar`), dependencies are resolved automatically by calling `predict()` on each dependency first. However, this requires the `gait` and `gait_aux` dicts to be correctly populated for all dependency chains — for example, `DoubleSupport` depends on both `InitialDoubleSupport` and `TerminalDoubleSupport`, each of which depends on FC data being present. Missing any upstream input raises a silent error (NaN-filled output) rather than an explicit exception, making debugging difficult.
