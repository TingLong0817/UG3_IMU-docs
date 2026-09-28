# Pipeline options explained

What every control on the **Pipeline** tab of the GUI (`python scripts/imu_pipeline.py`) actually does,
and how to choose. For the step-by-step of loading data and pressing Run, see
[Running an analysis](../scripts/README.md).

---

## Presets

Three buttons that fill in every option below at once. Start from the one that matches your recording,
then adjust if needed.

| Preset | Use it for | What it sets |
|--------|-----------|--------------|
| **At-Home** | Free-living recordings that contain many walking bouts with rest in between | Auto-detect the walking bouts, aggregate daily mobility outcomes (DMO), no reference comparison |
| **Lab** | Synchronised lab recordings that have a Vicon (V3D) or INDIP reference | Use the reference's own window, compare results against it |
| **Functional Test** | Short structured tests (TUG, 10-metre walk, 6-minute walk, …) | Treat the whole recording as one walking bout, no detection step |

---

## Windowing — where the pipeline looks for walking

| Option | Meaning | When |
|--------|---------|------|
| **GSD (auto-detect bouts)** | An algorithm scans the whole recording and finds the stretches that are walking | At-Home / free-living |
| **Reference window** | Use the start/end window that the mocap or INDIP reference system already defined | Lab, when you have a reference |
| **Full recording** | The entire file is treated as one continuous walking bout | Functional tests — you already know the person was walking the whole time |

---

## GSD engine — how walking bouts are found

Only relevant when Windowing is **GSD**.

| Engine | Algorithms | Notes |
|--------|-----------|-------|
| **MobGap** | `GsdIluz`, `GsdIonescu`, `GsdAdaptiveIonescu` | Signal-processing detectors. `GsdIluz` is a good default; the two Ionescu variants are tuned for impaired gait, `GsdAdaptiveIonescu` adapts its threshold to the recording. |
| **SKDH** | `PredictGaitLumbarLgbm` | A machine-learning classifier that labels 3-second windows as walking / not. |

Pick **All** to run every algorithm in the chosen engine and compare.

---

## Gait engine — how steps and gait parameters are computed

This is the biggest choice. It sets initial-contact (heel-strike) detection **and** left/right labelling,
cadence, stride length and walking speed — all together.

| Engine | What you get | Trade-off |
|--------|--------------|-----------|
| **MobGap** | Pick an IC detector (`IcdIonescu` / `IcdShinImproved` / `IcdHKLeeImproved`) and a left/right method (`LrcUllrich` / `LrcMansour` / `LrcMcCamley`) separately. Stride length uses an inverted-pendulum model. | Most flexible. Keeps detecting steps even through slow, low-amplitude walking. |
| **SKDH** | Pick a method (`AP CWT` or `Vertical CWT`). Everything — ICs, left/right, cadence, stride length, walking speed — comes from one `GaitLumbar` call. | Cannot mix in a MobGap sub-step. Its IC detector can go quiet for a few seconds during near-stationary / transition gait, which can split a walking bout in two. If you see fragmented bouts, this is usually why. |

You **can** cross engines — e.g. MobGap GSD + SKDH gait. The GSD result feeds whichever gait engine you pick.

### Fine-tuning an algorithm — the "⚙ Params" button

Next to the SKDH method row and the MobGap LRC row is a small **⚙ Params** button. It opens a popup with a
few numeric knobs for that engine — leave any field blank to use the library's own default.

| Engine | Knobs | Notes |
|--------|-------|-------|
| **SKDH** | IC prom. factor, FC prom. factor | Only affect `AP CWT` — no effect when `Vertical CWT` is selected |
| | Wavelet scale, Use CWT scale relation | Only affect `Vertical CWT` — no effect when `AP CWT` is selected |
| | Height factor | Always applies (leg length ≈ height × this factor) |
| | Max stride time | Applies to either SKDH method |
| **MobGap** | Step length scaling factor | Applies to MobGap's stride-length model |

You don't need to know the exact mechanics — the popup's own hint text next to each field says which
method it applies to. This replaced an older list of "preset" algorithm choices (e.g. "AP CWT (low-prom)")
that silently stopped working once you also tried to type a value into the same knob — pick the plain
method name and dial in the value you want here instead.

Running the same algorithm choice twice with different knob values never overwrites the first run's output
files — the filename automatically gets a short tag appended whenever a value differs from default, and
every output row also carries the actual values used as real columns, so nothing needs to be
cross-referenced by hand. In `report_app.py`, every results table shows a **Params** column right next to
Algorithm, and each tab has its own algorithm/parameter filter at the top — so comparing "default" against
a tuned run is a matter of selecting both (or just one) in that filter, in whichever tab you're looking at.

---

## Turn

Checkbox. When on, turns are detected (`TdElGohary`) and each stride/bout is labelled as turning or not.
It does **not** remove turns from the results — it only tags them. Independent of the gait engine.

---

## DMO Aggregation

Rolls the per-bout parameters up into daily mobility outcomes (median walking speed, longest bout, etc.).
Only bouts of 10 s or longer contribute. Meant for At-Home recordings; leave off for Lab / Functional Test.

---

## Evaluation

Turn on **Compare against reference** to score the results against a ground-truth system after the run
(the separate **Evaluate Results** button).

| Control | Meaning |
|---------|---------|
| **Reference type** | `Mocap (V3D TXT)` for Vicon, `INDIP` for the Mobilise-D in-lab reference system |
| **Reference Root** | Folder holding the reference files, laid out as `{root}/{subject}/…` |
| **WB overlap** | *(INDIP only)* How much a detected walking bout must overlap a reference bout to count as a match — see below |

### The "WB overlap" number

Range 0.5–1.0, default **0.8**. A detected walking bout is counted as matching a reference bout only if
they overlap by at least this fraction **of both** the detected bout's length and the reference bout's
length.

At home the default of 0.8 gives a low match count — often only a dozen or so out of 60+ reference bouts,
for either gait engine. That is expected, not a bug, and comes from a definition difference:

- The INDIP reference keeps a **turn inside one continuous bout** (walk → turn → walk is one bout).
- This pipeline follows the Mobilise-D standard, which **breaks a bout at the turn** (the short pivot
  steps get filtered out, leaving a >3 s gap that ends the bout).

So a correctly detected walking bout is systematically a bit shorter than the reference bout it lines up
with, and fails the strict 0.8 test.

**What to do:** lower it to **0.6–0.7** if you need a usable number of matched bouts to analyse. The
matches you gain are mostly cases where the pipeline captured the walking cleanly but excluded the turn —
a reasonable match to keep. The value is remembered between sessions and only affects INDIP At-Home
WB evaluation.

---

## "All" batch runs

Any engine/algorithm dropdown set to **All** expands into every combination and runs them in sequence,
writing one set of output files per combination. Use it to compare algorithms on the same data in one go.

---

For how the two engines are wired together internally, the exact filter thresholds, and the
turn-exclusion mechanics, see the developer docs:
[Pipelines](../src/ug3imu/pipelines/README.md) and
[Metrics & Evaluation](../src/ug3imu/metrics/README.md).
