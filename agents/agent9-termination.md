# Agent 9：终止判断 Agent

**职责**：综合 Agent 7（综合输出）和 Agent 8（边界守卫）的结果，判断当前报告是否满足终止条件，决定：
- ✅ 终止：输出最终报告
- 🔄 继续：列出优先缺口，返回 Phase 1 补一轮
- ❌ 失败：报告存在根本性问题，需重新检索

---

## 输入

```json
{
  "domain": "家用按摩仪",
  "date_range": "2026.6.1-2026.6.30",
  "report_type": "monthly",
  "constraints": {
    "strict_date_range": true,
    "min_per_dim": 3,
    "max_iterations": 2,  // 最大迭代轮数（防止无限循环）
    "current_iteration": 1
  },
  "agent7_output": {
    "markdown": "...",
    "metadata": {...}
  },
  "agent8_output": {
    "overall_status": "✓ pass / ⚠ needs-correction / ✗ fail",
    "checks": [...],
    "boundary_items_accepted": [...],
    "fixes": [...]
  }
}
```

---

## 终止判断流程

```
Step 1: 检查 agent8.overall_status
  → ✓ pass → 进入 Step 2
  → ⚠ needs-correction → 进入 Step 3
  → ✗ fail → 直接进入 Step 4（补一轮）

Step 2: 检查 3 个核心终止条件（必须全部满足）
  ① 6 个维度都有 ≥ min_per_dim 条目
  ② 边界守卫的 fixes 已被应用或显式接受
  ③ 一手源占比 ≥ 40%
  → 全部满足 → 终止，输出最终报告
  → 任一未满足 → 进入 Step 3

Step 3: 检查是否已达 max_iterations
  → current_iteration < max_iterations → 进入 Step 4
  → current_iteration >= max_iterations → 强制终止（诚实标注缺口）

Step 4: 列出优先缺口，返回 Phase 1 补一轮
  - 哪些维度条目不足？
  - 哪些一手源缺失？
  - 哪些边界情况需要进一步验证？
```

---

## 终止条件详解

### 条件 ① 6 个维度都有 ≥ min_per_dim 条目

```python
counts = {"1": 3, "2": 4, "3": 5, "4": 6, "5": 4, "6": 6}
for dim in counts:
    if counts[dim] < min_per_dim:  # 默认 3
        # 这一维度不够，需补
        deficit_dimensions.append(dim)
if deficit_dimensions:
    print(f"⚠ dimensions with insufficient coverage: {deficit_dimensions}")
    terminate = False
```

**日报的特殊情况**：如果报告类型是日报（信息密度低），允许 min_per_dim=2；月报必须 ≥ 3。

### 条件 ② 边界守卫的 fixes 已被应用或显式接受

```python
fixes = agent8.fixes  # 修正建议列表
for fix in fixes:
    if fix.action == "rewrite_summary":
        # 检查 markdown 中该条目概述是否已被重写
        if not is_summary_rewritten(fix.target_entry_id):
            terminate = False
    elif fix.action == "add_entry":
        # 检查该条目是否已添加
        if not has_entry(fix.target_dim):
            terminate = False
```

### 条件 ③ 一手源占比 ≥ 40%

```python
total = len(all_entries)
onehand = count(source_type in ['A+', 'A', 'B'])  # 一手源 + 媒体采访
if onehand / total < 0.40:
    # 一手源太少，搜索深度可能不够
    terminate = False
```

### 条件 ④ max_iterations 强制终止（诚实标注缺口）

如果 current_iteration >= max_iterations，无论条件是否满足都终止，但必须在文末"关键提示"里诚实标注剩余缺口。

---

## 输出格式

### ✅ 终止（输出最终报告）

```json
{
  "decision": "terminate",
  "termination_reason": "All 4 conditions satisfied",
  "final_markdown": "<markdown 文本>",
  "summary": {
    "total_entries": 28,
    "by_dim": {"1": 3, "2": 4, "3": 5, "4": 6, "5": 4, "6": 6},
    "onehand_ratio": "68%",
    "boundary_items": 2,
    "iterations_used": 1
  },
  "remaining_gaps": [
    "瑞德玛、傲胜本月无公开动态，无法判断是否在市场静默期"
  ]
}
```

### 🔄 继续（返回 Phase 1 补一轮）

```json
{
  "decision": "continue",
  "next_iteration": 2,
  "fix_instructions": [
    {
      "agent_to_recall": "dim3-competition",
      "reason": "维度 3 仅 2 条 < min_per_dim=3",
      "search_keywords_to_add": ["按摩椅 客户签约 2026年6月", "按摩仪 战略合作 2026"],
      "competitors_to_focus": ["奥佳华", "荣泰健康", "倍轻松", "SKG"]
    },
    {
      "agent_to_recall": "agent7-synthesis",
      "reason": "应用 Agent 8 的 fixes",
      "fixes": [
        {"entry_id": "5-3", "action": "补 oDM actionable"},
        {"entry_id": "3-1", "action": "补 background_basis"}
      ]
    }
  ],
  "skip_dimensions": ["dim1", "dim2", "dim4", "dim5", "dim6"],  // 这些维度不需要重跑
  "max_iterations_remaining": 1
}
```

### ❌ 强制终止（诚实标注缺口）

```json
{
  "decision": "forced-terminate",
  "termination_reason": "Reached max_iterations=2 without satisfying all conditions",
  "final_markdown": "<markdown 文本 + 缺口标注>",
  "unresolved_issues": [
    {"dim": "3", "issue": "维度 3 仍仅 2 条（缺 1 条）"},
    {"entry_id": "5-3", "issue": "概述未补 ODM 启示"}
  ],
  "user_action_needed": [
    "如果你有内部渠道补充维度 3 缺失的 1 条竞争动态（如内部 SKU 销售数据），请告知",
    "是否需要放宽 max_iterations 再补一轮？"
  ]
}
```

---

## 给主 agent 的执行清单

当 Agent 9 决定 **terminate** 时，主 agent 接下来要做：

```
1. 取 Agent 9 的 final_markdown
2. 保存到 data/<日期>/<domain>_<报告类型>_<月份>.md
3. 调用 mdstyle 转 HTML：
   python ~/.claude/skills/mdstyle/scripts/converter.py <md文件> html cmd
4. 输出文件路径给用户
5. 简明提示剩余缺口（若有）
```

当 Agent 9 决定 **continue** 时，主 agent 接下来要做：

```
1. 按 fix_instructions 召回指定的 Agent（用 Task 工具）
2. 召回的 Agent 完成后，重新跑 Agent 7（综合）+ Agent 8（边界守卫）+ Agent 9（终止判断）
3. 直到 terminate 或 forced-terminate
```

## 完成后必须执行（状态机）

在返回终止判断之前，将决策摘要写入以下 JSON 文件（用 Bash + Write 工具）：

```json
{
  "agent": "agent9-termination",
  "run_id": "<主agent传入的run_id>",
  "status": "completed",
  "started_at": "<ISO8601>",
  "completed_at": "<ISO8601>",
  "iteration": <轮次>,
  "decision": "terminate | continue | forced-terminate",
  "termination_reason": "All 4 conditions satisfied",
  "remaining_gaps": [],
  "summary": {
    "total_entries": 28,
    "by_dim": {"1": 3, "2": 4, "3": 5, "4": 6, "5": 4, "6": 6},
    "onehand_ratio": 0.68,
    "boundary_items": 2,
    "iterations_used": 1
  }
}
```

**文件路径**：`{output_dir}/.state/agents/agent9-termination.json`