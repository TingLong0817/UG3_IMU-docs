# `scripts/`

Entry points — GUIs and batch/report tools built on top of the [`ug3imu`](../src/ug3imu) library. This
folder has no `__init__.py`; each file is a standalone script.

[← Back to project README](../README.md)

## Files

| File | Purpose |
|------|---------|
| [imu_pipeline.py](imu_pipeline.py) | **Recommended entry point.** Unified GUI for Lab / At-Home / Functional Test, MobGap + SKDH tabs |
| [report_app.py](report_app.py) | Streamlit results dashboard — see [REPORT_APP_DOC.md](REPORT_APP_DOC.md) |
| [aggregate_ic_results.py](aggregate_ic_results.py) | CLI: aggregates per-subject `Evaluation/` CSVs into cross-subject summary tables |
| [task_config.json](task_config.json) | Task-keyword presets edited via `imu_pipeline.py`'s **Settings…** dialog — see below |
| [project_settings.json](project_settings.json) | Last-used GUI field values (paths, subject, device); read/written automatically, local to this machine |

## `imu_pipeline.py` — Unified GUI

```bash
conda activate ug3imu
python scripts/imu_pipeline.py
```

Single entry point for all three workflows (Lab, At-Home, Functional Test), both MobGap and SKDH engines.
The GUI is a thin layer: it discovers files, builds a dataset, and calls into
[`ug3imu.pipelines`](../src/ug3imu/pipelines/README.md) — almost no gait-analysis logic lives in this
file itself.

### Shared controls

| Control | Description |
|---------|-------------|
| **Input Root** | Root folder whose direct children are subject folders |
| **Output Root** | Where results are written; defaults to `{Input Root}/Results` |
| **Metadata** | Participant CSV — see [pipelines/README.md — Metadata CSV format](../src/ug3imu/pipelines/README.md#metadata-csv-format) |
| **Subject ID** | Subject folder name (e.g. `TB017`); auto-uppercased |
| **Batch…** | Checklist to select multiple subjects; disables the single Subject ID entry while active |
| **Device** | `AX6` / `OPAL` / `DP7` — sets sampling rate automatically (`DEVICE_FS` in this file) |
| **Input Format** | `NPZ` / `CSV` — from `INPUT_FORMATS` (see [preprocessing/README.md](../src/ug3imu/preprocessing/README.md#input-formats)) |
| **Task** | Keyword preset for file discovery, editable via **Settings…** → writes `task_config.json` |

File discovery (`discover_files_by_keyword`) searches `{Input Root}/{Subject}/` recursively, matching
whole underscore-delimited tokens of the filename stem against the task's keyword list — e.g. keyword
`home` matches `TB017_20250521_ax6_home_gait1.npz`. Full discovery details (including the placement-label
whitelist that keeps only the lumbar sensor file when a trial has multiple placements) are in
[pipelines/README.md — File discovery](../src/ug3imu/pipelines/README.md#file-discovery-athome_dataset_generationpy).

### Preset buttons (MobGap and SKDH tabs)

| Preset | Windowing | DMO | Evaluation | Typical use |
|--------|-----------|-----|------------|-------------|
| **At-Home** | GSD (auto-detect bouts) | On | Off | Free-living multi-bout recordings |
| **Lab** | Mocap window | Off | On | Synchronised lab recordings with V3D/INDIP reference |
| **Functional Test** | Full recording (no GSD) | Off | Off | Short structured tasks (TUG, 10MWT, 6MWT, etc.) |

Presets are read from `PRESETS` in [pipeline_factory.py](../src/ug3imu/pipelines/pipeline_factory.py) —
see [pipelines/README.md](../src/ug3imu/pipelines/README.md#presets) for the full config each preset sets.

### `task_config.json`

Editable via the **Settings…** button next to the Task dropdown. Each entry is
`{"name": "...", "keywords": ["...", ...]}`; keywords are matched case-insensitively against
underscore-delimited filename tokens (not substrings — a keyword `"lab"` won't match `"carpwa"`, but
splitting `TB017_20250521_carpwa_ax6.npz` gives the token `carpwa`, so a task actually needs that literal
token listed, e.g. `"carpwa"` not `"carp"`).

## `report_app.py` — Streamlit dashboard

```bash
conda activate ug3imu
streamlit run scripts/report_app.py
# or on Windows: run_report.bat
```

Reads `Evaluation/` CSVs written by `imu_pipeline.py`'s **Evaluate Results** button and renders
cross-subject comparison tables/plots (bias, RMSE, ICC, Bland-Altman) per algorithm. Depends on
`aggregate_ic_results.py` for shared summary-statistics helpers (`_STRIDE_PARAMS`, `_add_f1`,
`_detection_summary`, `_icc21`). Full file-naming conventions, expected CSV columns, and dashboard
architecture are documented separately in **[REPORT_APP_DOC.md](REPORT_APP_DOC.md)**.

## `aggregate_ic_results.py` — cross-subject CLI aggregation

```bash
python scripts/aggregate_ic_results.py --output_root R:/... --device OPAL
python scripts/aggregate_ic_results.py          # interactive prompts
```

Scans `{output_root}/*/{device}/Evaluation/` for `ic_metrics_*.csv` / `ic_error_*.csv` /
`stride_rmse_*.csv` / `stride_error_*.csv` and writes six aggregate CSVs to `{output_root}`:
`IC_metrics_all_{device}.csv`, `IC_errors_all_{device}.csv`, `IC_summary_{device}.csv` (per-algorithm
detection mean±SD + timing MAE/RMSE), and the `stride_*` equivalents. This is a standalone precursor to
`report_app.py` — useful when you want static CSVs rather than an interactive dashboard, or want to script
further analysis on top of the aggregated tables.

## Legacy At-Home GUI

`athome_monitoring.py` predates the unified `imu_pipeline.py` and has been moved to
[`legacy_code/`](../legacy_code/athome_monitoring.py). It imports `create_athome_pipeline` /
`run_athome_pipeline_on_dataset` from [`athome_pipeline.py`](../src/ug3imu/pipelines/athome_pipeline.py),
which is itself marked legacy in [pipelines/README.md](../src/ug3imu/pipelines/README.md#files) — use
`imu_pipeline.py`'s At-Home preset instead.
