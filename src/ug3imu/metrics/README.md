# `ug3imu.metrics`

Evaluates IMU-detected gait events (ICs, strides, walking bouts) against ground truth — V3D mocap in the
lab, or INDIP in-lab reference system. All matching is built on MobGap's `categorize_ic_list` /
`calculate_matched_icd_performance_metrics`, so MobGap and SKDH results are always directly comparable and
INDIP evaluation (in [`ug3imu.indip`](../indip/README.md)) reuses the exact same core logic.

[← Back to project README](../../../README.md)

## Files

| File | Purpose |
|------|---------|
| [ic_evaluation.py](ic_evaluation.py) | Initial-contact detection accuracy (precision/recall/F1, timing error) |
| [stride_evaluation.py](stride_evaluation.py) | Per-stride parameter matching + error (bias/RMSE) |
| [wb_evaluation.py](wb_evaluation.py) | Trial-level walking-bout aggregate comparison (Lab only) |
| [gsd_evaluation.py](gsd_evaluation.py) | WB-to-WB "true positive WB" matching + timeline plot (At-Home only) |

## Evaluation model

All three files follow the same shape:

1. **`process_*_trial` / `match_strides`** — evaluate one trial: match IMU output to a single reference
   file, compute per-event error columns.
2. **`batch_*_analysis`** — evaluate a whole folder: group files by 4-part trial key
   (`{subject}_{date}_{task}_{...}`), match each IMU output file to its reference, concatenate results
   across trials and algorithms.

Every `batch_*` function accepts `exclude_tags` / `include_tags` to filter files by substring — the GUI's
**Evaluate Results** button uses this to separate SKDH runs (`exclude_tags=["indip"]` when comparing to
V3D) from INDIP runs (`exclude_tags=["mocap"]`), and to scope evaluation to the currently selected task
via `include_tags=self._current_keywords`.

## IC evaluation (`ic_evaluation.py`)

```python
from ug3imu.metrics import batch_ic_analysis_multi_algo, process_ic_trial, extract_trial_key
```

- `process_ic_trial(imu_path, txt_path, imu_fs, mocap_fs=255, buffer_s=1.0, tolerance_samples=None)` —
  reads one IMU IC CSV + one V3D TXT, strips the ±1 s processing buffer (see
  [`ug3imu.mocap`](../mocap/README.md#time-alignment--the-core-problem-this-module-solves)), matches ICs
  within `tolerance_samples` (default `0.2 × imu_fs`), returns `(error_df, performance_metrics)`.
  `error_df` has `detected`, `reference`, `error_samples`, `error_s` columns; `performance_metrics` is
  MobGap's TP/FP/FN/precision/recall dict.
- `batch_ic_analysis_multi_algo(imu_folder, txt_folder, imu_fs, algorithm=None, exclude_tags=None, include_tags=None, ...)` —
  runs `process_ic_trial` across every matching IC csv in `imu_folder`, matched by trial key to a V3D
  `.txt` file. Returns `(error_all, metrics_all)` concatenated across trials/algorithms. IC-detection
  accuracy only depends on the GSD (windowing) + ICD pipeline stages, not LRC/etc (see
  [pipelines — per-stage algorithm columns](../pipelines/README.md#per-stage-algorithm-columns)), so within
  each trial only one file per `("{gsd_algorithm}_{icd_algorithm}")` combination is evaluated — read from
  the file's own `gsd_algorithm`/`icd_algorithm` columns (falls back to the full filename tag for files
  that predate them). Otherwise the same IC result would be scored once per LRC choice it happened to be
  generated alongside.
- `extract_trial_key(name)` — `"_".join(stem.split("_")[:4])`. This 4-part key convention (subject, date,
  task, device/run — the exact meaning of parts 2–4 varies by naming scheme) is how every evaluation
  function pairs an IMU output file to its reference file.

## Stride evaluation (`stride_evaluation.py`)

```python
from ug3imu.metrics import match_strides, process_stride_trial, batch_stride_analysis
```

- `match_strides(v3d_df, imu_df, tolerance_samples=20, min_overlap_ratio=0.0)` — the core matcher. Stride
  start == IC position, so this reuses `categorize_ic_list` on the `start` column directly. For each
  matched pair it emits `{param}_v3d`, `{param}_imu`, `{param}_error` for all 12 params in `_STRIDE_PARAMS`
  (`stride_duration_s, cadence_spm, stride_length_m, walking_speed_mps, stance_time_s, swing_time_s,
  step_time_s, step_length_m, single_support_s, initial_double_support_s, terminal_double_support_s,
  double_support_s`). Params absent from one side become `NaN` rather than raising — e.g. `stance_time_s`
  is `NaN` for MobGap since it doesn't compute that parameter. `is_turn`/`is_step` are carried straight
  through from the reference side's columns when present (`if flag in v3d_matched.columns`) — this is
  generic, so it works the same whether the reference is V3D (see
  [`ug3imu.mocap`](../mocap/README.md#public-api)) or INDIP (see
  [`ug3imu.indip`](../indip/README.md#loading)); INDIP has no `is_step` since it has no obstacle-step
  concept.
- **IoU overlap filter** (`min_overlap_ratio > 0.0`): after IC-position matching, pairs are further
  filtered by window overlap —
  `IoU = intersection / union` where `intersection = min(end_ref, end_imu) − max(start_ref, start_imu)`
  and `union = max(end_ref, end_imu) − min(start_ref, start_imu)`. Pairs with `IoU < min_overlap_ratio`
  are dropped. This catches cases where the *start* IC matched within tolerance but the detected stride
  window over/under-shoots the reference window. The count removed is returned in
  `performance_metrics["n_iou_removed"]`. The GUI's **Evaluate Results** button uses `min_overlap_ratio=0.8`
  and writes `n_iou_removed` into the trial's `qc/qc_summary_*.txt` block (see
  [`pipelines/qc_templates.py`](../pipelines/qc_templates.py)).
- `process_stride_trial(...)` / `batch_stride_analysis(...)` — single-trial / folder-batch wrappers around
  `match_strides`, same batching convention as the IC module. `batch_stride_analysis` additionally computes
  per-trial RMSE per parameter in the returned `summary_all`.
- `normalize_athome_stride(stride_df, imu_fs)` — converts an At-Home `*_stride.csv` to the common
  `start`/`end` (IMU frame) schema `match_strides` expects. MobGap's At-Home output already has these
  columns and passes through unchanged; SKDH's At-Home output keeps the raw GaitLumbar shape (`IC Time`,
  `stride time`, `Bout N`, ...) and is converted via
  [`pipelines.skdh_lab_pipeline.gait_lumbar_df_to_stride_df`](../pipelines/skdh_lab_pipeline.py) — the same
  conversion Lab SKDH uses before writing its own stride CSV. Used by
  [`indip.batch_stride_analysis_indip_athome`](../indip/__init__.py) — see
  ["At-Home stride/IC evaluation"](../indip/README.md#at-home-strideic-evaluation-incl-singledouble-support)
  in the indip README.

### Stride CSV columns by pipeline

| Column | MobGap | SKDH | V3D (ground truth) |
|--------|:------:|:----:|:------------------:|
| `stride_duration_s`, `cadence_spm`, `stride_length_m`, `walking_speed_mps` | ✓ | ✓ | ✓ |
| `step_time_s` | ✓ (derived post-hoc from consecutive ICs) | ✓ | ✓ |
| `stance_time_s`, `swing_time_s` | — | ✓ | ✓ |
| `step_length_m` | — | ✓ (inverted pendulum, Zijlstra & Hof 2003) | ✓ (kinematic) |
| `single_support_s`, `initial_double_support_s`, `terminal_double_support_s`, `double_support_s` | — | ✓ | ✓ (computed from raw IC/FC event times — see [`ug3imu.mocap`](../mocap/README.md#public-api)) |

MobGap's `step_time_s` isn't a native pipeline output — [`pipelines/pipeline_factory.py`](../pipelines/pipeline_factory.py)
derives it post-hoc as `(IC[i+1] − IC[i]) / fs` from `raw_ic_list_` before writing the stride CSV.

## WB (walking-bout) evaluation (`wb_evaluation.py`)

```python
from ug3imu.metrics import batch_wb_analysis_v3d, batch_wb_analysis_indip
```

Trial-level rather than event-level: multiple IMU WBs within a trial are aggregated to one row
(`_aggregate_imu_wbs` — `n_strides`/`duration_s` summed, other params stride-count-weighted mean) before
comparing to a single reference row. This is LAB-scenario evaluation only (mocap-windowed trials produce
one comparable "trial" per recording); At-Home free-living data has no per-trial ground truth to compare
against.

- `batch_wb_analysis_v3d(wb_folder, txt_folder, imu_fs, mocap_fs=255, ...)` — reference from
  `mocap_txt_to_wb_df` (see [`ug3imu.mocap`](../mocap/README.md)).
- `batch_wb_analysis_indip(wb_folder, mat_path, imu_fs, ...)` — reference from
  `load_indip_cwp_map` (see [`ug3imu.indip`](../indip/README.md)); uses `ContinuousWalkingPeriod`, not
  `MicroWB` — MicroWB has no QC applied, so CWP is the reference used everywhere in this codebase.
  Reference rows are matched by task name (3rd component of the trial key) rather than trial key
  directly, then aggregated the same way. `stride_duration_s` in the reference row is the arithmetic
  mean of the bout's own `Stride_Duration` array, not `60/StrideFrequency` — the latter works out to a
  harmonic mean under Mobilise-D's own `StrideFrequency` definition (`mean(60/Stride_Duration_k)`), which
  is biased low relative to the arithmetic mean, more so in bouts with variable stride durations.

Both return `(error_df, rmse_df)` — `error_df` has one row per matched trial with `_ref/_imu/_error`
(or `_indip` in the INDIP case) columns per `_WB_PARAMS`; `rmse_df` adds TP/FP/FN/precision/recall/F1 via
`_perf_row` (in LAB mode: TP=1 if any IMU WBs exist, FN=1 if none, FP=0 — there's no "spurious extra
trial" case to count as FP).

## GSD / WB evaluation for At-Home (`gsd_evaluation.py`)

```python
from ug3imu.metrics.gsd_evaluation import batch_gsd_analysis_indip, plot_gsd_wb_timeline
```

At-Home has no per-task trial structure — one subject's recording yields dozens of independent WBs from
both the algorithm and INDIP's own CWP list, so `wb_evaluation.py`'s trial-aggregate approach doesn't
apply. Instead:

- `normalize_athome_wb(wb_df, imu_fs)` — converts either pipeline's `*_wb.csv` (MobGap: `start`/`end` in
  samples; SKDH: `Bout Starts`/`Bout Duration` in seconds — both share the same t=0 = recording start
  reference as INDIP) into a common `start_s`/`end_s`/`duration_s`/... schema.
- Matching/aggregation is done with **mobgap's own** `mobgap.gait_sequences.evaluation` functions directly
  (not hand-rolled overlap or sum calculations), at three levels:
  - `_match_wb_bouts` (via mobgap's `categorize_intervals` + `get_matching_intervals`) — bout-to-bout
    matching for the gait-parameter comparison. A (reference, detected) pair matches if their overlap
    covers >= `overlap_threshold` (default 0.8) of *both* sides' own duration (mobgap's own bidirectional
    rule) — real-world bout-count match rates in the 20-40% range are expected, not a bug, since fragmented
    or oversized WBs on either side fail this threshold easily.
  - `_gsd_sample_level_metrics` (via mobgap's `categorize_intervals_per_sample` +
    `calculate_matched_gsd_performance_metrics`) — every timestamp independently classified tp/fp/fn/tn,
    giving precision/recall/f1/specificity/accuracy/npv that measure how much real walking *time* got
    captured regardless of bout-boundary disagreement. Usually far more forgiving (and more informative)
    than the bout-count match rate above. `n_overall_samples` (needed for tn/specificity/accuracy/npv) is
    approximated as the last WB end time across both systems.
  - `_gsd_unmatched_metrics` (via mobgap's `calculate_unmatched_gsd_performance_metrics`) — no matching at
    all, just total WB count and total walking-time duration on each side plus their error/relative error/
    log-error (`reference_num_gs`/`detected_num_gs`/`reference_gs_duration_s`/`detected_gs_duration_s`/
    `gs_duration_error_s`/`num_gs_error`/...). This replaced an earlier hand-written `.sum()`/`len()` version.
- `batch_gsd_analysis_indip(wb_folder, mat_path, imu_fs, overlap_threshold=0.8, ...)` — reference from
  `load_indip_cwp_athome` (see [`ug3imu.indip`](../indip/README.md); at-home INDIP data is a struct
  *array* of independent bouts, not the Lab per-task struct). Uses the `ContinuousWalkingPeriod` (CWP)
  bout list, **not** `MicroWB` — CWP is the coarser definition that bridges short gaps MicroWB splits on.
  Returns `(error_all, metrics_all)` — **labeled by different algorithm identities**, because they depend
  on different parts of the pipeline (see
  [pipelines — per-stage algorithm columns](../pipelines/README.md#per-stage-algorithm-columns)):
  - `error_all` — one row per bout-level matched WB pair with `{param}_ref/_imu/_error` (same schema as
    `wb_evaluation.py`'s output, so it's written to the same `wb_error_indip_*.csv` filename and consumed
    by the existing Walking Bouts tab unmodified). `algorithm` is the **full** `{gsd}_{icd}_{lrc}` pipeline
    name — every `*_wb.csv` file is evaluated, since per-bout parameter values depend on the whole chain.
  - `metrics_all` — one row per trial × **GSD algorithm** (not the full pipeline name) combining all three
    functions above: bout-level `tp_wb`/`fp_wb`/`fn_wb`; unmatched totals `reference_num_gs`/
    `detected_num_gs`/`reference_gs_duration_s`/`detected_gs_duration_s`/... (mobgap's own naming); and
    sample-level `tp_samples`/`fp_samples`/`fn_samples`/`precision`/`recall`/`f1_score`/`tn_samples`/
    `specificity`/`accuracy`/`npv` (same `tp_samples`/`fp_samples`/`fn_samples` convention as
    `ic_metrics_*.csv` — genuinely sample counts this time, safe to sum across trials — reused by
    `aggregate_ic_results._detection_summary()` for the GSD Detection tab's sample-level panel). Bout
    boundaries don't depend on ICD/LRC, so only one `*_wb.csv` file per `(trial, gsd_algorithm)` is used
    here — evaluating every file the way `error_all` does would score the same GSD result once per ICD/LRC
    combination it was tested alongside, and the GSD Detection tab would show far more "algorithms" than
    were actually tested.
- `plot_gsd_wb_timeline(wb_folder, mat_path, imu_fs, plot_dir, ...)` — saves one
  `{trial}_{algorithm}_gsd_wb.png` per algorithm found in `wb_folder`, using
  [`ug3imu.plotting.plot_wb_timeline`](../plotting/README.md)'s `ref_wb_intervals` overlay to show the
  algorithm's WBs and INDIP's CWPs on a shared time axis. Generated during evaluation (not at
  pipeline-run time) because that's the first point where both the algorithm's `wb.csv` and the INDIP
  `.mat` path are available together.

## Where this is consumed

The GUI's **Evaluate Results** button (both MobGap and SKDH tabs, Lab preset) in
[`scripts/imu_pipeline.py`](../../../scripts/imu_pipeline.py) calls all three `batch_*` families — once
per detected algorithm, against whichever reference type (Mocap V3D or INDIP) is selected — and writes
results to `{Output Root}/{Subject}/{Device}/Evaluation/`. For At-Home tasks (task keyword `outoflab`),
the same button instead calls `indip.batch_ic_analysis_indip_athome` +
`indip.batch_stride_analysis_indip_athome` + `batch_gsd_analysis_indip` + `plot_gsd_wb_timeline`. All four
At-Home output files use a fixed `outoflab` suffix instead of the task label — `ic_error_indip_outoflab.csv`,
`stride_error_indip_outoflab.csv`, `wb_error_indip_outoflab.csv`, `gsd_metrics_indip_outoflab.csv`, ... —
since At-Home matches every IC/stride/WB file against one continuous per-subject reference rather than a
per-task one (see
[`ug3imu.indip`](../indip/README.md#at-home-strideic-evaluation-incl-singledouble-support)).
