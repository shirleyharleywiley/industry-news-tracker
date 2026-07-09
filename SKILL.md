---
name: industry-news-tracker
description: "行业新闻、情报生成工具。当用户请求生成某一细分领域（家用按摩仪、电子烟、储能、机器人、智能门锁、清洁电器等 ToC/ToB 行业）的新闻日报/周报/月报/季报时触发。"
user-invocable: true
disable-model-invocation: false
context: fork
agent: general-purpose
allowed-tools: WebSearch, WebFetch, mcp__MiniMax__web_search, Task, TodoWrite, Read, Write, Edit, Bash, AskUserQuestion
argument-hint: "<细分领域> <日期范围> "
---

# industry-news-tracker

## Instructions

内部使用 6 维度并行检索 + 综合 Agent + 边界守卫 + 终止判断的 4 阶段流水线，输出供 ODM 厂商老板等读者人群阅读的结构化情报简报，并自动调用 mdstyle 转 HTML。

调用方式

```bash
/industry-news-tracker 家用按摩仪 2026.6.1-2026.6.30 
/industry-news-tracker 储能 2026.6
/industry-news-tracker 某细分领域 2026.Q2
```

或自然语言：

> "帮我收集家用按摩仪2026 年 6 月的新闻"

**适用品类示例**（不限于此）：
- 电子制造：家用按摩仪、电子烟、储能、机器人、智能门锁、清洁电器
- 消费品：美妆个护、母婴、家居、日化
- 医疗健康：家用医疗器械、保健品、医药器械
- 工业品：仪器仪表、化工原料、零部件
- 纯软件 / SaaS：电子签、协同办公、CRM、HR SaaS、垂直 SaaS（如医疗 SaaS、法律 SaaS）

**不适用场景**：宏观金融政策分析、纯学术论文综述。

### step1：运行前自检（主 agent 启动后第一件事）

在收到用户输入后、启动 Phase 1 之前，主 agent 必须逐条确认：

1. 明确工作流程：3 阶段 × 4 类 Agent

```
用户输入（细分领域 + 日期范围）
  ↓
【Phase 1: 6 维度并行检索】
  ├─ Agent 1 → 维度 1: 市场与规模       
  ├─ Agent 2 → 维度 2: 前景与趋势       
  ├─ Agent 3 → 维度 3: 竞争态势        
  ├─ Agent 4 → 维度 4: 客户与渠道       
  ├─ Agent 5 → 维度 5: 运营与利润       
  └─ Agent 6 → 维度 6: 政策与合规      
  ↓
【Phase 2: 综合】
  └─ Agent 7（综合 Agent）               
       → 6 维度合并 + 同源去重 + 按日期排序 + 5 要素概述 + 输出 markdown
  ↓
【Phase 3: 边界守卫 + 终止判断】
  ├─ Agent 8（边界守卫）                
  │     → 检查日期范围 / 链接质量 / 维度完整性 / 5 要素完整性
  └─ Agent 9（终止判断）                 
        → 判断是否需要补一轮 / 输出最终报告
  ↓
【Phase 4: mdstyle 转 HTML】
  ↓
 结束
```

2. 理解最终输出格式 = Agent 7 的表格体，不是自行撰写的章节体
3. 理解我的角色 = 编排者，不是写手——不自行撰写报告正文
4. 理解关闸 = Agent 9 判定 terminate 之前，绝不用 Write 输出报告文件
5. 工作目录确认 = PWD 不是 iCloud 归档路径，是真实桌面路径
```

**工作目录检测脚本**（必须在 SKILL.md 第一项就执行）：

```bash
EXPECTED="/Users/mac/Desktop/aitask/daily-news-ee"
CURRENT="$(pwd)"

if [[ "$CURRENT" != "$EXPECTED" && ! "$CURRENT" =~ /iCloud云盘（归档）/ ]]; then
  echo "OK: $CURRENT"
elif [[ "$CURRENT" =~ /iCloud云盘（归档）/ ]]; then
  echo "WRONG_DIR: $CURRENT 包含 iCloud 归档路径"
  echo "FIX: cd /Users/mac/Desktop/aitask/daily-news-ee 后再继续"
  exit 1
fi
```

> **iCloud 归档路径的根因**：macOS 优化存储功能自动生成的副本目录，与真实桌面是不同的物理目录。VSCode 打开 iCloud 归档路径下的项目时，shell PWD 也是 iCloud 路径。**所有 Write 操作必须在 cd 到真实桌面后再执行。**

### step2：创建agents

**Agent 文件映射**：每个 agent 的完整提示词（输入参数、关键词策略、输出 json 格式、关键约束、避坑清单）都在 `agents/<file>.md` 中，具体对应关系为：

  Agent 1 → agents/dim1-market.md
  Agent 2 → agents/dim2-trend.md
  Agent 3 → agents/dim3-competition.md
  Agent 4 → agents/dim4-channel.md
  Agent 5 → agents/dim5-operation.md
  Agent 6 → agents/dim6-policy.md
  Agent 7 → agents/agent7-synthesis.md
  Agent 8 → agents/agent8-boundary.md
  Agent 9 → agents/agent9-termination.md

**主 agent 在调用时直接 Read 该文件**，把内容作为 Task 工具的 prompt。主 agent 通过 `Task` 工具并行启动 9 个子 agent。

### step3：从Phase 1-Phase4 按照流程开始工作

Phase 1: 6 维度并行检索（关键约束）

**输入**：细分领域、日期范围 统一传给 Agent 1到Agent 6 每个 agent 。

启动前 —— 创建 run.json 🔴

主 agent 启动 6 个 dim agent **之前**，必须先 Write 初始 `run.json`：

```json
{
  "run_id": "<domain_slug>-<YYYYMMDD>-<HHMMSS>",
  "domain": "<细分领域>",
  "date_range": "<日期范围>",
  "reader": "<读者>",
  "report_type": "weekly",
  "created_at": "<ISO8601>",
  "updated_at": "<ISO8601>",
  "status": "phase1",
  "current_phase": 1,
  "iteration": 1,
  "phases": {
    "phase1": {"status": "in_progress", "agents": {
      "dim1-market": {"status": "dispatched"}, "dim2-trend": {"status": "dispatched"},
      "dim3-competition": {"status": "dispatched"}, "dim4-channel": {"status": "dispatched"},
      "dim5-operation": {"status": "dispatched"}, "dim6-policy": {"status": "dispatched"}
    }}
  },
  "output_dir": "/Users/mac/Desktop/aitask/daily-news-ee/output"
}
```

> **绝对路径**：`{output_dir}/.state/run.json`。不是 iCloud 归档路径！

**输出**：Agent 1到Agent 6 每个 agent 返回 json 数组（5-15 条候选）+ 元信息（候选数 / 搜索关键词 / 边界情况）。

每次收到 task-notification 时 —— 更新 run.json 

```
1. task-notification 到达 → Read task output 获取子 agent 结果
2. Read 检查 agents/dimX.json 是否存在（子 agent 应该已写入）
3. 用 Edit/Write 更新 run.json 中对应 agent 的 status → "completed"
4. 更新 updated_at 时间戳
5. 检查 6 个 agent 是否全部 status=="completed" → 是 → 切 status→"phase2"
```

防重复启动检查：

在启动**第二轮** dim agent 补搜之前，必须先检查：

```
if agents/dimX.json 存在且 iteration ≥ 当前轮次:
    跳过，不重复启动该 dim agent
elif agents/dimX.json 存在但状态 != completed:
    该 agent 可能卡死 → 重新启动，iteration+1
else:
    正常启动
```

> **关键约束（详细版见各子 agent md）**：
> - 严格日期过滤（看发布日，不是事件日）
> - 关键词必须带日期修饰词
> - 优先一手源（公告 > 数据机构 > 媒体 > 转载）
> - **窗口内候选数 5-15 条，不必找全**——若窗口内不足 3 条，按下方"检索命中不足时的强制确认"询问用户
> - **日期不够时必须先询问用户**（见下方"检索命中不足时的强制确认"），不要自行扩大搜索窗口

---

Phase 2: 综合 Agent（去重 + 排序 + 5 要素概述 + 表格体输出）

**输入**：细分领域、日期范围、Agent 1到Agent 6 每个 agent 返回 json 数组

```json
{
  "domain": "家用按摩仪",
  "date_range": "2026.6.1-2026.6.30",
  "dim1_market": [...],
  "dim2_trend": [...],
  "dim3_competition": [...],
  "dim4_channel": [...],
  "dim5_operation": [...],
  "dim6_policy": [...]
}
```
**输出**：markdown文件

Agent 7 完成后 —— 更新 run.json 

主 agent 收到 Agent 7 task-notification 后，立即：

```
1. Read agents/agent7-synthesis.json 确认完成
2. Edit run.json：
   - status → "phase3"
   - current_phase → 3
   - phases.phase2.status → "completed"
   - phases.phase2.agent7.status → "completed"
   - updated_at → 当前时间
3. 判断输出格式是否合法：

```markdown
## 【问题 1：市场与规模】
| 日期 | 标题 | 概述 | 链接 |
|---|---|---|---|
| 7.4 | xxx | 【背景】...【数据】...【冲击】...【ODM启示】... | [来源](具体URL) |
```
如果不合法重新执行Phase 2。

---

Phase 3: 边界守卫 + 终止判断 

执行流程:
step1： Agent 8（边界守卫）

**输入**: Agent 7 输出的 markdown、细分领域、日期范围
**输出**：边界守卫的检查结果

step2： Agent 9（终止判断）

**输入**: Agent 8的边界守卫检查结果

当 Agent 9 决定 **terminate** 时，主 agent 接下来要做：

```
1. 取 Agent 9 的 final_markdown
2. 保存到 data/<日期>/<domain>_<报告类型>_<月份>.md
3. 简明提示剩余缺口（若有）
```

当 Agent 9 决定 **continue** 时，主 agent 接下来要做：

```
1. 按 fix_instructions 召回指定的 Agent（用 Task 工具）
2. 召回的 Agent 完成后，重新跑 Phase 2 和 Phase 3 
3. 直到 terminate 或 forced-terminate
```

Phase 4: mdstyle 转 HTML,（使用cmd版式）

1. 调用 mdstyle 转 HTML：
   python ~/.claude/skills/mdstyle/scripts/converter.py <md文件> html cmd
2. 输出md和html两个文件路径给用户


## Performance Notes（主 agent 必读，违反任一即不可输出）

### 铁律 1：流程执行
1. **Phase 1 数据到齐后，主 agent 的唯一合法动作是调用 Agent 7**——不得使用 Write/Edit 自行撰写报告。
2. **Agent 7 输出后必须立刻跑 Agent 8**——Agent 8 检查全部 7 项（日期/覆盖度/源质量/5要素/合并/链接真实性/链接可访问性）+ 1 项链接文本合规性(8 项总和,详见 `agents/agent8-boundary.md` 检查 8),任一 fail 不可输出。
3. **Agent 8 pass 后必须跑 Agent 9**——3 个终止条件全部满足才可输出最终报告。
4. **主 agent 不可跳过 Agent 7/8/9 中的任何一个**——Agent 7 提供格式保障（表格体），Agent 8 提供质量保障，Agent 9 提供完整性保障。三者缺一不可。


### 铁律 2：不得绕过 Agent 7/8/9 直接写文件 

### 铁律 3：检索命中不足时的强制确认（重要）

**触发条件**：Phase 1 检索完成后，任一维度在用户给定日期范围内的搜索命中 < min_per_dim（默认 3 条）。

**主 agent 必须执行**：

1. 立即停止后续 Phase，调用 `AskUserQuestion` 询问用户：
   - 选项 A：**保持原范围**，接受该维度数据不足 / 强制终止
   - 选项 B：**向前扩展 N 天**（建议值 7-14 天）
   - 选项 C：**向前扩展到上一档粒度**（如周报→月报、日报→周报）
   - 选项 D：用户自定义范围
2. **不得自行扩大搜索时间范围**——这是用户的决策权。
3. 用户确认后，按用户授权的新范围重新执行该维度检索。
4. 用户拒绝扩大，诚实标注"XX 维度命中不足，forced-terminate"，但不要放弃整个报告。

**为什么必须有这一步**：用户对时间范围的控制优先级高于"完成报告"。缺失比错误更可接受——错误纳入超范围条目会让用户怀疑整份报告的严谨性。

---


## 工具降级策略（重要）

Phase 1 检索时若主搜索引擎工具故障，按以下顺序降级，避免单点失败导致整份报告中断：

| 优先级 | 工具 | 适用场景 |
|--------|------|---------|
| 1（主） | `WebSearch` | 默认，国内外新闻通用 |
| 2（备用） | `mcp__MiniMax__web_search` | WebSearch 报 400/500/超时等后端错误时切换 |
| 3（备用） | `WebFetch` | 直接抓取已知 URL（用于备份特定来源、补救特定维度） |
| 4（最终） | 告知用户放弃 | 前三级全部失败 |

**触发降级的判定条件**：
- `WebSearch` 连续 3 次调用返回 API 错误（4xx/5xx）
- 或所有调用均返回空结果（关键词合理但 0 命中）——此时仍可降级到备用源验证是否真的没数据

**降级时主 agent 必须**：

1. **明确告知用户**："主工具（WebSearch）异常，已切换到备用源 [MiniMax web_search / WebFetch]"
2. 在报告元信息中标注：`primary_search_tool` + `fallback_used: true/false`
3. 备用源返回的结果也必须遵守日期标注规范（不能因为是降级就放宽日期要求）
4. 切勿在工具故障时硬凑数据——保持诚实

**最终放弃条件**（指不生成本次报告）：
- WebSearch + mcp__MiniMax__web_search 全部故障
- 或 WebFetch 仅返回已知 URL 而用户未提供备份来源
- 此时告知用户："工具后端异常，已取消本次报告生成。请稍后重试。"

**Why**: 2026-07-01 重跑超声清洗机周报时，WebSearch 连续 8 次调用（含 'test'）全部返回 400 invalid params，单点故障导致无法验证任何日期的实际新闻命中。降级策略可避免这类情况完全卡死 skill。

---

## 配合使用的其他 Skill

- **mdstyle**：将 markdown 报告转 HTML（默认 `cmd` 版式）
- **brainstorming**：报告生成前，对细分领域的"重点关注维度"做头脑风暴
- **pdf-to-markdown**：若用户提供历史报告 PDF，可先转 markdown 作为参考

---

## 与 `research-multi-agent` 的关系

| 维度 | research-multi-agent | industry-news-tracker |
|------|---------------------|-----------------------|
| 适用范围 | 通用深度调研（8 种类型） | 行业细分领域新闻日报/月报 |
| 角色数 | 7 角色（4 层架构） | 精简为 4 类 |
| 迭代 | 多轮深度迭代 | 通常 1 轮 |
| 信息源 | 爬虫 + 搜索 + Task 并行 | 主要是搜索引擎（WebSearch / mcp__MiniMax__web_search） |
| 终止判断 | 4 个复杂条件 | 3 个简化条件 |
| 输出 | 多读者多版本报告 | 单一报告 + mdstyle 转 HTML |

> **精简原因**：新闻日报信息密度低、可信度天花板低（多为媒体报道），多轮迭代的边际收益不大；6 维度并行检索已覆盖主要工作量。

---

## 🔵 运行状态机（v2 增强）

> **设计 spec**：[docs/specs/2026-07-06-state-machine-design.md](docs/specs/2026-07-06-state-machine-design.md)
> **目的**：进度可见 + 断点恢复。解决 2026.7.6 事故中"30 分钟执行过程不可观测"的问题。

### 目录结构

```
output/
├── .state/
│   ├── run.json                    # 主控状态文件
│   └── agents/                     # 子 agent 结果摘要
│       ├── dim1-market.json        # ... dim6-policy.json
│       ├── agent7-synthesis.json   # ... agent9-termination.json
└── <报告文件>.md / .html
```

> `.state/` 加入 `.gitignore`，不提交。

### 启动检查（主 agent 第一步）

主 agent 在启动后、Phase 1 之前必须先检查 `output/.state/run.json` 是否存在：

| 情况 | 动作 |
|------|------|
| 文件不存在 | 创建新文件，`status: init` → `phase1` |
| 文件存在，`status: completed` | 直接返回报告路径给用户，不重跑 |
| 文件存在，其他状态 | 读取 `status` 和 `phases`，从断点继续 |

### 状态枚举

`init` / `phase1` / `phase2` / `phase3` / `fix` / `phase4` / `completed` / `failed`

> `fix` 是 phase3 内部子状态（手动修正时），不独立占 phase 编号。

### 写者职责

| 文件 | 写者 | 时机 |
|------|------|------|
| `run.json` | 主 agent | 每次 task-notification 到达、phase 切换、修复完成时 |
| `agents/dimX.json` | 6 个 dim 子 agent | 完成时（在 prompt 里已要求） |
| `agents/agent7-9.json` | Agent 7/8/9 | 完成时 |
| 报告 MD/HTML | 主 agent | Phase 4 写文件 |

### 断点恢复流程

```
1. 主 agent 启动 → 检查 .state/run.json
2. 存在 → 读取
3. 根据 status：
   phase1 → 读 agents/dim1-dim6.json 看哪些已完成，只补跑缺失的
   phase2 → 看 agent7-synthesis.json，不存在则重跑 Agent 7
   phase3 → 看 agent8/9.json，从当前检查点继续
   phase4 / completed → 输出文件已存在或任务完成
4. 补跑时 prompt 附带："你的上一轮结果在 agents/dim1-market.json，"
   "本轮请在此基础上只补搜缺失日期 / 补充指定关键词"
```

### 进度查看命令

```bash
cat output/.state/run.json | python3 -c "
import json, sys
r = json.load(sys.stdin)
print(f'状态: {r[\"status\"]}  |  阶段: {r[\"current_phase\"]}  |  轮次: {r[\"iteration\"]}')
for p, info in r['phases'].items():
    if info.get('status') == 'in_progress':
        agents = info.get('agents', {})
        done = sum(1 for a in agents.values() if a.get('status') == 'completed')
        total = len(agents)
        print(f'{p}: [{done}/{total}]')
"
```

---

