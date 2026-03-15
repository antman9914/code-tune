# MODULE 4: 接受/拒绝与终止检查

## 4.1 判断接受或拒绝

从 `result.json` 读取 `metric_after`，从 `manifest.json` 读取 `best_metric` 和 `metric_config.primary_metric_direction`。

**接受条件**：
```
direction = "max"：metric_after > best_metric
direction = "min"：metric_after < best_metric
```

任何严格改善（哪怕微小）都接受——保持贪心策略，微小改善也可能是后续更大改善的基础。

## 4.2 接受分支

**更新 baseline 快照**：

将当前项目中 `mutable_files` 的所有文件复制到 `.autoresearch/baseline/`，覆盖之前的快照：

```
对 mutable_files 中的每个 file_path：
  src = TARGET_DIR/{file_path}
  dest = .autoresearch/baseline/{file_path 相对 TARGET_DIR 的路径}
  确保 dest 父目录存在
  复制 src → dest
```

若本次假设新增了文件（已加入 mutable_files），它们同样被纳入快照，
之后的 reject 也会恢复这些文件。

**更新 result.json**：`status` 改为 `"accept"`。

**更新 manifest.json**：
```json
"best_metric": {metric_after},
"best_exp_id": "{exp_id}"
```

输出：
```
✓ 接受：{metric_name} {best_metric} → {metric_after}（+{|delta|}）
  baseline 已更新
```

## 4.3 拒绝分支

**恢复项目文件**：

将 `.autoresearch/baseline/` 的所有文件复制回项目对应位置：

```
对 .autoresearch/baseline/ 下的每个文件：
  相对路径 = 该文件相对于 .autoresearch/baseline/ 的路径
  dest = TARGET_DIR/{相对路径}
  确保 dest 父目录存在
  复制文件 → dest
```

若本次假设新增了文件（不在 baseline/ 中），从 Module 2 上下文中保留的 `CURRENT_DIFF`
或 `experiment_log.jsonl` 最新一条的 `files_changed` 字段找出新增文件列表，
逐一删除，并从 `manifest.json` 的 `mutable_files` 中移除这些路径。

**更新 result.json**：`status` 改为 `"reject"`。

输出：
```
✗ 拒绝：{metric_name} {metric_after}（基线：{best_metric}，delta：{delta}）
  项目文件已恢复到 baseline 状态
```

## 4.4 追加实验日志

将以下内容追加到 `TARGET_DIR/.autoresearch/experiment_log.jsonl`（每行一条）：

```json
{
  "exp_id": "{exp_id}",
  "timestamp": "{ISO 8601}",
  "hypothesis_summary": "{改动一句话描述，来自 Module 2 生成的假设内容}",
  "files_changed": ["{相对路径}"],
  "metric_before": {metric_before},
  "metric_after": {metric_after},
  "delta": {delta},
  "status": "{accept / reject / error}",
  "curve_diagnosis": "{curve_diagnosis}",
  "overfitting": "{无 / 轻微 / 中等 / 严重 / 未知}"
}
```

更新 `manifest.json`：
```json
"experiment_count": {当前值 + 1}
```

## 4.5 更新搜索窗口

将本次实验的 `metric_after` 追加到 `manifest.json` 的 `recent_metrics` 列表（滑动窗口）：

```
recent_metrics.append(metric_after)
若 len(recent_metrics) > K：移除最旧的元素（只保留最近 K 条）
```

`K` 从 `manifest.json` 的 `termination_window` 字段读取（初始化时设为 5）。

将更新后的 `recent_metrics` 写回 `manifest.json`。

## 4.6 终止条件检查

参考 param-tune 的终止逻辑，基于滑动窗口判断搜索是否收敛。

**前提**：`len(recent_metrics) >= K`（窗口未满则跳过终止检查，继续搜索）。

**计算 δ（改善阈值）**：

```
metric_range：
  对 accuracy / F1 等归一化指标 → metric_range = 1.0
  对 loss 等无固定上界指标      → metric_range = max(全部 metric_after) - min(全部 metric_after)
  （从 experiment_log.jsonl 中所有记录的 metric_after 计算）

δ = 0.01 × metric_range   # 1% of metric range
```

**终止判断**：

```
window_best = max(recent_metrics)   # direction=max 时
             （或 min(recent_metrics)，direction=min 时）

global_best = manifest.best_metric

终止条件：window_best 相对 global_best 无显著改善
  direction=max：window_best < global_best + δ
  direction=min：window_best > global_best - δ

含义：最近 K 次实验中，没有任何一次实现了超过 δ 的改善
```

若满足终止条件：
→ 将 `manifest.json` 中 `termination_triggered` 设为 `true`，写回文件。

否则继续（返回主流程 Step 1）。

## 4.7 写入实验日志文件

将本次实验的完整信息写入 `TARGET_DIR/.autoresearch/run_logs/{exp_id}.md`
（若 `run_logs/` 目录不存在，先创建）。

日志内容直接由 Claude 输出，**不依赖外部文件**——假设描述来自 Module 2 生成时的内容，
diff 来自 Module 2 保留的 `CURRENT_DIFF`，训练结果来自 Module 3 读取的数据：

```markdown
# {exp_id} — {ISO timestamp}

## 假设
**诊断依据**：{curve_diagnosis 及具体观察}
**改动描述**：{一句话}
**改动理由**：{2-3 句话}

## 改动内容
{CURRENT_DIFF 的完整内容}

## 训练结果
| 指标 | 值 |
|---|---|
| {primary_metric_name}（best） | {metric_after} |
| best_train_acc | {overfitting_train 或 N/A} |
| 过拟合诊断 | {无 / 轻微 / 中等 / 严重} |
| 曲线诊断 | {curve_diagnosis} |

## 决策
{✓ 接受 / ✗ 拒绝}（delta：{+/-值}）

## 搜索状态
- 历史最优 : {best_exp_id}（{best_metric}）
- 初始基线 : {baseline_metric}
- 窗口     : {recent_metrics}（K={K}）
- 实验总数 : {experiment_count}
- 终止状态 : {未触发 / 已触发}
```

## 4.8 输出本轮完整摘要

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
实验 {exp_id} 总结
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
结果     : {✓ 接受 / ✗ 拒绝}
metric   : {metric_after}  （基线：{baseline_metric} | 历史最优：{best_metric}）
窗口状态  : recent_metrics = {recent_metrics}（K={K}，δ={δ:.4f}）
总实验数  : {experiment_count}

已接受的改动（累计）：
  {从 experiment_log.jsonl 列出所有 accept 的 hypothesis_summary}

下一步：{继续搜索 / 搜索终止（窗口内 K 次无显著改善）}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```
