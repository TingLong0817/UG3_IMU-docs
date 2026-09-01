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
| [skdh_athome_pipeline.py](skdh_athome_pipeline.py) | `create_skdh_pipeline()`, `run_skdh_athome_pipeline()` — SKDH bout detection (`PredictGaitLumbarLgbm`) + `GaitLumbar`, DMO aggregation |
| [mobilised_wb.py](mobilised_wb.py) | `WBA_RULES`, `trim_edge_strides_and_summarize()` — the shared Mobilise-D-standard WB-assembly rules and first/last-stride trim, used identically by `pipeline_factory.py`, `skdh_lab_pipeline.py`, and `skdh_athome_pipeline.py` (see [Stride selection & walking-bout assembly](#stride-selection--walking-bout-assembly-mobilised_wbpy) below) |
| [qc_templates.py](qc_templates.py) | Shared QC text-block builders — MobGap and SKDH both render through these so output format is identical |

`build_dataset_from_folder()` (dataset_generation.py), `build_athome_dataset_from_files()`
(athome_dataset_generation.py), and `athome_pipeline.py` in its entirety (a MobGap at-home pipeline
predating `pipeline_factory.py`, superseded by `create_pipeline(windowing="gsd")` +
`run_pipeline_on_dataset()`) were removed as dead code — none were imported by `scripts/imu_pipeline.py`.
`legacy_code/athome_monitoring.py` still imports from the now-deleted `athome_pipeline.py`; that script is
archived/not run, so its import breaking is expected, not a regression.

## Unified engine — GSD / Gait / Turn algorithms freely combinable

`unified_engine.run_unified_pipeline(dataset, config, output_path, ...)` is a single entry point that
supersedes `run_pipeline_on_dataset` (MobGap) + `run_skdh_lab_pipeline` + `run_skdh_athome_pipeline`
(SKDH). It lets a MobGap stage and an SKDH stage be mixed in one run — e.g. **MobGap `GsdIluz` for
gait-sequence detection + SKDH `GaitLumbar` (`AP CWT`) for IC detection and per-stride parameters**.

| File | Role |
|------|------|
| [stage_registry.py](stage_registry.py) | `PipelineConfig` dataclass, `GSD_ENGINES` / `GAIT_ENGINES` registries, `PRESETS` (unified shape), `resolve_algorithm_name()`, `stage_algorithm_values()` |
| [unified_engine.py](unified_engine.py) | `run_unified_pipeline()`, `detect_gsd()`, `_ReplayGSD`, `expand_configs()` |

### Stage model & the one hard constraint

| Stage | `mobgap` engine | `skdh` engine |
|-------|-----------------|---------------|
| **GSD** (`windowing="gsd"`) | `GsdIluz` / `GsdIonescu` / `GsdAdaptiveIonescu` | `PredictGaitLumbarLgbm` |
| **Gait** = IC + laterality + cadence + stride length + walking speed | `IcdIonescu`/`IcdShinImproved`/`IcdHKLeeImproved` × `LrcUllrich`/`LrcMansour`/`LrcMcCamley`, then `CadFromIc` / `SlZijlstra` / `WsNaive` | `GaitLumbar(gait_event_method="AP CWT" \| "Vertical CWT")` — one atomic block |
| **Turn** | `TdElGohary` (or off) — independent of the gait engine | ″ |
| Stride selection + WB assembly + first/last-stride trim + DMO | fixed, shared ([mobilised_wb.py](mobilised_wb.py)) | ″ |

`windowing="ref"` (mocap / INDIP crop) and `windowing="full"` (whole recording) replace the GSD
detector with a single fixed window, exactly as `DummyGSD` / `FullWindowGSD` did.

**Why "Gait" is one block, not four:** SKDH's `GaitLumbar` detects its own initial contacts and
computes laterality / cadence / stride length / walking speed in a single `predict()` call and does
**not** accept externally supplied ICs. So "use SKDH's IC timing but MobGap's `SlZijlstra`" is not
possible; picking the gait engine picks the whole IC→parameters block. GSD and Turn stay freely
mixable because both sides expose them as standalone windows-in / table-out steps.

### How the two engines are bridged

1. The GSD stage always runs first → a canonical `gs_list` DataFrame (`start` / `end` IMU frame
   indices, `gs_id`-named index).
2. **`gait_engine="mobgap"`** — `gs_list` is replayed into `GenericMobilisedPipeline` via
   `_ReplayGSD` (a stand-in GSD object whose `detect()` just returns the pre-computed list, keyed by
   `dp_group`). MobGap's own per-GS iteration then runs unchanged, so this path is byte-identical to
   the old `create_pipeline` + `run_pipeline_on_dataset` when the GSD engine is also MobGap.
3. **`gait_engine="skdh"`** — `skdh.gait.GaitLumbar(min_bout_time=0.0, max_bout_separation_time=3.0)
   .predict(..., gait_bouts=[[0, len]], gait_pred=False)`, run **once per GSD window** on a
   `± 2 s`-padded crop of the recording (ICs outside the true window are discarded afterwards; the pad
   only gives `GaitLumbar`'s CWT clean edges). Per-window isolation means one bad window can't abort the
   others. `min_bout_time=0.0` disables SKDH's own 8 s bout floor so the GSD stage's windows stay
   authoritative (plan decision D1); `max_bout_separation_time=3.0` matches the shared
   `MaxBreakCriteria(max_break_s=3)` downstream (leaving it at `0` fragments each window at every turn
   or brief pause). The shared Mobilise-D tail (`StrideSelection` → `WbAssembly(WBA_RULES)` →
   `trim_edge_strides_and_summarize` → `_compute_dmo`) then runs exactly as for MobGap.

   **Known limitation:** `GaitLumbar`'s CWT IC detector (`AP CWT` / `Vertical CWT`) gates peaks on a
   *bout-global* prominence (`k · std` of the CWT coefficients over the whole window) and can go silent
   for several seconds in low-amplitude / near-stationary gait — a gap that then trips the 3 s WB break
   and fragments the bout. MobGap's threshold-free `IcdIonescu` does not have this failure mode, so on
   an *identical* MobGap GSD, SKDH gait tends to match fewer INDIP CWP bouts than MobGap gait. It can be
   reduced by lowering `GaitLumbar`'s `ic_prom_factor` / `fc_prom_factor` (AP CWT only; ~0.3 recovers
   most dropped ICs without inflating spurious strides) — currently hard-coded at SKDH's `0.6` default in
   `_run_skdh_gait`. See
   [metrics/README.md — Why At-Home matched-WB counts run low](../metrics/README.md#why-at-home-matched-wb-counts-run-low).

Accelerometer data is loaded once (`build_dataset_from_file_list` → `dp.data_ss`, m/s²); the SKDH
calls get `g` by dividing by 9.80665 — no second file read, so GSD frame indices always line up with
what `GaitLumbar` sees.

### Per-stage algorithm columns / output layout

Unchanged from the split pipelines — see
[Per-stage algorithm columns](#per-stage-algorithm-columns) and
[Output directory layout](#output-directory-layout) below. `stage_algorithm_values(config)` fills
`gsd_algorithm` / `icd_algorithm` / … with the same conventions
(`"GsdIluz"`, `"SKDH_AP CWT"`, `"SKDH_PredictGaitLumbarLgbm"`, `"mocap_windowed"`, `"full_window"`,
`"TdElGohary"`, …), so a cross-engine run such as `gsd_algorithm="GsdIluz"` +
`icd_algorithm="SKDH_AP CWT"` is grouped/deduplicated correctly by the evaluation code with no
changes there.

### Usage

```python
from ug3imu.pipelines import (
    build_dataset_from_file_list, run_unified_pipeline, PipelineConfig, expand_configs,
)

dataset = build_dataset_from_file_list(file_list=files, metadata_csv="participants.csv",
                                       sampling_rate_hz=100, device="AX6")

cfg = PipelineConfig(
    windowing="gsd",
    gsd_engine="mobgap", gsd_algorithm="GsdIluz",     # MobGap GSD
    gait_engine="skdh",  icd_algorithm="AP CWT",       # SKDH IC + parameters
    turn=True, enable_dmo=True,
)
run_unified_pipeline(dataset, cfg, output_path="results/TB017/AX6", imu_fs=100)

# batch: cartesian-expand "All" selections
for cfg in expand_configs(windowing="gsd", gsd_engine="mobgap",
                          gsd_algorithms=["GsdIluz", "GsdIonescu"],
                          gait_engine="mobgap",
                          icd_algorithms=["IcdIonescu", "IcdShinImproved"],
                          lrc_algorithms="LrcUllrich", turn=True, enable_dmo=False):
    run_unified_pipeline(dataset, cfg, output_path="results/TB017/AX6", imu_fs=100)
```

### Status

`run_unified_pipeline` is now the only pipeline entry point. `scripts/imu_pipeline.py` has a single
**Pipeline** tab (was MobGap + SKDH tabs) with a per-stage engine × algorithm selector, presets, and
"All" batch expansion (`expand_configs`). `ug3imu.pipelines` no longer exports
`run_skdh_lab_pipeline` / `run_skdh_athome_pipeline` / `create_skdh_pipeline`; those modules stay only
as the home of helpers the engine reuses (`gait_lumbar_df_to_stride_df`, `_ic_time_to_abs_s`,
`_compute_dmo`, `_skdh_athome_qc_kwargs`, `_plot_ic_overlay`). `create_pipeline` /
`run_pipeline_on_dataset` remain exported as the low-level MobGap builder the engine calls internally.
`legacy_code/athome_monitoring.py` imports the removed `run_skdh_athome_pipeline`; that script is
archived / not run, so the broken import is expected, not a regression.

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

### Stride selection & walking-bout assembly (`mobilised_wb.py`)

Every scenario (Lab, At-Home, Functional Test) and both engines (MobGap, SKDH) select strides and assemble
walking bouts (WBs) using the same rules, matching the official Mobilise-D / INDIP validation-paper
definition exactly — this is what makes results directly comparable to INDIP. Not currently exposed as a
GUI setting.

**1. Stride selection** — `mobgap.wba.StrideSelection()`, mobgap's own unmodified `"mobilised"` default:

- Duration: `0.2–3.0 s`
- Length: `≥ 0.15 m`
- No cadence rule (an earlier, non-standard 0.6–2.0 s + 60–200 spm rule used to apply to Lab/Functional
  Test only — removed so every scenario now uses the identical, standard-compliant rule)

**2. Walking-bout assembly** — `mobgap.wba.WbAssembly(rules=WBA_RULES)`, also mobgap's own unmodified
`"mobilised"` default rules:

- `NStridesCriteria(min_strides=4, min_strides_left=3, min_strides_right=3)` — a WB needs ≥4 strides total
  to be included. (`min_strides_left`/`min_strides_right` are never actually evaluated — mobgap's own
  inclusion check short-circuits on `min_strides` alone whenever it's set — kept only for parity with
  mobgap's own default.)
- `MaxBreakCriteria(max_break_s=3)` — a gap of >3.0 s between consecutive same-side strides ends the WB.

Left/right stride sequences are built and broken on gaps internally by `WbAssembly` itself — this module
just supplies the rule thresholds (`WBA_RULES`) and calls mobgap's classes directly, so MobGap and SKDH
produce byte-identical WBs from the same stride list.

**3. First/last-stride trim** — the standard also discards the first and last stride of every WB (they're
transition strides, not reliable for per-stride parameters like cadence/length/speed). mobgap's own
`WbAssembly` only has a mechanism for dropping the *last* one (`MaxBreakCriteria(remove_last_ic=True)`), and
that mechanism has a real bug in the installed version: when the extra trim collapses a preliminary WB to
zero/negative length (common with short, isolated At-Home bouts), `WbAssembly.assemble()` crashes inside its
own exclusion-reason bookkeeping. So `remove_last_ic` stays at mobgap's default (`False`), and
`trim_edge_strides_and_summarize()` does both edges as its own post-processing step on top of `WbAssembly`'s
unmodified output instead:

- The returned per-stride table (→ `stride.csv`) excludes the first and last stride of every WB.
- A WB's own `start`/`end`/`duration_s` still span the **full, untrimmed** stride sequence (matching
  mobgap's own `wb_meta_parameters_`) — only `n_strides` and the mean of the per-WB gait parameters are
  computed from the trimmed interior strides. This matters for At-Home GSD-detection evaluation, which
  matches these WB boundaries against INDIP's CWP reference by time overlap — narrowing the boundary by the
  edge strides' own duration would make every WB look artificially shorter/later-starting than it actually
  was, and a WB is never dropped outright just because few strides remain after trimming.

`create_pipeline()` wires `stride_selection=StrideSelection()` and `wba=WbAssembly(rules=WBA_RULES)` into
`GenericMobilisedPipeline`; `run_pipeline_on_dataset()` calls `trim_edge_strides_and_summarize()` right after
`pipeline.run(dp)` and overwrites `res.per_stride_parameters_`/`res.per_wb_parameters_` in place (also
re-running the DMO threshold mask + aggregation against the trimmed WBs when DMO is enabled, mirroring what
`GenericMobilisedPipeline.run()` does internally). `skdh_lab_pipeline.py` and `skdh_athome_pipeline.py` call
`StrideSelection()` → `WbAssembly(rules=WBA_RULES)` → `trim_edge_strides_and_summarize()` on their own
GaitLumbar-derived stride tables the same way. SKDH At-Home pools strides from every bout
`PredictGaitLumbarLgbm` detects (rather than keeping SKDH's own per-bout grouping as the final WB boundary)
and lets `WbAssembly`'s own break rule re-segment them — this mirrors how MobGap decouples its GSD stage
(candidate windows) from WB assembly (the final bout definition), and means a Lab trial can also now
legitimately split into more than one WB if it contains a >3 s pause.

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

`skdh_athome_pipeline.py`/`skdh_lab_pipeline.py` call mobgap's own `StrideSelection`/`WbAssembly` directly
for stride selection and WB assembly (see
[Stride selection & walking-bout assembly](#stride-selection--walking-bout-assembly-mobilised_wbpy) above) —
SKDH itself has no equivalent step. DMO aggregation (`_compute_dmo`, At-Home only) is still SKDH-specific
(same Mobilise-D thresholds as MobGap's `MobilisedAggregator`), since SKDH doesn't ship that either.

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
| Stride | Duration 0.2–3.0 s | MobGap & SKDH | MobGap & SKDH | MobGap & SKDH |
| Stride | Length ≥ 0.15 m | MobGap & SKDH | MobGap & SKDH | MobGap & SKDH |
| WB | ≥ 4 strides per bout | MobGap & SKDH | MobGap & SKDH | MobGap & SKDH |
| WB | Gap between strides ≤ 3.0 s | MobGap & SKDH | MobGap & SKDH | MobGap & SKDH |
| WB | First/last stride excluded from parameters (full extent kept for `start`/`end`) | MobGap & SKDH | MobGap & SKDH | MobGap & SKDH |
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
