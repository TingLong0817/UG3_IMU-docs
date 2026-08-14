# `ug3imu.mocap`

Parses Vicon Nexus V3D TXT exports and reconciles the mocap time reference with the IMU sample-index
reference, so lab-validation pipelines can compare IMU output against mocap ground truth without any
further offset correction.

[← Back to project README](../../../README.md)

## Files

| File | Purpose |
|------|---------|
| [mocap_utils.py](mocap_utils.py) | V3D TXT parsing, frame-window conversion, IC/stride/step extraction |

## The V3D TXT format

V3D exports are tab-separated, with the real header on the **second** line (row 0 is a title row) and
non-numeric trailer rows at the bottom. Every function here re-parses the file the same way:

```python
df_raw = pd.read_csv(txt_path, sep="\t", header=1)
# first column may or may not already be named "ITEM"
df_raw["ITEM_num"] = pd.to_numeric(df_raw["ITEM"], errors="coerce")
df = df_raw[df_raw["ITEM_num"].notna()].copy()   # drop trailer rows
```

Relevant columns (not all present in every export):

| Column | Meaning |
|--------|---------|
| `Cropped Measurement Start/End Frame` | Mocap frame bounds of the validated window |
| `RIC`, `LIC` (legacy: `RHS`, `LHS`) | Right/left heel-strike (initial contact) times, in **seconds** |
| `RFC`, `LFC` (legacy: `RTO`, `LTO`) | Right/left toe-off (final contact) times |
| `Right_*`, `Left_*` | Per-side stride parameters (cadence, stride length, stance/swing time, …) |
| `Cycle_Time_Count`, `Cycle_Time_Mean`, `Stride_Length_Mean`, `Speed`, `*_Steps_Per_Minute_Mean` | Trial-level (bilateral) summary stats |

## Time alignment — the core problem this module solves

Three time axes need to be reconciled:

| System | Reference | Unit |
|--------|-----------|------|
| IMU recording | Sample 0 = recording start | Frame index |
| V3D mocap | Relative to *Cropped Measurement Start* | Seconds from crop start |
| SKDH `GaitLumbar` | Inherits from the `time` array passed in | Seconds |

`parse_mocap_frame_window` reads `Cropped Measurement Start/End Frame`, converts to IMU frame indices via
`ratio = imu_fs / mocap_fs`, and pads ±1 s on each side as a processing buffer:

```python
start_frame_imu = round(start_frame * ratio) - imu_fs
end_frame_imu   = round(end_frame   * ratio) + imu_fs
```

MobGap consumes these as `DummyGSD` window boundaries (see [`pipelines/lab_pipeline.py`](../pipelines/lab_pipeline.py));
SKDH consumes an absolute time array built from the same boundaries:

```python
time_crop = np.arange(start_frame, end_frame) / sampling_rate_hz  # absolute seconds
```

Both outputs land in the same reference frame as `mocap_txt_to_ic_df`'s output, so downstream evaluation
code ([`ug3imu.metrics`](../metrics/README.md)) can compare IMU and mocap events directly. The ±1 s buffer
is trimmed back out during evaluation so edge detections near the crop boundary aren't counted as false
positives/negatives.

## Public API

```python
from ug3imu.mocap import (
    parse_mocap_frame_window, mocap_txt_to_ic_df,
    mocap_txt_to_stride_df, mocap_txt_to_wb_df,
)
```

| Function | Returns | Notes |
|----------|---------|-------|
| `parse_mocap_frame_window(txt_path, mocap_fs, imu_fs)` | `(start_frame_imu, end_frame_imu)` | Falls back to min/max of `RHS/RTO/LHS/LTO` if `Cropped Measurement *` columns are absent |
| `mocap_txt_to_ic_df(txt_path, mocap_fs, imu_fs)` | `DataFrame[ic, lr_label]` | One row per heel strike (RHS + LHS merged), `ic` in IMU frame index |
| `mocap_txt_to_stride_df(txt_path, mocap_fs, imu_fs)` | `DataFrame[start, end, lr_label, stride_duration_s, cadence_spm, stride_length_m, walking_speed_mps, stance_time_s, swing_time_s, step_time_s, step_length_m, single_support_s, initial_double_support_s, terminal_double_support_s, double_support_s, is_turn, is_step]` | A stride = two consecutive same-side heel strikes (`RIC[i] → RIC[i+1]`); `step_time_s`/`step_length_m` come from the same per-side V3D columns a dedicated per-step extractor would use, so there's no separate step-level function |
| `mocap_txt_to_wb_df(txt_path, mocap_fs)` | One-row `DataFrame` | Trial-level bilateral means, shaped to match IMU WB output columns |

`mocap_txt_to_stride_df` pulls most per-stride values from the `_STRIDE_METRICS` column map (right/left →
V3D column name). The four support-phase columns are the exception: V3D only exports
`Right_Initial/Terminal_Double_Limb_Support_Time` (nothing for Left), but double support is a
whole-gait-cycle event, not a side-specific one, so instead of reading those columns
`_support_phase_durations` computes all four **directly from the raw RIC/LIC/RFC/LFC event times**, the
same formula for both sides:

```
initial_double_support_s = own_ic[i]  -> first opposite-foot FC after own_ic[i]
single_support_s         = that FC    -> first opposite-foot IC after it
terminal_double_support_s= that IC    -> first own-foot FC after it
double_support_s         = initial_double_support_s + terminal_double_support_s
```

Verified against V3D's own `Right_Initial/Terminal_Double_Limb_Support_Time` /
`Right_Single_Limb_Support_Time_Manual` columns on real data (exact match), which also confirmed the
left/right timing relationship: a given stride's `initial_double_support_s` equals the *previous*
opposite-side stride's `terminal_double_support_s`, and vice versa — i.e. adjacent left/right strides
share double-support periods, they just get attributed to whichever stride's window contains them.
[`ug3imu.indip`](../indip/README.md)'s `_cwp_to_stride_df` reuses this exact function on INDIP's own raw
IC/FC events, so V3D and INDIP report these four metrics on the same footing.

### `is_turn` / `is_step` labeling

Both are computed by **time-interval overlap**, not by matching a stride's boundary IC against a discrete
event timestamp:

- `is_turn` — `Pelvis_Turn_On`/`Pelvis_Turn_Off` are paired, row-aligned columns giving the `[start, end]`
  time (seconds) of each turn, one row per turn. Not per-side — a turn is a whole-body pelvis-rotation
  event.
- `is_step` — same shape, from `step_start`/`step_end` (obstacle-crossing intervals; present only in the
  obstacle tasks, e.g. STEPWA/STEPWB).

A stride is flagged if its own `[IC_i, IC_i+1]` span overlaps *any* such interval — `_paired_intervals`
builds the `(start, end)` list (dropping unmatched rows, e.g. a turn that started but has no recorded end
because the trial was cut mid-turn), `_overlaps_any` does the inclusive-bounds overlap check. No
left/right distinction. This replaced an earlier approach of matching a stride's boundary IC against
discrete `RIC_TURN`/`LIC_TURN` or `RIC_STEP`/`LIC_STEP` timestamps — those columns are still present in
some exports but no longer read. **Files without the new columns get `is_turn`/`is_step` = `False` for
every stride** — there's no fallback to the old columns.

[`ug3imu.indip`](../indip/README.md)'s `_cwp_to_stride_df` uses the same overlap rule for its own
`is_turn` (from INDIP's `Turn_Start`/`Turn_End`); INDIP has no obstacle-step concept, so it has no
`is_step`.

## Where this is consumed

- [`pipelines/dataset_generation.py`](../pipelines/dataset_generation.py) / [`pipelines/pipeline_factory.py`](../pipelines/pipeline_factory.py) — `parse_mocap_frame_window` for lab windowing (MobGap `DummyGSD`, SKDH crop array)
- [`pipelines/skdh_lab_pipeline.py`](../pipelines/skdh_lab_pipeline.py) — same, for the SKDH lab path
- [`metrics/ic_evaluation.py`](../metrics/ic_evaluation.py) — `mocap_txt_to_ic_df` as the IC reference
- [`metrics/stride_evaluation.py`](../metrics/stride_evaluation.py) — `mocap_txt_to_stride_df` as the stride reference
- [`metrics/wb_evaluation.py`](../metrics/wb_evaluation.py) — `mocap_txt_to_wb_df` as the trial-level WB reference

For the equivalent reference-system module when using INDIP `.mat` files instead of V3D TXT, see
[`ug3imu.indip`](../indip/README.md).
