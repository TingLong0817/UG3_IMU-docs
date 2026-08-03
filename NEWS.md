# News

Notable changes to the toolbox, newest first. Maintained by hand alongside the repository — not a
live feed of the GitHub history.

<!-- end-to-end sync test marker -->

## 2026-08-01 — Algorithm-name parsing fixed everywhere; SKDH At-Home gains IC output

The `algorithm` column across every IC/stride/WB/GSD evaluation function was picking up extra
device/sync/reftype tokens from the filename (e.g. `ax6_sync_indip_IcdHKLeeImproved` instead of
`IcdHKLeeImproved`) — the underscore-offset used to split the filename was wrong. Fixed at every
call site. A second, separate bug in `imu_pipeline.py`'s `_detect_ic_algorithms` was independently
re-computing the same dirty value and overwriting the already-correct result specifically for
V3D-referenced Lab IC/stride — fixed too.

Separately, `skdh_athome_pipeline.py` never wrote an `IC/` output folder at all (unlike MobGap and
the Lab SKDH pipeline), so SKDH At-Home IC-level evaluation had nothing to compare against; it now
writes per-IC `ic`/`lr_label` csvs the same way the Lab pipeline does.

`70b14b8` `68afe72`

## 2026-07-31 — R-friendly long-format export tool for Evaluation CSVs

New `scripts/long_format_exports/export_long_format.py` reshapes a subject's `Evaluation/` CSVs
into long format (one row per device per stride/bout/trial) for R. Ten exporters cover stride/WB/IC
against both V3D and INDIP in Lab, and stride/WB/IC/GSD for At-Home — filenames follow
`{metric}_{ref}_{scenario}_long_{subject}.csv` with `scenario` only ever `lab` or `outoflab`. Every
row also carries a `sensor` column (AX6/OPAL/DP7), inferred automatically from the eval folder path.

`27fe84f` `978b60c` `5a33e75`

## 2026-07-30 — At-Home INDIP stride/IC evaluation, incl. single/double support

The At-Home `*Outoflab.mat` `ContinuousWalkingPeriod` array turned out to carry the same per-stride
fields as the Lab CWP struct (verified against real data) — the existing at-home loader just read
the bout-level aggregates and discarded them. New `load_indip_stride_athome`/`load_indip_ic_athome`
plus `batch_stride_analysis_indip_athome`/`batch_ic_analysis_indip_athome` (no task-name keying,
mirrors the existing GSD/WB at-home matching) wire this up end to end, including a
`normalize_athome_stride` shim for SKDH's raw GaitLumbar stride csv shape. Also fixes At-Home IC
evaluation always matching against the Lab `.mat` file (silently empty results). All four At-Home
Evaluation outputs (IC/stride/WB/GSD) now consistently use a fixed `outoflab` filename suffix
instead of the task label; `report_app.py` updated to match.

`081b844`

## 2026-07-28 — Single/double support metrics, for both V3D and INDIP

Added `single_support_s`, `initial_double_support_s`, `terminal_double_support_s`, and
`double_support_s` to the stride evaluation pipeline. Both reference systems compute all four
directly from raw IC/FC event times rather than trusting either system's own pre-aggregated fields
— INDIP's native `SingleSupport_Duration` turned out to use a different convention than V3D's
per-side one, which the event-based approach sidesteps entirely. `report_app.py`,
`report_app_indipvsmocap.py`, and `aggregate_ic_results.py` were updated to surface the new columns.

`27f334c`

## 2026-07-28 — SKDH `height_factor` now configurable

`run_skdh_lab_pipeline` exposes `height_factor` (default 0.53, SKDH's own default) as a parameter
instead of hardcoding it.

`d3cc28c`

## 2026-07-20 — Obstacle-step labels, INDIP CWP at-home reference, report app fixes

Strides are now flagged `is_turn`/`is_step` from V3D's turn/obstacle event columns and carried
through matching. At-home GSD/WB evaluation switched to INDIP's `ContinuousWalkingPeriod` instead of
`MicroWB`. Stride matching gained a `buffer_frames` trim so IMU strides detected outside mocap's own
annotated window no longer count as false positives.

`e2b9dc7`
