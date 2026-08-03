# Report App 说明文档

## 快速定位

主文件：`scripts/report_app.py`  
依赖：`scripts/aggregate_ic_results.py`（提供 `_STRIDE_PARAMS`, `_add_f1`, `_detection_summary`, `_icc21`）  
启动：`streamlit run scripts/report_app.py`

姊妹 app：`scripts/report_app_indipvsmocap.py` — 验证 INDIP 自身相对 V3D mocap 的精度（而非 IMU 算法相对参考系），
详见文末「INDIP vs Mocap 验证」一节。

---

## 评估输出文件结构

评估文件存放在：
```
{out_root}/{subject}/{device}/Evaluation/
```

### 文件命名规律

```
{metric_type}_{ref_suffix}_{scenario}.csv
```

| 字段 | 取值 | 说明 |
|---|---|---|
| `metric_type` | `ic_error`, `ic_metrics`, `stride_error`, `stride_rmse`, `wb_error`, `wb_rmse`, `gsd_metrics` | 评估类型 |
| `ref_suffix` | `v3d`, `indip` | 参考系：v3d = 动捕 (V3D TXT)；indip = INDIP mat 文件 |
| `scenario` | `lab`, `outoflab` | 采集环境。lab 场景取自 `_task_label()`（任务名小写化，可能是 lab 下的任意任务名）；**At-Home 场景固定写死 `outoflab`**，不管 `task_config.json` 里这个任务实际叫什么名字 |

示例：`ic_metrics_v3d_lab.csv`、`wb_error_indip_outoflab.csv`、`stride_error_indip_outoflab.csv`

**为什么 At-Home 固定用 `outoflab`，不用任务名**：一个 subject 只有一份连续的 at-home recording，
IC/stride/WB/GSD 四种评估都是对着同一份、不按 task 拆分的 INDIP reference 匹配（详见下方「At-Home / GSD
说明」），所以没有必要、也不应该像 lab 场景那样按 task_lbl 分文件——固定用 `outoflab` 避免了"同一份
at-home 数据因为任务改名而在 report_app 里散落成两个不同 scenario"的问题。

---

## 各文件列结构

### `ic_metrics_{ref}_{scenario}.csv`（IC 检测指标，每行 = 一个 trial × algo）
```
trial, algorithm, TP, FP, FN, n_detected, n_reference, precision, recall, f1
```

### `ic_error_{ref}_{scenario}.csv`（IC 时间误差，每行 = 一个匹配的 IC 对）
```
detected, reference,        ← 绝对 IMU 帧号
error_samples, error_s,     ← 时间差（error_s 单位秒）
trial, algorithm
```

### `stride_error_{ref}_{scenario}.csv`（步态参数误差，每行 = 一个匹配的 stride 对）
```
lr_label,
start_{ref}, start_imu, end_{ref}, end_imu,
{param}_{ref}, {param}_imu, {param}_error   ← 对所有 _STRIDE_PARAMS
trial, algorithm
```
**参考列命名随参考系而变**：  
- v3d 文件：`{param}_v3d`（`stride_evaluation.py` 直接输出）  
- INDIP 文件：`{param}_indip`（`indip/__init__.py` 在 `batch_stride_analysis_indip` 里 rename `_v3d → _indip`）  

app 用 `_stride_ref_col(param, df)` 自动检测实际列名（按 v3d → indip → ref 顺序查找）。

`_STRIDE_PARAMS` = `stride_duration_s, cadence_spm, stride_length_m, walking_speed_mps, stance_time_s, swing_time_s, step_time_s, step_length_m, single_support_s, initial_double_support_s, terminal_double_support_s, double_support_s`

`step_time_s` / `step_length_m` 在 INDIP 侧（`_indip` 列）恒为空 —— INDIP 没有对应的per-stride数据。
`single_support_s` / `initial_double_support_s` / `terminal_double_support_s` / `double_support_s` 两侧（V3D 和 INDIP）都有值，
都是直接用各自的原始IC/FC事件时间算的（不是INDIP自带的Single/DoubleSupport_Duration字段，那两个字段口径和V3D对不上）。

### `wb_error_{ref}_{scenario}.csv`（WB 聚合参数误差，每行 = 一个 trial × algo）
```
trial, algorithm,
{param}_ref, {param}_imu, {param}_error   ← 对所有 _WB_PARAMS
```
`_WB_PARAMS` = `n_strides, duration_s, stride_duration_s, stride_length_m, cadence_spm, walking_speed_mps`

**注意**：WB 文件参考列统一命名为 `{param}_ref`（hardcoded in `wb_evaluation.py`），与 ref_suffix 无关。

---

## App 架构

### 侧边栏选择顺序
1. Results root 路径
2. Device（OPAL/AX6/DP7）
3. Subject 多选
4. **Scenario**（LAB / Outoflab）← 先检测再选
5. **Reference**（v3d / indip）← 基于 scenario 检测可用的
6. Export PDF 按钮

### 数据加载

`_load_all_data(root, dev, subjects_key, ref_suffix, scenario)` 返回 5 个 DataFrame：
```python
ic_filt, stride_filt, ic_error_filt, wb_filt, gsd_filt = _load_all_data(...)
```

`_load_subject_files` 使用 glob 模式匹配（支持通配符），合并多个 subject 的数据，加 `subject` 列。

### 关键 Helper 函数

| 函数 | 作用 |
|---|---|
| `_stride_ref_col(param, df)` | 自动检测 stride 数据中参考列名（`_v3d` / `_indip` / `_ref`） |
| `_stride_summary_with_ref(df)` | 按 algorithm 计算 bias/SD/MAE/RMSE/LOA/ICC，无需 ref_suffix |
| `_wb_summary(df)` | 同上，适用于 WB 数据（固定用 `{p}_ref` 列） |
| `_detect_available_scenarios(out_root, device, subjects)` | 扫描 `ic_metrics_*.csv` / `wb_error_*.csv` / `gsd_metrics_*.csv` 获取可用 scenario（lab 场景取值多样，at-home 场景现在固定只会是 `outoflab`） |
| `_detect_available_refs(out_root, device, subjects, scenario)` | 扫描 `*_{scenario}.csv`，按 `stem.split("_")[2]` 取参考系 token（用 token 解析而不是子串匹配，避免文件名结构以后再变时踩坑） |
| `_ic_stride_glob_patterns(ref_suffix, scenario)` | 五种 metric 统一走 `{metric}_{ref}_{scenario}.csv`——现在 at-home 全部四种指标（IC/stride/WB/GSD）都固定用 `scenario="outoflab"`，跟 lab 场景用同一套公式，不需要额外特殊处理。`_eval_data_sig`/`_load_all_data` 都用这个统一构建 glob 列表 |

---

## Tabs 结构

| Tab | 内容 | 数据来源 |
|---|---|---|
| IC Detection | 检测指标（TP/FP/FN/P/R/F1）+ IC 时间误差（bias/SD/MAE） | `ic_metrics`, `ic_error` |
| Stride | stride 级数据，结构与 Walking Bouts 一致：每个 stride 参数的摘要表（Ref mean±SD, IMU mean±SD, Bias[LOA], MAE, ICC）+ 算法/参数选择器驱动的散点图/BA 图（单色：scatter=#4C72B0, BA=#DD8452）+ 按 condition 明细 | `stride_error` |
| Walking Bouts | WB 参数摘要表 + 算法/参数选择器驱动的散点图/BA 图。Outoflab 场景下每行是一个**匹配上的 WB 对**（不是 lab 那种按 trial 聚合成一行），数据点更丰富 | `wb_error` |
| GSD Detection | 有 `gsd_metrics` 数据时显示（At-Home 场景）：WB 数量/总步行时长对比表 + 样本级检测指标（TP/FP/FN/P/R/F1/Specificity/Accuracy/NPV，复用 `_detection_summary`）+ bout 级匹配指标（TP/FP/FN/P/R） | `gsd_metrics` |

**注意**：Gait Measures 与 Bland-Altman 曾是两个独立 tab（都基于 stride 级数据），现已合并为一个 Stride tab，布局对齐 Walking Bouts（先摘要表，后交互式散点/BA 图），避免同一级别数据分散在两个模块、而 WB 级别数据又单独一个模块的不一致逻辑。

---

## At-Home / GSD 说明

- At-home 场景比 lab 多一个 **GSD（Gait Sequence Detection）** 评估步骤，参考系固定为 INDIP（V3D 是 lab-only 参考系统，不适用于 at-home）
- **At-Home 场景的文件后缀固定写死 `outoflab`**，不用 `_task_label()`（任务显示名小写化）——这是特意这么设计的：at-home 的 IC/stride/WB/GSD 四种评估全都是对着同一份不按 task 拆分的连续 reference 匹配，任务在 `task_config.json` 里叫什么名字跟输出文件名无关，这样同一个 subject 的 at-home 数据不会因为任务改名而分裂成 report_app 里两个不同的 scenario。GSD tab 是否显示则是纯数据驱动（看 `gsd_metrics_*.csv` 有没有数据，即 `_has_gsd`），不依赖 scenario 字符串本身
- `imu_pipeline.py` 里判断"这是不是 at-home 任务"用的是 `"outoflab" in self._current_keywords`（即 `task_config.json` 里该任务配置的 keyword，目前 At-Home 任务的 keyword 是 `["outoflab"]`，对应原始文件名里的 `OutofLab` 标记）——这是比较稳定的判断依据，不依赖任务显示名怎么改
- At-home 没有 lab 那种按 task 分 trial 的结构，一个 subject 一天只有一段连续记录；INDIP 参考数据来自 `.mat` 里的 `data.TimeMeasure1.{Recording*}.Standards.INDIP.ContinuousWalkingPeriod`（CWP，结构体数组，每个元素是一个独立的 walking bout），**用 CWP 不用 MicroWB**——CWP 是更粗的 bout 定义，会把 MicroWB 拆开的短间隙桥接起来（P04：56 个 CWP vs 64 个 MicroWB）；这跟 lab 用的 `{test}.{trial}.testName` 结构不同，两者不能混用同一个 loader
- 匹配逻辑：**直接用 mobgap 自带的 `mobgap.gait_sequences.evaluation` 模块**，不再手写重叠计算，分两个层次回答两个不同的问题：
  - **bout 级**（`_match_wb_bouts`，调 mobgap 的 `categorize_intervals` + `get_matching_intervals`）：哪个 ref WB 具体配对哪个 det WB，用于比步态参数（stride length/cadence 这些一对一比较）。mobgap 自己的重叠阈值（默认 0.8，`overlap_threshold`）是**双向**都要满足（重叠必须同时占 ref WB 和 det WB 各自时长的 ≥80%），比之前 Kirk et al. 单向定义更严——真实环境下 bout 计数匹配率在 20%~40% 是符合预期的，因为碎片化或过长的 WB 很容易在双向阈值下失配。
  - **样本级**（`_gsd_sample_level_metrics`，调 mobgap 的 `categorize_intervals_per_sample` + `calculate_matched_gsd_performance_metrics`）：把整段记录逐 sample 打上 tp/fp/fn/tn，不管两边怎么切分 bout，算出 precision/recall/f1/specificity/accuracy/npv——这个衡量的是"真实行走时间到底被捕捉了多少"，通常比 bout 级匹配率宽容得多、也更能说明问题（`n_overall_samples` 用两边最后一个 WB 的结束时间近似）。
  - 核心代码在 `src/ug3imu/metrics/gsd_evaluation.py`（`normalize_athome_wb` 统一 MobGap/SKDH 两种 wb.csv 格式的时间单位，`_match_wb_bouts`/`_gsd_sample_level_metrics` 分别对应上面两层，`batch_gsd_analysis_indip` 是批处理入口，两层指标都会算）
- INDIP at-home mat 的查找用 `find_indip_mat_athome`（找 `*Outoflab.mat`），跟 lab 用的 `find_indip_mat`（找 `*Inlab.mat`）是两个不同函数 —— 同一个 subject 的 INDIP 文件夹下两个文件都存在，必须分开找
- 输出两个文件（固定 `outoflab` 后缀，不带 task_lbl）：
  - `wb_error_indip_outoflab.csv` —— 每行是一个**bout 级匹配上的 WB 对**（`{param}_ref/_imu/_error`），跟 lab 版 `wb_error_*.csv` 同一套 schema，因此直接被 Walking Bouts tab 的现有摘要表 + 散点/BA 图逻辑消费，不需要额外改 tab_wb 的代码
  - `gsd_metrics_indip_outoflab.csv` —— 每行是一个 trial × algorithm，bout 级 `tp_wb/fp_wb/fn_wb` + 样本级 `tp_samples/fp_samples/fn_samples/precision/recall/f1_score/tn_samples/specificity/accuracy/npv`（mobgap 原生命名，`tp_samples` 这套跟 `ic_metrics_*.csv` 同名但现在是真正的样本计数，可以安全跨 trial 求和）+ `n_ref_wb/n_imu_wb/total_walking_time_ref_s/total_walking_time_imu_s`，GSD tab 用样本级的 `tp_samples/fp_samples/fn_samples` 直接复用 `_detection_summary()`，bout 级的 `tp_wb/fp_wb/fn_wb` 另外单独一张表展示
- 生成入口：`scripts/imu_pipeline.py` 的 `_eval_common()` 里，当前任务的 keyword 含 `outoflab` 时会自动走 `batch_gsd_analysis_indip` 而不是 lab 用的 `batch_wb_analysis_indip`（MobGap/SKDH 两条评估路径都走 `_eval_common`，因此两边都覆盖到了）
- **IC / Stride 级别现在也支持 at-home**（原先的已知局限已解决）：at-home 的 `*Outoflab.mat` 里
  `ContinuousWalkingPeriod` 虽然是"每个 bout 一个元素"的结构体数组，但**每个 bout 元素自己带着跟 lab 版
  CWP 一样丰富的 per-stride 字段**（`Stride_InitialContacts`、`SingleSupport_Duration`/
  `DoubleSupport_Duration`、`InitialContact_Event`/`FinalContact_Event` 等），只是 `load_indip_cwp_athome`
  只读了每个 bout 的聚合标量字段（Start/End/NumberStrides…），把这些 per-stride 数组扔掉了——这才是之前
  "读不到东西"的真正原因，不是数据本身没有这个粒度（用真实 P04 数据现场打开 `.mat` 验证过）。
  - `load_indip_stride_athome(mat_path)` / `load_indip_ic_athome(mat_path)`（`src/ug3imu/indip/__init__.py`）：
    遍历每个 bout，复用 lab 现成的 `_cwp_to_stride_df`/`_cwp_to_ic_df`（原样不改），拼成一张覆盖整段
    recording 的连续表，含全部 4 个 support 列。
  - `batch_stride_analysis_indip_athome`/`batch_ic_analysis_indip_athome`：跟 `batch_gsd_analysis_indip`
    一样**不按 task_name 匹配**——每个 stride/IC 文件都对着同一份连续 reference 匹配。SKDH 的 at-home
    stride csv 还保留原始 GaitLumbar 列（`IC Time` 等，没转成 start/end 帧号），先经过
    `metrics.stride_evaluation.normalize_athome_stride` 转换（复用 lab SKDH 那套
    `gait_lumbar_df_to_stride_df` 转换逻辑）再匹配；MobGap 的 at-home 输出已经有 start/end，原样通过。
  - 输出文件：`ic_error_indip_outoflab.csv`、`ic_metrics_indip_outoflab.csv`、
    `stride_error_indip_outoflab.csv`、`stride_rmse_indip_outoflab.csv`——跟 WB/GSD 一样固定 `outoflab`
    后缀，四种 at-home 指标命名规律完全统一。report_app.py 这边不需要任何特殊处理，`_ic_stride_glob_patterns`
    走跟 lab 场景一样的 `{metric}_{ref}_{scenario}.csv` 公式，IC/Stride tab 不用改代码就能自动显示。
  - **已知问题**：真实数据里 support 指标（尤其 `single_support_s`）尾部偶尔有物理上不合理的异常值
    （比如负数），这是复用的 `_support_phase_durations`（lab 也在用、已验证过的同一套逻辑）在个别 bout
    边界/稀疏 stride 情况下的固有毛刺，中位数/四分位是正常的，暂未加额外过滤。

---

## PDF 导出

`_build_pdf_report(ic_filt, stride_filt, ic_error_filt, wb_filt, device, selected, ref_label, scenario)`

包含：IC Detection → IC Timing Error → Stride 摘要表 → Bland-Altman（stride）→ Walking Bouts 摘要表 → WB Bland-Altman

---

## 已知问题 / 注意事项

- stride_evaluation.py 中参考列固定命名为 `{p}_v3d`，与实际参考系无关。`_stride_ref_col` 用于绕过此问题。
- WB 评估要求先跑 SKDH/MobGap pipeline 生成 `wb/` 文件夹，才会有 `wb_error_*.csv`。
- `_icc21` 计算 ICC(2,1)，需要至少 3 个样本；少于 3 个时返回 NaN。

---

## INDIP vs Mocap 验证

验证 INDIP 参考系统本身相对 V3D mocap 的精度（而不是某个 IMU 算法相对参考系的精度）。只依赖
`{ref_root}/{subject}/V3D/*.txt` + `{ref_root}/{subject}/INDIP/*.mat`，与设备/算法输出无关，因此只覆盖
Lab 场景（V3D 和 INDIP 都是 lab-only 参考系统）。

### 生成评估数据

```
python scripts/evaluate_indip_vs_mocap.py --ref_root R:/... --output_root R:/...
python scripts/evaluate_indip_vs_mocap.py --subjects P03 P04   # 只跑指定 subject
python scripts/evaluate_indip_vs_mocap.py                       # 用 project_settings.json 里的 ref_root/output_root
```

核心匹配逻辑复用 `src/ug3imu/indip/__init__.py` 里新增的三个函数（把 INDIP 当作"被检测系统"、
V3D 当作参考，跟 `batch_ic_analysis_indip` / `batch_stride_analysis_indip` 的检测/参考角色刚好相反）：
`batch_ic_analysis_indip_vs_mocap`, `batch_stride_analysis_indip_vs_mocap`, `batch_wb_analysis_indip_vs_mocap`。
匹配用的公共帧率默认是 INDIP 原生的 100 Hz（`fs=100.0`），只对 V3D 一侧做频率转换，INDIP 一侧不需要重采样。

输出到 `{output_root}/{subject}/INDIP/Evaluation/`，文件命名和 `report_app.py` 的 v3d 参考文件完全一致
（`ic_metrics_v3d_lab.csv` 等），`algorithm` 列固定为 `"INDIP"`。

### 查看结果

```
streamlit run scripts/report_app_indipvsmocap.py
```

侧边栏只有 Results root + Subject 多选（没有 Device / Reference / Scenario 选择器，因为都是固定的）。
Tabs 结构跟 `report_app.py` 完全一致：IC Detection / Stride / Walking Bouts，散点图/BA 图同样带
subject/trial 悬浮提示。没有 GSD tab（outoflab 不适用）。
