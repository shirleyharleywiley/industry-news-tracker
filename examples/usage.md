# 示例：调用 industry-news-tracker skill

## 调用方式

```bash
# 方式 1：通过 Skill 调用
/industry-news-tracker 家用按摩仪 2026.6.1-2026.6.30 --reader ODM老板
/industry-news-tracker 储能 2026.6 --output 月报
/industry-news-tracker 智能门锁 2026.Q2 --output 季报
/industry-news-tracker 美妆个护 2026.6 --output 月报

# 方式 2：通过自然语言触发
"帮我做一份家用按摩仪 2026 年 6 月的月报，读者是 ODM 老板"
"做一份储能 2026 年 6 月的周报"
"分析下智能门锁 2026 年 Q2 的市场情况"
```

> skill 适用于**任何 ToC/ToB 细分领域**（不限电子行业）。各领域的关键词和"头部企业清单"在子 agent 的 md 中维护。

## 主 agent 执行的完整流程

### Step 1: 参数解析

```python
domain = "家用按摩仪"  # 或 "储能" / "智能门锁" 等
date_range = "2026.6.1-2026.6.30"
reader = "ODM 老板"
report_type = "monthly"  # daily / weekly / monthly / quarterly
constraints = {
  "strict_date_range": True,
  "min_per_dim": 3,
  "max_iterations": 2,
  "current_iteration": 1
}
```

### Step 2: 头部企业识别（领域知识，调用前完成）

每个细分领域都有自己的头部企业清单，**主 agent 应在调用本 skill 前先识别**该领域的 Top 5-10 头部企业作为检索锚点。

示例：
- **家用按摩仪**：奥佳华、荣泰健康、倍轻松、SKG、松下、傲胜
- **储能**：宁德时代、比亚迪、亿纬锂能、阳光电源、派能科技
- **智能门锁**：德施曼、凯迪仕、鹿客、华为、小米
- **电子烟**：思摩尔、雾芯科技、悦刻、绿豆、铂德

### Step 3: Phase 1 — 并行启动 6 个维度 agent

```python
from Task import Task

tasks = []
for dim in [1, 2, 3, 4, 5, 6]:
    prompt = read_file(f"agents/dim{dim}-<name>.md")
    prompt = prompt.replace("{domain}", domain)
    prompt = prompt.replace("{date_range}", date_range)
    if reader:
        prompt = prompt.replace("{reader}", reader)
    task = Task(
        agent="general-purpose",
        prompt=prompt,
        description=f"维度 {dim} 检索"
    )
    tasks.append(task)

results = parallel(tasks)  # 6 个并行执行
```

每个 agent 返回 ~5-15 条候选，共 ~30-90 条。

### Step 4: Phase 2 — 综合 Agent

```python
synthesis_result = Task(
    agent="general-purpose",
    prompt=read_file("agents/agent7-synthesis.md"),
    input={
        "domain": domain,
        "date_range": date_range,
        "reader": reader,
        "report_type": report_type,
        "dim1_market": results[0],
        "dim2_trend": results[1],
        "dim3_competition": results[2],
        "dim4_channel": results[3],
        "dim5_operation": results[4],
        "dim6_policy": results[5]
    }
)
```

### Step 5: Phase 3 — 边界守卫 + 终止判断

```python
boundary_result = Task(
    agent="general-purpose",
    prompt=read_file("agents/agent8-boundary.md"),
    input={
        "constraints": constraints,
        "markdown": synthesis_result.markdown
    }
)

termination_result = Task(
    agent="general-purpose",
    prompt=read_file("agents/agent9-termination.md"),
    input={
        "domain": domain,
        "constraints": constraints,
        "agent7_output": synthesis_result,
        "agent8_output": boundary_result
    }
)
```

### Step 6: 根据终止判断决定行动

```python
if termination_result.decision == "terminate":
    final_md = termination_result.final_markdown
    save_to = f"data/{today}/{domain}_{report_type}_{period}.md"
    write_file(save_to, final_md)

    # 转 HTML
    html_path = run_cmd(
        f"python ~/.claude/skills/mdstyle/scripts/converter.py "
        f"{save_to} html cmd"
    )

    print(f"✅ 报告生成完成：")
    print(f"   Markdown: {save_to}")
    print(f"   HTML: {html_path}")

elif termination_result.decision == "continue":
    # 按 fix_instructions 召回指定 agent
    for fix in termination_result.fix_instructions:
        recall_agent(fix)
    # 重新跑 Phase 2 + Phase 3

elif termination_result.decision == "forced-terminate":
    # 输出报告 + 缺口标注 + 询问用户
    final_md = termination_result.final_markdown
    write_file(...)
    print(f"⚠ 报告完成但有缺口：")
    print(f"   {termination_result.unresolved_issues}")
    print(f"   {termination_result.user_action_needed}")
```

---

## 真实示例：家用按摩仪 2026 年 6 月月报

### 输入
```
/industry-news-tracker 家用按摩仪 2026.6.1-2026.6.30 --reader ODM老板
```

### 6 个维度 agent 检索关键词

| Agent | 关键词示例 |
|-------|----------|
| 维度 1 市场 | 按摩仪 市场规模 2026年6月 / 奥维云网 按摩仪 / 观研天下 按摩小器具 |
| 维度 2 趋势 | 按摩仪 4D 机芯 2026 / 按摩仪 AI 智能化 / 按摩仪 节日送礼 |
| 维度 3 竞争 | 奥佳华 2026 / 荣泰健康 一季报 / 倍轻松 专利 / SKG 母公司 上市 |
| 维度 4 渠道 | 按摩仪 618 销售 / 按摩仪 TikTok / 按摩仪 直播带货 |
| 维度 5 运营 | 铜价 2026年6月 / ABS 价格 / 奥佳华 财报 / MCU 报价 |
| 维度 6 政策 | FDA 510(k) 按摩仪 / 按摩仪 CE 认证 / 按摩仪 UDI |

### 预期输出文件

```
data/<日期>/
├── 家用按摩仪_月报_2026-06.md         (源 markdown)
└── 家用按摩仪_月报_2026-06-cmd.html   (mdstyle 转 HTML)
```

### 报告结构（30-50 条新闻）

按 6 维度分布（假设）：
- 维度 1（市场与规模）：3-5 条
- 维度 2（前景与趋势）：5-8 条
- 维度 3（竞争态势）：8-12 条
- 维度 4（客户与渠道）：6-10 条
- 维度 5（运营与利润）：4-6 条
- 维度 6（政策与合规）：5-8 条

合计 ~31-49 条。

---

## 跨领域适配示例

### 示例 1：储能行业 2026 Q2 季报

```
/industry-news-tracker 储能 2026.Q2 --output 季报
```

Phase 2 步骤会自动识别头部企业（宁德时代、比亚迪等），关键词策略在 dim1-dim6 中按储能领域适配。

### 示例 2：智能门锁 2026 年 6 月月报

```
/industry-news-tracker 智能门锁 2026.6.1-2026.6.30
```

关键词：智能锁 出货量、智能门锁 摄像头、指纹识别 集成、AI 门锁、TikTok 智能锁 等。

### 示例 3：美妆个护 2026.6 月报

```
/industry-news-tracker 美妆个护 2026.6
```

关键词：化妆品 GMV、618 美妆、抖音 美妆 KOL、男士护肤、纯净美妆 等。

---

## 常见错误及避免

### 错误 1：维度 5 把"4 月底财报"误纳为 6 月新闻

**症状**：奥佳华 4 月 29 日发布的 2025 年报 + 2026 一季报，被纳入 6 月月报
**避免**：
- 维度 5 agent 严格检查 publish_date
- 若必须保留（如持续影响市场），显式标注 original_publish_date="2026-04-29" + boundary_reason="6 月被券商持续引用"
- Agent 8 边界守卫会标 ⚠ 让 Agent 9 决定是否保留

### 错误 2：维度 4 把带货稿当多条独立新闻

**症状**：6.22/6.25/6.26 三个 PZPE 头部按摩仪测评稿被当成 3 条
**避免**：
- 维度 4 agent 标 `potential_duplicate: true`
- Agent 7 合并为 1 条，标题写"（多源多日期报道）"

### 错误 3：概述与标题几乎一样

**症状**：标题"按摩仪市场规模 153 亿元"，概述"按摩仪市场规模 153 亿元"
**避免**：
- Agent 7 必须用 5 要素范式撰写概述
- Agent 8 检查 5 要素完整性

### 错误 4：维度 1-6 内容不平衡

**症状**：维度 3 有 15 条，维度 6 只有 1 条
**避免**：
- Agent 9 检查 `min_per_dim=3`
- 不满足时召回对应维度 agent 补一轮

### 错误 5：把软件/SaaS 产品套用本 skill

**症状**：用本 skill 分析某 SaaS 产品的市场动态
**避免**：本 skill 主要面向硬件/制造/消费品领域。SaaS/纯软件请用 research-multi-agent 的"竞品分析"或"市场研究"类型。

---

## 配合使用的 skill

- **mdstyle**：转 HTML（默认 `cmd` 版式）
- **brainstorming**：报告生成前，对"重点关注维度"做头脑风暴（可选）
- **pdf-to-markdown**：若用户提供历史报告 PDF，可先转 markdown 作为参考（可选）

---

## 维护说明

- **每个 agent 提示词独立可修改**：根据细分领域调整关键词部分即可
- **同源合并规则**：当发现新的同源模式时，更新 agent 7 的合并策略
- **边界守卫规则**：根据用户对"严格 vs 宽松"的偏好调整
- **终止判断阈值**：根据报告类型（日报/周报/月报/季报）调整 min_per_dim 和 max_iterations
- **新增细分领域**：复制 `agents/dimX-*.md` 文件，替换关键词示例和领域头部企业清单即可