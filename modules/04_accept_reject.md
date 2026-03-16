# MODULE 4: 接受/拒绝与终止检查

## 4.1 判断接受或拒绝

从 Module 3 上下文读取 `metric_after`，从 `manifest.json` 读取 `best_metric` 和 `metric_config.primary_metric_direction`。

将此时读取到的 `best_metric` 记为 `metric_before`（后续 4.2 会更新 manifest，需提前保留此值用于 4.4 日志）。

**接受条件**：
```
direction = "max"：metric_after > best_metric
direction = "min"：metric_after < best_metric
```

任何严格改善（哪怕微小）都接受——保持贪心策略，微小改善也可能是后续更大改善的基础。

## 4.1.5 决策解释与用户确认（仅 explain_decisions: true 时执行）

若 `manifest.explain_decisions: false`，跳过本节直接进入 4.2/4.3。

若 `manifest.explain_decisions: true`，在执行任何文件操作之前，输出以下结构化解释，**停止并等待用户回复后再继续**：

---

**格式模板（用通俗语言输出，避免专业术语堆砌）**：

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
实验 {exp_id} 决策说明
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

【本次改了什么，为什么这样改】
{2-4 句。用类比或直觉解释改动思路。
例："我们把 dropout 比例从 0.3 提高到 0.5。
  Dropout 就像让部分神经元'随机请假'，迫使网络不依赖某几个特征，
  从而减少过拟合——即训练表现好但测试表现差的现象。
  由于上一轮你的模型在训练集准确率远高于验证集，我们判断过拟合较严重，
  所以加大 dropout 力度。"}

【实验结果说明什么】
{2-3 句。解读 metric 变化，结合曲线诊断。
例："验证准确率从 87.2% 上升到 88.6%，提升了约 1.4 个百分点。
  同时训练集与验证集的差距从 9% 缩小到 6%，说明过拟合确实有所缓解。
  训练曲线较平稳，无明显震荡，说明这次改动方向是正确的。"}

【为什么建议{接受/拒绝}】
{1-2 句。直接说明理由。
例："验证 metric 有明确提升，且过拟合改善，符合我们的优化目标，因此建议接受。"
或："验证 metric 下降了 0.8%，改动没有带来提升，需要回到之前的版本重新尝试。"}

【相关背景（选读）】
{列出 1-3 条最相关的参考，每条格式：[作者等, 年份] 标题 — 1-2 句说明与本次改动的关联。
只引用领域内广泛引用的经典或重要文献，不要捏造引用。
若本次改动无强对应文献（如简单调整 batch_size），此部分可省略。
例：
• [Srivastava et al., 2014] "Dropout: A Simple Way to Prevent Neural Networks from Overfitting"
  — 提出 Dropout 机制的原始论文，说明随机丢弃神经元如何作为正则化手段抑制过拟合。
• [Goodfellow et al., 2016] Deep Learning (Chapter 7: Regularization)
  — 系统介绍 L2、Dropout、early stopping 等正则化方法的原理与适用场景。}

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
推荐决策：{✓ 接受 / ✗ 拒绝}

是否执行此决策？[Y/n]（直接回车 = 执行推荐决策）
```

---

**在用户回复之前不得执行任何文件操作（baseline 快照/恢复）。**

- 用户回复 `Y` 或直接回车 → 按推荐决策执行（进入 4.2 或 4.3）
- 用户回复 `N` → 询问用户希望改为{接受/拒绝}，按用户意见执行，并在 run_log 的"决策"一节注明"用户覆盖推荐决策"
- 用户回复其他内容（如追问解释） → 回答后重新展示确认提示，继续等待

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
找出新增文件列表，逐一删除，并从 `manifest.json` 的 `mutable_files` 中移除这些路径。

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
  "epochs_used": {epochs_used},
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

## 4.7 写入 run_log

**此步骤不可跳过，无论本轮结果是 accept / reject / error / timeout，都必须写入。**

将本次实验的完整信息写入 `TARGET_DIR/.autoresearch/run_logs/{exp_id}.md`
（若 `run_logs/` 目录不存在，先创建）。

日志内容直接由 Claude 输出，不依赖外部文件——假设描述来自 Module 2 生成时的内容，
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

## 决策
{✓ 接受 / ✗ 拒绝}（delta：{+/-值}）
{若用户覆盖了推荐决策，注明：用户覆盖推荐决策，实际执行：{接受/拒绝}}
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
