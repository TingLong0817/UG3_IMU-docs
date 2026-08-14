# `ug3imu.plotting`

Two shared plot functions used by every pipeline (MobGap and SKDH, lab and at-home) so output figures
look identical regardless of which algorithm produced them.

[← Back to project README](../../../README.md)

## Files

| File | Purpose |
|------|---------|
| [visualization.py](visualization.py) | IC overlay plot + walking-bout timeline plot |

## Public API

```python
from ug3imu.plotting import plot_ic_overlay_lab, plot_wb_timeline
```

### `plot_ic_overlay_lab(accel_outer, outer_start, imu_fs, ic_frames, mocap_ic_frames, save_path, ...)`

Plots `|accel|` (norm of the 3-axis signal) over the trial window, with reference ICs (green triangles)
and algorithm-detected ICs (red dots) overlaid. `buf_samples` trims the ±1 s processing buffer from each
end before plotting (see [`ug3imu.mocap`](../mocap/README.md#time-alignment--the-core-problem-this-module-solves));
pass `0` when there's no buffer (INDIP mode, where windows come directly from `ContinuousWalkingPeriod`
bounds rather than a padded mocap crop). Saved as `plot/*_ic.png`.

### `plot_wb_timeline(res=None, imu_fs=None, wb_intervals=None, ic_times_s=None, ref_wb_intervals=None, ref_label="INDIP", ...)`

No left/right distinction — draws the algorithm's own WB intervals as green patches in an upper row, IC
timestamps as small purple dots, and (when `ref_wb_intervals` is given) a reference system's WB intervals
(e.g. INDIP MicroWB) as red patches in a lower row directly underneath, so fragmentation/misalignment
between the two systems' bout boundaries is visible at a glance.

Two modes, selected by which arguments are passed:

- **MobGap mode** (`res` + `imu_fs`): algorithm WB intervals come from `res.per_wb_parameters_`, IC times
  from `res.per_stride_parameters_` (unless `ic_times_s` is given directly).
- **SKDH / generic mode** (`wb_intervals`, `ic_times_s`): algorithm WB intervals and IC times passed
  directly.

`run_pipeline_on_dataset` in [`pipelines/pipeline_factory.py`](../pipelines/pipeline_factory.py) and
`skdh_athome_pipeline.run_skdh_athome_pipeline` call this at pipeline-run time (no `ref_wb_intervals` —
the INDIP reference isn't resolved until evaluation). The algorithm-vs-INDIP comparison plot is generated
separately, during evaluation, by `gsd_evaluation.plot_gsd_wb_timeline` (see
[`metrics/README.md`](../metrics/README.md)), which has both the algorithm's `wb.csv` and the INDIP `.mat`
path available at that point. Saved as `plot/*_wba.png` (at-home, algorithm-only) or embedded in the
per-trial IC plot layout (lab); the INDIP-overlay version is saved separately as `plot/*_gsd_wb.png`.

## Where this is consumed

Both functions are called from every pipeline runner in [`ug3imu.pipelines`](../pipelines/README.md):
`pipeline_factory.run_pipeline_on_dataset` (MobGap, all three scenarios),
`skdh_lab_pipeline.run_skdh_lab_pipeline`, and `skdh_athome_pipeline.run_skdh_athome_pipeline`.
