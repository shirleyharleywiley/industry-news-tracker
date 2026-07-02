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

### ✅ 检查 7: 链接完整性（每条必须有可点击链接）🔴

```python
# 这是硬性检查，不通过直接标 ✗ fail
for entry in all_entries:
    if not entry.source_url or entry.source_url == "N/A":
        flag(entry, "⛔ missing-link: 无任何链接")
    elif entry.source_url in ["跨境行业渠道综合", "海关公告综合", "亚马逊卖家后台", 
                               "跨境物流渠道", "行业渠道综合", "综合报道",
                               "跨境电商综合", "海关/物流渠道", "电商平台"]:
        flag(entry, "⛔ fuzzy-source: 出处为模糊描述而非可点击URL，必须替换为真实链接")
    elif not entry.source_url.startswith("http"):
        flag(entry, "⛔ invalid-url: 链接不是有效 URL（需以 http/https 开头）")

# 检查规则
- 禁止出现"跨境行业渠道综合""海关公告综合""亚马逊卖家后台"等无链接的模糊出处
- 禁止出现"N/A""内部工具日志"等非链接文本作为出处
- 出处必须是可点击的超链接（`<a href="...">来源名</a>` 或 markdown `[来源名](url)`）
- 如果确实无法找到一手链接，也必须至少附一个二次转载/聚合页链接
- 如果某条新闻在所有搜索引擎都无法找到任何可访问链接 → 该条目不得收入报告
- 来源描述文字（链接文本）不能是"来源名待查""待补充"等占位符
```

**判定标准**：
- 所有条目通过 → `✓ pass`
- 任一条目无链接或模糊出处 → 直接 `✗ fail`，必须修正后才能进入 Agent 9

---

## 输出格式

```json
{
  "overall_status": "✓ pass / ⚠ needs-correction / ✗ fail",
  "overall_status_rule": "检查7(链接完整性)任一不通过 → 直接 ✗ fail；其他检查不通过 → ⚠ needs-correction",
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
    },
    {
      "name": "link_integrity",
      "status": "✗ fail",
      "details": [
        {"entry_id": "6-2", "issue": "missing-link: 出处为'跨境行业渠道综合', 无URL", "fix": "替换为真实链接"},
        {"entry_id": "6-3", "issue": "fuzzy-source: 出处为'亚马逊卖家后台', 无URL", "fix": "替换为真实链接"}
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