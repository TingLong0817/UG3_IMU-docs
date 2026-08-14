# News

Notable changes to the toolbox, newest first. Maintained by hand alongside the repository — not a
live feed of the GitHub history.

## 2026-08-13 — Reports show GSD/ICD/LRC as separate columns; reference cadence QC; INDIP-vs-mocap dashboard catches up

`report_app.py`'s IC Detection, Stride, and Walking Bouts tabs (interactive and PDF export) no longer
display the per-stage algorithm columns added earlier today as one flattened string — each stage that
actually ran gets its own column (`GSD Algorithm` / `ICD Algorithm` / `LRC Algorithm`), so different
pipeline stages are never mixed together when scanning a table. A stage that didn't really run — Lab's
mocap-windowed crop or a Functional Test's full-recording window, neither of which is a selectable GSD
algorithm — is hidden from display entirely rather than shown as a constant `GSD Algorithm` column, which
was misleading readers into thinking a GSD algorithm had been evaluated in Lab. `ic_evaluation.py`,
`wb_evaluation.py`, `stride_evaluation.py`, `gsd_evaluation.py`, and `indip`'s
`batch_ic_analysis_indip*`/`batch_stride_analysis_indip*` now write these per-stage columns into their own
output rows (previously only the pipeline-run step did).

New **reference cadence QC**: a sidebar control (default 80–180 spm, adjustable) drops stride/WB rows whose
*reference* (mocap/INDIP) cadence is out of range before computing bias/RMSE/ICC — unlike IMU-side strides,
the reference cadence had no equivalent upstream filter. Applies to both `report_app.py` and the sibling
`report_app_indipvsmocap.py`.

`report_app_indipvsmocap.py` also gained the turn/obstacle-step stride exclusion filter and
Straight/Turn/Step color-coded scatter + Bland-Altman plots that `report_app.py` already had — the
underlying data (`is_turn`/`is_step`, carried through by `match_strides` from the V3D reference side) was
already there, this dashboard just hadn't been updated to show it.

Fixed a pre-existing bug in `aggregate_ic_results.py` (the standalone CLI aggregator): `_load_with_meta`
was overwriting each file's correct per-row `algorithm` column with a garbage value derived from the
filename (e.g. `"v3d_lab"`), so the CLI's `IC_summary_*.csv`/`stride_summary_*.csv` output couldn't group
by algorithm correctly. Now only used as a fallback for files that predate the per-row column.

## 2026-08-13 — Per-stage algorithm columns; GSD/IC results no longer duplicate across ICD/LRC choices

Every pipeline output (`gs`/`ic`/`stride`/`wb`/`turn` CSVs, MobGap and SKDH, Lab and At-Home) now carries
one column per pipeline stage — `gsd_algorithm`, `icd_algorithm`, `lrc_algorithm`, plus four constant
columns for the non-selectable stages — alongside the existing full-pipeline `algorithm` filename tag.

This fixes a real bug in At-Home evaluation: GSD-detection accuracy only depends on the GSD stage and
IC-detection accuracy only depends on GSD+ICD, but both were previously labeled and grouped by the full
`{gsd}_{icd}_{lrc}` pipeline name — so testing one GSD algorithm against several ICD choices made the
*same* GSD result show up as several different "algorithms" in the GSD Detection tab, cluttering it with
what looked like IC algorithm names. `gsd_evaluation.batch_gsd_analysis_indip`,
`ic_evaluation.batch_ic_analysis_multi_algo`, and `indip`'s `batch_ic_analysis_indip`/
`batch_ic_analysis_indip_athome` now read the new columns directly and dedup to one result per
`(trial, gsd_algorithm)` or `(trial, gsd_algorithm, icd_algorithm)`, while per-bout/per-stride *parameter*
comparisons (which do depend on the whole chain) keep evaluating every file under the full pipeline name.
`report_app.py` and the long-format exports needed no changes — they already group/read by whatever
`algorithm` column the source data provides.

`athome_pipeline.py` (a separate, older At-Home MobGap pipeline implementation) was found to be dead code
during this work — `imu_pipeline.py` has exclusively used `pipeline_factory.py`'s unified
`run_pipeline_on_dataset` (via `windowing="gsd"`) for both Lab and At-Home for some time.

## 2026-08-13 — Dead code cleanup

Removed code with zero remaining callers, found while auditing the pipeline output columns above:
`pipelines/athome_pipeline.py` (superseded by `pipeline_factory.run_pipeline_on_dataset`, see above),
`ug3imu/io/` (empty package), `dataset_generation.build_dataset_from_folder`,
`athome_dataset_generation.build_athome_dataset_from_files`, and `mocap_utils.mocap_txt_to_step_df`
(stride-level step metrics already come from `_STRIDE_METRICS`). `legacy_code/athome_monitoring.py` still
imports from the now-deleted `athome_pipeline.py`; it's archived and not run, so its import is left broken
rather than fixed — use `imu_pipeline.py`'s At-Home preset instead.

## 2026-08-12 — Turn/step stride labels now come from time-interval overlap; INDIP gets `is_turn` too

V3D's `is_turn`/`is_step` now come from new `Pelvis_Turn_On`/`Pelvis_Turn_Off` and `step_start`/`step_end`
paired-interval columns — a stride is flagged if its own `[IC_i, IC_i+1]` span overlaps any turn/step
interval, no left/right distinction. Replaces matching a stride's boundary IC against discrete
`RIC_TURN`/`LIC_TURN` or `RIC_STEP`/`LIC_STEP` timestamps. V3D files without the new columns get
`is_turn`/`is_step = False` for every stride — there's no fallback to the old columns.

INDIP's `ContinuousWalkingPeriod` strides now carry the same `is_turn` label, computed from each bout's
own `Turn_Start`/`Turn_End` — covers Lab and At-Home in one change, since At-Home reuses the Lab
per-stride parser. INDIP has no obstacle-step concept, so there's no `is_step`. Long-format exports and
`report_app.py`'s turn/step filtering and Bland-Altman coloring pick this up for INDIP automatically, no
further changes needed there.

`d9b6354`

## 2026-08-12 — INDIP WB-level evaluation now uses CWP, not MicroWB

`batch_wb_analysis_indip` and `batch_wb_analysis_indip_vs_mocap` now read INDIP's
`ContinuousWalkingPeriod` instead of `MicroWB` for WB-level comparisons — MicroWB has no QC applied, so
CWP is the reference used everywhere else already. Also fixed `stride_duration_s` in the WB-level output
to be the arithmetic mean of each bout's own `Stride_Duration` array rather than `60/StrideFrequency`:
per Mobilise-D's own `StrideFrequency` definition (`mean(60/Stride_Duration_k)`), inverting it back gives
a harmonic mean of the stride durations, biased low relative to their true arithmetic mean — more so in
bouts with variable stride durations (turns, free-living data).

`372e457`

## 2026-08-01 — Clean algorithm names everywhere; SKDH At-Home now has IC output

The `algorithm` column in every IC/stride/WB/GSD evaluation output is now always a clean algorithm
name (e.g. `IcdHKLeeImproved`), consistent across every reference system and scenario, including
V3D-referenced Lab IC/stride results.

SKDH's At-Home pipeline now also writes an `IC/` output folder, matching MobGap and the Lab SKDH
pipeline — per-IC `ic`/`lr_label` csvs, enabling IC-level evaluation for SKDH At-Home runs.

`70b14b8` `68afe72`

## 2026-07-31 — R-friendly long-format export tool for Evaluation CSVs

New `scripts/long_format_exports/export_long_format.py` reshapes a subject's `Evaluation/` CSVs
into long format (one row per device per stride/bout/trial) for R. Ten exporters cover stride/WB/IC
against both V3D and INDIP in Lab, and stride/WB/IC/GSD for At-Home — filenames follow
`{metric}_{ref}_{scenario}_long_{subject}.csv` with `scenario` only ever `lab` or `outoflab`. Every
row also carries a `sensor` column (AX6/OPAL/DP7), inferred automatically from the eval folder path.

`27fe84f` `978b60c` `5a33e75`

## 2026-07-30 — At-Home INDIP stride/IC evaluation, incl. single/double support

At-Home stride- and IC-level evaluation against the INDIP reference is now supported, including all
four single/double support metrics. New `load_indip_stride_athome`/`load_indip_ic_athome` loaders
plus `batch_stride_analysis_indip_athome`/`batch_ic_analysis_indip_athome` match every stride/IC
file against one continuous At-Home reference (mirroring the existing GSD/WB at-home matching),
with a `normalize_athome_stride` shim covering both MobGap's and SKDH's At-Home stride formats. All
four At-Home Evaluation outputs (IC/stride/WB/GSD) consistently use a fixed `outoflab` filename
suffix.

`081b844`

## 2026-07-28 — Single/double support metrics, for both V3D and INDIP

Added `single_support_s`, `initial_double_support_s`, `terminal_double_support_s`, and
`double_support_s` to the stride evaluation pipeline. Both reference systems compute all four
directly from raw IC/FC event times, giving a consistent definition across V3D and INDIP.
`report_app.py`, `report_app_indipvsmocap.py`, and `aggregate_ic_results.py` were updated to
surface the new columns.

`27f334c`

## 2026-07-28 — SKDH `height_factor` now configurable

`run_skdh_lab_pipeline` exposes `height_factor` (default 0.53, SKDH's own default) as a parameter
instead of hardcoding it.

`d3cc28c`

## 2026-07-20 — Obstacle-step labels, INDIP CWP at-home reference, report app improvements

Strides are now flagged `is_turn`/`is_step` from V3D's turn/obstacle event columns and carried
through matching. At-home GSD/WB evaluation now uses INDIP's `ContinuousWalkingPeriod` reference.
Stride matching gained a `buffer_frames` trim to keep IMU strides detected outside mocap's own
annotated window from being scored as false positives.

`e2b9dc7`
