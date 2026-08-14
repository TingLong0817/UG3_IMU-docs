# `ug3imu.pipelines`

The core of the toolbox: builds MobGap/SKDH-compatible datasets from raw IMU files, runs gait-detection
pipelines on them, and writes results + QC reports in a consistent directory layout across all three
scenarios (Lab, At-Home, Functional Test) and both engines (MobGap, SKDH).

[← Back to project README](../../../README.md)

## Files

| File | Role |
|------|------|
| [pipeline_factory.py](pipeline_factory.py) | **Unified MobGap factory** — `create_pipeline()`, `run_pipeline_on_dataset()`, algorithm registries, presets. Used for all three MobGap scenarios. |
| [dataset_generation.py](dataset_generation.py) | `build_dataset_from_file_list()` — the dataset builder used by the GUI (works for lab, at-home, and functional test alike) |
| [athome_dataset_generation.py](athome_dataset_generation.py) | `INPUT_FORMATS` registry, file discovery (`discover_files_by_keyword`, `discover_athome_files`) |
| [lab_pipeline.py](lab_pipeline.py) | `DummyGSD` — treats the mocap crop window as a single gait sequence (MobGap lab windowing) |
| [skdh_lab_pipeline.py](skdh_lab_pipeline.py) | `run_skdh_lab_pipeline()` — SKDH `GaitLumbar` on the V3D-cropped window |
| [skdh_athome_pipeline.py](skdh_athome_pipeline.py) | `create_skdh_pipeline()`, `run_skdh_athome_pipeline()` — SKDH bout detection (`PredictGaitLumbarLgbm`) + `GaitLumbar`, stride filtering, WB assembly, DMO aggregation |
| [qc_templates.py](qc_templates.py) | Shared QC text-block builders — MobGap and SKDH both render through these so output format is identical |

`build_dataset_from_folder()` (dataset_generation.py), `build_athome_dataset_from_files()`
(athome_dataset_generation.py), and `athome_pipeline.py` in its entirety (a MobGap at-home pipeline
predating `pipeline_factory.py`, superseded by `create_pipeline(windowing="gsd")` +
`run_pipeline_on_dataset()`) were removed as dead code — none were imported by `scripts/imu_pipeline.py`.
`legacy_code/athome_monitoring.py` still imports from the now-deleted `athome_pipeline.py`; that script is
archived/not run, so its import breaking is expected, not a regression.

## Three windowing modes, one factory

`create_pipeline(windowing=...)` in `pipeline_factory.py` builds a `mobgap.pipeline.GenericMobilisedPipeline`
that differs only in how the gait-sequence-detection (GSD) step is configured:

| `windowing` | GSD component | Requires | Use case |
|-------------|----------------|----------|----------|
| `"gsd"` | A real algorithm from `GSD_ALGO_MAP` (default `GsdIluz`) | — | At-Home: auto-detect bout boundaries |
| `"mocap"` | `DummyGSD` (from [lab_pipeline.py](lab_pipeline.py)) | V3D TXT via `mocap_folder=` | Lab: window = mocap crop, no detection |
| `"full"` | `FullWindowGSD` (defined in `pipeline_factory.py`) | — | Functional Test: entire recording = one window |

Everything downstream (IC detection, laterality, cadence, stride length, stride selection/filtering, WB
assembly, turn detection, optional DMO) is identical across the three modes — only the GSD step changes.
This is why Lab/At-Home/Functional-Test share one code path instead of three.

### Stride quality filter (applied in all three modes)

Hard-coded in `create_pipeline()`'s `StrideSelection` rules — **not** currently exposed as a GUI setting:

- Duration: `0.6–2.0 s` inclusive → cadence 60–200 steps/min (`cadence_spm = 120 / duration_s`)
- Length: `≥ 0.15 m`

The SKDH pipelines apply the identical thresholds via their own `_filter_strides()` (see
[skdh_athome_pipeline.py](skdh_athome_pipeline.py)) so filtering behaves the same regardless of engine.

## PRESETS

`PRESETS` in `pipeline_factory.py` is the single source of truth the GUI reads to populate preset buttons:

```python
PRESETS = {
    "At-Home":          {"windowing": "gsd",   "enable_dmo": True,  "evaluation": False, ...},
    "Lab":               {"windowing": "mocap", "enable_dmo": False, "evaluation": True,  ...},
    "Functional Test":   {"windowing": "full",  "enable_dmo": False, "evaluation": False, ...},
}
```

`data_layout` (`"athome"` vs `"lab"`) tells the GUI which folder-discovery convention to use — see
[athome_dataset_generation.py](#file-discovery--athome_dataset_generationpy) below.

## Building a dataset

```python
from ug3imu.pipelines import build_dataset_from_file_list, create_pipeline, run_pipeline_on_dataset

dataset = build_dataset_from_file_list(
    file_list=files, metadata_csv="participants.csv",
    sampling_rate_hz=100, device="AX6",
    mocap_folder="mocap/TB017/V3D/",   # omit for at-home / functional test
)
```

`build_dataset_from_file_list` (dataset_generation.py) is format/layout-agnostic — it works for flat lab
folders, nested at-home folders, or a mixed file list, because the caller (usually
`discover_files_by_keyword`) has already resolved which files to include. Per file:

1. Subject ID = first `_`-split token of the stem; looked up in `metadata_csv` (must contain
   `Record ID`, `Height (meters)`, and a device-specific height column — see
   [Metadata CSV format](#metadata-csv-format) below). Missing sensor height falls back to
   `height × 0.55` with a logged warning.
2. IMU data loaded via [`preprocessing.load_imu_for_mobgap`](../preprocessing/README.md).
3. If `mocap_folder` is given, the file's 4-part trial key is matched against V3D `.txt` files to get a
   crop window (`parse_mocap_frame_window`, see [`ug3imu.mocap`](../mocap/README.md)); if `frame_windows`
   is given instead (pre-built `{key4: (start, end)}`, e.g. from
   [`indip.build_frame_windows_for_files`](../indip/README.md)), those frames are used directly and take
   priority. Files with neither fall back to the full recording as the window.

## Running a pipeline

```python
pipeline = create_pipeline(windowing="gsd", gsd_algorithm="GsdIluz",
                           icd_algorithm="IcdIonescu", enable_dmo=True)
run_pipeline_on_dataset(dataset, pipeline, output_path="results/TB017/AX6",
                        algorithm_name="GsdIluz_IcdIonescu", imu_fs=100,
                        enable_dmo=True, plot_wb=True, plot_ic=True,
                        gsd_algorithm="GsdIluz", icd_algorithm="IcdIonescu")
```

`run_pipeline_on_dataset` iterates every trial in the dataset, runs the pipeline, and writes whatever the
result object has (`gs_list_`, `raw_ic_list_`, `per_stride_parameters_`, `per_wb_parameters_`,
`aggregated_parameters_`, `raw_turn_list_`) to the matching subfolder — see
[Output directory layout](#output-directory-layout) below. `step_time_s` is computed here as a post-hoc
addition to the stride table, since MobGap doesn't produce it natively:
`step_time_s[i] = (IC[i+1] − IC[i]) / imu_fs`.

### Per-stage algorithm columns

Besides `algorithm_name` (the full pipeline identity baked into every output filename, e.g.
`GsdIluz_IcdIonescu_LrcUllrich`), every row of `gs`/`ic`/`stride`/`wb`/`turn` output also carries one column
per pipeline stage — `gsd_algorithm`, `icd_algorithm`, `lrc_algorithm`, `cadence_algorithm`,
`stride_length_algorithm`, `walking_speed_algorithm`, `turn_algorithm` — written by
`_stage_algorithm_columns()` from the `gsd_algorithm`/`icd_algorithm`/`lrc_algorithm` arguments passed to
`run_pipeline_on_dataset` (the last four stages aren't user-selectable in this codebase, so those columns
are constant today; kept for schema uniformity).

This exists because different evaluation questions depend on different, *smaller* subsets of the full
pipeline than the flat `algorithm_name` implies:

- GSD-detection accuracy (did the algorithm find the right walking-bout windows) only depends on the GSD
  stage — it's unaffected by which ICD/LRC a `wb.csv` happened to be generated with.
- IC-detection accuracy depends on GSD (windowing) + ICD, not LRC/etc.
- Per-stride/per-bout *parameter* accuracy (stride length, cadence, ...) genuinely depends on the whole
  chain.

Evaluation functions that only care about a result's GSD or IC identity read these columns directly instead
of parsing the full `algorithm_name` out of the filename, and deduplicate down to one file per
`(trial, gsd_algorithm)` or `(trial, gsd_algorithm, icd_algorithm)` — otherwise the *same* GSD (or IC)
result would get evaluated once per ICD/LRC combination it was tested alongside, and turn up as several
seemingly-different "algorithms" in the GSD Detection / IC tabs. See
[`metrics/gsd_evaluation.py`](../metrics/README.md#gsd--wb-evaluation-for-at-home-gsd_evaluationpy) and
[`indip.batch_ic_analysis_indip_athome`](../indip/README.md).

`windowing != "gsd"` (Lab modes without a real GSD run) still gets a `gsd_algorithm` value rather than a
blank one: `"mocap_windowed"` for `windowing="mocap"` (DummyGSD — window comes from the mocap crop), or
`"full_window"` for `windowing="full"` (FullWindowGSD). SKDH's `gsd_algorithm`/`icd_algorithm` columns
follow its own architecture instead (context detection vs. `gait_event_method`) — see
[SKDH pipelines](#skdh-pipelines) below.

## SKDH pipelines

SKDH doesn't have a `GenericMobilisedPipeline`-equivalent single abstraction, so `skdh_lab_pipeline.py` and
`skdh_athome_pipeline.py` each implement the full read → detect → filter → save loop directly, mirroring
the MobGap output format so both engines' CSVs have compatible columns (see
[metrics/README.md — stride columns by pipeline](../metrics/README.md#stride-csv-columns-by-pipeline)).

```python
from ug3imu.pipelines import run_skdh_athome_pipeline, run_skdh_lab_pipeline

gs, ic, stride, wb, dmo = run_skdh_athome_pipeline(
    file_list=files, metadata_csv="participants.csv",
    sampling_rate_hz=100, device="AX6", output_path="results/",
)
# Functional test: same call with use_gsd=False, enable_dmo=False
# (GaitLumbar.predict() runs on the full recording — no PredictGaitLumbarLgbm bout detection)

all_ic, all_stride = run_skdh_lab_pipeline(
    imu_folder="imu/TB017/AX6_Sync/", txt_folder="mocap/TB017/V3D/",
    metadata_csv="participants.csv", sampling_rate_hz=100,
    device="AX6", output_path="results/",
)
```

| | GSD step | Gait step | DMO |
|---|----------|-----------|-----|
| At-Home (`use_gsd=True`) | `PredictGaitLumbarLgbm` (bout detection) | `GaitLumbar` | Yes (WB ≥ 10 s) |
| Functional Test (`use_gsd=False`) | — (full recording = one bout) | `GaitLumbar.predict()` directly | No |
| Lab (`run_skdh_lab_pipeline`) | — (V3D crop window via `parse_mocap_frame_window`) | `GaitLumbar` | No |

`skdh_athome_pipeline.py` also implements its own WB assembly (`_build_wb_from_strides`,
`_filter_wbs` — same ≥4-strides / ≤3.0 s-gap rules as MobGap's `WbAssembly`) and DMO aggregation
(`_compute_dmo`, same Mobilise-D thresholds as MobGap's `MobilisedAggregator`) since SKDH doesn't ship
these itself.

**Per-stage algorithm columns** (see [Running a pipeline](#per-stage-algorithm-columns) above) follow
SKDH's own architecture rather than MobGap's, since SKDH bundles GSD (context/bout detection) and ICD
(`gait_event_method`) into one `pipeline.run()` call: `gsd_algorithm` = `"SKDH_PredictGaitLumbarLgbm"`
(constant — bout boundaries don't depend on `gait_event_method`) or `"full_window"` when `use_gsd=False`
(At-Home) / always for Lab (mocap/INDIP-cropped, no real GSD step — same `"mocap_windowed"` convention
MobGap's `windowing="mocap"` uses); `icd_algorithm` = `f"SKDH_{gait_event_method}"` (varies: `"SKDH_AP CWT"`
or `"SKDH_Vertical CWT"`). `lrc_algorithm`/`cadence_algorithm`/`stride_length_algorithm`/
`walking_speed_algorithm` are all `"SKDH_GaitLumbar"` — bundled into the same call, no user-selectable
alternative; `turn_algorithm` is `"N/A"` since this pipeline doesn't do turn detection.

## File discovery (`athome_dataset_generation.py`)

```python
from ug3imu.pipelines import INPUT_FORMATS, discover_files_by_keyword, discover_athome_files
```

- **`INPUT_FORMATS`** — `{"NPZ": ".npz", "CSV": ".csv"}` registry; the GUI's **Input Format** dropdown is
  populated from this dict directly. See [Adding a new input format](../preprocessing/README.md#adding-a-new-input-format).
- **`discover_files_by_keyword(project_root, subject, keywords, device=None, file_ext=None)`** — the
  discovery function actually used by the GUI. Recursively searches `{root}/{subject}/`, works for both
  the flat lab layout (`subject/{Device}_Sync/`) and the nested at-home layout
  (`subject/{date}/{device}/At Home/`), excludes known pipeline-output subfolders (`IC/`, `stride/`, `wb/`,
  etc. — see `_OUTPUT_DIRS`), and filters out non-lumbar sensor-placement files via `_is_extra_placement`.
  `keywords` matches whole underscore-delimited tokens in the filename stem (task presets in
  `scripts/task_config.json` define these).
- **`_is_extra_placement(stem, device)`** — filters out files with a body-location suffix (`LF`, `RF`,
  `rightlowerleg`, `leftlowerleg`) appearing anywhere after the device token in the filename, so only the
  lumbar sensor file is kept when a trial has multiple sensor placements. Uses an **explicit whitelist**
  (`_PLACEMENT_LABELS`) rather than inferring from token position or alphabetic-ness — scenario/task tags
  (`home`, `monitoring`, `lab`, `gait1`, …) can legitimately appear as bare tokens after the device
  identifier too (e.g. `H100_20250625_ax6_home_gait1.npz`), and must not be mistaken for a placement
  suffix. If a new placement label shows up and gets misfiltered, add it to `_PLACEMENT_LABELS`.
- **`discover_athome_files(project_root, subject, device)`** — the older, structure-specific discovery
  function (walks `{root}/{subject}/{date}/{device}/At Home/*.csv`, requires `"athome"` in the filename).
  Superseded by `discover_files_by_keyword` for GUI use but still exported for scripts relying on the
  strict at-home layout.
## QC reports (`qc_templates.py`)

Every pipeline run writes `qc/qc_summary_*.txt`. Both MobGap and SKDH call into
`build_lab_qc_text` / `build_functional_qc_text` / `build_athome_qc_text` so the three scenarios have a
consistent block format regardless of engine — callers extract counts from their own result objects and
pass them in as keyword arguments; the QC module has no pipeline-specific knowledge.

### Filtering rules reported

| Level | Rule | At-Home | Lab | Functional Test |
|-------|------|:-------:|:---:|:---------------:|
| Stride | Duration 0.6–2.0 s | MobGap & SKDH | MobGap & SKDH | MobGap & SKDH |
| Stride | Length ≥ 0.15 m | MobGap & SKDH | MobGap & SKDH | MobGap & SKDH |
| WB | ≥ 4 strides per bout | MobGap & SKDH | — | — |
| WB | Gap between strides ≤ 3.0 s | MobGap & SKDH | — | — |
| DMO | Only WBs ≥ 10 s contribute (`DMO_MIN_WB_DURATION_S`) | MobGap & SKDH | — | — |
| Evaluation | IoU overlap ≥ 0.8 (stride matching) | — | MobGap & SKDH | — |

The IoU overlap filter is implemented in [`metrics.stride_evaluation.match_strides`](../metrics/README.md#stride-evaluation-stride_evaluationpy),
not here — `qc_templates.py` only renders the `n_iou_removed` count that evaluation passes back.

`DMO_VALUE_COLS` is the ordered list of (column, display label) pairs used to render the at-home DMO
summary block — keep this in sync with whatever columns `_compute_dmo` (SKDH) and MobGap's
`MobilisedAggregator` actually produce if you add a new DMO metric.

## Output directory layout

### Lab (per trial)

```
{OutputRoot}/{SubjectID}/{Device}/
├── IC/          *.csv                        # Initial contacts (absolute IMU frame index)
├── stride/      *_stride.csv                 # Per-stride parameters
├── wb/          *_wb.csv                     # Per walking-bout summary
├── turn/        *_turn.csv                   # Detected turns (MobGap only)
├── plot/        *_ic.png                     # Acc-norm + WB panels, mocap (green) vs IMU (red) ICs
├── qc/          qc_summary_*.txt
└── Evaluation/  ic_error_{algo}.csv, ic_metrics_{algo}.csv,
                 stride_error_{algo}.csv, stride_rmse_{algo}.csv
```

### At-Home / Functional Test (per trial)

```
{OutputRoot}/{SubjectID}/{Device}/
├── GS/       *_GS.csv            # Detected gait sequences
├── IC/       *.csv               # Raw initial contacts (MobGap + SKDH)
├── stride/   *_stride.csv        # Quality-filtered per-stride parameters
├── wb/       *_wb.csv
├── turn/     *_turn.csv          # MobGap only
├── dmo/      *_dmo.csv           # When DMO enabled
├── plot/     *_ic.png / *_wba.png
└── qc/       qc_summary_*.txt
```

## Supported algorithms

| Pipeline | Stage | Algorithm | Registry |
|----------|-------|-----------|----------|
| MobGap lab | GSD | `DummyGSD` | [lab_pipeline.py](lab_pipeline.py) |
| MobGap | GSD | `GsdIluz` (default), `GsdIonescu`, `GsdAdaptiveIonescu` | `GSD_ALGO_MAP` |
| MobGap | IC | `IcdIonescu` (default), `IcdShinImproved`, `IcdHKLeeImproved` | `ICD_ALGO_MAP` |
| MobGap | Laterality | `LrcUllrich` (default), `LrcMansour`, `LrcMcCamley` | `LRC_ALGO_MAP` |
| SKDH lab | IC | `GaitLumbar` on V3D window (no GSD) | — |
| SKDH | GSD + IC | `PredictGaitLumbarLgbm` + `GaitLumbar` | — |

### Adding a new MobGap algorithm

- **GSD**: add the class to `GSD_ALGO_MAP` in [pipeline_factory.py](pipeline_factory.py).
- **IC**: add the class to `ICD_ALGO_MAP`.
- **Laterality**: add the class to `LRC_ALGO_MAP`.
- **Other stages** (stride length, cadence, walking speed, turn detection): swap the corresponding
  argument in `create_pipeline()` — any class implementing the matching MobGap interface can be dropped
  in directly (e.g. replace `SlZijlstra()` with another `stride_length_calculation` implementation).

## Metadata CSV format

Every dataset builder in this module expects a metadata CSV (default filename
`UG3Dev-Participantbasic.csv`) with:

| Column | Description |
|--------|--------------|
| `Record ID` | Subject ID, matching the first `_`-delimited part of filenames |
| `Height (meters)` | Participant standing height |
| `AX6 height (meters)` | Sensor mounting height for AX6 |
| `Dynaport height (meters)` | Sensor mounting height for DP7 |
| `APDM3 lumbar sensor height (meters)` | Sensor mounting height for OPAL |

If a device-specific column is missing or `NaN` for a subject, the builder falls back to
`height × 0.55` and logs a warning rather than failing the whole batch.
