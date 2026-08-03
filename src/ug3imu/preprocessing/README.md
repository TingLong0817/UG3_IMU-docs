# `ug3imu.preprocessing`

Device-specific IMU loading: reads raw device exports (CSV) or pre-processed archives (NPZ) and
converts them into the column/unit conventions that MobGap and SKDH expect.

[← Back to project README](../../../README.md)

## Files

| File | Purpose |
|------|---------|
| [imu_preprocessing.py](imu_preprocessing.py) | All loaders, axis transforms, and format dispatchers |

## Why this module exists

MobGap and SKDH expect different things from the same raw signal:

| | MobGap | SKDH |
|---|--------|------|
| Acceleration unit | m/s² | **g** |
| Axis orientation | Device-specific rotation applied | Raw axes (SKDH does its own vertical-axis detection) |
| Columns | `acc_x, acc_y, acc_z, gyr_x, gyr_y, gyr_z` | `(time, accel)` arrays |

This module hides that difference behind two dispatchers, so pipeline code never has to know
whether the input file is CSV or NPZ, or whether it's headed to MobGap or SKDH.

## Public API

```python
from ug3imu.preprocessing import load_imu_for_mobgap, load_imu_for_skdh
```

| Function | Returns | Used by |
|----------|---------|---------|
| `load_imu_for_mobgap(path, device)` | `pd.DataFrame` with `[acc_x, acc_y, acc_z, gyr_x, gyr_y, gyr_z]` (m/s², device-corrected) | [`pipelines/dataset_generation.py`](../pipelines/dataset_generation.py), [`pipelines/athome_dataset_generation.py`](../pipelines/athome_dataset_generation.py) |
| `load_imu_for_skdh(path, device, sampling_rate_hz)` | `(time, accel)` — `accel` in **g**, no axis correction | [`pipelines/skdh_lab_pipeline.py`](../pipelines/skdh_lab_pipeline.py), [`pipelines/skdh_athome_pipeline.py`](../pipelines/skdh_athome_pipeline.py) |

Both dispatch on file extension (`.csv` → `preprocess_imu_for_*`, `.npz` → `preprocess_npz_for_mobgap` /
`load_npz_for_skdh`). Extending to a new format means adding a branch here — see
[Adding a new input format](#adding-a-new-input-format) below.

### Lower-level functions (rarely called directly)

`preprocess_imu_for_mobgap`, `preprocess_npz_for_mobgap`, `preprocess_imu_for_skdh`, `load_npz_for_skdh` —
the per-format implementations behind the two dispatchers above. Import these directly only if you're
bypassing the format registry (e.g. writing a one-off script against a known file type).

## Input formats

| Format | Extension | Raw accel unit | NPZ/CSV keys or columns |
|--------|-----------|-----------------|--------------------------|
| NPZ | `.npz` | g | `accel (N×3)`, `gyro (N×3, deg/s)`, `fs`, `time` |
| CSV | `.csv` | g (AX6/DP7) or m/s² (OPAL) | `Acc_X/Y/Z`, `Gyro_X/Y/Z` |

The NPZ time vector is read from the `time` key (Unix timestamps, shifted to start at 0); if absent it
falls back to `fs`, then to the caller-supplied `sampling_rate_hz`.

The registry mapping display name → extension (`INPUT_FORMATS`) actually lives in
[`pipelines/athome_dataset_generation.py`](../pipelines/athome_dataset_generation.py) — the GUI's
**Input Format** dropdown is populated from it directly.

## Supported devices

| Device | Model | Accel preprocessing (MobGap path) | Gyro preprocessing (MobGap path) | SKDH path |
|--------|-------|--------------------------------------|-----------------------------------|-----------|
| `AX6` | Axivity AX6 | g → m/s², 180° rotation around X (flip Y, Z) | flip Y, Z | g, no rotation |
| `OPAL` | APDM Opal v3 | 180° rotation around Y (flip X, Z) | flip X, Z; rad/s → deg/s | m/s² → g |
| `DP7` | Dynaport Hybrid | g → m/s² | — (no gyro) | g, no rotation |

All MobGap outputs use column order `[acc_x, acc_y, acc_z, gyr_x, gyr_y, gyr_z]`. For SKDH, OPAL is the
only device needing a unit conversion (m/s² → g); AX6/DP7 NPZ and CSV are already in g.

## Adding a new input format

1. Add `"FormatName": ".ext"` to `INPUT_FORMATS` in [`pipelines/athome_dataset_generation.py`](../pipelines/athome_dataset_generation.py).
2. Implement here:
   - `preprocess_<fmt>_for_mobgap(path, device) -> pd.DataFrame`
   - `load_<fmt>_for_skdh(path, device, sampling_rate_hz) -> (time, accel)`
3. Add the corresponding `.ext` branches to `load_imu_for_mobgap` and `load_imu_for_skdh`.

The GUI's **Input Format** dropdown picks up the new option automatically — no other changes needed.

## Adding a new device

1. Add axis-transform and unit-conversion branches for the new device in both
   `preprocess_imu_for_mobgap` / `preprocess_npz_for_mobgap` (MobGap path) and
   `preprocess_imu_for_skdh` / `load_npz_for_skdh` (SKDH path) in [imu_preprocessing.py](imu_preprocessing.py).
2. Add the device to `DEVICE_FS` in [`scripts/imu_pipeline.py`](../../../scripts/imu_pipeline.py) (sets the
   default sampling rate shown in the GUI).
3. Add to `DEVICE_HEIGHT_MAP` in [`pipelines/athome_dataset_generation.py`](../pipelines/athome_dataset_generation.py)
   and [`pipelines/dataset_generation.py`](../pipelines/dataset_generation.py) (maps device → metadata CSV
   column for sensor mounting height).
4. Add to `_SUPPORTED_DEVICES` in [`pipelines/skdh_athome_pipeline.py`](../pipelines/skdh_athome_pipeline.py).
