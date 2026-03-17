---
name: code-tune
description: 通用 ML 模型代码级自动优化工具。针对完整项目（含代码+配置）进行
             假设驱动的自动搜索：每次提出一个代码/配置修改假设，运行验证，
             按 metric 决定接受或回滚，直到搜索收敛。
             支持多文件项目，备份机制与项目结构无关。
             触发词：code-tune、代码调参、架构搜索、自动优化
user-invocable: true
---

# ML 代码级自动优化工作流

## 使用方式
- `/code-tune` — 在当前目录启动
- `/code-tune ~/my_project` — 指定项目路径

TARGET_DIR = $ARGUMENTS（若为空则用当前目录）

启动前确认 TARGET_DIR 下存在至少一个 `.py` 或 `.sh` 文件，否则报错退出。
（具体训练脚本的发现与选择由 Module 1 的 1.1 节处理）

---

## 启动检查

检查 `TARGET_DIR/.autoresearch/manifest.json` 是否存在：

**不存在** → 首次运行，执行 **初始化流程**

**已存在** → 读取 manifest，按以下条件分支：
- `termination_triggered: true` → 上次搜索已完成，输出提示后退出：
  ```
  本项目的 code-tune 搜索已在上次运行中完成（共 {experiment_count} 次实验）。
  若需重新搜索，请删除 .autoresearch/ 目录后重新运行。
  ```
- `baseline/` 目录为空或缺失 → 初始化未完成，重新执行初始化流程
- 否则 → 执行 **循环流程**（从上次中断处继续）

---

## 初始化流程（仅首次运行执行一次）

### Init — 项目发现与基线快照

读取并执行 `~/.claude/skills/code-tune/modules/01_init.md`

初始化完成后，展示以下摘要：
```
项目已初始化：
  训练脚本        : {train_script}
  可修改文件      : {mutable_files}
  运行环境        : {execution_env.type}（{集群时显示 submit_script}）
  Primary Metric  : {metric_name} ({direction})
  Epoch 预算/轮   : {epoch_budget}
  用户约束        : {user_constraints 摘要 | 无}
  人工审查模式    : {human_review ? "已启用（每次修改前确认）" : "关闭"}
  决策解释模式    : {explain_decisions ? "已启用（每次决策前说明依据）" : "关闭"}
  日志注入        : {logging_injected 时显示"已向 {train_script} 注入日志代码" | 否则"项目已有输出文件"}
  基线 metric     : {baseline_metric}（来自首次训练）
```

**若 `human_review: true`**：追加"即将开始自动优化，是否继续？[Y/n]"，等待确认后进入循环。
**若 `human_review: false`**：直接进入循环，不等待。

---

## 循环流程

以下步骤循环执行，直到触发终止条件：

### Step 1 — 生成并应用假设

读取并执行 `~/.claude/skills/code-tune/modules/02_hypothesis.md`

若生成假设失败（约束冲突、无可探索空间等），停止循环，报告原因。

**人工审查模式**（可选）：
若 manifest.json 中 `human_review: true`，在应用修改前展示具体的文件变更并等待用户确认。
默认 `human_review: false`，Claude 直接应用修改。

### Step 2 — 运行实验

读取并执行 `~/.claude/skills/code-tune/modules/03_experiment.md`

若训练失败（Python 报错、SLURM 失败、TIMEOUT 等），执行以下操作：

1. 从 `.autoresearch/baseline/` 恢复所有可修改文件（回滚本次假设）

2. 将本次实验追加到 `experiment_log.jsonl`（完整字段，不可省略）：
   ```json
   {
     "exp_id": "{exp_id}",
     "timestamp": "{ISO 8601}",
     "hypothesis_summary": "{来自 Module 2 生成的改动一句话描述}",
     "files_changed": ["{Module 2 应用的文件列表}"],
     "metric_before": "{manifest.best_metric}",
     "metric_after": null,
     "delta": null,
     "epochs_used": null,
     "status": "{timeout 或 error}",
     "curve_diagnosis": "{来自 Module 2 的诊断，若无则填 N/A}",
     "overfitting": "未知"
   }
   ```

3. 将 `manifest.json` 中的 `experiment_count` +1（此步骤通常由 Module 4 完成，error/timeout 时须在此手动执行）

4. **写入 run_log**（`run_logs/{exp_id}.md`），格式如下，此步不可跳过：
   ```markdown
   # {exp_id} — {ISO timestamp}

   ## 假设
   **诊断依据**：{curve_diagnosis}
   **改动描述**：{一句话}
   **改动理由**：{2-3 句话}

   ## 改动内容
   {CURRENT_DIFF 的完整内容}

   ## 训练结果
   训练失败：{error / timeout}
   失败信息（最后 20 行）：
   {错误日志内容}

   ## 决策
   ✗ 回滚（训练失败，无 metric）
   ```

5. 若是 TIMEOUT：**直接继续循环**（不等待用户），在下一轮假设中考虑减少 epoch 或换更轻量架构
6. 若是其他错误（Python 报错等）：停止循环，等待用户处理

### Step 3 — 接受/拒绝与终止检查

读取并执行 `~/.claude/skills/code-tune/modules/04_accept_reject.md`

### Step 4 — 终止检查

读取 `.autoresearch/manifest.json` 中的 `termination_triggered`：
- `true` → 执行 **收尾流程**，退出循环
- `false` → 回到 Step 1

**收尾流程**：

1. 确认项目文件已处于最优状态（reject 分支已恢复，或 accept 分支文件本身就是最优）

2. **清除注入的日志代码**（若 `manifest.logging_injected: true`）：
   读取 `train_script`，删除所有以下标记之间的代码块（含标记行本身）：
   ```
   开始标记：# >>> code-tune: logging injection start
   结束标记：# <<< code-tune: logging injection end
   ```
   写回文件。输出：
   ```
   ⚙ 已从 {train_script} 移除注入的日志代码，训练脚本已恢复原始状态
   ```

3. 删除 baseline 目录：`rm -rf TARGET_DIR/.autoresearch/baseline/`

4. 输出摘要：

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
code-tune 搜索完成
共进行实验：{N} 次
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
最优实验：{exp_id}
  metric  : {metric_name} = {值}（相对基线提升 {delta}）
  修改文件 : {files_changed}
  变更摘要 : {hypothesis_summary}

项目文件已处于最优状态，注入代码已清除，.autoresearch/baseline/ 已清理。
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## 状态文件说明

所有状态文件存储在 `TARGET_DIR/.autoresearch/` 目录下：

```
.autoresearch/
  manifest.json          # 项目配置、约束、搜索状态（主控状态）
  experiment_log.jsonl   # 每行一条实验记录（追加写入，不修改已有行）
  baseline/              # 当前已接受的最优项目文件快照（搜索结束后自动删除）
    config.yaml
    train.py
    model.py
    ...（mutable_files 中的所有文件）
  run_logs/
    exp_001.md           # 假设描述 + diff + 训练结果 + 决策（完整记录）
    exp_002.md
    ...
```

---

## 手动干预

- 调整约束：直接编辑 `manifest.json` 的 `user_constraints` 字段，下一轮生效
- 重置搜索窗口：将 `manifest.json` 中的 `recent_metrics` 清空为 `[]`
