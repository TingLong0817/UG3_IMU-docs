# `scripts/long_format_exports/`

Small standalone utility (not part of the `ug3imu` library) that reshapes a subject's
`Evaluation/` CSVs into R-friendly long format — one row per device (`mocap`/`indip` vs. `ax6`)
per stride/bout/trial, instead of the evaluation pipeline's paired `_v3d`/`_imu`/`_ref` columns.
Built for an ad-hoc P04 data request; kept here since the same reshaping will likely be needed
for other subjects.

[← Back to project README](../../README.md)

## Usage

```bash
python export_long_format.py --eval_dir "R:\...\Results\P04\AX6\Evaluation" --subject P04 --out_dir "C:\UG3_IMU\UG3_IMU\testdata"

# Run a single exporter instead of all ten:
python export_long_format.py --eval_dir ... --subject P04 --out_dir ... --only stride_lab
```

`--eval_dir` is the subject's `{Output Root}/{Subject}/{Device}/Evaluation/` folder (the folder
`imu_pipeline.py`'s **Evaluate Results** button writes into). Missing source files are skipped with a
`[skip]` message rather than raising — not every subject has out-of-lab data, for example.

Every output row also carries a `sensor` column (`AX6`/`OPAL`/`DP7`) — the physical recording device,
inferred automatically from `--eval_dir`'s parent folder name, not passed separately. Don't confuse this
with the `device` column `_melt_devices` adds to the stride/WB exports — that one means "which measurement
system" (the `mocap`/`indip` reference vs. the `ax6`-labeled IMU algorithm output row), a different axis
from which physical sensor recorded the data.

## What each exporter does

Output filenames are all `{metric}_{ref}_{scenario}_long_{subject}.csv` — no `error` in the name (these
are the reshaped long-format files, not the pipeline's own `_error`/`_rmse` csvs), and `scenario` is only
ever `lab` or `outoflab` (matching the fixed suffix the evaluation pipeline itself writes — see
[`ug3imu.indip`](../../src/ug3imu/indip/README.md#at-home-strideic-evaluation-incl-singledouble-support)).

| Function | Source CSV | Output | Rows are... |
|----------|-----------|--------|--------------|
| `export_stride_lab` | `stride_error_v3d_lab.csv` | `stride_v3d_lab_long_{subject}.csv` | one device's params for one matched stride |
| `export_wb_lab` | `wb_error_v3d_lab.csv` | `wb_v3d_lab_long_{subject}.csv` | one device's params for one matched in-lab walking bout (1 per trial) |
| `export_ic_metrics_lab` | `ic_metrics_v3d_lab.csv` | `ic_v3d_lab_long_{subject}.csv` | one trial × algorithm's IC detection performance (already tidy, no device split) |
| `export_stride_indip_lab` | `stride_error_indip_lab.csv` | `stride_indip_lab_long_{subject}.csv` | same as `export_stride_lab`, but for a Lab run evaluated against INDIP instead of V3D — has `is_turn`, no `is_step` (INDIP has no obstacle-step concept) |
| `export_wb_indip_lab` | `wb_error_indip_lab.csv` | `wb_indip_lab_long_{subject}.csv` | same as `export_wb_lab`, but against INDIP |
| `export_ic_metrics_indip_lab` | `ic_metrics_indip_lab.csv` | `ic_indip_lab_long_{subject}.csv` | same as `export_ic_metrics_lab`, but against INDIP |
| `export_stride_outoflab` | `stride_error_indip_outoflab.csv` | `stride_indip_outoflab_long_{subject}.csv` | one device's params for one matched out-of-lab stride, incl. `single_support_s`/`initial_double_support_s`/`terminal_double_support_s`/`double_support_s` — has `is_turn`, no `is_step` (INDIP has no obstacle-step concept) |
| `export_ic_metrics_outoflab` | `ic_metrics_indip_outoflab.csv` | `ic_indip_outoflab_long_{subject}.csv` | one trial × algorithm's IC detection performance for out-of-lab (already tidy, no device split) |
| `export_wb_outoflab` | `wb_error_indip_outoflab.csv` | `wb_indip_outoflab_long_{subject}.csv` | one device's params for one matched out-of-lab walking bout |
| `export_gsd_outoflab` | `gsd_metrics_indip_outoflab.csv` | `gsd_indip_outoflab_long_{subject}.csv` | one session × algorithm's sample-level GSD detection performance (bout-count/unmatched-total columns intentionally dropped) |

Column-by-column meaning, the `_v3d`/`_imu`/`_ref` naming convention, and important caveats about
matched-vs-all-detected data are documented in `testdata/*_README.md` and
`testdata/P04_evaluation_notes.md` (generated alongside the first P04 export — read those before
trusting a comparison built on this output).

## Design notes (carried over from the P04 conversation)

- **`bout`** is the task code only (e.g. `CARPWA`), extracted from the pipeline's `trial` string
  (`{subject}_{date}_{task}_{run}`) by taking the 3rd underscore-delimited part — subject/date/run are
  stripped so the label is reusable across subjects and sessions. Out-of-lab data has no per-task
  structure, so `bout` is currently always `OutofLab` there.
- **`stride_number` / `wb_number`** re-pair the two device rows produced from one matched pair (ordered
  by the reference device's start time within each `bout` + `algorithm` group) — added back after an
  earlier revision dropped all position/index columns and turned out to need *some* join key.
- **No `_error` columns** in the stride/WB exports — compute error yourself by pivoting `mocap`/`indip`
  vs `ax6` rows back out (`pivot_wider` in R, keyed on `subjectID`+`bout`+`algorithm`+`stride_number`/`wb_number`).
- **Blank parameter cells are expected**, not missing data — they mean the specific algorithm on that
  row doesn't compute that gait parameter (e.g. MobGap-style IC algorithms don't produce
  `stance_time_s`/`swing_time_s`/`step_length_m`). Reference (`mocap`/`indip`) rows are always complete.
- **`algorithm` should be a bare algorithm name** (`IcdIonescu`, `skdh_apcwt`, `GsdIluz_IcdIonescu_LrcUllrich`,
  ...) — if a source Evaluation csv still has device/sync/reftype tokens glued onto the front
  (`ax6_sync_indip_IcdIonescu`), it was written before the algorithm-name parsing fix in
  `ug3imu.indip`/`ug3imu.metrics.*`; re-run **Evaluate Results** in the GUI to regenerate it before
  exporting.
