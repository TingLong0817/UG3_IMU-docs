# News

Notable changes to the toolbox, newest first. Maintained by hand alongside the repository — not a
live feed of the GitHub history.

## 2026-11-05 — QC summary gets a final zero-WB / failed-trial tally; one bad trial no longer aborts a batch

Two related fixes to `qc_summary_{algorithm}.txt` (written by both `unified_engine._run_skdh_gait` and
`pipeline_factory.run_pipeline_on_dataset`):

- Each per-trial body is now wrapped in its own `try/except` — an exception partway through one trial
  (bad GSD window, a pipeline step raising, …) is logged and that trial is skipped, instead of aborting
  every trial after it in the same batch call (and losing the qc_summary file for all of them, since it
  used to only get written after the full loop finished without raising).
- A final `Summary` section is appended after all per-trial blocks, via the new
  `qc_templates.build_qc_summary_footer`: lists which trials ended up with 0 walking bouts, and which
  failed outright, so a scan of just the last few lines tells you whether a batch run needs follow-up.

## 2026-11-04 — Two new tunable knobs: fixed SKDH `v_axis`/`ap_axis`, for a real dual-task false-positive bug

Diagnosed a bad SKDH `"AP CWT"` result on a real trial: UG035's `DTWTEXT_02` (dual-task, texting while
walking) reported SKDH cadence over 400 spm, with `ic_metrics_v3d_lab.csv` showing precision 0.44 (49 FP
out of 88 detected ICs) against MobGap's 0.83 on the same trial. Root cause, confirmed by reproducing the
exact mocap-cropped window (`parse_mocap_frame_window`'s 29 s crop) standalone: `GaitLumbar`'s own per-bout
AP-axis estimate picked axis 1 on that crop vs. axis 2 on the full 42 s file — texting-related arm/torso
motion apparently competes with the true AP oscillation once the window is tightened to just the walking
span. The wrong axis fed `mean step freq = 5 Hz` (true rate ~1.6 Hz) into the AP-CWT wavelet-scale formula,
fragmenting what should be one stride into several spurious ICs. Confirmed this is *not* a regression from
the 0.17.18 upgrade above — reproduced byte-identical under a side-installed 0.17.9.

Added `skdh_v_axis`/`skdh_ap_axis` to `PipelineConfig`, threaded through to `GaitLumbar.predict()`'s
`v_axis`/`ap_axis` kwargs in `unified_engine._run_skdh_gait`, `expand_configs()`, and the GUI's SKDH
"⚙ Params" dialog — same self-describing-output/filename-tag pattern as the other tunable knobs (see
[pipelines/README.md — Tunable SKDH/MobGap parameters](src/ug3imu/pipelines/README.md#tunable-skdhmobgap-parameters)).

**Unlike every other tunable knob here, the default is a fixed axis (`v_axis=0, ap_axis=2`), not a
`None`/"let the library decide" sentinel.** Reasoning: `unified_engine._accel_g` feeds SKDH the *same*
array MobGap's own pipeline already rotated into the Mobilise-D convention (`acc_x` = up, `acc_z` =
forward) — true for every device, since `build_dataset_from_file_list` always loads through
`load_imu_for_mobgap` first. So the axis mapping is already fixed and identical across devices by the time
SKDH runs; re-estimating it per bout from the signal is pure downside, confirmed by the UG035 case above.
Pass `skdh_v_axis=None`/`skdh_ap_axis=None` explicitly to opt back into SKDH's own per-bout estimate.

Initially implemented this as an opt-in override (default `None`, matching the other knobs) before
reconsidering: that would have required deriving the axis per device from each one's raw sign-flip
convention in `preprocessing/imu_preprocessing.py` (confirmed those also resolve to axis 0/2 for all four
supported devices — AX6/OPAL/DP7/EMPATICA — since none of their rotation recipes swap axes, only flip
signs) — a real but unnecessary detour, since `_run_skdh_gait` never actually sees that raw, per-device
frame; it only ever sees the already-unified rotated one. The GUI's SKDH params dialog now pre-fills these
two fields with `"0"`/`"2"` to match (clearing a field opts back into the per-bout estimate, the reverse of
every other field in that dialog).

## 2026-11-04 — Upgraded scikit-digital-health 0.17.9 → 0.17.18

Checked every release between the two (GitHub release notes + a line-by-line diff of
`skdh/gait/core.py` and `gait_metrics.py`) before bumping the pin in `requirements.txt`:

- **Breaking, fixed here**: `GaitLumbar`'s output column `"IC Time"` was renamed to
  `"IC Datetime"` in 0.17.18 (plus two new columns, `"IC Timestamp"`/`"IC Rel Timestamp"`,
  that we don't use). Without a fix, `gait_lumbar_df_to_stride_df` would have silently
  returned `None` for every SKDH-gait run — no error, just empty stride/WB output. Updated
  the 4 call sites (`pipelines/skdh_lab_pipeline.py`, `pipelines/unified_engine.py` ×2,
  `metrics/stride_evaluation.py`) to the new name; `stride_evaluation.normalize_athome_stride`
  still accepts the old `"IC Time"` name too, since it reads *historical* `*_stride.csv`
  files that may have been written under 0.17.9.
- **Not breaking**: `GaitLumbar.predict()` gained an optional `leg_length=` kwarg (0.17.16,
  takes priority over `height`×`height_factor` if both given) — our existing
  `height=`/`height_factor=` call pattern is the documented fallback path and needed no
  change. The old `provide_leg_length` constructor flag was removed; we never used it.
  `gait_metrics.py` (the actual stride/step-length/cadence/speed formulas) is byte-for-byte
  identical between the two versions — no numeric-formula changes in this range despite
  0.17.10's changelog note about "gait results changed slightly" (that change must live in
  a dependency outside this file).

Verified end-to-end on a real P17 lab AX6 recording (`LWA_01`) through
`run_unified_pipeline` for both SKDH IC methods (`"AP CWT"`, `"Vertical CWT"`) and MobGap —
all three produce non-empty GS/IC/stride/WB output after the fix.

## 2026-10-29 — Raw `.cwa` (Axivity binary) input format, AX6 only

AX6 data can now be fed into the pipeline straight from the raw `.cwa` file the device writes, with no
sync/export step first — added for a headless cluster batch job over at-home recordings. New
`preprocess_cwa_for_mobgap` / `load_cwa_for_skdh` / `read_cwa_raw` / `read_cwa_fs` in
[preprocessing/imu_preprocessing.py](src/ug3imu/preprocessing/imu_preprocessing.py), reading the binary via
`skdh.io.ReadCwa`; wired into `load_imu_for_mobgap`/`load_imu_for_skdh` and `INPUT_FORMATS`. See
[preprocessing/README.md — CWA](src/ug3imu/preprocessing/README.md#cwa-raw-axivity-binary-ax6-only):
the axis-orientation transform is carried over unverified from the lab AX6 CSV path and is explicitly
flagged as a placeholder (`TODO(cwa-orientation)`) pending a check against a real at-home recording; each
file's own actual device sampling rate is used throughout rather than a nominal rate, since Axivity units
drift from nominal differently per device/recording.

## 2026-10-15 — EMPATICA (Embrace Plus) support; several MobGap/report_app fixes surfaced by real data

A fourth device, **EMPATICA** (Embrace Plus, lumbar placement), is now supported end-to-end: raw-file
discovery, MobGap/SKDH preprocessing, GUI device selector, and `report_app.py`'s device dropdown. See
[preprocessing/README.md — Supported devices](src/ug3imu/preprocessing/README.md#supported-devices) for
the axis-rotation derivation (180° rotation around Y, same pattern as OPAL — derived from the device's
documented raw mounting orientation against the Mobilise-D sensor frame, and cross-checked against AX6's
raw gravity reading at rest). The internal device key is `EMPATICA_lumbar` (not bare `EMPATICA`) so file
discovery can tell the lumbar placement apart from the same subject's `EMPATICA_ankleL/ankleR/wrist/BVP`
folders, but results still land under a plain `EMPATICA` output folder — `imu_pipeline._output_device_name`
maps between the two.

Also fixed, all found while bringing up OPAL and EMPATICA data through paths that AX6/DP7 had exercised for
months without hitting them:

- **MobGap gait crashed on any non-integer sampling rate** (`ValueError: A value (...) in x_new is below
  the interpolation range's minimum`) — `SlZijlstra`'s internal per-second resampling assumes an integer
  Hz; AX6's real rate (99.51 Hz) left a sub-sample bin-edge offset that could land exactly on a stride
  boundary. Fixed by rounding only the sampling rate mobgap itself reads
  (`pipeline_factory._round_fs_for_mobgap`); crop frames, evaluation, and output timestamps elsewhere still
  use the device's true rate.
- **`discover_files_by_keyword`'s device match was a loose substring** — `device="OPAL"` matched
  `Opal_Sync_L`/`Opal_Sync_R` (secondary foot-placement folders) as well as the real `Opal_Sync`. Now
  requires an exact match on `{device}` or `{device}_Sync`.
- **OPAL's ICD algorithms collapsed together in evaluation** — `imu_pipeline._detect_ic_algorithms` and
  `ic_evaluation.py`'s empty-file fallback both assumed a fixed 7-token filename prefix before the
  algorithm name; OPAL's raw filename has no separate `_sync` token (just `opal8`), so the split landed one
  token early and merged 3 distinct MobGap ICD algorithms into one (`IcdIonescu`/`IcdShinImproved`/
  `IcdHKLeeImproved` all read back as `LrcUllrich`). Both now locate the `mocap`/`indip` reftype token
  itself instead of assuming a fixed count.
- **SKDH's IC overlay plot never drew the reference ICs**, only the algorithm's own detections — the
  `mocap_folder`/`indip_map` parameters were already threaded into `_run_skdh_gait` but unused. It now
  builds the same V3D/INDIP lookup as the MobGap path and calls the same `plot_ic_overlay_lab`, so both
  engines' IC plots look the same.

New **Stride Length QC** sidebar filter in `report_app.py`, mirroring the existing Cadence QC: drops
stride/WB rows whose stride length (reference or IMU) falls outside a configurable range (default
0.15–2.5 m) before computing bias/RMSE/ICC.

## 2026-09-28 — SKDH/MobGap parameters are tunable from the GUI, and correctly tracked through evaluation

Seven previously-hardcoded knobs (SKDH `GaitLumbar`'s prominence/wavelet/height/max-stride-time knobs;
MobGap `SlZijlstra`'s step-length scaling factor) are now adjustable per run via a "⚙ Params" button in the
GUI's **Pipeline** tab, instead of requiring a code edit. See
[pipelines — Tunable SKDH/MobGap parameters](src/ug3imu/pipelines/README.md#tunable-skdhmobgap-parameters)
and [guide — Fine-tuning an algorithm](docs-site/guide/pipeline-options.md#fine-tuning-an-algorithm-the-params-button).
SKDH's IC-detection algorithm list also dropped from 5 entries to the 2 real algorithms (`AP CWT`,
`Vertical CWT`) — the other 3 were fixed presets of these same knobs and could silently conflict with a
manually-typed value.

A parameter-sweep run no longer overwrites its own earlier output (filename gets a tag when values differ
from default; the values used are written as real columns on every output row). Several evaluation
functions that deduped by algorithm *choice* only were fixed to fold these knobs into their dedup/labeling
key too, so two sweep runs sharing a choice no longer collide (`ug3imu.indip`, `metrics.ic_evaluation`,
`metrics.wb_evaluation`, `metrics.stride_evaluation`, `metrics.gsd_evaluation`).

`report_app.py` now shows a **Params** column next to every Algorithm column, and each results tab has its
own algorithm/parameter filter, so parameter combinations can be compared within the tab instead of only in
a separate ledger.

Also fixed: the SKDH pipeline-run-time WB timeline plot was built from raw GSD candidate windows instead of
the final assembled walking bouts; and a pandas `.groupby().apply()` edge case in `report_app.py` crashed
whenever exactly one algorithm/parameter combination was selected.

## 2026-09-01 — At-Home WB match threshold is adjustable in the GUI

The bidirectional bout-overlap threshold used to match a detected walking bout against an INDIP CWP bout
(`batch_gsd_analysis_indip`'s `overlap_threshold`) is now a **WB overlap** field in the GUI's Evaluation
row (INDIP reference only), default `0.8`, clamped to `> 0.5`, remembered in `project_settings.json`. The
`Evaluate Results` log line now reports the value it used.

Why it's needed: At-Home matched-WB counts are structurally low at `0.8` for **both** gait engines, and
lowering the threshold to ~0.6–0.7 is the intended remedy. Two causes, now written up in
[metrics/README.md — Why At-Home matched-WB counts run low](src/ug3imu/metrics/README.md#why-at-home-matched-wb-counts-run-low):
(1) INDIP's `ContinuousWalkingPeriod` keeps a turn inside one continuous bout, while the shared
Mobilise-D `WbAssembly` drops the turn strides and splits the bout at the resulting >3 s gap — so a
correctly detected IMU bout is systematically shorter than the CWP it should match; (2) SKDH's
`GaitLumbar` CWT IC detector gates on a bout-global prominence and goes silent for several seconds in
low-amplitude gait, fragmenting bouts further (MobGap's threshold-free `IcdIonescu` does not). Cause 2 is
separately reducible via `GaitLumbar`'s `ic_prom_factor` / `fc_prom_factor` (AP CWT) — now exposed as a GUI
knob, see the 2026-09-28 entry above.

## 2026-08-14 — Stride selection & walking-bout assembly unified to the Mobilise-D standard everywhere

Every scenario (MobGap Lab/At-Home/Functional Test, SKDH Lab/At-Home) now selects strides and assembles
walking bouts with the same rules: duration 0.2–3.0 s + stride length ≥ 0.15 m (no cadence rule), and WB
assembly via mobgap's own `StrideSelection`/`WbAssembly` (min 4 strides/bout, max 3 s break) — previously
Lab/Functional Test used a stricter, non-standard 0.6–2.0 s + cadence 60–200 spm rule, and SKDH filtered
strides and grouped bouts with its own hand-rolled logic instead of MobGap's. SKDH now calls mobgap's
classes directly, so both engines produce identical WBs from the same stride list. The shared rule set and
trim logic live in a new `pipelines/mobilised_wb.py`, used by all three pipelines.

Per the standard, the first and last stride of every WB (unreliable transition strides) are now excluded
from per-stride output and from each WB's mean gait parameters. A WB's own `start`/`end`/`duration_s` keep
the *full*, untrimmed stride-sequence extent, though — only `n_strides` and the parameter means come from
the trimmed interior strides — so At-Home GSD-detection evaluation (which compares these WB boundaries
against INDIP's CWP reference) isn't penalized with artificially narrow bout windows, and a short WB is no
longer dropped outright just because few strides remain after trimming.

SKDH's At-Home `wb.csv`/`stride.csv` also switch to the same unified column names MobGap and SKDH Lab
already use (`start`/`end`/`n_strides`/`duration_s`/... instead of raw SKDH names like `Bout Starts`), so
`gsd_evaluation.py`'s format detection and `report_app.py` treat all three pipelines identically.

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
