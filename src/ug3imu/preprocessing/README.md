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
from ug3imu.preprocessing import read_cwa_fs  # .cwa callers: get the file's actual device fs
```

| Function | Returns | Used by |
|----------|---------|---------|
| `load_imu_for_mobgap(path, device)` | `pd.DataFrame` with `[acc_x, acc_y, acc_z, gyr_x, gyr_y, gyr_z]` (m/s², device-corrected) | [`pipelines/dataset_generation.py`](../pipelines/dataset_generation.py), [`pipelines/athome_dataset_generation.py`](../pipelines/athome_dataset_generation.py) |
| `load_imu_for_skdh(path, device, sampling_rate_hz)` | `(time, accel)` — `accel` in **g**, no axis correction | Not called inside `ug3imu.pipelines` any more — the unified engine loads accel once via `load_imu_for_mobgap`/`dp.data_ss` (m/s²) and converts to g locally (`unified_engine._accel_g`) instead of re-reading the file. Kept exported as a standalone SKDH-facing utility for scripts/notebooks. |

Both dispatch on file extension (`.csv` → `preprocess_imu_for_*`, `.npz` → `preprocess_npz_for_mobgap` /
`load_npz_for_skdh`, `.cwa` → `preprocess_cwa_for_mobgap` / `load_cwa_for_skdh`). Extending to a new format
means adding a branch here — see [Adding a new input format](#adding-a-new-input-format) below.

### Lower-level functions (rarely called directly)

`preprocess_imu_for_mobgap`, `preprocess_npz_for_mobgap`, `preprocess_imu_for_skdh`, `load_npz_for_skdh`,
`preprocess_cwa_for_mobgap`, `load_cwa_for_skdh` — the per-format implementations behind the two dispatchers
above. Import these directly only if you're bypassing the format registry (e.g. writing a one-off script
against a known file type). `read_cwa_raw`/`read_cwa_fs` (see [CWA](#cwa-raw-axivity-binary-ax6-only)
below) are lower-level still — the per-engine preprocessors call them internally.

## Input formats

| Format | Extension | Raw accel unit | NPZ/CSV keys or columns |
|--------|-----------|-----------------|--------------------------|
| NPZ | `.npz` | g | `accel (N×3)`, `gyro (N×3, deg/s)`, `fs`, `time` |
| CSV | `.csv` | g (AX6/DP7/EMPATICA) or m/s² (OPAL) | `Acc_X/Y/Z`, `Gyro_X/Y/Z` |
| CWA | `.cwa` | g (AX6 only) | raw Axivity binary — read via `skdh.io.ReadCwa`, see below |

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
| `EMPATICA_lumbar` | Empatica Embrace Plus (lumbar placement) | g → m/s², 180° rotation around Y (flip X, Z) | flip X, Z | g, no rotation |

All MobGap outputs use column order `[acc_x, acc_y, acc_z, gyr_x, gyr_y, gyr_z]`. For SKDH, OPAL is the
only device needing a unit conversion (m/s² → g); AX6/DP7/EMPATICA NPZ and CSV are already in g.

`EMPATICA_lumbar`'s rotation was derived from its documented raw mounting orientation (Forward = -Z,
Up = -X, Right = +Y) against the Mobilise-D sensor frame (+acc_x = up, +acc_y = right, +acc_z = forward),
and cross-checked against AX6's raw gravity reading at rest (same target convention, independently
verified). The device key is `EMPATICA_lumbar`, not bare `EMPATICA` — the same subject folder also has
`EMPATICA_ankleL/ankleR/wrist/BVP_Sync` placements, and `discover_files_by_keyword`'s exact folder-name
match (device or `{device}_Sync`) needs the full `EMPATICA_lumbar_Sync` name to avoid picking those up too.

## CWA (raw Axivity binary, AX6 only)

Added for batch-processing raw `.cwa` dumps directly (no sync/export step), e.g. for a
headless cluster job over at-home recordings. `read_cwa_raw(path)` wraps
`skdh.io.ReadCwa` and returns `{"accel": (N,3) g, "gyro": (N,3) deg/s or None, "fs": float,
"time": (N,) s from 0}`; `read_cwa_fs(path)` returns just the fs. `preprocess_cwa_for_mobgap`
and `load_cwa_for_skdh` build on it the same way the CSV/NPZ loaders do.

**Use the file's own actual fs, not a nominal rate.** `fs` here is whatever `ReadCwa` read
out of the file's header (e.g. 99.97 Hz), not a flat 100 — Axivity units drift from nominal
differently per device/recording, and that error compounds over multi-day at-home
recordings if a nominal rate is assumed instead. `load_cwa_for_skdh`'s `sampling_rate_hz`
parameter exists only for interface parity with the other `load_*_for_skdh` functions and is
ignored in favor of the file's own rate.

**Orientation is an unverified placeholder.** Both `preprocess_cwa_for_mobgap` and
`load_cwa_for_skdh` reuse the AX6 CSV path's axis transform (g→m/s² + 180° rotation around X)
verbatim — that transform was derived from `*_ax6_sync.csv` lab exports, not checked against
a raw `.cwa` read directly off the device. Before trusting absolute values (walking speed,
stride length — detection/counts are more robust to a wrong sign/axis swap than magnitudes
are), verify on one file: a quiet/lying-still segment should read ≈1 g on the axis the code
treats as "up" after the flip. Both functions have a `TODO(cwa-orientation)` comment marking
exactly where to fix this if it's wrong.

Only AX6 is supported — both functions raise `ValueError` for any other device, since the
other devices here don't produce `.cwa` files.

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
   column for sensor mounting height; `dataset_generation.py`'s device-validation check is driven by this
   map's keys, so this step alone is what makes a new device "known").
