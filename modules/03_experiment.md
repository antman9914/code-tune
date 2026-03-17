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

作业结束后检查日志文件（`--output`/`-o` 指定的路径），按以下优先级判断状态：

1. **TIMEOUT**：日志末尾含 `DUE TO TIME LIMIT`、`TIME LIMIT`，或 `squeue`/`sacct` 状态为 `TIMEOUT`
   → 返回 `status: timeout` 给主流程（主流程自动继续，无需用户干预）

2. **其他失败**：日志末尾含 `Error`、`Traceback`、`CANCELLED`、`FAILED`、`OOM`、`Killed`
   → 输出末尾 20 行，返回 `status: error` 给主流程（主流程停止等待用户）

3. **无输出文件**：训练脚本无报错但 metric 文件未生成（见 3.3）
   → 返回 `status: error`

4. **正常完成**：无上述关键词，且 metric 文件已生成
   → 返回 `status: success`

**本地环境**：

```bash
cd TARGET_DIR && python {TRAIN_SCRIPT}
```

若脚本不是 `.py`，直接执行（去掉 `python`）。

本地训练结束后，按与集群环境相同的优先级判断状态：检查标准输出/标准错误中是否含 `Error`、`Traceback`、`Killed` 等关键词，若有则返回 `status: error`；无报错且 metric 文件已生成则返回 `status: success`。本地环境不存在 TIMEOUT 状态。

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

## 3.5 向 Module 4 传递实验结果

将以下数据保留在当前对话上下文中，供 Module 4 直接使用（不写入任何文件）：

```
metric_before      = manifest.best_metric（在本次实验前从 manifest 读取，用于 delta 计算和日志）
metric_after       = {读取到的 metric 值}
curve_diagnosis    = {训练曲线诊断描述}
overfitting_train  = {值或 null}
overfitting_val    = {值或 null}
epochs_used        = {epoch_budget_per_exp}
```

所有持久化写入由 Module 4 统一完成。

## 3.6 Module 3 输出摘要

```
训练完成：
  metric ({metric_name}) : {metric_after}（基线：{metric_before}，delta：{metric_after - metric_before 的有符号值}）
  曲线诊断               : {curve_diagnosis}
  过拟合诊断             : {无 / 轻微 / 中等 / 严重}（若可获取）
```
