# UG3_IMU

IMU data processing toolbox for the UG3 project at MAI Lab. Supports laboratory validation against V3D
mocap or INDIP ground truth, and at-home free-living monitoring, across multiple devices and input
formats, using both the MobGap and SKDH gait-analysis engines.

> **Algorithm reference**: For a full comparison of all available algorithms across MobGap and SKDH (GSD, ICD, stride length, symmetry metrics, etc.) see [ALGORITHM_COMPARISON.md](ALGORITHM_COMPARISON.md).

---

## Documentation

This README covers installation and a high-level tour. Each module folder has its own README with
implementation details, public API, and "how do I extend this" notes — start here, then follow a link
below into whichever part you're actually touching.

| Doc | Covers |
|-----|--------|
| [scripts/README.md](scripts/README.md) | GUI usage (`imu_pipeline.py`), preset buttons, task keywords, `report_app.py` dashboard, CLI aggregation tool |
| [scripts/REPORT_APP_DOC.md](scripts/REPORT_APP_DOC.md) | `report_app.py` file-naming conventions and CSV schema (中文) |
| [src/ug3imu/preprocessing/README.md](src/ug3imu/preprocessing/README.md) | Device loaders, axis/unit conversion, input-format registry, adding a device or format |
| [src/ug3imu/mocap/README.md](src/ug3imu/mocap/README.md) | V3D TXT parsing, the IMU/mocap time-alignment problem and how it's solved |
| [src/ug3imu/indip/README.md](src/ug3imu/indip/README.md) | INDIP `.mat` reference system — loader, windowing, batch evaluation |
| [src/ug3imu/pipelines/README.md](src/ug3imu/pipelines/README.md) | Dataset building, the three windowing modes, MobGap/SKDH pipeline runners, QC reports, output layout, adding an algorithm |
| [src/ug3imu/metrics/README.md](src/ug3imu/metrics/README.md) | IC/stride/WB evaluation against ground truth, IoU overlap filter, stride column reference |
| [src/ug3imu/plotting/README.md](src/ug3imu/plotting/README.md) | IC overlay and walking-bout timeline plots |

---

## Features

- **Unified GUI** (`imu_pipeline.py`): single entry point for Lab, At-Home, and Functional Test workflows — MobGap and SKDH tabs, task-keyword discovery, evaluation
- **Batch processing**: select multiple subjects at once; subjects with missing files or empty mocap folders are skipped with a clean warning
- **Multi-format input**: NPZ (pre-processed) and CSV via an extensible format registry
- **Two reference systems**: V3D mocap (Vicon Nexus) and INDIP (Mobilise-D in-lab reference)
- **Two gait-analysis engines**: MobGap (`GenericMobilisedPipeline`) and SKDH (`GaitLumbar`), with matching output formats so results are directly comparable
- **Stride quality filtering**: duration 0.6–2.0 s and stride length ≥ 0.15 m, consistently applied in all pipelines
- **QC reports**: written for every run across all pipelines, with library version header
- **Streamlit report app**: cross-subject dashboard for evaluation results (bias, RMSE, ICC, Bland-Altman)

See the module docs above for how each of these actually works.

---

## Project Structure

```
UG3_IMU/
├── scripts/                                   → scripts/README.md
│   ├── imu_pipeline.py              # Unified GUI — recommended entry point
│   ├── report_app.py                # Streamlit results dashboard
│   └── aggregate_ic_results.py      # CLI cross-subject aggregation
├── src/ug3imu/
│   ├── preprocessing/                → preprocessing/README.md
│   ├── mocap/                        → mocap/README.md
│   ├── indip/                        → indip/README.md
│   ├── pipelines/                    → pipelines/README.md
│   ├── metrics/                      → metrics/README.md
│   └── plotting/                     → plotting/README.md
├── testdata/                        # Sample files for smoke-testing (gitignored)
├── requirements.txt
└── setup.py
```

---

## Installation

**Prerequisites**: Python 3.9+, Conda (recommended)

```bash
git clone https://github.com/TingLong0817/UG3_IMU.git
cd UG3_IMU

conda create -n ug3imu python=3.12
conda activate ug3imu

pip install -r requirements.txt
```

## Quick Start

```bash
conda activate ug3imu
python scripts/imu_pipeline.py
```

See [scripts/README.md](scripts/README.md) for the full GUI walkthrough (shared controls, presets, task
keywords), or [API Usage](#api-usage) below to drive the pipelines programmatically.

---

## Workflows at a glance

Full detail — directory layout, output columns, QC filtering rules — lives in
[pipelines/README.md](src/ug3imu/pipelines/README.md).

| Workflow | Windowing | Reference | Typical input |
|----------|-----------|-----------|----------------|
| **Lab Validation** | Mocap-defined crop window | V3D TXT or INDIP `.mat` | Synchronised lab recordings |
| **At-Home Monitoring** | Auto-detected gait sequences (GSD) | None | Free-living multi-day recordings |
| **Functional Test** | Entire recording = one window | None | Short structured tasks (TUG, 10MWT, 6MWT) |

---

## API Usage

### Unified pipeline (programmatic)

```python
from ug3imu.pipelines import (
    INPUT_FORMATS, discover_files_by_keyword,
    build_dataset_from_file_list, create_pipeline, run_pipeline_on_dataset,
)

files = discover_files_by_keyword(
    project_root="R:/UG3", subject="TB017",
    keywords=["home"], device="AX6",
    file_ext=INPUT_FORMATS["NPZ"],   # or INPUT_FORMATS["CSV"]
)
dataset = build_dataset_from_file_list(
    file_list=files, metadata_csv="participants.csv",
    sampling_rate_hz=100, device="AX6",
)
pipeline = create_pipeline(windowing="gsd", gsd_algorithm="GsdIluz",
                           icd_algorithm="IcdIonescu", enable_dmo=True)
run_pipeline_on_dataset(dataset, pipeline, output_path="results/TB017/AX6",
                        algorithm_name="GsdIluz_IcdIonescu", imu_fs=100,
                        enable_dmo=True, plot_wb=True, plot_ic=True)
```

### Preprocessing dispatchers

```python
from ug3imu.preprocessing import load_imu_for_mobgap, load_imu_for_skdh

# Automatically selects NPZ or CSV loader based on file extension
df   = load_imu_for_mobgap("recording.npz", device="AX6")   # → DataFrame
t, a = load_imu_for_skdh("recording.npz",   device="AX6", sampling_rate_hz=100)
```

### Lab pipeline

```python
from ug3imu.pipelines import build_dataset_from_file_list, create_pipeline, run_pipeline_on_dataset

dataset = build_dataset_from_file_list(
    file_list=files, metadata_csv="participants.csv",
    sampling_rate_hz=100, device="AX6",
    mocap_folder="mocap/TB017/V3D/",
)
pipeline = create_pipeline(windowing="mocap", icd_algorithm="IcdIonescu", enable_dmo=False)
run_pipeline_on_dataset(dataset, pipeline, output_path="results/",
                        algorithm_name="mocap_IcdIonescu", imu_fs=100,
                        plot_ic=True, mocap_folder="mocap/TB017/V3D/")
```

### SKDH pipelines

```python
from ug3imu.pipelines import run_skdh_athome_pipeline, run_skdh_lab_pipeline

# At-home (with GSD and DMO)
gs, ic, stride, wb, dmo = run_skdh_athome_pipeline(
    file_list=files, metadata_csv="participants.csv",
    sampling_rate_hz=100, device="AX6", output_path="results/",
)

# Functional test (full recording, no GSD, no DMO)
gs, ic, stride, wb, dmo = run_skdh_athome_pipeline(
    file_list=files, metadata_csv="participants.csv",
    sampling_rate_hz=100, device="AX6", output_path="results/",
    use_gsd=False, enable_dmo=False,
)

# Lab
all_ic, all_stride = run_skdh_lab_pipeline(
    imu_folder="imu/TB017/AX6_Sync/", txt_folder="mocap/TB017/V3D/",
    metadata_csv="participants.csv", sampling_rate_hz=100,
    device="AX6", output_path="results/",
)
```

### Evaluation

```python
from ug3imu.metrics import batch_ic_analysis_multi_algo, batch_stride_analysis

ic_error, ic_metrics = batch_ic_analysis_multi_algo(
    imu_folder="results/IC/", txt_folder="mocap/V3D/",
    imu_fs=100, algorithm="IcdIonescu", exclude_tags=["athome"],
)
stride_error, stride_rmse = batch_stride_analysis(
    stride_folder="results/stride/", txt_folder="mocap/V3D/",
    imu_fs=100, algorithm="IcdIonescu", exclude_tags=["athome"],
    min_overlap_ratio=0.8,   # IoU threshold (default 0.0 = disabled)
)
```

See [metrics/README.md](src/ug3imu/metrics/README.md) for the full evaluation model (matching, IoU filter,
column reference) and [indip/README.md](src/ug3imu/indip/README.md) for the INDIP equivalents.

---

## Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| mobgap | 1.2.0 | MobGap / Mobilise-D gait analysis |
| scikit-digital-health | 0.17.9 | SKDH gait analysis (PredictGait + GaitLumbar) |
| pandas | — | Data manipulation |
| numpy | — | Numerical computing |
| matplotlib | — | Visualisation |
| scipy | — | Scientific computing; also used for INDIP `.mat` parsing |

## License

MIT License — see [LICENSE](LICENSE) for details.

## Contact

For questions or issues, open a GitHub issue or contact the MAI Lab team.
