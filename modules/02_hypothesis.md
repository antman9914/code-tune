# MODULE 2: 诊断与假设生成

## 2.1 读取当前状态

从 `TARGET_DIR/.autoresearch/manifest.json` 读取：
- `best_metric`：历史最优 metric
- `baseline_metric`：初始基线
- `user_constraints`：用户约束文本
- `metric_config`：metric 读取方式

从 `experiment_log.jsonl` 读取全部实验记录（用于上下文和避免重复）。

读取 `.autoresearch/baseline/` 中的所有文件内容（这是当前最优的项目代码状态）。

## 2.2 读取训练曲线并诊断

按照 `metric_config.peak_value.file_pattern` 找到最新一次训练的输出文件，
读取完整训练曲线（全部 epoch 的 train/val metric）。

**若无训练记录**（初始化后首次运行 Module 2）：跳过诊断，标注"无历史曲线"。

**曲线诊断**：对训练曲线做开放式观察，用自然语言描述所有值得注意的现象。

不使用预定义标签，而是直接描述观察到的事实，例如：
- train/val 之间的差距及其变化趋势
- val metric 是否有拐点，拐点位置
- 收敛速度（前期/后期）
- 震荡幅度
- early stopping 是否触发，触发时的状态
- 其他任何异常形态

将自由描述的诊断结果记为 `curve_diagnosis`，作为假设生成的核心输入。

## 2.3 生成假设

综合以下信息，生成**一个**具体的代码/配置修改假设：

**输入信息：**
1. `curve_diagnosis`：本次训练曲线的问题诊断
2. `experiment_log.jsonl`：所有历史实验的假设和结果（避免重复已测试的改动）
3. `user_constraints`：用户设定的约束（架构限制、参数范围限制等）
4. 当前 baseline 代码状态（`.autoresearch/baseline/` 中的文件内容）

**假设生成原则：**

- **代价敏感**：优先尝试低代价改动（修改数值 < 修改训练逻辑 < 修改模型结构 < 修改训练目标），在简单改动被充分探索之前不建议跳到复杂的架构修改。具体判断依据是 `experiment_log.jsonl`：若数值类改动（learning_rate、weight_decay 等）在历史中测试不足 3 次，优先补足这类实验。

- **诊断导向**：根据 `curve_diagnosis` 的自由描述，推断最可能的问题根因，选择对应的改动方向。以下为常见现象与行动的对应关系，仅作参考，不做强制查表：
  - val 震荡明显 → 降低 learning_rate，或加 warmup
  - 收敛极慢 → 提高 learning_rate，或换 scheduler
  - train/val 差距持续扩大 → 增强正则化，或减少模型容量
  - train/val 都低且平稳 → 减弱正则化，或增加模型容量，或换更强激活函数
  - val 有明显拐点后下降 → 调整 early stopping patience，或强化正则
  - val 末尾仍在改善 → 增加 epoch 预算，或调整 scheduler
  - 正常收敛但仍有提升空间 → 精细搜索当前参数邻域，或尝试不同归一化/激活
  - 其他复合问题 → 优先处理最突出的单一现象，不同时调多个维度

- **不重复**：从 `experiment_log.jsonl` 检查近期历史，不重复测试已验证过的改动方向（相同参数的相近取值、已测试过的 scheduler 类型等）

- **约束优先**：在生成任何假设前，先对照 `user_constraints` 全文检查合规性。对于架构类改动，额外确认不引入约束中明确禁止的模块。

**假设类型参考**（按复杂度排序，不做强制分层，仅作参考）：

| 类型 | 典型改动 |
|------|---------|
| 超参数值 | learning_rate, weight_decay, dropout_rate, batch_size 的数值调整 |
| 训练动态 | optimizer 类型、lr_scheduler 类型/参数、gradient clipping、warmup |
| 模型架构 | 层数、模型容量相关参数（读 baseline 代码确定字段名）、activation 函数、normalization 层（受 user_constraints 约束）|
| 训练过程 | loss 函数、class weighting、label smoothing、正则化策略组合 |

每次只做**一个方向**的改动，不同时修改多个正交维度（例如不同时改 lr 和架构）。

### 约束检查

1. 读取 `user_constraints` 全文，识别所有限制条件
2. 若改动涉及架构，检查是否在允许的模块范围内（如"仅限 MLP"则不引入 Transformer 等）
3. 若改动涉及参数范围，检查是否满足数值约束（如"hidden_dims 每层不超过 512"）
4. 若假设违反任何约束，放弃该假设，生成替代假设

### 假设输出格式

```
假设：{exp_id}
诊断依据：{curve_diagnosis + 具体观察，如"val_acc 在末 20 epoch 震荡，std=0.023 > mean×0.15"}
改动描述：{一句话，如"将 learning_rate 从 0.01 降至 0.002"}
改动理由：{2-3 句话说明为什么这个改动应该有帮助}

文件改动：
  文件 1：{文件相对路径}
    改动类型：{值修改 / 代码修改 / 新增代码}
    具体内容：{精确的 old_content → new_content，或要添加的代码片段}
  文件 2：{若有}
    ...
```

## 2.4 应用假设到项目文件

**前置步骤：确保项目文件处于 baseline 状态**

将 `.autoresearch/baseline/` 中的每个文件复制回项目对应位置：
```
对 mutable_files 中的每个 file_path：
  src = .autoresearch/baseline/{file_path 相对 TARGET_DIR 的路径}
  复制 src → TARGET_DIR/{file_path}
```

这一步确保每次实验都从最优的已知状态出发，而不是上一次实验（无论接受还是拒绝）的状态。

**应用文件改动**：

按假设描述，对项目文件执行精确的编辑操作。
若改动涉及新建文件，将该文件路径加入 `manifest.json` 的 `mutable_files` 列表。

**记录 diff（内存中保留，供 Module 4 写入 run_log）**：

对每个被修改的文件，用以下命令生成 diff 文本并保存在当前对话上下文中：
```bash
diff .autoresearch/baseline/{相对路径} {项目文件路径} || true
```
（`|| true` 确保 diff 找到差异时返回的 exit code 1 不被误判为错误）

将所有文件的 diff 合并为 `CURRENT_DIFF` 变量，格式：
```
文件：{相对路径}
{diff 输出内容}
```

不创建 experiments/ 子目录，不单独保存文件。Module 4 写 run_log 时直接使用 `CURRENT_DIFF`。

## 2.5 Module 2 输出摘要

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
实验 {exp_id}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
曲线诊断  : {curve_diagnosis}
假设      : {改动一句话描述}
改动文件  : {文件列表}
改动理由  : {2-3 句话}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```
