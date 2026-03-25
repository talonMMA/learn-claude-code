# Learn Claude Code -- 学习计划

## 总览

- **总课程数：** 12 课 + 1 个总纲
- **总代码量：** ~4,600 行 Python
- **建议总周期：** 3-4 周（每天 1-2 小时）
- **前置要求：** Python 基础、了解 API 调用概念

---

## 第一阶段：基础循环（第 1-3 天）

| 天 | 课程 | 核心概念 | 学习方式 |
|---|---|---|---|
| 1 | **s01 Agent 循环** | while 循环 + stop_reason 判断 | 先读文档，再读 107 行代码，跑起来 |
| 2 | **s02 Tool Use** | dispatch map（工具名→处理函数） | 在 s01 基础上加工具，理解循环不变 |
| 3 | 复习 + 动手 | 自己从零写一遍 s01+s02 | 不看参考代码，凭理解重写 |

**里程碑：** 能独立写出一个多工具的 agent loop

### 学习资料
- 文档: [s01](./docs/zh/s01-the-agent-loop.md) | [s02](./docs/zh/s02-tool-use.md)
- 代码: `agents/s01_agent_loop.py` | `agents/s02_tool_use.py`

---

## 第二阶段：规划与知识管理（第 4-8 天）

| 天 | 课程 | 核心概念 | 学习方式 |
|---|---|---|---|
| 4 | **s03 TodoWrite** | TodoManager 状态机 + nag 提醒注入 | 重点理解"先规划再行动"的设计思想 |
| 5 | **s04 子智能体** | 独立 messages[] + 上下文隔离 | 理解为什么子任务需要干净的上下文 |
| 6 | **s05 Skill 加载** | 两层注入（system prompt 摘要 + tool_result 全文）| 动手写一个自定义 SKILL.md |
| 7 | **s06 上下文压缩** | 三层压缩策略（微压缩/自动压缩/手动压缩）| 这课概念较多，画图理解三层关系 |
| 8 | 复习 + 实验 | 把 s03-s06 串联起来跑 | 观察 todo 提醒、子 agent、压缩的实际效果 |

**里程碑：** 理解 agent 如何管理计划、知识和记忆

### 学习资料
- 文档: [s03](./docs/zh/s03-todo-write.md) | [s04](./docs/zh/s04-subagent.md) | [s05](./docs/zh/s05-skill-loading.md) | [s06](./docs/zh/s06-context-compact.md)
- 代码: `agents/s03_todo_write.py` | `agents/s04_subagent.py` | `agents/s05_skill_loading.py` | `agents/s06_context_compact.py`

---

## 第三阶段：持久化与后台（第 9-12 天）

| 天 | 课程 | 核心概念 | 学习方式 |
|---|---|---|---|
| 9 | **s07 任务系统** | 文件持久化 + 依赖图（blockedBy/blocks）| 重点看 .tasks/ 目录的 JSON 结构 |
| 10 | **s08 后台任务** | 守护线程 + 通知队列 | 理解"慢操作丢后台"的异步模式 |
| 11 | 复习 + 动手 | 自己实现一个带依赖的任务系统 | 用 s07+s08 做一个真实的多步骤任务 |
| 12 | 阶段回顾 | 梳理 s01-s08 的完整 harness 架构 | 画一张架构图：循环→工具→规划→持久化 |

**里程碑：** 任务能落盘、后台能跑、依赖能自动解析

### 学习资料
- 文档: [s07](./docs/zh/s07-task-system.md) | [s08](./docs/zh/s08-background-tasks.md)
- 代码: `agents/s07_task_system.py` | `agents/s08_background_tasks.py`

---

## 第四阶段：团队协作与自治（第 13-20 天）

| 天 | 课程 | 核心概念 | 学习方式 |
|---|---|---|---|
| 13-14 | **s09 智能体团队** | TeammateManager + JSONL 邮箱 | 这课代码 406 行，分两天消化 |
| 15-16 | **s10 团队协议** | request-response FSM + request_id 关联 | 重点理解关机协议和计划审批的状态机 |
| 17-18 | **s11 自治智能体** | 空闲轮询 + 自动认领任务 | 理解从"被动接指令"到"主动找活干" |
| 19-20 | **s12 Worktree 隔离** | 控制面（任务板）+ 执行面（worktree）| 最大的一课(781行)，需要理解 git worktree |

**里程碑：** 理解多 agent 如何协作、自组织、互不干扰

### 学习资料
- 文档: [s09](./docs/zh/s09-agent-teams.md) | [s10](./docs/zh/s10-team-protocols.md) | [s11](./docs/zh/s11-autonomous-agents.md) | [s12](./docs/zh/s12-worktree-task-isolation.md)
- 代码: `agents/s09_agent_teams.py` | `agents/s10_team_protocols.py` | `agents/s11_autonomous_agents.py` | `agents/s12_worktree_task_isolation.py`

---

## 第五阶段：融会贯通（第 21-24 天）

| 天 | 内容 | 方式 |
|---|---|---|
| 21 | 阅读 `s_full.py` 总纲 (736 行) | 对照 12 课逐一识别每个机制 |
| 22 | 用 `web/` 交互平台可视化复习 | `cd web && npm install && npm run dev` |
| 23 | 动手项目：为一个新领域设计 harness | 比如：为"每日新闻摘要"设计 agent harness |
| 24 | 总结笔记 + 延伸阅读 | 看 Kode CLI / Kode SDK / claw0 姊妹项目 |

---

## 每课学习方法（四步法）

1. **读文档**（docs/zh/sXX-*.md）— 理解心智模型和 ASCII 架构图
2. **读代码**（agents/sXX_*.py）— 对照文档看实现，关注增量部分
3. **跑代码** — `python agents/sXX_*.py` 实际运行，观察行为
4. **改代码** — 做一个小修改（加个工具、改个策略），验证理解

## 详细讲解生成指令

在新 session 中使用以下 prompt 让 Claude 为指定课程生成详细讲解：

> 请阅读 STUDY_PLAN.md，然后为我生成第 N 课的详细讲解。

### 讲解格式模板

每课讲解需包含以下章节，内容比原文档只多不少：

1. **课程标题 + 原文引言** — 保留原文档的 quote 和定位
2. **问题：为什么需要这个机制？** — 用直白的语言和具体场景解释，不是原文的简单一句话，而是展开说清楚痛点
3. **解决方案的架构** — 保留原文的 ASCII 架构图，并加上对架构图的逐元素解读和设计哲学说明
4. **代码逐行详解** — 这是最核心的部分，要求：
   - 把代码按功能分段（如"文件头和导入"、"环境配置"、"System Prompt"、"工具定义"、"工具执行函数"、"核心循环"、"交互式 REPL"等）
   - 每一段给出完整代码块 + 逐行/逐块解释
   - 对关键设计决策解释"为什么这么做"而不只是"做了什么"
   - 涉及 API 格式的地方要解释字段含义（如 tool_use_id 的配对机制、stop_reason 的取值等）
   - 与前几课的增量对比（从 s02 开始）：明确标出"这是新增的"vs"这是从上一课继承的"
5. **完整执行流程示例** — 用一个具体的用户输入，画出多轮循环中 messages 列表的变化过程，标明每一轮的 stop_reason
6. **消息列表 / 状态的完整结构** — 用表格或缩进展示循环结束后的数据结构全貌
7. **变更总结表** — 保留原文档的变更表
8. **关键设计原则** — 总结本课体现的设计思想，每条原则用一句话概括 + 一句话解释
9. **试一试** — 保留原文档的运行指令和推荐 prompt

## 环境准备（开始前做一次）

```sh
pip install -r requirements.txt
cp .env.example .env   # 填入 ANTHROPIC_API_KEY
```

## 进度追踪

- [x] s01 Agent 循环
- [x] s02 Tool Use
- [ ] s03 TodoWrite
- [ ] s04 子智能体
- [ ] s05 Skill 加载
- [ ] s06 上下文压缩
- [ ] s07 任务系统
- [ ] s08 后台任务
- [ ] s09 智能体团队
- [ ] s10 团队协议
- [ ] s11 自治智能体
- [ ] s12 Worktree 隔离
- [ ] s_full 总纲通读
- [ ] 动手项目
