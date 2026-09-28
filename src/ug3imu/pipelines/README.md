# `ug3imu.pipelines`

The core of the toolbox: builds MobGap/SKDH-compatible datasets from raw IMU files, runs gait detection
on them through **one configurable engine** (`run_unified_pipeline`), and writes results + QC reports in
a consistent directory layout across all three scenarios (Lab, At-Home, Functional Test) and both
gait-analysis engines (MobGap, SKDH).

[← Back to project README](../../../README.md)

> **Read this first:** [Unified engine](#unified-engine--gsd--gait--turn-algorithms-freely-combinable)
> below is the current entry point, and
> [Tunable SKDH/MobGap parameters](#tunable-skdhmobgap-parameters) covers the numeric knobs on top of it.
> The three older sections ([Three windowing modes](#three-windowing-modes-one-factory),
> [SKDH pipelines](#skdh-pipelines), [PRESETS](#presets)) describe the internal MobGap builder and the
> deleted standalone SKDH functions the engine's logic used to live in — kept for reference, not the
> public API.

## Files

| File | Role |
|------|------|
| [stage_registry.py](stage_registry.py) | `PipelineConfig`, `GSD_ENGINES` / `GAIT_ENGINES` registries, `PRESETS` (single source of truth for the GUI), `resolve_algorithm_name()`, `stage_algorithm_values()` |
| [unified_engine.py](unified_engine.py) | **The public entry point** — `run_unified_pipeline()`, `detect_gsd()`, `_ReplayGSD`, `expand_configs()` |
| [pipeline_factory.py](pipeline_factory.py) | Low-level MobGap builder the engine calls — `create_pipeline()`, `run_pipeline_on_dataset()`, `GSD_ALGO_MAP` / `ICD_ALGO_MAP` / `LRC_ALGO_MAP`, `FullWindowGSD` |
| [dataset_generation.py](dataset_generation.py) | `build_dataset_from_file_list()` — the dataset builder used by the GUI (works for lab, at-home, and functional test alike) |
| [athome_dataset_generation.py](athome_dataset_generation.py) | `INPUT_FORMATS` registry, file discovery (`discover_files_by_keyword`, `discover_athome_files`) |
| [lab_pipeline.py](lab_pipeline.py) | `DummyGSD` — treats a fixed crop window as a single gait sequence (`windowing="ref"`) |
| [skdh_lab_pipeline.py](skdh_lab_pipeline.py) | SKDH output-conversion helpers the engine reuses — `gait_lumbar_df_to_stride_df()`, `_ic_time_to_abs_s()`. The standalone `run_skdh_lab_pipeline()` this module used to hold was dead code (nothing called it once the unified engine shipped) and was deleted. |
| [skdh_athome_pipeline.py](skdh_athome_pipeline.py) | More SKDH output helpers — `_compute_dmo()`, `_skdh_athome_qc_kwargs()`, `_plot_ic_overlay()`. `run_skdh_athome_pipeline()` / `create_skdh_pipeline()` were the same kind of dead code and were deleted. |
| [mobilised_wb.py](mobilised_wb.py) | `WBA_RULES`, `trim_edge_strides_and_summarize()` — the shared Mobilise-D-standard WB-assembly rules and first/last-stride trim, applied identically to the MobGap and SKDH gait paths (see [Stride selection & walking-bout assembly](#stride-selection--walking-bout-assembly-mobilised_wbpy) below) |
| [qc_templates.py](qc_templates.py) | Shared QC text-block builders — the MobGap and SKDH gait paths both render through these so output format is identical |

`ug3imu.pipelines` exports: `build_dataset_from_file_list`, `INPUT_FORMATS`, `discover_files_by_keyword`,
`discover_athome_files`, `run_unified_pipeline`, `PipelineConfig`, `expand_configs`, `detect_gsd`,
`PRESETS`, `GSD_ENGINES`, `GAIT_ENGINES`, `SKDH_METHOD_SUFFIX`, `stage_algorithm_values`,
`resolve_algorithm_name`, `param_override_tag`, `record_param_run`, plus the low-level `create_pipeline` /
`run_pipeline_on_dataset` / `FullWindowGSD` / `DummyGSD`. `run_skdh_lab_pipeline` / `run_skdh_athome_pipeline`
/ `create_skdh_pipeline` are gone entirely (deleted, not just unexported) — `skdh_lab_pipeline.py` /
`skdh_athome_pipeline.py` now only hold the small output-conversion helpers `unified_engine` reuses.

`build_dataset_from_folder()`, `build_athome_dataset_from_files()`, and `athome_pipeline.py` in its
entirety were removed as dead code. `legacy_code/athome_monitoring.py` still imports the removed
`athome_pipeline.py` / `run_skdh_athome_pipeline`; that script is archived / not run, so its import
breaking is expected, not a regression.

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
   most dropped ICs without inflating spurious strides) — GUI-tunable per run, not hard-coded, since
   [Tunable SKDH/MobGap parameters](#tunable-skdhmobgap-parameters) below shipped. See
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
**Pipeline** tab (was MobGap + SKDH tabs) with a per-stage engine × algorithm selector, presets, "All"
batch expansion (`expand_configs`), and a small "⚙ Params" dialog per gait engine for the tunable knobs
(see [Tunable SKDH/MobGap parameters](#tunable-skdhmobgap-parameters) below). `run_skdh_lab_pipeline` /
`run_skdh_athome_pipeline` / `create_skdh_pipeline` have been deleted outright (not just unexported) —
they were dead code once the unified engine shipped; `skdh_lab_pipeline.py` / `skdh_athome_pipeline.py`
stay only as the home of the output-conversion helpers the engine reuses (`gait_lumbar_df_to_stride_df`,
`_ic_time_to_abs_s`, `_compute_dmo`, `_skdh_athome_qc_kwargs`, `_plot_ic_overlay`). `create_pipeline` /
`run_pipeline_on_dataset` remain exported as the low-level MobGap builder the engine calls internally.
`legacy_code/athome_monitoring.py` imports the long-removed `athome_pipeline.py` / `run_skdh_athome_pipeline`;
that script is archived / not run, so the broken import is expected, not a regression.

## Tunable SKDH/MobGap parameters

Seven numeric/boolean knobs on `PipelineConfig` (all default `None` = "use the library's own class
default", except `skdh_height_factor` which is a real float since SKDH itself has no `None` sentinel for
it):

| Field | Reaches | Only applies when | Class default |
|-------|---------|--------------------|----------------|
| `skdh_ic_prom_factor` | `GaitLumbar`'s `ApCwtGaitEvents` substep | `icd_algorithm == "AP CWT"` (silent no-op under `"Vertical CWT"` — `GaitLumbar` never wires it into `VerticalCwtGaitEvents`) | 0.6 |
| `skdh_fc_prom_factor` | same substep | same as above | 0.6 |
| `skdh_height_factor` | `GaitLumbar` itself (leg-length ≈ `height_factor * height_m`) | always, when `gait_engine == "skdh"` | 0.53 |
| `skdh_wavelet_scale` | `GaitLumbar`'s `VerticalCwtGaitEvents` substep | `icd_algorithm == "Vertical CWT"` (silent no-op under `"AP CWT"`) | adaptive (`'default'`, computed per-bout from the bout's own mean step frequency) |
| `skdh_use_cwt_scale_relation` | same substep | same as above | `True` (adaptive) |
| `skdh_max_stride_time` | `GaitLumbar`'s `CreateStridesAndQc` substep | **either** `icd_algorithm` — this substep always runs, unlike the two rows above | adaptive (`2.0 * mean_step_time + 1.0`, from the bout's own step-frequency estimate) |
| `mobgap_step_length_scaling_factor` | MobGap `SlZijlstra` | `gait_engine == "mobgap"` | 1.14675 |

The "only applies when" column matters because `GaitLumbar.__init__` dispatches `ic_prom_factor`/
`fc_prom_factor` into `ApCwtGaitEvents` and `wavelet_scale`/`use_cwt_scale_relation` into
`VerticalCwtGaitEvents` inside an `if gait_event_method.lower() == "ap cwt": ... elif ... "vertical cwt":
...` block — each pair only reaches the branch it's named after, so setting e.g. `skdh_wavelet_scale`
while `icd_algorithm == "AP CWT"` is accepted but has zero effect (confirmed by reading the installed
`skdh` package's source, not guessed). `max_stride_time` is different: it's consumed by
`CreateStridesAndQc`, which both branches run unconditionally after the if/elif, so it's the one knob
that affects either SKDH IC-detection algorithm.

**Why only two SKDH IC-detection algorithms exist (`stage_registry.GAIT_ENGINES["skdh"]["icd"]`):**
`GAIT_ENGINES["skdh"]["icd"]` used to list three more entries — `"AP CWT (low-prom)"`, `"Vertical CWT
(lab-w)"`, `"Vertical CWT (lab-fw)"` — that were never separate algorithms, just fixed presets of the
same knobs above (e.g. `ic_prom_factor=0.3` for "low-prom"). They were removed because they actively
conflicted with the tunable-knob feature: picking a preset and then also typing a value into the params
dialog silently did nothing whenever the typed value happened to equal the *unrelated* class default
(0.6), since the override logic couldn't distinguish "field left blank" from "explicitly re-entered the
default". Reproduce any of the old presets by picking the plain algorithm and dialing the same numbers
into the params dialog — see `stage_registry.SKDH_METHOD_SUFFIX`'s comment for the exact removed values.

**Filename collisions** — `resolve_algorithm_name(config)` only encodes the gsd/icd/lrc algorithm
*choices*, not these numeric knobs, so two runs with the same choices but different knob values would
otherwise land on the same output filename and silently overwrite each other. `param_override_tag(config)`
(`stage_registry.py`) fixes this: it returns `""` when every knob above is at its default (so today's
default runs are byte-for-byte unaffected — same filenames as before this feature existed) and otherwise
returns `"_p" + <8 hex chars>` hashed from the overridden values, appended to `algorithm_name` by both the
GUI (`imu_pipeline.py`'s `_run_unified`/`_run_batch_unified`) and `run_unified_pipeline`'s own fallback.

**Self-describing output** — `stage_algorithm_values(config)` already writes every knob (as a real value,
or the literal string `"default"` when unset) into every `ic`/`stride`/`wb` CSV row, reusing the exact
same convention as `gsd_algorithm`/`icd_algorithm`/etc. — no separate lookup needed to know which
parameters produced a given row. `record_param_run(output_path, config, algorithm_name)` additionally
upserts one row per distinct `algorithm_name` into `{output_path}/param_runs.csv` (a small per-subject/
device ledger of every parameter combination tried, for browsing history — not the primary way to answer
"what produced this row").

**GUI** (`imu_pipeline.py`) — each gait engine's algorithm row has a "⚙ Params" button (not always-visible
inline entry fields — that clutters the row regardless of which algorithm is selected and invites the
exact preset/override collision described above) that opens a small popup (`_open_param_settings`) with
the relevant knobs for that engine. Getters (`_skdh_ic_prom_factor()` etc.) parse the field and return
`None` on blank/unparseable input, matching `PipelineConfig`'s own "unset" sentinel.

**report_app** (`scripts/report_app.py`) — `_split_algo_cols(df)`, the one function every IC/Stride/WB
table renders its "Algorithm" column through, also derives a "Params" column from these same raw CSV
columns (`_params_summary_col`) — e.g. `"ic_prom_factor=0.3"`, or `"default"` when every knob is unset.
On top of that, every tab (IC/Stride/WB/GSD) opens with an `_algo_param_filter(df, key=...)` multiselect
(built from `_algo_filter_options`, which combines `_algo_stage_map` + `_params_summary_col` into one
readable label like `"GsdIluz / IcdIonescu  [step_length_scaling_factor=1.5]"`) — so a parameter sweep can
be narrowed down to specific combinations *within* whichever tab you're already looking at, not just
listed passively. This replaced an earlier "Parameter Runs"-tab-only design after feedback that comparing
parameter results actually requires switching at each results level (IC/Stride/WB/GSD), not a separate
ledger tab — the "Parameter Runs" tab (reading `param_runs.csv`) is kept as a secondary historical-overview
list, not the primary way to compare runs.

**Three gaps found and fixed after this shipped** (all surfaced by actually running a parameter sweep
through evaluation, not just the pipeline step):

1. The MobGap gait path never wrote any of these 7 columns onto its `ic`/`stride`/`wb` output at all —
   `_run_mobgap_gait` built its own filename tag correctly but never passed the tunable values through
   to `pipeline_factory.run_pipeline_on_dataset`, which only knew about the older 7-key
   `_stage_algorithm_columns()`. Fixed by adding a `run_pipeline_on_dataset(..., extra_stage_cols=)`
   parameter that gets merged into its `stage_cols` dict before every per-row write; `_run_mobgap_gait`
   now passes the tunable-knob subset of `stage_algorithm_values(config)` there. The SKDH gait path's
   own IC table had a narrower version of the same gap — it wrote only `gsd_algorithm`/`icd_algorithm`,
   not the full `stage_vals` dict `stride_df`/`wb_df` already got — fixed the same way.
2. Every evaluation function that groups/dedups by algorithm identity — `indip.batch_ic_analysis_indip`,
   `indip.batch_ic_analysis_indip_athome`, `metrics.ic_evaluation.batch_ic_analysis_multi_algo` (IC-level,
   one file per `(trial, gsd_algorithm, icd_algorithm)` since IC detection doesn't depend on LRC/etc — see
   [Per-stage algorithm columns](#per-stage-algorithm-columns)) — keyed **only** on that coarse tuple, with
   no idea these 7 knobs existed. Two parameter-sweep runs sharing the same algorithm *choice* collided on
   the same key, and `dict.setdefault` silently kept only one, dropping the other's row entirely — the
   result a run of "evaluate" only ever showing the last parameter combination tried. Fixed by folding a
   short param-tag (derived from whichever `_PARAM_COLS` are present and non-default, mirroring
   `param_override_tag`) into the dedup key in all three functions. The stride/WB-level equivalents
   (`indip._stage_algo_from_df_or_name`, `metrics.wb_evaluation._stage_algo_from_df_or_name`,
   `metrics.stride_evaluation._stage_algo_from_path`) were never dropping rows (every file there is kept,
   dedup doesn't apply at that level), but still needed the same param-tag folded into their `"algorithm"`
   label — otherwise two rows survived but were indistinguishable to a reader or to report_app's grouping.
   GSD-level dedup (`metrics.gsd_evaluation.batch_gsd_analysis_indip`'s `seen_gsd`, used for `metrics_all`)
   was **not** touched — none of these 7 knobs affect GSD detection, so collapsing to one row per
   `gsd_algorithm` there remains correct.
3. `metrics.gsd_evaluation.batch_gsd_analysis_indip` has its *own*, separate `_names()` helper that labels
   the per-bout matched-WB output (`error_all`, written to `wb_error_indip_outoflab.csv`) — distinct from
   the `seen_gsd` dedup in point 2, and initially missed when that fix went in. It read only
   `gsd_algorithm`/`icd_algorithm`/`lrc_algorithm` off each file's header row, so `wb_error_indip_*.csv`
   carried no way to tell two parameter-sweep runs' bouts apart even though `error_all` itself was never
   dropping rows. Fixed the same way as point 2's stride/WB helpers: folds a param tag into `algorithm` and
   copies the raw `_PARAM_COLS` values onto each row.

## Three windowing modes, one factory

> Internal detail — `PipelineConfig.windowing` is the user-facing knob; the values below are what
> `create_pipeline` sees after `unified_engine` translates the config. `"ref"` in the config maps to
> `"mocap"` here.

`create_pipeline(windowing=...)` in `pipeline_factory.py` builds a `mobgap.pipeline.GenericMobilisedPipeline`
that differs only in how the gait-sequence-detection (GSD) step is configured:

| `windowing` | `PipelineConfig` value | GSD component | Use case |
|-------------|------------------------|----------------|----------|
| `"gsd"` | `"gsd"` | `_ReplayGSD` replaying the pre-computed `gs_list` (from any GSD engine), or a `GSD_ALGO_MAP` class via `gsd_override` | At-Home: auto-detected bout boundaries |
| `"mocap"` | `"ref"` | `DummyGSD` (from [lab_pipeline.py](lab_pipeline.py)) — one window = the mocap / INDIP crop | Lab: no detection |
| `"full"` | `"full"` | `FullWindowGSD` (defined in `pipeline_factory.py`) | Functional Test: entire recording = one window |

`create_pipeline` also takes a `gsd_override=` argument (added for the unified engine): when set, that
object is used as the GSD component directly, which is how `_ReplayGSD` injects a `gs_list` produced by
*any* engine (MobGap `GsdIluz`/…, SKDH `PredictGaitLumbarLgbm`) into MobGap's per-GS iteration.

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

For the MobGap gait path, `create_pipeline()` wires `stride_selection=StrideSelection()` and
`wba=WbAssembly(rules=WBA_RULES)` into `GenericMobilisedPipeline`; `run_pipeline_on_dataset()` calls
`trim_edge_strides_and_summarize()` right after `pipeline.run(dp)` and overwrites
`res.per_stride_parameters_`/`res.per_wb_parameters_` in place (also re-running the DMO threshold mask +
aggregation against the trimmed WBs when DMO is enabled). For the SKDH gait path,
`unified_engine._run_skdh_gait()` calls `StrideSelection()` → `WbAssembly(rules=WBA_RULES)` →
`trim_edge_strides_and_summarize()` on the GaitLumbar-derived stride table the same way. Both engines
pool strides across all GSD windows and let `WbAssembly`'s own break rule define the final WB boundaries
(rather than treating a GSD window as one WB), so a Lab trial with a >3 s pause can legitimately split
into more than one WB.

## PRESETS

`PRESETS` lives in [`stage_registry.py`](stage_registry.py) (the old `pipeline_factory.PRESETS` was
removed). It's the single source of truth the GUI reads to populate the preset buttons — one entry per
preset, in the unified `PipelineConfig` shape plus a few GUI-only flags:

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

`_apply_preset(name)` in `scripts/imu_pipeline.py` copies these values into the GUI's state variables
and refreshes which conditional rows are shown.

## Building a dataset

```python
from ug3imu.pipelines import build_dataset_from_file_list, run_unified_pipeline, PipelineConfig

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
cfg = PipelineConfig(
    windowing="gsd",
    gsd_engine="mobgap", gsd_algorithm="GsdIluz",
    gait_engine="mobgap", icd_algorithm="IcdIonescu", lrc_algorithm="LrcUllrich",
    turn=True, enable_dmo=True,
)
run_unified_pipeline(dataset, cfg, output_path="results/TB017/AX6", imu_fs=100,
                     algorithm_name=resolve_algorithm_name(cfg))
```

`run_unified_pipeline` validates the config, runs the GSD stage, dispatches to the MobGap or SKDH gait
path, and writes `gs_list` / raw ICs / per-stride / per-WB / aggregated DMO / turn tables to the matching
subfolder — see [Output directory layout](#output-directory-layout) below. Internally the MobGap gait path
delegates to `create_pipeline` + `run_pipeline_on_dataset` (iterating every trial, running
`GenericMobilisedPipeline`, writing `gs_list_` / `raw_ic_list_` / `per_stride_parameters_` /
`per_wb_parameters_` / `aggregated_parameters_` / `raw_turn_list_`). `step_time_s` is added post-hoc to
the stride table since MobGap doesn't produce it natively: `step_time_s[i] = (IC[i+1] − IC[i]) / imu_fs`.

### Per-stage algorithm columns

Besides `algorithm_name` (the full pipeline identity baked into every output filename, e.g.
`GsdIluz_IcdIonescu_LrcUllrich` or `GsdIluz_SKDH_apcwt`), every row of `gs`/`ic`/`stride`/`wb`/`turn`
output also carries one column per pipeline stage — `gsd_algorithm`, `icd_algorithm`, `lrc_algorithm`,
`cadence_algorithm`, `stride_length_algorithm`, `walking_speed_algorithm`, `turn_algorithm` — filled by
`stage_registry.stage_algorithm_values(config)` (MobGap path) / the equivalent SKDH labels. For the
MobGap engine `cadence`/`stride_length`/`walking_speed` aren't user-selectable so those columns are
constant (`CadFromIc`/`SlZijlstra`/`WsNaive`); for the SKDH engine they're all `SKDH_GaitLumbar`.

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

`windowing != "gsd"` (Lab / Functional modes without a real GSD run) still gets a `gsd_algorithm` value
rather than a blank one: `"mocap_windowed"` for `windowing="ref"` (DummyGSD — window comes from the
mocap / INDIP crop) or `"full_window"` for `windowing="full"` (FullWindowGSD). When the GSD engine is
SKDH, `gsd_algorithm` is `"SKDH_PredictGaitLumbarLgbm"`. When the gait engine is SKDH, `icd_algorithm`
is `f"SKDH_{gait_event_method}"` (`"SKDH_AP CWT"` / `"SKDH_Vertical CWT"`) and
`lrc_algorithm`/`cadence_algorithm`/`stride_length_algorithm`/`walking_speed_algorithm` are all
`"SKDH_GaitLumbar"`.

## SKDH pipelines

> Internal detail — the standalone functions this section used to describe (`run_skdh_lab_pipeline`,
> `run_skdh_athome_pipeline`, `create_skdh_pipeline`) have been **deleted** — they were dead code once
> the unified engine shipped. The SKDH gait path now runs through `unified_engine._run_skdh_gait()` (see
> [How the two engines are bridged](#how-the-two-engines-are-bridged) above). `skdh_lab_pipeline.py` /
> `skdh_athome_pipeline.py` survive only as the home of the output-conversion helpers `unified_engine`
> reuses: `gait_lumbar_df_to_stride_df`, `_ic_time_to_abs_s`, `_compute_dmo`, `_skdh_athome_qc_kwargs`,
> `_plot_ic_overlay`.

SKDH has no `GenericMobilisedPipeline`-equivalent abstraction. Its `GaitLumbar.predict()` bundles IC
detection (`gait_event_method` = `"AP CWT"` / `"Vertical CWT"`), laterality, cadence, stride length and
walking speed into one atomic call and does not accept externally supplied ICs. The unified engine feeds
it the GSD stage's windows via `gait_bouts=` (one call per window), converts the per-IC output to a
`start`/`end` stride table with `gait_lumbar_df_to_stride_df`, then runs the same shared Mobilise-D tail
(`StrideSelection` → `WbAssembly(WBA_RULES)` → `trim_edge_strides_and_summarize`) plus `_compute_dmo` for
At-Home DMO aggregation (same Mobilise-D thresholds as MobGap's `MobilisedAggregator`). Output columns
mirror the MobGap format so both engines' CSVs are directly comparable (see
[metrics/README.md — stride columns by pipeline](../metrics/README.md#stride-csv-columns-by-pipeline)).

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
