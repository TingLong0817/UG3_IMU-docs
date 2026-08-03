# `ug3imu.indip`

Everything needed to use **INDIP** (the Mobilise-D in-lab reference system) as an alternative to V3D
mocap: parsing INDIP `.mat` files, extracting per-task windows and reference events, and batch evaluation
against IMU output. Mirrors [`ug3imu.mocap`](../mocap/README.md) + parts of
[`ug3imu.metrics`](../metrics/README.md) so V3D and INDIP are interchangeable reference systems from the
GUI's point of view — same core matching logic (`mobgap.initial_contacts.evaluation.categorize_ic_list`
and [`metrics.stride_evaluation.match_strides`](../metrics/stride_evaluation.py)), different file format.

[← Back to project README](../../../README.md)

## Files

| File | Purpose |
|------|---------|
| [\_\_init\_\_.py](__init__.py) | Everything — loader, windowing, discovery, batch evaluation (single file, ~630 lines; not split further because every function shares the same `.mat` struct-navigation helpers) |

Requires `scipy` (`.mat` parsing via `scipy.io.loadmat`) — raises `ImportError` at call time if missing,
not at import time, so the rest of the package works without `scipy` installed.

## The INDIP `.mat` structure

```
mat["data"].TimeMeasure1
  .{test_name}                              # e.g. "Test5"
    .{trial_name}                           # e.g. "Trial2"
      .testName                             # e.g. "28MWA_" → stripped to "28MWA"
      .Standards.INDIP
        .MicroWB                            # walking-bout-level struct arrays
          .Start, .End, .Duration, .NumberStrides, .Cadence,
          .WalkingSpeed, .AverageStrideLength, .StrideFrequency
        .ContinuousWalkingPeriod (CWP)       # stride-level struct
          .Start, .End                       # window bounds, seconds
          .Stride_InitialContacts             # (N, 2) start/end IC pairs, 100 Hz frames
          .Stride_Duration, .Stride_Length, .Stride_Speed,
          .Stance_Duration, .Swing_Duration
          .SingleSupport_Duration, .DoubleSupport_Duration  # present but NOT used directly — see below
          .Step_Duration                      # different length than Stride_Duration, not 1:1 with strides
          .InitialContactEvent               # ALL IC timestamps (seconds) — authoritative IC list
          .InitialContact_LeftRight          # label per InitialContactEvent entry
          .FinalContact_Event, .FinalContact_LeftRight       # ALL FC (toe-off) timestamps + labels
```

`infoForCalibration` under `TimeMeasure1` is skipped everywhere — it's metadata, not a trial. Everything
else is defensive (`hasattr(..., "_fieldnames")`, `getattr(..., None)` checks) because not every trial has
an INDIP standard, and not every INDIP trial has both `MicroWB` and `ContinuousWalkingPeriod`.

**Sample rate**: INDIP frame indices (`Stride_InitialContacts`, IC events) are at a fixed **100 Hz**,
independent of the external IMU's sampling rate — every function that returns frame indices takes
`imu_fs` and rescales via `scale = imu_fs / 100.0`.

## Public API

```python
from ug3imu.indip import (
    load_indip_mat, load_indip_ic_map, load_indip_microwb_map,
    extract_indip_ic_df, get_cwp_windows_s, get_indip_windows,
    build_frame_windows_for_files, find_indip_mat,
    batch_ic_analysis_indip, batch_stride_analysis_indip,
)
```

### Loading

| Function | Returns | Source struct |
|----------|---------|----------------|
| `load_indip_mat(mat_path)` | `{task_name: stride_df}` — `start`, `end` (100 Hz frames), `lr_label`, stride params incl. `single_support_s`/`initial_double_support_s`/`terminal_double_support_s`/`double_support_s` | `ContinuousWalkingPeriod` |
| `load_indip_ic_map(mat_path)` | `{task_name: DataFrame[ic, lr_label]}` — **authoritative** full IC list | `CWP.InitialContactEvent` + `InitialContact_LeftRight` |
| `load_indip_microwb_map(mat_path)` | `{task_name: microwb_df}` — one row per walking bout | `MicroWB` |
| `extract_indip_ic_df(stride_df, imu_fs)` | `DataFrame[ic, lr_label]` reconstructed from stride start/end | Fallback only — use `load_indip_ic_map` when possible; this approximates the IC list from stride boundaries when `InitialContactEvent` isn't usable |

`lr_label` resolution in `_cwp_to_stride_df` is nontrivial: each stride's start IC frame is matched
against `InitialContactEvent` (nearest within 2 frames) to look up its `InitialContact_LeftRight` label,
because stride order in `Stride_InitialContacts` isn't guaranteed to align positionally with
`InitialContact_LeftRight`. There's a direct-index fallback for older files where that assumption held.

**Support-phase columns are computed from raw events, not from `SingleSupport_Duration` /
`DoubleSupport_Duration`.** Those two CWP fields exist and are 1:1-aligned with `Stride_Duration`, but on
real data `SingleSupport_Duration` turned out to be ~2× the V3D per-side `single_support_s` value —
apparently the combined left+right single-support time for the full gait cycle, not "this stride's own
side" the way V3D/SKDH report it (`DoubleSupport_Duration` *did* match V3D's scale, for what it's worth,
since double support is inherently a whole-cycle metric already — see
[`ug3imu.mocap`](../mocap/README.md#public-api)). Rather than trust an INDIP-specific convention that
doesn't line up with the V3D-referenced comparisons this same file feeds, `_cwp_to_stride_df` reuses
`mocap._support_phase_durations` directly on INDIP's own `InitialContact_Event`/`InitialContact_LeftRight`
+ `FinalContact_Event`/`FinalContact_LeftRight` (per-side IC/FC arrays sorted and fed through the same
formula as V3D). Each stride's computed value is looked up by matching its start frame back against the
per-side event array (nearest match, 2-frame tolerance — same pattern as the `lr_label` lookup above).
This also fills in `initial_double_support_s`/`terminal_double_support_s`, which INDIP has no native split
for (only the combined `DoubleSupport_Duration`). `Step_Duration` is left unused for `step_time_s` for a
different reason — its length doesn't match `Stride_Duration` 1:1 (steps ≠ strides), so there's no safe
positional alignment; `step_time_s`/`step_length_m` stay V3D-only.

### Windowing (mirrors `mocap.parse_mocap_frame_window`)

| Function | Returns |
|----------|---------|
| `get_cwp_windows_s(mat_path)` | `{task_name: (start_s, end_s)}` — raw, **unpadded** `CWP.Start`/`CWP.End` in seconds |
| `get_indip_windows(mat_path, imu_fs)` | `{task_name: (start_frame, end_frame)}` in IMU sample space — `frame = round(seconds × imu_fs)`, **±1 s padded** on each side |
| `build_frame_windows_for_files(file_list, task_windows)` | `{key4: (start, end)}` — maps each file's 3rd stem component (task name) to its window, for passing as `frame_windows=` to `dataset_generation.build_dataset_from_file_list` |

`CWP.Start`/`CWP.End` coincide exactly with the first/last IC of the walking period — there's no built-in
lead-in or lead-out the way a V3D mocap-crop window has. Running detection right up against that boundary
gives the algorithm no signal context at the edges, so `get_indip_windows` pads ±1 s on each side, mirroring
`parse_mocap_frame_window`'s V3D buffer. That padding is processing-only (no INDIP ground truth exists
there), so it's trimmed back out in two places: `batch_ic_analysis_indip` (via its `buffer_s` parameter,
default `1.0`, same mechanism as [`ic_evaluation.process_ic_trial`](../metrics/README.md#ic-evaluation-ic_evaluationpy))
and the IC overlay plot (`plot_ic_overlay_lab`'s default `buf_samples` strip). Stride/WB evaluation
(`batch_stride_analysis_indip`, `batch_wb_analysis_indip`) do **not** trim the padding — matching V3D's
existing behavior, where only IC-level evaluation applies the edge trim.

### Discovery

`find_indip_mat(ref_root, subject)` — locates the `.mat` file for a subject, checking in order:
`{ref_root}/{subject}/INDIP/*Inlab.mat` → `{ref_root}/{subject}*Inlab.mat` → `{ref_root}/{subject}/*.mat`
→ `{ref_root}/{subject}*.mat`. Mirrors the V3D subfolder convention so both reference types can live under
the same `Reference Root` setting in the GUI.

### Batch evaluation

| Function | Mirrors | Reference source |
|----------|---------|-------------------|
| `batch_ic_analysis_indip(ic_folder, mat_path, imu_fs, ...)` | [`metrics.ic_evaluation.batch_ic_analysis_multi_algo`](../metrics/ic_evaluation.py) | `load_indip_ic_map` (falls back to `extract_indip_ic_df` if the event field is absent) |
| `batch_stride_analysis_indip(stride_folder, mat_path, imu_fs, ...)` | [`metrics.stride_evaluation.batch_stride_analysis`](../metrics/stride_evaluation.py) | `load_indip_mat` |

Both take the same `algorithm` / `exclude_tags` / `include_tags` filters as their V3D counterparts, and
`batch_stride_analysis_indip` calls [`metrics.stride_evaluation.match_strides`](../metrics/stride_evaluation.py)
directly (including the same IoU overlap filter via `min_overlap_ratio`) — the only difference from the
V3D path is where the reference DataFrame comes from and that error columns are renamed `_v3d` → `_indip`
for output clarity.

### At-Home stride/IC evaluation (incl. single/double support)

At-Home's `*Outoflab.mat` `ContinuousWalkingPeriod` is an array of per-bout structs (one per detected
walking bout), but **each bout element carries the same rich per-stride fields as a Lab trial's CWP
struct** — `Stride_InitialContacts`, `Stride_Duration`, `SingleSupport_Duration`/`DoubleSupport_Duration`,
`InitialContact_Event`/`InitialContact_LeftRight`, `FinalContact_Event`/`FinalContact_LeftRight`, etc.,
all in absolute time within the whole continuous recording. `load_indip_cwp_athome` (above) only reads
each bout's scalar aggregates and discards these per-stride arrays — the functions below read them.

| Function | Mirrors | Reference source |
|----------|---------|-------------------|
| `load_indip_stride_athome(mat_path)` | `load_indip_mat` | Walks every bout via `_iter_athome_cwp_bouts` and runs the existing `_cwp_to_stride_df` on each element unmodified, concatenated into **one continuous stride table** for the whole recording (incl. the 4 support columns) |
| `load_indip_ic_athome(mat_path)` | `load_indip_ic_map` | Same bout walk, using `_cwp_to_ic_df` per bout — one continuous IC table |
| `batch_stride_analysis_indip_athome(stride_folder, mat_path, imu_fs, ...)` | `batch_stride_analysis_indip` | `load_indip_stride_athome` — **no task-name keying**: every `*_stride.csv` matches against the same continuous reference (mirrors `batch_gsd_analysis_indip`'s at-home matching pattern, not the Lab per-task one) |
| `batch_ic_analysis_indip_athome(ic_folder, mat_path, imu_fs, ...)` | `batch_ic_analysis_indip` | `load_indip_ic_athome` — same no-keying pattern; no `buffer_s` edge trim (that trim strips the padded-window margin around each Lab per-task crop window, which At-Home doesn't have) |

SKDH's At-Home `*_stride.csv` keeps the raw GaitLumbar shape (`IC Time`, `stride time`, `Bout N`, ...) with
no `start`/`end` columns yet, unlike the Lab SKDH pipeline which converts before writing. `batch_stride_analysis_indip_athome`
runs every stride file through [`metrics.stride_evaluation.normalize_athome_stride`](../metrics/stride_evaluation.py)
first, which passes MobGap files through unchanged and converts SKDH files via
[`pipelines.skdh_lab_pipeline.gait_lumbar_df_to_stride_df`](../pipelines/skdh_lab_pipeline.py) (the same
conversion the Lab SKDH pipeline uses, factored out for reuse).

## Where this is consumed

The GUI's **Evaluate Results** button in [`scripts/imu_pipeline.py`](../../../scripts/imu_pipeline.py)
calls `find_indip_mat` + `batch_ic_analysis_indip` + `batch_stride_analysis_indip` +
[`metrics.wb_evaluation.batch_wb_analysis_indip`](../metrics/wb_evaluation.py) when **Reference: INDIP** is
selected (as an alternative to Mocap V3D). `get_indip_windows` / `build_frame_windows_for_files` feed into
dataset construction so INDIP-referenced trials use the same lab-windowing code path as V3D-referenced
ones — see [`ug3imu.pipelines`](../pipelines/README.md).

For At-Home runs (task keyword `outoflab`), `_eval_common` swaps in `find_indip_mat_athome` +
`batch_ic_analysis_indip_athome` + `batch_stride_analysis_indip_athome` +
[`metrics.gsd_evaluation.batch_gsd_analysis_indip`](../metrics/gsd_evaluation.py) instead. All four At-Home
output files — `ic_error_indip_outoflab.csv`, `ic_metrics_indip_outoflab.csv`,
`stride_error_indip_outoflab.csv`, `stride_rmse_indip_outoflab.csv`, plus `wb_error_indip_outoflab.csv` /
`gsd_metrics_indip_outoflab.csv` (see [`ug3imu.metrics`](../metrics/README.md)) — use a fixed `outoflab`
suffix rather than the task label: one subject has one continuous At-Home recording matched as a whole
against a single reference, so (unlike the Lab path's `..._{task_lbl}.csv` files, where `task_lbl` is
meaningful) there's nothing per-task to key by.
