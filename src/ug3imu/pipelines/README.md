# `ug3imu.pipelines`

Builds MobGap/SKDH-compatible datasets from raw IMU files, runs gait detection through **one configurable
engine** (`run_unified_pipeline`), and writes results + QC reports in a consistent directory layout across
all three scenarios (Lab, At-Home, Functional Test) and both engines (MobGap, SKDH).

[← Back to project README](../../../README.md)

## Files

| File | Role |
|------|------|
| [stage_registry.py](stage_registry.py) | `PipelineConfig`, `GSD_ENGINES` / `GAIT_ENGINES` registries, `PRESETS`, `resolve_algorithm_name()`, `stage_algorithm_values()`, `param_override_tag()` |
| [unified_engine.py](unified_engine.py) | **Public entry point** — `run_unified_pipeline()`, `detect_gsd()`, `expand_configs()`, `record_param_run()` |
| [pipeline_factory.py](pipeline_factory.py) | Low-level MobGap builder the engine calls — `create_pipeline()`, `run_pipeline_on_dataset()`, `GSD_ALGO_MAP` / `ICD_ALGO_MAP` / `LRC_ALGO_MAP`, `FullWindowGSD` |
| [dataset_generation.py](dataset_generation.py) | `build_dataset_from_file_list()` — dataset builder used by the GUI (lab, at-home, and functional test alike) |
| [athome_dataset_generation.py](athome_dataset_generation.py) | `INPUT_FORMATS` registry, file discovery (`discover_files_by_keyword`, `discover_athome_files`) |
| [lab_pipeline.py](lab_pipeline.py) | `DummyGSD` — treats a fixed crop window as one gait sequence (`windowing="ref"`) |
| [skdh_lab_pipeline.py](skdh_lab_pipeline.py) | SKDH output-conversion helpers reused by the engine — `gait_lumbar_df_to_stride_df()`, `_ic_time_to_abs_s()` |
| [skdh_athome_pipeline.py](skdh_athome_pipeline.py) | More SKDH helpers — `_compute_dmo()`, `_skdh_athome_qc_kwargs()`, `_plot_ic_overlay()` |
| [mobilised_wb.py](mobilised_wb.py) | `WBA_RULES`, `trim_edge_strides_and_summarize()` — shared Mobilise-D WB-assembly rules + first/last-stride trim, used by both gait paths |
| [qc_templates.py](qc_templates.py) | Shared QC text-block builders, so MobGap and SKDH output format is identical |

Exports: `build_dataset_from_file_list`, `INPUT_FORMATS`, `discover_files_by_keyword`,
`discover_athome_files`, `run_unified_pipeline`, `PipelineConfig`, `expand_configs`, `detect_gsd`,
`PRESETS`, `GSD_ENGINES`, `GAIT_ENGINES`, `SKDH_METHOD_SUFFIX`, `stage_algorithm_values`,
`resolve_algorithm_name`, `param_override_tag`, `record_param_run`, plus the low-level `create_pipeline` /
`run_pipeline_on_dataset` / `FullWindowGSD` / `DummyGSD`.

## Unified engine — GSD / Gait / Turn algorithms freely combinable

`run_unified_pipeline(dataset, config, output_path, ...)` is the one entry point for every scenario. It
lets a MobGap stage and an SKDH stage be mixed in one run — e.g. **MobGap `GsdIluz` for gait-sequence
detection + SKDH `GaitLumbar` (`AP CWT`) for IC detection and per-stride parameters**.

### Stage model & the one hard constraint

| Stage | `mobgap` engine | `skdh` engine |
|-------|-----------------|---------------|
| **GSD** (`windowing="gsd"`) | `GsdIluz` / `GsdIonescu` / `GsdAdaptiveIonescu` | `PredictGaitLumbarLgbm` |
| **Gait** = IC + laterality + cadence + stride length + walking speed | `IcdIonescu`/`IcdShinImproved`/`IcdHKLeeImproved` × `LrcUllrich`/`LrcMansour`/`LrcMcCamley`, then `CadFromIc` / `SlZijlstra` / `WsNaive` | `GaitLumbar(gait_event_method="AP CWT" \| "Vertical CWT")` — one atomic block |
| **Turn** | `TdElGohary` (or off) — independent of the gait engine | ″ |
| Stride selection + WB assembly + trim + DMO | fixed, shared ([mobilised_wb.py](mobilised_wb.py)) | ″ |

`windowing="ref"` (mocap / INDIP crop) and `windowing="full"` (whole recording) replace the GSD detector
with a single fixed window.

**Why "Gait" is one block, not four:** SKDH's `GaitLumbar` detects its own initial contacts and computes
laterality / cadence / stride length / walking speed in one `predict()` call, and doesn't accept externally
supplied ICs — so picking the gait engine picks the whole IC→parameters block. GSD and Turn stay freely
mixable because both expose a plain windows-in / table-out interface on either side.

### How the two engines are bridged

1. The GSD stage always runs first → a canonical `gs_list` DataFrame (`start`/`end` IMU frame indices).
2. **`gait_engine="mobgap"`** — `gs_list` is replayed into `GenericMobilisedPipeline` via `_ReplayGSD` (a
   stand-in GSD object whose `detect()` returns the pre-computed list). MobGap's own per-GS iteration then
   runs unchanged.
3. **`gait_engine="skdh"`** — `GaitLumbar(min_bout_time=0.0, max_bout_separation_time=3.0).predict(...)`
   runs once per GSD window on a `±2 s`-padded crop (ICs outside the true window are discarded after; the
   pad only gives the CWT clean edges). `min_bout_time=0.0` disables SKDH's own 8 s bout floor so the GSD
   stage's windows stay authoritative; `max_bout_separation_time=3.0` matches the shared
   `MaxBreakCriteria(max_break_s=3)` downstream. The shared Mobilise-D tail (`StrideSelection` →
   `WbAssembly(WBA_RULES)` → `trim_edge_strides_and_summarize` → `_compute_dmo`) then runs exactly as for
   MobGap.

**Known limitation:** `GaitLumbar`'s CWT IC detector gates peaks on a *bout-global* prominence and can go
silent for several seconds in low-amplitude/near-stationary gait, fragmenting the bout at the resulting >3 s
gap. MobGap's threshold-free `IcdIonescu` doesn't have this failure mode, so SKDH gait tends to match fewer
INDIP CWP bouts than MobGap gait on an identical GSD. Reducible via `ic_prom_factor`/`fc_prom_factor` (AP
CWT only, GUI-tunable — see [Tunable SKDH/MobGap parameters](#tunable-skdhmobgap-parameters)). Details:
[metrics/README.md — Why At-Home matched-WB counts run low](../metrics/README.md#why-at-home-matched-wb-counts-run-low).

Accelerometer data is loaded once (`build_dataset_from_file_list` → `dp.data_ss`, m/s²); SKDH calls get
`g` by dividing by 9.80665 — no second file read, so GSD frame indices always match what `GaitLumbar` sees.

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

`scripts/imu_pipeline.py` has a single **Pipeline** tab with a per-stage engine × algorithm selector,
presets, "All" batch expansion (`expand_configs`), and a "⚙ Params" dialog per gait engine for the tunable
knobs below.

## Tunable SKDH/MobGap parameters

Seven numeric/boolean knobs on `PipelineConfig` (default `None` = "use the library's own class default",
except `skdh_height_factor`, a real float since SKDH has no `None` sentinel for it):

| Field | Reaches | Only applies when | Class default |
|-------|---------|--------------------|----------------|
| `skdh_ic_prom_factor` | `GaitLumbar`'s `ApCwtGaitEvents` substep | `icd_algorithm == "AP CWT"` | 0.6 |
| `skdh_fc_prom_factor` | same substep | same as above | 0.6 |
| `skdh_height_factor` | `GaitLumbar` (leg-length ≈ `height_factor * height_m`) | always, `gait_engine == "skdh"` | 0.53 |
| `skdh_wavelet_scale` | `GaitLumbar`'s `VerticalCwtGaitEvents` substep | `icd_algorithm == "Vertical CWT"` | adaptive, per-bout |
| `skdh_use_cwt_scale_relation` | same substep | same as above | `True` (adaptive) |
| `skdh_max_stride_time` | `GaitLumbar`'s `CreateStridesAndQc` substep | either `icd_algorithm` (this substep always runs) | adaptive (`2.0 * mean_step_time + 1.0`) |
| `mobgap_step_length_scaling_factor` | MobGap `SlZijlstra` | `gait_engine == "mobgap"` | 1.14675 |

`ic_prom_factor`/`fc_prom_factor` and `wavelet_scale`/`use_cwt_scale_relation` only reach the AP-CWT /
Vertical-CWT branch they're named after (`GaitLumbar.__init__` dispatches them inside an `if/elif` on
`gait_event_method`) — setting one under the other method is accepted but has zero effect. `max_stride_time`
is different: `CreateStridesAndQc` runs unconditionally after that `if/elif`, so it's the one knob that
affects either SKDH IC-detection method.

**Only two SKDH IC-detection algorithms exist** (`stage_registry.GAIT_ENGINES["skdh"]["icd"]`) — `"AP CWT"`
and `"Vertical CWT"`. Three preset variants that used to exist here (fixed combinations of the knobs above,
e.g. `ic_prom_factor=0.3`) were removed: picking a preset and then typing a value into the params dialog
could silently do nothing whenever the typed value equalled the *unrelated* class default. Reproduce an old
preset by picking the plain algorithm and dialing the same numbers into the params dialog (see
`stage_registry.SKDH_METHOD_SUFFIX`'s comment for the removed values).

**No filename collisions** — `param_override_tag(config)` returns `""` when every knob is at its default
(so default runs keep today's filenames) and otherwise `"_p" + <8 hex chars>` hashed from the overridden
values, appended to `algorithm_name`. Two runs with the same gsd/icd/lrc choice but different knob values
therefore never overwrite each other.

**Self-describing output** — `stage_algorithm_values(config)` writes every knob (real value, or the literal
string `"default"` when unset) into every `ic`/`stride`/`wb` CSV row, same convention as `gsd_algorithm` /
`icd_algorithm`. `record_param_run(output_path, config, algorithm_name)` additionally upserts one row per
distinct `algorithm_name` into `{output_path}/param_runs.csv`, a small per-subject/device ledger for
browsing history.

**GUI** — each gait engine's algorithm row has a "⚙ Params" button opening a small popup
(`imu_pipeline._open_param_settings`) with the relevant knobs. Getters (`_skdh_ic_prom_factor()` etc.)
return `None` on blank/unparseable input, matching `PipelineConfig`'s "unset" sentinel.

**report_app** — `_split_algo_cols(df)` derives a "Params" column next to every "Algorithm" column
(`_params_summary_col`), and every tab (IC/Stride/WB/GSD) opens with an `_algo_param_filter` multiselect so
a parameter sweep can be narrowed to specific combinations within the tab. See
[REPORT_APP_DOC.md](../../../scripts/REPORT_APP_DOC.md).

Evaluation functions that group/dedup results by algorithm identity (`ug3imu.indip`,
`metrics.ic_evaluation`, `metrics.wb_evaluation`, `metrics.stride_evaluation`, `metrics.gsd_evaluation`)
fold a param tag into that key too, so parameter-sweep runs never collide or lose their identity — see
[NEWS.md](../../../NEWS.md) for the fixes that got this working end to end.

## Stride selection & walking-bout assembly (`mobilised_wb.py`)

Every scenario and both engines select strides and assemble walking bouts (WBs) using the same rules,
matching the Mobilise-D / INDIP validation-paper definition exactly. Not currently exposed as a GUI
setting.

**1. Stride selection** — `mobgap.wba.StrideSelection()`, unmodified `"mobilised"` default: duration
0.2–3.0 s, length ≥ 0.15 m, no cadence rule.

**2. Walking-bout assembly** — `mobgap.wba.WbAssembly(rules=WBA_RULES)`, also unmodified `"mobilised"`
defaults: `NStridesCriteria(min_strides=4, ...)` (≥4 strides per WB) and `MaxBreakCriteria(max_break_s=3)`
(gap >3.0 s between consecutive same-side strides ends the WB).

**3. First/last-stride trim** — the standard also discards the first/last stride of every WB (transition
strides, unreliable for per-stride parameters). mobgap's own `WbAssembly(remove_last_ic=True)` has a real
crash bug in the installed version when the extra trim collapses a short WB to zero/negative length, so
`remove_last_ic` stays `False` and `trim_edge_strides_and_summarize()` does both edges as a post-processing
step instead:
- The per-stride table (`stride.csv`) excludes the first and last stride of every WB.
- A WB's own `start`/`end`/`duration_s` still span the full, untrimmed stride sequence — only `n_strides`
  and the per-WB parameter means come from the trimmed interior strides. This keeps At-Home GSD-detection
  evaluation (which matches WB boundaries against INDIP's CWP reference by time overlap) from seeing
  artificially narrow bouts.

Both engines pool strides across all GSD windows and let `WbAssembly`'s own break rule define the final WB
boundaries (rather than treating a GSD window as one WB), so a Lab trial with a >3 s pause can legitimately
split into more than one WB.

## PRESETS

`PRESETS` (in [stage_registry.py](stage_registry.py)) is the single source of truth the GUI reads to
populate the preset buttons — one entry per preset, in `PipelineConfig` shape plus a few GUI-only flags:

```python
PRESETS = {
    "At-Home": {
        "windowing": "gsd", "gsd_engine": "mobgap", "gsd_algorithm": "GsdIluz",
        "gait_engine": "mobgap", "icd_algorithm": "IcdIonescu", "lrc_algorithm": "LrcUllrich",
        "turn": True, "enable_dmo": True,
        "evaluation": False, "plot_wb": True, "plot_ic": False,
    },
    "Lab":             {"windowing": "ref",  "enable_dmo": False, "evaluation": True,  ...},
    "Functional Test": {"windowing": "full", "enable_dmo": False, "evaluation": False, ...},
}
```

`_apply_preset(name)` in `scripts/imu_pipeline.py` copies these values into the GUI's state variables and
refreshes which conditional rows are shown.

## Building a dataset

```python
from ug3imu.pipelines import build_dataset_from_file_list, run_unified_pipeline, PipelineConfig

dataset = build_dataset_from_file_list(
    file_list=files, metadata_csv="participants.csv",
    sampling_rate_hz=100, device="AX6",
    mocap_folder="mocap/TB017/V3D/",   # omit for at-home / functional test
)
```

Format/layout-agnostic — works for flat lab folders, nested at-home folders, or a mixed file list, because
the caller (usually `discover_files_by_keyword`) has already resolved which files to include. Per file:

1. Subject ID = first `_`-split token of the stem; looked up in `metadata_csv` (must contain `Record ID`,
   `Height (meters)`, and a device-specific height column — see [Metadata CSV format](#metadata-csv-format)
   below). Missing sensor height falls back to `height × 0.55` with a logged warning.
2. IMU data loaded via [`preprocessing.load_imu_for_mobgap`](../preprocessing/README.md).
3. If `mocap_folder` is given, the file's 4-part trial key is matched against V3D `.txt` files to get a
   crop window (`parse_mocap_frame_window`, see [`ug3imu.mocap`](../mocap/README.md)); if `frame_windows`
   is given instead (pre-built `{key4: (start, end)}`, e.g. from
   [`indip.build_frame_windows_for_files`](../indip/README.md)), those frames are used directly and take
   priority. Files with neither fall back to the full recording as the window.

### Per-stage algorithm columns

Besides `algorithm_name` (the full pipeline identity baked into the output filename, e.g.
`GsdIluz_IcdIonescu_LrcUllrich`), every row of `gs`/`ic`/`stride`/`wb`/`turn` output also carries one column
per pipeline stage — `gsd_algorithm`, `icd_algorithm`, `lrc_algorithm`, `cadence_algorithm`,
`stride_length_algorithm`, `walking_speed_algorithm`, `turn_algorithm` — filled by
`stage_registry.stage_algorithm_values(config)`. For MobGap, `cadence`/`stride_length`/`walking_speed`
aren't user-selectable so those columns are constant; for SKDH they're all `SKDH_GaitLumbar`.

This exists because different evaluation questions depend on different, smaller subsets of the full
pipeline than the flat `algorithm_name` implies:
- GSD-detection accuracy only depends on the GSD stage.
- IC-detection accuracy depends on GSD + ICD, not LRC/etc.
- Per-stride/per-bout parameter accuracy depends on the whole chain.

Evaluation functions that only care about a result's GSD or IC identity read these columns directly and
deduplicate down to one file per `(trial, gsd_algorithm)` or `(trial, gsd_algorithm, icd_algorithm)` —
otherwise the same GSD (or IC) result would get evaluated once per ICD/LRC combination it was tested
alongside. See [metrics/gsd_evaluation.py](../metrics/README.md#gsd--wb-evaluation-for-at-home-gsd_evaluationpy)
and [indip.batch_ic_analysis_indip_athome](../indip/README.md).

`windowing != "gsd"` still gets a `gsd_algorithm` value rather than a blank one: `"mocap_windowed"` for
`windowing="ref"`, `"full_window"` for `windowing="full"`. When the gait engine is SKDH, `icd_algorithm` is
`f"SKDH_{gait_event_method}"` and `lrc_algorithm`/`cadence_algorithm`/`stride_length_algorithm`/
`walking_speed_algorithm` are all `"SKDH_GaitLumbar"`.

## Internals: the three-mode MobGap builder

`create_pipeline(windowing=...)` in `pipeline_factory.py` builds a `mobgap.pipeline.GenericMobilisedPipeline`
that differs only in how GSD is configured — this is what `run_unified_pipeline` calls into for the MobGap
gait path, and it's also usable directly for low-level/advanced scripting.

| `windowing` | GSD component | Use case |
|-------------|----------------|----------|
| `"gsd"` | `_ReplayGSD` replaying a pre-computed `gs_list` (from any engine), or a `GSD_ALGO_MAP` class via `gsd_override` | At-Home: auto-detected bout boundaries |
| `"mocap"` | `DummyGSD` ([lab_pipeline.py](lab_pipeline.py)) — one window = the mocap/INDIP crop | Lab: no detection |
| `"full"` | `FullWindowGSD` (`pipeline_factory.py`) | Functional Test: entire recording = one window |

`create_pipeline` also takes `gsd_override=`: when set, that object is used as the GSD component directly —
how `_ReplayGSD` injects a `gs_list` produced by any engine into MobGap's per-GS iteration.

For the SKDH gait path, `unified_engine._run_skdh_gait()` has no `GenericMobilisedPipeline`-equivalent
abstraction to call into — `GaitLumbar.predict()` bundles IC detection, laterality, cadence, stride length
and walking speed into one atomic call. The unified engine feeds it the GSD stage's windows via
`gait_bouts=` (one call per window), converts the per-IC output to a `start`/`end` stride table with
`gait_lumbar_df_to_stride_df`, then runs the same shared Mobilise-D tail as MobGap. Output columns mirror
the MobGap format so both engines' CSVs are directly comparable (see
[metrics/README.md — stride columns by pipeline](../metrics/README.md#stride-csv-columns-by-pipeline)).

## File discovery (`athome_dataset_generation.py`)

```python
from ug3imu.pipelines import INPUT_FORMATS, discover_files_by_keyword, discover_athome_files
```

- **`INPUT_FORMATS`** — `{"NPZ": ".npz", "CSV": ".csv"}` registry; the GUI's **Input Format** dropdown reads
  this directly. See [Adding a new input format](../preprocessing/README.md#adding-a-new-input-format).
- **`discover_files_by_keyword(project_root, subject, keywords, device=None, file_ext=None)`** — discovery
  function actually used by the GUI. Recursively searches `{root}/{subject}/`, works for both the flat lab
  layout and the nested at-home layout, excludes known pipeline-output subfolders (`IC/`, `stride/`, `wb/`,
  etc.), and filters out non-lumbar sensor-placement files via `_is_extra_placement`. `keywords` matches
  whole underscore-delimited tokens in the filename stem (task presets in `scripts/task_config.json` define
  these).
- **`_is_extra_placement(stem, device)`** — filters out files with a body-location suffix (`LF`, `RF`,
  `rightlowerleg`, `leftlowerleg`) after the device token, using an explicit whitelist (`_PLACEMENT_LABELS`)
  rather than inferring from token position, since scenario/task tags (`home`, `gait1`, …) can also appear
  as bare tokens there.
- **`discover_athome_files(project_root, subject, device)`** — older, structure-specific discovery
  (`{root}/{subject}/{date}/{device}/At Home/*.csv`, requires `"athome"` in the filename). Superseded by
  `discover_files_by_keyword` for GUI use, kept for scripts relying on the strict at-home layout.

## QC reports (`qc_templates.py`)

Every pipeline run writes `qc/qc_summary_*.txt`. Both MobGap and SKDH call
`build_lab_qc_text`/`build_functional_qc_text`/`build_athome_qc_text` so the three scenarios share one
block format regardless of engine — callers extract counts from their own result objects and pass them in
as keyword arguments.

### Filtering rules reported

| Level | Rule | At-Home | Lab | Functional Test |
|-------|------|:-------:|:---:|:---------------:|
| Stride | Duration 0.2–3.0 s | ✓ | ✓ | ✓ |
| Stride | Length ≥ 0.15 m | ✓ | ✓ | ✓ |
| WB | ≥ 4 strides per bout | ✓ | ✓ | ✓ |
| WB | Gap between strides ≤ 3.0 s | ✓ | ✓ | ✓ |
| WB | First/last stride excluded from parameters | ✓ | ✓ | ✓ |
| DMO | Only WBs ≥ 10 s contribute (`DMO_MIN_WB_DURATION_S`) | ✓ | — | — |
| Evaluation | IoU overlap ≥ 0.8 (stride matching) | — | ✓ | — |

(All rules apply identically to MobGap and SKDH.) The IoU overlap filter itself lives in
[`metrics.stride_evaluation.match_strides`](../metrics/README.md#stride-evaluation-stride_evaluationpy) —
`qc_templates.py` only renders the `n_iou_removed` count evaluation passes back.

`DMO_VALUE_COLS` is the ordered `(column, display label)` list used to render the at-home DMO summary block
— keep in sync with whatever columns `_compute_dmo` (SKDH) / MobGap's `MobilisedAggregator` produce if you
add a new DMO metric.

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

| Stage | Engine | Algorithms | Registry |
|-------|--------|------------|----------|
| GSD | `mobgap` | `GsdIluz` (default), `GsdIonescu`, `GsdAdaptiveIonescu` | `GSD_ALGO_MAP` / `stage_registry.GSD_ENGINES` |
| GSD | `skdh` | `PredictGaitLumbarLgbm` | `stage_registry.GSD_ENGINES` |
| GSD | — | `windowing="ref"` → `DummyGSD`; `windowing="full"` → `FullWindowGSD` | [lab_pipeline.py](lab_pipeline.py) / [pipeline_factory.py](pipeline_factory.py) |
| Gait — IC | `mobgap` | `IcdIonescu` (default), `IcdShinImproved`, `IcdHKLeeImproved` | `ICD_ALGO_MAP` / `stage_registry.GAIT_ENGINES` |
| Gait — laterality | `mobgap` | `LrcUllrich` (default), `LrcMansour`, `LrcMcCamley` | `LRC_ALGO_MAP` |
| Gait — cadence / stride length / walking speed | `mobgap` | `CadFromIc` / `SlZijlstra` / `WsNaive` (fixed) | `create_pipeline()` args |
| Gait — IC + laterality + all parameters | `skdh` | `GaitLumbar(gait_event_method="AP CWT" \| "Vertical CWT")` — one atomic block | `stage_registry.GAIT_ENGINES` |
| Turn | `mobgap` | `TdElGohary` (or off) | `create_pipeline()` / `unified_engine` |

### Adding a new MobGap algorithm

- **GSD**: add the class to `GSD_ALGO_MAP` in [pipeline_factory.py](pipeline_factory.py) **and** to the
  `"mobgap"` list in `stage_registry.GSD_ENGINES`.
- **IC**: add to `ICD_ALGO_MAP` **and** `stage_registry.GAIT_ENGINES["mobgap"]["icd"]`.
- **Laterality**: add to `LRC_ALGO_MAP` **and** `stage_registry.GAIT_ENGINES["mobgap"]["lrc"]`.
- **Other stages** (stride length, cadence, walking speed, turn detection): swap the corresponding argument
  in `create_pipeline()` — any class implementing the matching MobGap interface can be dropped in directly.

## Metadata CSV format

Every dataset builder in this module expects a metadata CSV (default `UG3Dev-Participantbasic.csv`) with:

| Column | Description |
|--------|--------------|
| `Record ID` | Subject ID, matching the first `_`-delimited part of filenames |
| `Height (meters)` | Participant standing height |
| `AX6 height (meters)` | Sensor mounting height for AX6 |
| `Dynaport height (meters)` | Sensor mounting height for DP7 |
| `APDM3 lumbar sensor height (meters)` | Sensor mounting height for OPAL |

If a device-specific column is missing or `NaN` for a subject, the builder falls back to `height × 0.55`
and logs a warning rather than failing the whole batch.
