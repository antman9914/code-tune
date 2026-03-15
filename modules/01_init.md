# MODULE 1: 项目初始化与基线快照

## 1.1 定位训练脚本

在 TARGET_DIR 下按优先级查找入口文件：
`train.py` → `main.py` → `run.py` → `scripts/train.py` → `train.sh` → `main.sh`

若以上均不存在，列出 TARGET_DIR 下所有可执行脚本（`.py` / `.sh`）：
- 若唯一，自动选择
- 若有多个，询问用户

记为 `TRAIN_SCRIPT`。

## 1.2 发现可修改文件（mutable_files）

扫描 TARGET_DIR，识别属于"项目代码"的文件（以下为默认规则，可被用户覆盖）：

**纳入 mutable_files（可修改）：**
- Python 代码文件：`*.py`（排除 `__pycache__/`）
- 配置文件：`*.yaml`、`*.yml`、`*.json`（排除 package.json、lock 文件）、`*.toml`（排除 pyproject.toml、poetry.lock）
- 推断逻辑：通过 TRAIN_SCRIPT 的 import 语句和配置加载路径，确定哪些文件直接影响训练行为

**排除（不可修改）：**
- `data/`、`datasets/`、`*.csv`、`*.npz`、`*.pt`（数据文件）
- `logs/`、`checkpoints/`、`outputs/`（训练输出）
- `__pycache__/`、`*.pyc`、`*.egg-info/`（编译产物）
- `requirements.txt`、`setup.py`、`pyproject.toml`（依赖管理）
- `*.sh`（提交脚本为基础设施，不修改）
- `.autoresearch/`（本工具的状态目录）

向用户展示发现的文件列表并确认：
```
发现的可修改文件（mutable_files）：
  [1] config.yaml      ← 配置文件
  [2] train.py         ← 训练逻辑
  [3] model.py         ← 模型定义
  ...

是否需要增删文件？[Enter 继续 / 输入编号删除 / 输入路径添加]
```

将确认后的列表记为 `mutable_files`。

## 1.3 分析训练脚本

读取 TRAIN_SCRIPT 和所有 mutable_files 的完整内容，提取：

**A. 目标 Metric**：
- 任务类型（分类/回归）
- primary_metric_name（如 val_acc、val_loss、mAP 等）
- primary_metric_direction（max / min）
- 过拟合诊断字段（若可获取，如 train_acc vs val_acc）

**B. Epoch 预算**：
- 读取配置文件或代码中的 `epochs` 字段（或等效字段名，如 `num_epochs`、`max_epochs`、`range(200)` 等）
- 记为 `epoch_budget_per_exp`
- 若无法确定，询问用户

**C. 可修改参数清单**（用于 Module 2 假设生成的参考）：
- 数值型超参数及其当前值、所在文件和位置（行号或字段名）
- 模型结构参数（层数、宽度、激活函数等）
- 训练动态参数（optimizer 类型、scheduler 类型等）
- 参数可在任意位置：config yaml/json、argparse defaults、代码硬编码——均记录

若发现 loss 函数设计有根本性问题（优化目标与任务不一致、NaN 风险等），
输出具体说明并**终止初始化**，等待用户修复。

## 1.4 训练输出检测与日志注入

检测 TRAIN_SCRIPT 是否在训练结束后写入**可解析的 metric 文件**（CSV、JSON、JSONL 等）。

### 情况 A：已有结构化输出

若训练脚本已写入包含 per-epoch 指标的文件（如 CSV 或 JSON），确认：
- 文件路径模式（如 `logs/exp_*.csv`、`results/*.json`）
- 字段名（如 `val_acc`、`val_loss`）
- 聚合方式（`max` / `min` / `direct`）

将上述信息填入 `metric_config`，记录 `logging_injected: false`，直接进入 1.5。

### 情况 B：无结构化输出（仅 stdout / 仅 checkpoint）

若训练脚本不写入可解析文件，**自动注入日志代码到 TRAIN_SCRIPT**：

1. 读取 TRAIN_SCRIPT，定位训练循环（epoch 迭代），识别每个 epoch 结束时可用的变量（train_loss、val_loss、train_acc、val_acc 或其等效变量名）
2. 在文件中进行精确编辑，注入以下功能：
   - 文件顶部：补充 `import csv, json, os` 及 `from datetime import datetime`（已有则跳过）
   - 训练循环前：初始化时间戳、`logs/` 目录、行列表
   - 每 epoch 末尾：将 `{epoch, train_loss, val_loss, train_acc, val_acc}` 追加到行列表
   - 训练循环后：写入 `logs/exp_{timestamp}.csv` 和 `logs/exp_{timestamp}_summary.json`

   summary JSON 至少包含：
   ```json
   {
     "best_{metric_name}": <全程最优值>,
     "final_{metric_name}": <最后一 epoch 的值>,
     "epochs_run": <实际跑的 epoch 数>,
     "log_file": "logs/exp_{timestamp}.csv",
     "timestamp": "..."
   }
   ```

3. 注入完成后输出：
   ```
   ⚙ 已向 {TRAIN_SCRIPT} 注入日志代码（一次性，不影响后续实验）
     写入文件：logs/exp_{timestamp}.csv（per-epoch 曲线）
               logs/exp_{timestamp}_summary.json（摘要指标）
   ```

4. 将 `metric_config` 配置为读取注入后的输出格式，记录 `logging_injected: true`

**注意**：注入的代码将成为 `mutable_files` 的一部分，纳入 baseline 快照，后续 reject 时会随其他文件一起恢复。

## 1.5 检测运行环境

按优先级检测以下调度器是否可用（`which` 或 `command -v`）：

| 调度器 | 检测命令 | 提交命令 | 状态查询 |
|--------|---------|---------|---------|
| SLURM  | `sbatch` | `sbatch {submit_script}` | `squeue -j {jobid} -h` |
| PBS    | `qsub`（+检查 `qconf` 不存在） | `qsub {submit_script}` | `qstat {jobid}` |
| LSF    | `bsub` | `bsub < {submit_script}` | `bjobs {jobid}` |
| SGE    | `qsub`（+检查 `qconf` 存在） | `qsub {submit_script}` | `qstat -j {jobid}` |

若无调度器 → 本地环境。

**集群环境下查找提交脚本**（在 TARGET_DIR 下按优先级查找）：
1. `submit_train.sh`
2. `submit.sh`
3. `run.pbs` / `run.lsf` / `run.sge`
4. 任意 `*.sh` 文件中含有 `#SBATCH`、`#PBS`、`#BSUB`、`#$` 指令的文件

若集群环境下找不到提交脚本，提示用户并停止初始化。

将结果记为 `execution_env`：
```json
{
  "type": "<slurm | pbs | lsf | sge | local>",
  "submit_script": "<相对路径，集群时有效，本地时为 null>"
}
```

输出检测结果：
```
运行环境检测：{type}（{集群时：提交脚本 submit_script | 本地时：直接运行 python TRAIN_SCRIPT}）
```

## 1.6 获取用户约束

询问用户（若不回答则使用默认值）：

```
请设置搜索约束（直接回车使用默认值）：

1. 架构约束（如"仅限 MLP，不引入 Transformer/CNN"）
   默认：无约束（Claude 自主判断合理的架构改动）
   > _

2. 参数约束（如"dropout_rate 不超过 0.5，hidden_dims 每层不超过 512"）
   默认：无约束
   > _

3. 是否启用人工审查（每次修改前需要确认）？[y/N]
   默认：N
   > _
```

将用户输入记录为 `user_constraints`（纯文本，供 Module 2 在生成假设时参考）。

## 1.7 创建目录结构并写入初始 manifest.json

**在基线训练之前**先完成此步骤，以便训练时可以读取 `execution_env`。

```bash
mkdir -p TARGET_DIR/.autoresearch/baseline
mkdir -p TARGET_DIR/.autoresearch/run_logs
```

写入 `TARGET_DIR/.autoresearch/manifest.json`
（`baseline_metric` 和 `best_metric` 暂填 `null`，1.8 完成后更新）：

```json
{
  "project_dir": "<TARGET_DIR 的绝对路径>",
  "train_script": "<TRAIN_SCRIPT 的相对路径>",
  "mutable_files": ["<相对路径列表>"],
  "metric_config": { "<由 1.3 + 1.4 分析结果填写>" },
  "epoch_budget_per_exp": <值>,
  "logging_injected": <true | false>,
  "baseline_metric": null,
  "human_review": <用户设定，默认 false>,
  "user_constraints": "<用户约束的完整文本>",
  "execution_env": {
    "type": "<slurm | pbs | lsf | sge | local>",
    "submit_script": "<相对路径或 null>"
  },
  "termination_triggered": false,
  "termination_window": 5,
  "recent_metrics": [],
  "experiment_count": 0,
  "best_metric": null,
  "best_exp_id": null,
  "created_at": "<ISO 8601>"
}
```

## 1.8 运行基线训练

检查是否已有训练结果文件（`metric_config.peak_value.file_pattern` 匹配的文件）：

- **有结果文件**：读取最新一次的 metric 作为基线，跳过训练，提示用户：
  ```
  检测到已有训练结果（{最新文件名}），直接使用作为基线。
  baseline metric ({metric_name}) = {值}
  ```

- **无结果文件**：按 `manifest.json` 中的 `execution_env` 执行一次训练
  （使用 Module 3 的 3.2 节逻辑）。若训练失败，终止初始化，报告错误。

读取 metric 后，将 `manifest.json` 中的 `baseline_metric` 和 `best_metric` 更新为该值。
记为 `baseline_metric`。

## 1.9 快照基线文件

将 `mutable_files` 中的每个文件复制到 `.autoresearch/baseline/`，
保留相对于 TARGET_DIR 的目录结构：

```
对 mutable_files 中的每个 file_path：
  dest = .autoresearch/baseline/{file_path 相对 TARGET_DIR 的路径}
  确保 dest 的父目录存在
  复制 file_path → dest
```

## 1.10 Module 1 输出摘要

- 训练脚本路径
- 可修改文件列表
- Primary Metric 及读取方式（含 logging_injected 状态）
- Epoch 预算
- 运行环境（local / slurm / pbs 等）
- 用户约束摘要
- 基线 metric 值
- 初始化完成确认
