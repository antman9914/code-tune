# MODULE 3: 运行实验与读取结果

## 3.1 读取运行环境

从 `manifest.json` 的 `execution_env` 字段读取运行环境（已在 Module 1 初始化时检测并保存）：

```
env_type      = manifest.execution_env.type        # slurm / pbs / lsf / sge / local
submit_script = manifest.execution_env.submit_script  # 集群时有效，本地时为 null
```

## 3.2 运行训练

**集群环境**：

提交作业并捕获 job ID：
```
SLURM: jobid = sbatch {submit_script} 输出中的数字
PBS:   jobid = qsub {submit_script} 输出（格式如 12345.hostname）
LSF:   jobid = bsub < {submit_script} 输出中 "Job <NNNN>" 的数字
SGE:   jobid = qsub {submit_script} 输出中 "Your job NNNN" 的数字
```

轮询循环（每 60 秒一次）：
```
判断作业是否结束：
  SLURM: squeue -j {jobid} -h → 输出为空则结束
  PBS:   qstat {jobid}         → 非零退出码则结束
  LSF:   bjobs {jobid}         → 含 DONE 或 EXIT 则结束
  SGE:   qstat -j {jobid}      → 非零退出码则结束

每次输出：[{时间戳}] 作业 {jobid} 运行中，已等待 {N} 分钟...
```

作业结束后检查日志文件（`--output`/`-o` 指定的路径），
若末尾含 `Error`、`Traceback`、`CANCELLED`、`FAILED`，
输出末尾 20 行，返回错误状态给主流程。

**本地环境**：

```bash
cd TARGET_DIR && python {TRAIN_SCRIPT}
```

若脚本不是 `.py`，直接执行（去掉 `python`）。

## 3.3 确认训练输出文件已生成

按 `metric_config.peak_value.file_pattern` 查找最新的训练输出文件。

若文件未出现（或时间戳未更新），等待最多 30 秒后再次检查。
若仍未出现，返回错误状态。

## 3.4 读取实验 Metric

按照 `metric_config` 描述的方式读取本次实验的 metric 值：

1. 按 `file_pattern` 定位最新训练输出文件
2. 按 `aggregation` 决定读取方式：
   - `direct`：直接读取 `field_or_column` 字段的值
   - `max` / `min`：扫描该列所有值，取极值
3. 读取诊断字段（若 `overfitting_diagnosis.available = true`）

记为 `metric_after`。

同时读取训练曲线，提取 `curve_diagnosis`（复用 Module 2 的诊断逻辑）。

## 3.5 保存实验结果

将以下内容写入 `TARGET_DIR/.autoresearch/experiments/{exp_id}/result.json`：

```json
{
  "exp_id": "{exp_id}",
  "metric_before": {manifest.best_metric},
  "metric_after": {metric_after},
  "delta": {metric_after - metric_before（注意 direction：max 时正为好）},
  "curve_diagnosis": "{curve_diagnosis}",
  "overfitting_train": {值或 null},
  "overfitting_val": {值或 null},
  "epochs_used": {epoch_budget_per_exp},
  "status": "pending"
}
```

状态此时为 `"pending"`，由 Module 4 更新为 `"accept"` 或 `"reject"`。

## 3.6 Module 3 输出摘要

```
训练完成：
  metric ({metric_name}) : {metric_after}（基线：{metric_before}，delta：{+/-值}）
  曲线诊断               : {curve_diagnosis}
  过拟合诊断             : {无 / 轻微 / 中等 / 严重}（若可获取）
```
