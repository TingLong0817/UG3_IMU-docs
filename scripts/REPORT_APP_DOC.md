# Report App Reference

## Quick orientation

Main file: `scripts/report_app.py`
Dependency: `scripts/aggregate_ic_results.py` (provides `_STRIDE_PARAMS`, `_add_f1`, `_detection_summary`, `_icc21`)
Run: `streamlit run scripts/report_app.py`

Sibling app: `scripts/report_app_indipvsmocap.py` — validates INDIP itself against V3D mocap (rather than
an IMU algorithm against a reference system); see "INDIP vs Mocap validation" at the end of this doc.

---

## Evaluation output file layout

Evaluation files live at:
```
{out_root}/{subject}/{device}/Evaluation/
```

### Filename convention

```
{metric_type}_{ref_suffix}_{scenario}.csv
```

| Field | Values | Meaning |
|---|---|---|
| `metric_type` | `ic_error`, `ic_metrics`, `stride_error`, `stride_rmse`, `wb_error`, `wb_rmse`, `gsd_metrics` | Evaluation type |
| `ref_suffix` | `v3d`, `indip` | Reference system: v3d = mocap (V3D TXT); indip = INDIP `.mat` file |
| `scenario` | `lab`, `outoflab` | Collection setting. `lab` comes from `_task_label()` (the lowercased task name, could be any Lab task); **At-Home is always the fixed literal `outoflab`**, regardless of what the task is actually named in `task_config.json` |

Examples: `ic_metrics_v3d_lab.csv`, `wb_error_indip_outoflab.csv`, `stride_error_indip_outoflab.csv`

**Why At-Home always uses `outoflab` instead of the task name**: a subject has only one continuous
at-home recording, and all four At-Home evaluations (IC/stride/WB/GSD) match against that same
un-split-by-task INDIP reference (see "At-Home / GSD notes" below) — so there's no reason, and no
benefit, to split by `task_lbl` the way Lab does. A fixed `outoflab` suffix avoids the same at-home
data getting scattered across two different scenarios in report_app just because a task got renamed.

---

## File column reference

### `ic_metrics_{ref}_{scenario}.csv` (IC detection metrics, one row per trial × algo)
```
trial, algorithm, TP, FP, FN, n_detected, n_reference, precision, recall, f1
```

### `ic_error_{ref}_{scenario}.csv` (IC timing error, one row per matched IC pair)
```
detected, reference,        ← absolute IMU frame index
error_samples, error_s,     ← timing difference (error_s in seconds)
trial, algorithm
```

### `stride_error_{ref}_{scenario}.csv` (stride parameter error, one row per matched stride pair)
```
lr_label,
start_{ref}, start_imu, end_{ref}, end_imu,
{param}_{ref}, {param}_imu, {param}_error   ← for every _STRIDE_PARAMS entry
trial, algorithm
```
**Reference column naming depends on the reference system**:
- v3d files: `{param}_v3d` (written directly by `stride_evaluation.py`)
- INDIP files: `{param}_indip` (`indip/__init__.py` renames `_v3d → _indip` inside `batch_stride_analysis_indip`)

The app uses `_stride_ref_col(param, df)` to auto-detect the actual column name (checking v3d → indip → ref, in that order).

`_STRIDE_PARAMS` = `stride_duration_s, cadence_spm, stride_length_m, walking_speed_mps, stance_time_s, swing_time_s, step_time_s, step_length_m, single_support_s, initial_double_support_s, terminal_double_support_s, double_support_s`

`step_time_s` / `step_length_m` are always empty on the INDIP side (`_indip` columns) — INDIP has no
corresponding per-stride data.
`single_support_s` / `initial_double_support_s` / `terminal_double_support_s` / `double_support_s` are
populated on both sides (V3D and INDIP), both computed directly from each system's own raw IC/FC event
times (not from INDIP's native `Single/DoubleSupport_Duration` fields — those use a different
convention than V3D's and don't line up).

### `wb_error_{ref}_{scenario}.csv` (WB aggregate parameter error, one row per trial × algo)
```
trial, algorithm,
{param}_ref, {param}_imu, {param}_error   ← for every _WB_PARAMS entry
```
`_WB_PARAMS` = `n_strides, duration_s, stride_duration_s, stride_length_m, cadence_spm, walking_speed_mps`

**Note**: WB file reference columns are uniformly named `{param}_ref` (hardcoded in `wb_evaluation.py`), independent of `ref_suffix`.

---

## App architecture

### Sidebar selection order
1. Results root path
2. Device (OPAL/AX6/DP7)
3. Subject multi-select
4. **Scenario** (LAB / Outoflab) ← detected first, then selected
5. **Reference** (v3d / indip) ← detected based on the chosen scenario
6. Export PDF button

### Data loading

`_load_all_data(root, dev, subjects_key, ref_suffix, scenario)` returns 5 DataFrames:
```python
ic_filt, stride_filt, ic_error_filt, wb_filt, gsd_filt = _load_all_data(...)
```

`_load_subject_files` uses glob-pattern matching (wildcards supported), merges data across multiple
subjects, and adds a `subject` column.

### Key helper functions

| Function | Purpose |
|---|---|
| `_stride_ref_col(param, df)` | Auto-detects the reference column name in stride data (`_v3d` / `_indip` / `_ref`) |
| `_stride_summary_with_ref(df)` | Computes bias/SD/MAE/RMSE/LOA/ICC by algorithm, independent of `ref_suffix` |
| `_wb_summary(df)` | Same as above, for WB data (always uses the `{p}_ref` column) |
| `_detect_available_scenarios(out_root, device, subjects)` | Scans `ic_metrics_*.csv` / `wb_error_*.csv` / `gsd_metrics_*.csv` to find available scenarios (Lab has many possible values; At-Home is now always just `outoflab`) |
| `_detect_available_refs(out_root, device, subjects, scenario)` | Scans `*_{scenario}.csv` and reads the reference-system token from `stem.split("_")[2]` (token-based parsing rather than substring matching, to avoid breaking if the filename structure changes later) |
| `_ic_stride_glob_patterns(ref_suffix, scenario)` | All five metrics uniformly use `{metric}_{ref}_{scenario}.csv` — At-Home's four metrics (IC/stride/WB/GSD) now all fix `scenario="outoflab"`, so they follow the same formula as Lab with no extra special-casing needed. Both `_eval_data_sig` and `_load_all_data` build their glob lists from this one function |

---

## Tab structure

| Tab | Content | Data source |
|---|---|---|
| IC Detection | Detection metrics (TP/FP/FN/P/R/F1) + IC timing error (bias/SD/MAE) | `ic_metrics`, `ic_error` |
| Stride | Stride-level data, structured the same as Walking Bouts: a summary table per stride parameter (Ref mean±SD, IMU mean±SD, Bias[LOA], MAE, ICC) + scatter/BA plots driven by an algorithm/parameter selector (single color: scatter=#4C72B0, BA=#DD8452) + per-condition detail | `stride_error` |
| Walking Bouts | WB parameter summary table + scatter/BA plots driven by an algorithm/parameter selector. In the Outoflab scenario each row is a **matched WB pair** (not aggregated to one row per trial the way Lab is), so there are more data points | `wb_error` |
| GSD Detection | Shown when `gsd_metrics` data exists (At-Home scenario): a WB-count/total-walking-time comparison table + sample-level detection metrics (TP/FP/FN/P/R/F1/Specificity/Accuracy/NPV, reusing `_detection_summary`) + bout-level matching metrics (TP/FP/FN/P/R) | `gsd_metrics` |

**Note**: Gait Measures and Bland-Altman used to be two separate tabs (both built on stride-level data);
they're now merged into one Stride tab, laid out the same way as Walking Bouts (summary table first,
then the interactive scatter/BA plots) — this removes the inconsistency of having data at the same
level split across two tabs while WB-level data got its own single tab.

---

## At-Home / GSD notes

- At-home has one extra evaluation step compared to Lab: **GSD (Gait Sequence Detection)**, with the
  reference system fixed to INDIP (V3D is a Lab-only reference, not applicable at-home)
- **At-Home's file suffix is always the literal `outoflab`**, not `_task_label()` (the lowercased task
  display name) — this is deliberate: all four At-Home evaluations (IC/stride/WB/GSD) match against the
  same continuous, not-split-by-task reference, so whatever the task happens to be called in
  `task_config.json` has no bearing on the output filename. This keeps one subject's at-home data from
  splitting into two different report_app scenarios just because a task got renamed. Whether the GSD tab
  is shown is purely data-driven (whether `gsd_metrics_*.csv` has data, i.e. `_has_gsd`), independent of
  the scenario string itself
- `imu_pipeline.py` detects "is this an at-home task" via `"outoflab" in self._current_keywords` (the
  keyword configured for that task in `task_config.json` — currently the At-Home task's keyword is
  `["outoflab"]`, matching the `OutofLab` marker in the raw filenames) — this is a stable signal that
  doesn't depend on how the task's display name gets edited
- At-home has no Lab-style per-task trial structure — one subject has one continuous recording per day.
  The INDIP reference comes from `data.TimeMeasure1.{Recording*}.Standards.INDIP.ContinuousWalkingPeriod`
  in the `.mat` file (CWP, a struct array where each element is one independent walking bout).
  **CWP is used, not MicroWB** — CWP is a coarser bout definition that bridges the short gaps MicroWB
  splits apart (P04: 56 CWP bouts vs. 64 MicroWB bouts). This differs from Lab's
  `{test}.{trial}.testName` structure, so the two use different loaders and can't share one.
- Matching logic: **uses mobgap's own `mobgap.gait_sequences.evaluation` module directly** instead of a
  hand-rolled overlap calculation, answering two different questions at two levels:
  - **Bout level** (`_match_wb_bouts`, calls mobgap's `categorize_intervals` + `get_matching_intervals`):
    which reference WB pairs with which detected WB, used for comparing gait parameters (stride
    length/cadence etc., one-to-one). mobgap's own overlap threshold (default 0.8, `overlap_threshold`)
    must be satisfied **in both directions** (the overlap must cover ≥80% of both the reference WB's and
    the detected WB's own duration) — stricter than the older Kirk et al. one-directional definition. In
    real-world data, a bout-count match rate of 20%–40% is expected, since fragmented or overly long WBs
    fail easily under a bidirectional threshold.
  - **Sample level** (`_gsd_sample_level_metrics`, calls mobgap's `categorize_intervals_per_sample` +
    `calculate_matched_gsd_performance_metrics`): labels every sample in the whole recording tp/fp/fn/tn
    regardless of how either side segmented its bouts, then computes precision/recall/f1/specificity/
    accuracy/npv. This measures "how much of the actual walking time got captured," and is usually much
    more forgiving — and more informative — than the bout-level match rate (`n_overall_samples`
    approximates using the end time of the last WB on either side).
  - Core code lives in `src/ug3imu/metrics/gsd_evaluation.py` (`normalize_athome_wb` unifies the time
    units of MobGap's and SKDH's two different `wb.csv` formats; `_match_wb_bouts`/
    `_gsd_sample_level_metrics` correspond to the two levels above; `batch_gsd_analysis_indip` is the
    batch entry point, computing both)
- The At-Home INDIP `.mat` file is located via `find_indip_mat_athome` (looks for `*Outoflab.mat`), a
  separate function from Lab's `find_indip_mat` (looks for `*Inlab.mat`) — a given subject's INDIP folder
  contains both files, so they must be looked up separately
- Two output files (always the fixed `outoflab` suffix, no `task_lbl`):
  - `wb_error_indip_outoflab.csv` — one row per **bout-level matched WB pair** (`{param}_ref/_imu/_error`),
    the same schema as Lab's `wb_error_*.csv`, so it's consumed directly by the Walking Bouts tab's
    existing summary-table + scatter/BA logic with no extra tab_wb code needed
  - `gsd_metrics_indip_outoflab.csv` — one row per trial × algorithm, with bout-level `tp_wb/fp_wb/fn_wb`
    + sample-level `tp_samples/fp_samples/fn_samples/precision/recall/f1_score/tn_samples/specificity/
    accuracy/npv` (mobgap's native naming — `tp_samples` and friends share a name with `ic_metrics_*.csv`
    but are now true sample counts, safe to sum across trials) + `n_ref_wb/n_imu_wb/
    total_walking_time_ref_s/total_walking_time_imu_s`. The GSD tab reuses `_detection_summary()`
    directly on the sample-level `tp_samples/fp_samples/fn_samples`, and shows the bout-level
    `tp_wb/fp_wb/fn_wb` in a separate table
- Generation entry point: in `scripts/imu_pipeline.py`'s `_eval_common()`, when the current task's
  keyword contains `outoflab` it automatically routes to `batch_gsd_analysis_indip` instead of Lab's
  `batch_wb_analysis_indip` (both the MobGap and SKDH evaluation paths go through `_eval_common`, so
  both are covered)
- **IC/stride-level evaluation now also works at-home** (the previous known limitation is resolved): the
  At-Home `*Outoflab.mat`'s `ContinuousWalkingPeriod` is a struct array with "one element per bout," but
  **each bout element itself carries the same rich per-stride fields as the Lab CWP struct**
  (`Stride_InitialContacts`, `SingleSupport_Duration`/`DoubleSupport_Duration`,
  `InitialContact_Event`/`FinalContact_Event`, etc.) — `load_indip_cwp_athome` was only reading each
  bout's aggregate scalar fields (Start/End/NumberStrides…) and discarding these per-stride arrays. That
  turned out to be the actual reason nothing could be read before, not a genuine gap in the data itself
  (verified by opening the real P04 `.mat` file directly).
  - `load_indip_stride_athome(mat_path)` / `load_indip_ic_athome(mat_path)`
    (`src/ug3imu/indip/__init__.py`): walk every bout, reuse the existing Lab
    `_cwp_to_stride_df`/`_cwp_to_ic_df` unchanged, and concatenate into one continuous table spanning
    the whole recording, including all 4 support columns.
  - `batch_stride_analysis_indip_athome`/`batch_ic_analysis_indip_athome`: like
    `batch_gsd_analysis_indip`, **no task-name matching** — every stride/IC file matches against the same
    continuous reference. SKDH's at-home stride csv still has raw GaitLumbar columns (`IC Time`, etc.,
    not yet converted to start/end frame indices) and goes through
    `metrics.stride_evaluation.normalize_athome_stride` first (reusing the Lab SKDH
    `gait_lumbar_df_to_stride_df` conversion); MobGap's at-home output already has start/end and passes
    through unchanged.
  - Output files: `ic_error_indip_outoflab.csv`, `ic_metrics_indip_outoflab.csv`,
    `stride_error_indip_outoflab.csv`, `stride_rmse_indip_outoflab.csv` — same fixed `outoflab` suffix as
    WB/GSD, so all four At-Home metrics now follow one consistent naming convention. No special-casing
    needed in report_app.py — `_ic_stride_glob_patterns` uses the same `{metric}_{ref}_{scenario}.csv`
    formula as Lab, so the IC/Stride tabs pick this up automatically with no code changes.
  - **Known issue**: in real data, support metrics (`single_support_s` in particular) occasionally show
    physically implausible outliers (e.g. negative values) toward the tail. This is an inherent artifact
    of the shared `_support_phase_durations` logic (the same code Lab uses, already validated there) at
    certain bout boundaries / sparse-stride situations — the median/IQR are normal, and no extra
    filtering has been added yet.

---

## PDF export

`_build_pdf_report(ic_filt, stride_filt, ic_error_filt, wb_filt, device, selected, ref_label, scenario)`

Includes: IC Detection → IC Timing Error → Stride summary table → Bland-Altman (stride) → Walking Bouts summary table → WB Bland-Altman

---

## Known issues / caveats

- In `stride_evaluation.py` the reference column is always hardcoded as `{p}_v3d`, regardless of the actual reference system. `_stride_ref_col` works around this.
- WB evaluation requires the SKDH/MobGap pipeline to have already produced a `wb/` folder — otherwise there's no `wb_error_*.csv`.
- `_icc21` computes ICC(2,1) and needs at least 3 samples; returns NaN with fewer than 3.

---

## INDIP vs Mocap validation

Validates the INDIP reference system itself against V3D mocap (rather than validating an IMU algorithm
against a reference). Depends only on `{ref_root}/{subject}/V3D/*.txt` +
`{ref_root}/{subject}/INDIP/*.mat`, independent of any device/algorithm output — so it only covers the
Lab scenario (V3D and INDIP are both Lab-only reference systems).

### Generating evaluation data

```
python scripts/evaluate_indip_vs_mocap.py --ref_root R:/... --output_root R:/...
python scripts/evaluate_indip_vs_mocap.py --subjects P03 P04   # only run specific subjects
python scripts/evaluate_indip_vs_mocap.py                       # uses ref_root/output_root from project_settings.json
```

The core matching logic reuses three functions added to `src/ug3imu/indip/__init__.py` (treating INDIP
as the "system under test" and V3D as the reference — the opposite of `batch_ic_analysis_indip` /
`batch_stride_analysis_indip`'s detected/reference roles):
`batch_ic_analysis_indip_vs_mocap`, `batch_stride_analysis_indip_vs_mocap`,
`batch_wb_analysis_indip_vs_mocap`. The shared matching frame rate defaults to INDIP's native 100 Hz
(`fs=100.0`) — only the V3D side is resampled; INDIP doesn't need it.

Output goes to `{output_root}/{subject}/INDIP/Evaluation/`, with filenames identical to `report_app.py`'s
v3d-reference files (`ic_metrics_v3d_lab.csv`, etc.); the `algorithm` column is always `"INDIP"`.

### Viewing results

```
streamlit run scripts/report_app_indipvsmocap.py
```

The sidebar has only Results root + Subject multi-select (no Device / Reference / Scenario selectors,
since those are all fixed). Tab structure is identical to `report_app.py`: IC Detection / Stride /
Walking Bouts, with the same subject/trial hover tooltips on the scatter/BA plots. No GSD tab (outoflab
doesn't apply here).
