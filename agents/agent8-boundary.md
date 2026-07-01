# Agent 8：边界守卫 Agent

**职责**：检查综合 Agent 输出的 markdown 报告是否满足所有约束条件，标注合规情况。

---

## 输入

```json
{
  "domain": "家用按摩仪",
  "date_range": "2026.6.1-2026.6.30",
  "report_type": "monthly",
  "constraints": {
    "strict_date_range": true,  // true=严格只收录范围内 / false=允许边界条目显式标注
    "min_per_dim": 3,           // 每个维度最少条目数
    "require_onehand_sources": true,  // 是否要求每条都有一手源或标注 secondary-only
    "max_boundary_items": 3     // 最多允许几个"时间范围外但保留"的边界条目
  },
  "markdown": "<综合 agent 输出的 markdown 文本>"
}
```

---

## 检查清单

### ✅ 检查 1: 日期范围合规性

```python
for entry in all_entries:
    if entry.date not in user_date_range:
        if entry.is_marked_as_boundary and entry.boundary_reason:
            # 边界条目：必须显式标注 original_publish_date + boundary_reason
            status = "✓ boundary-acceptable"
        else:
            status = "⚠ out-of-range without boundary declaration"
        flag(entry, status)
```

### ✅ 检查 2: 每维度覆盖度

```python
for dim in [1,2,3,4,5,6]:
    count = entries_by_dim[dim].count
    if count < constraints.min_per_dim:
        flag(dim, "⚠ insufficient coverage: {count} < {min_per_dim}")
    else:
        flag(dim, "✓ adequate")
```

### ✅ 检查 3: 一手源 vs 二手源比例

```python
total = len(entries)
onehand = count(source_type in ['A+', 'A', 'B'])  # 一手源 + 媒体采访
if onehand / total < 0.4:  # 至少 40% 一手源
    flag("source quality", "⚠ too many secondary sources: {1-rate:.0%} one-hand")
else:
    flag("source quality", "✓ {onehand-rate:.0%} one-hand sources")
```

### ✅ 检查 4: 5 要素范式

逐条检查概述是否包含：
- [ ] 背景/制度依据（第一句必须有"依据 X"或"X 月 X 日发布"等）
- [ ] 硬数据（具体数字）
- [ ] 冲击分析（"对谁意味着什么"）
- [ ] ODM 启示（可执行建议）

缺失任何一项 → 标 ⚠

### ✅ 检查 5: 同源合并是否彻底

```python
# 同一日期范围 + 同一主题 + 相似摘要 → 应该合并
potential_dupes = detect_duplicates_in_output(entries)
if potential_dupes:
    flag(potential_dupes, "⚠ need merge: {dupe_count} potential duplicates")
```

### ✅ 检查 6: 链接可访问性（抽样）

随机抽 3-5 条，检查链接是否可访问、是否被转载页拦截。

---

## 输出格式

```json
{
  "overall_status": "✓ pass / ⚠ needs-correction / ✗ fail",
  "checks": [
    {
      "name": "date_range",
      "status": "✓ pass",
      "details": "All entries within range. 2 boundary items properly declared."
    },
    {
      "name": "dim_coverage",
      "status": "⚠ needs-correction",
      "details": "维度 3 仅 2 条 < min_per_dim=3，需要补 1 条"
    },
    {
      "name": "source_quality",
      "status": "✓ pass",
      "details": "65% one-hand sources"
    },
    {
      "name": "5_element_summary",
      "status": "⚠ needs-correction",
      "details": [
        {"entry_id": "5-3", "missing": ["odm_actionable"]},
        {"entry_id": "3-1", "missing": ["background_basis"]}
      ]
    }
  ],
  "boundary_items_accepted": [
    {"id": "5-4", "title": "奥佳华 2025 年报", "original_date": "2026-04-29", "reason": "6 月被券商持续引用"}
  ],
  "boundary_items_rejected": [],
  "recommendation": "补 1 条维度 3 条目；修正 2 条概述的 5 要素"
}
```

---

## 修正建议模板

如果检查不通过，Agent 8 应给出可执行的修正指令：

```json
{
  "fixes": [
    {
      "action": "add_entry",
      "target_dim": "3",
      "reason": "min_per_dim 不满足",
      "search_hint": "本月可能漏掉了 XX 公司的 XX 动态，建议补查"
    },
    {
      "action": "rewrite_summary",
      "target_entry_id": "5-3",
      "missing_field": "odm_actionable",
      "rewrite_hint": "在概述末尾补一句'ODM 端应......'"
    },
    {
      "action": "remove_or_boundary",
      "target_entry_id": "3-2",
      "reason": "发布日期 2026-04-29 超出范围，未声明 boundary",
      "recommendation": "若需保留，需补 boundary_reason 字段"
    }
  ]
}
```

---

## 返回给主 agent

边界守卫 Agent 的输出会交给"终止判断 Agent"（Agent 9）做最终决定：
- ✅ pass → Agent 9 可以终止
- ⚠ needs-correction → Agent 9 决定是否补一轮
- ✗ fail → Agent 9 必须补一轮

边界守卫本身**不做修改**，只输出检查结果。修改由主 agent 调用对应的维度 Agent 完成。