# News

Notable changes to the toolbox, newest first. Maintained by hand alongside the repository — not a
live feed of the GitHub history.

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
