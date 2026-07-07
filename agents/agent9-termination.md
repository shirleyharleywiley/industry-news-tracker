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
  "constraints": {
    "strict_date_range": true,
    "min_per_dim": 3,
    "max_iterations": 2,  // 最大迭代轮数（防止无限循环）
    "current_iteration": 1
  },
  "agent8_output": {
    "overall_status": "✓ pass / ⚠ needs-correction / ✗ fail",
    "checks": [...],
    "boundary_items_rejected": [],
    "recommendation": "..."
  }
}
```

---

## 终止判断流程

Step 1: 检查 agent8_output.overall_status
  → ✓ pass → 终止，输出最终报告
  → ⚠ needs-correction → 进入 Step 3
  → ✗ fail → 进入 Step 2

Step 2: 检查相应agent的current_iteration是否已达 max_iterations
  → current_iteration < max_iterations → 相应agent的current_iteration++；返回 Phase 1，相应agent重新搜索
  → current_iteration >= max_iterations → 强制认定该agent ✓ pass（诚实标注缺口）

Step 3: 返回 Phase 2，重写markdown报告

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

### 🔄 继续（返回 Phase 1 补一轮、修订）

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
## 返回结果给主 agent。


---
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