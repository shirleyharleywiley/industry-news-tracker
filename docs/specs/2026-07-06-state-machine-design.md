# 行业情报 — 运行状态机设计

**日期**：2026-07-06  
**状态**：已审批  
**上下文**：2026.7.6 空气炸锅周报事故根因分析后的改进项

---

## run_id 生成规则

`{domain_slug}-{YYYYMMDD}-{HHMMSS}`，例如 `kongqizhaguo-20260706-143012`。

domain_slug 由主 agent 从用户输入的细分领域转换为英文/拼音短标识（长度 ≤ 20 字符）。

---

## 动机

两次事故（2026.7.3 蓝牙耳机、2026.7.6 空气炸锅）暴露了执行过程中的两个问题：

1. **执行不透明**：Phase 1 持续 30 分钟、12 个子 agent、大量 task-notification，用户和主 agent 都难以判断当前进度和剩余时间
2. **无法恢复**：如果会话中途崩溃，已完成的 6 个维度检索结果在对话上下文中，重启后无法复用，只能重来

状态机文件解决这两个问题——**进度可见**（随时查看 / 让主 agent 读状态就知道该做什么）、**断点恢复**（子 agent 结果持久化到文件，重启后跳过已完成步骤）。

---

## 目录结构

```
output/
├── .state/                          # 新增：运行状态目录
│   ├── run.json                     # 主控状态文件
│   └── agents/                      # 子 agent 结果摘要
│       ├── dim1-market.json
│       ├── dim2-trend.json
│       ├── dim3-competition.json
│       ├── dim4-channel.json
│       ├── dim5-operation.json
│       ├── dim6-policy.json
│       ├── agent7-synthesis.json
│       ├── agent8-boundary.json
│       └── agent9-termination.json
```

> `.state/` 加入 `.gitignore`，不提交。

---

## 数据模型

### 主控文件 `run.json`

```json
{
  "run_id": "airfryer-20260706-143012",
  "domain": "空气炸锅",
  "date_range": "2026.6.29-2026.7.6",
  "reader": "ODM厂商决策层",
  "report_type": "weekly",
  "created_at": "2026-07-06T14:30:12",
  "updated_at": "2026-07-06T15:20:45",
  "status": "completed",
  "current_phase": 4,
  "iteration": 2,
  "phases": {
    "phase1": {
      "status": "completed",
      "started_at": "2026-07-06T14:30:15",
      "completed_at": "2026-07-06T14:55:00",
      "agents": {
        "dim1-market":    {"status": "completed", "iterations": 2},
        "dim2-trend":     {"status": "completed", "iterations": 2},
        "dim3-competition":{"status": "completed", "iterations": 2},
        "dim4-channel":   {"status": "completed", "iterations": 2},
        "dim5-operation": {"status": "completed", "iterations": 2},
        "dim6-policy":    {"status": "completed", "iterations": 2}
      }
    },
    "phase2": {
      "status": "completed",
      "agent7": {"status": "completed", "iterations": 2}
    },
    "phase3": {
      "status": "completed",
      "agent8": {"status": "completed", "result": "✓ pass"},
      "agent9": {"status": "completed", "result": "terminate"}
    },
    "phase4": {
      "status": "completed",
      "output_file": "空气炸锅周报-2026.6.29-7.6.md",
      "html_file": "空气炸锅周报-2026.6.29-7.6-cmd.html"
    }
  },
  "quality": {
    "total_entries": 28,
    "by_dim": {"1":5,"2":5,"3":6,"4":4,"5":6,"6":6},
    "boundary_items": 5,
    "onehand_ratio": 0.82
  },
  "output_dir": "/Users/mac/Desktop/aitask/daily-news-ee/output"
}
```

### 子 agent 摘要 `agents/dimX-xxx.json`

```json
{
  "agent": "dim1-market",
  "run_id": "airfryer-20260706-143012",
  "status": "completed",
  "started_at": "2026-07-06T14:30:15",
  "completed_at": "2026-07-06T14:42:31",
  "iteration": 2,
  "candidates_count": 7,
  "strictly_in_window": 3,
  "boundary_items": 4,
  "sources_used": 15,
  "warnings": ["2条边界条目略早于窗口"],
  "search_keywords_tried": ["空气炸锅 市场规模 2026年6月", "..."],
  "raw_output_path": "/private/tmp/claude-501/.../tasks/a51ff1bbbe1c1cdf1.output"
}
```

Agent 7/8/9 的摘要稍作适配——Agent 7 记录 `total_candidates` / `deduplicated_count` / `merged_sources`；Agent 8 记录 `overall_status` / `checks`；Agent 9 记录 `decision` / `remaining_gaps`。

---

## 状态流转

```
                    ┌─────────┐
                    │  init   │  ← 启动时检查：output/.state/run.json 是否存在？
                    └────┬────┘      存在 → 跳到 resume 流程
                         │            不存在 → 创建新文件，进入 phase1
                    ┌────▼────┐
                    │ phase1  │  6 维度并行检索
                    │  ┌──┐   │  每收到一个 task-notification → 子 agent 写自己的 agents/dimX.json
                    │  │ N│   │  主 agent 更新 run.json 中对应 agent 的 status
                    │  └──┘   │  N=6 时自动 → phase2
                    └────┬────┘
                    ┌────▼────┐
                    │ phase2  │  Agent 7 综合
                    │  ┌──┐   │  Agent 7 写 agents/agent7-synthesis.json + 更新 run.json
                    │  │ 1│   │  完成 → phase3
                    │  └──┘   │
                    └────┬────┘
                    ┌────▼────┐
                    │ phase3  │  Agent 8 → Agent 9
                    │  ┌──┐   │  各写各自 agentX.json
           ┌───────►│  │ 2│   │  Agent 8 ✗ fail → iteration++ → 回到 phase2 或 phase1
           │        │  └──┘   │  Agent 9 terminate → phase4
           │        └────┬────┘  Agent 9 continue → iteration++ → 回到指定 phase
           │        ┌────▼────┐
           └────────┤ fix     │  主 agent 修正（如手动替换链接）
                    │  ┌──┐   │  修正完成后更新 run.json + 重新跑 Agent 8/9
                    │  │ ?│   │
                    │  └──┘   │
                    └────┬────┘
                    ┌────▼────┐
                    │ phase4  │  Write MD + mdstyle → HTML
                    │  ┌──┐   │  更新 run.json quality
                    │  │ 1│   │  status → completed
                    │  └──┘   │
                    └─────────┘
```

状态枚举：`init` / `phase1` / `phase2` / `phase3` / `fix` / `phase4` / `completed` / `failed`

> **`fix` 状态说明**：当 Agent 8 返回 ✗ fail 且主 agent 需要手动修正（如替换链接）时，status 切为 `fix`。此时 `current_phase` 保持为 3（因为修正后仍需在 phase3 内重跑 Agent 8/9）。修正完成后 status 切回 `phase3`。`fix` 是 phase3 内部的子状态，不独立占一个 phase 编号。

---

## 履历上下文恢复

### 主 agent 在收到 task-notification 时的动作

```
1. task-notification 到达
2. Read task output 获取子 agent 结果
3. 子 agent 结果中已包含 agents/dimX.json 路径 → 主 agent 读取确认
4. 如果没有 → 主 agent 从 task output 中提取摘要，写入 agents/dimX.json
5. 更新 run.json 中对应 agent status
6. 检查所有 6 维度是否完成 → 是 → 进入 phase2
```

### 会话崩溃后的恢复流程

```
1. 主 agent 启动 → SKILL.md 运行前自检
2. 检查 output/.state/run.json 是否存在？
   是 → 读取 run.json
   否 → 正常从 init 开始
3. 根据 run.status：
   phase1 → 检查哪些维度已完成（读 agents/dimX.json），只补跑缺失的
   phase2 → 检查 agent7-synthesis.json 是否存在 → 否 → 重新跑 Agent 7
   phase3 → 检查 agent8/9.json → 从当前检查点继续
   phase4 → 输出文件已存在 → 直接返回路径给用户
   completed → 直接返回路径
4. 补跑时在主 agent 调用子 agent 的 prompt 中附带上一轮的搜索结果摘要：
   "你的上一轮检索结果在 agents/dim1-market.json，已找到 7 条候选（3 条严格窗口内）。本轮请在此基础上：只补搜缺失的日期 / 补充指定关键词"
5. 子 agent 先 Read agents/dim1-market.json 获取上下文，再决定本轮检索策略
```

---

## 文件写入职责

| 文件 | 写者 | 时机 |
|------|------|------|
| `run.json` | 主 agent | 每次 task-notification 到达、phase 切换、修复完成时更新 |
| `agents/dimX-xxx.json` | 子 agent | 完成时自动写入（在 prompt 中要求） |
| `agents/agent7-synthesis.json` | Agent 7 | 完成时 |
| `agents/agent8-boundary.json` | Agent 8 | 完成时 |
| `agents/agent9-termination.json` | Agent 9 | 完成时 |

---

## 子 agent prompt 修改

每个子 agent（dim1-dim6、agent7/8/9）的 md 文件需新增一段"完成后写状态文件"指令：

```markdown
## 完成后必须执行

在返回结果之前，将你的运行摘要写入以下 JSON 文件（用 Bash 执行）：

{ "agent": "dim1-market", "run_id": "<主agent传入的run_id>", "status": "completed", ... }

文件路径：output/.state/agents/dim1-market.json
```

主 agent 在调用子 agent 时传入 `run_id` 和 `.state/` 的实际绝对路径。

---

## `run.json` 的进度摘要命令

提供一个辅助命令供用户或主 agent 快速查看当前进度：

```bash
cat output/.state/run.json | python3 -c "
import json,sys
r=json.load(sys.stdin)
print(f'状态: {r[\"status\"]}  |  阶段: {r[\"current_phase\"]}  |  轮次: {r[\"iteration\"]}')
for p,info in r['phases'].items():
    if info['status']=='in_progress':
        agents=info.get('agents',{})
        done=sum(1 for a in agents.values() if a['status']=='completed')
        total=len(agents)
        print(f'{p}: [{done}/{total}]')
"
```

输出示例：
```
状态: phase1  |  阶段: 1  |  轮次: 1
phase1: [4/6]
```

---

## 非功能约束

- `.state/` 目录加入 `.gitignore`
- 单个 JSON 文件 < 10KB（所有文件合计 < 100KB）
- `run.json` 写入时用 `Write` 工具（原子覆盖），减少并发写冲突
- 历史状态文件**不主动清理**——用户手动删或累计超过 50 个时提示清理

---

## 显式不做的事

- 不做 Web UI / 进度条
- 不做多 run 并发（一次只跑一份报告）
- 不在子 agent 内部实现状态文件的 `Read`—`Edit` 并发安全（只有一个写者）
- 不做超时自动终止（依赖 Agent 9 的 max_iterations）
