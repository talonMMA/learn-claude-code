# s11: Autonomous Agents (自治智能体) -- 详细讲解

`s01 > s02 > s03 > s04 > s05 > s06 | s07 > s08 > s09 > s10 > [ s11 ] s12`

> *"队友自己看看板, 有活就认领"* -- 不需要领导逐个分配, 自组织。
>
> **Harness 层**: 自治 -- 模型自己找活干, 无需指派。

这是系列的**第十一课**，也是**第四阶段（团队协作与自治）**的第三课。s10 实现了结构化协议（关机、审批），但队友仍然是"被动"的 -- 必须由领导指派任务才动。s11 要回答的核心问题是：**怎么让队友从"等指令"变成"自己找活干"？** 答案是：**WORK/IDLE 双阶段生命周期 + 空闲轮询（收件箱 + 任务看板）+ 身份重注入**。

---

## 1. 问题：为什么需要自治？

### 回顾 s09-s10 的不足

s09-s10 已经有了团队通信和结构化协议，但队友本质上还是"被动执行者"：

**问题 1：领导是瓶颈**

```
s10 的工作方式:
  用户: "创建 10 个任务"
  领导: 好，任务已创建到看板

  用户: "让 alice 做任务 1"
  领导: spawn alice，分配任务 1

  用户: "让 bob 做任务 2"
  领导: spawn bob，分配任务 2

  用户: "让 alice 做任务 3"  ← alice 做完任务 1 后，还得等领导来分配
  领导: send_message 给 alice，分配任务 3

  ...重复 10 次...

  问题: 领导成了"任务调度员"，每个任务都要手动分配
  如果看板上有 100 个任务呢？领导就成了瓶颈。
```

**问题 2：队友做完就"死了"**

```
s10 中队友的生命周期:
  spawn → 执行 prompt → LLM 不再调用工具 → 线程结束

  alice 做完任务 1:
    t1: alice 收到 prompt "完成任务 1"
    t2: alice 工作...调用工具...
    t3: LLM 返回 stop_reason = "end_turn"
    t4: _loop() 结束 → 线程退出 → alice "死了"

  如果此时看板上还有 9 个任务:
    alice 不知道，因为她已经退出了
    得重新 spawn 一个新的 alice

  这就像: 工人干完一个活就辞职了，每次都要重新招聘
```

**问题 3：上下文压缩后"失忆"**

```
当对话历史太长，需要压缩（回忆 s06）:
  压缩前 messages = [
    {role: "user", content: "你是 alice，负责后端..."},
    {role: "assistant", content: "我是 alice，开始..."},
    ...（很多轮对话）...
  ]

  压缩后 messages = [
    {role: "user", content: "(压缩摘要)"},
    {role: "assistant", content: "(最近的回复)"},
  ]

  问题: 压缩后，"你是 alice" 这条信息没了！
  LLM 不知道自己叫什么、负责什么、属于哪个团队
  → 开始胡说，或者角色漂移
```

### 三个问题的解决思路

```
问题 1 (领导瓶颈)  → 队友自己扫描任务看板，自动认领
问题 2 (做完就死)  → WORK/IDLE 双阶段循环，做完不退出，进入空闲轮询
问题 3 (压缩失忆)  → 检测到上下文过短时，重新注入身份信息

三个问题，三个机制，合在一起 = "自治智能体"
```

---

## 2. 架构总览

```
Teammate lifecycle with idle cycle:

+-------+
| spawn |
+---+---+
    |
    v
+-------+   tool_use     +-------+
| WORK  | <------------- |  LLM  |
+---+---+                +-------+
    |
    | stop_reason != tool_use (or idle tool called)
    v
+--------+
|  IDLE  |  poll every 5s for up to 60s
+---+----+
    |
    +---> check inbox --> message? ----------> WORK
    |
    +---> scan .tasks/ --> unclaimed? -------> claim -> WORK
    |
    +---> 60s timeout ----------------------> SHUTDOWN

Identity re-injection after compression:
  if len(messages) <= 3:
    messages.insert(0, identity_block)
```

### 架构图解读

这张图是 s11 的核心，把它拆解开：

**1. spawn → WORK**：和 s10 一样，`spawn()` 创建线程，队友开始执行 prompt。

**2. WORK → IDLE 的触发条件**：
- `stop_reason != "tool_use"`：LLM 觉得任务做完了，不再调用工具
- `idle tool called`：LLM 主动说"我没事干了"（调用 `idle` 工具）

**3. IDLE 阶段的三条出路**：
- **收件箱有消息** → 回到 WORK（可能是领导发来新指令，或队友发来协作消息）
- **任务看板有未认领的任务** → 自动认领 → 回到 WORK
- **60 秒超时** → 真的没事干了 → 关机

**4. 身份重注入**：在认领新任务时，如果发现 messages 列表太短（说明被压缩过），在开头插入身份信息。

### 和 s10 的架构对比

```
s10 的队友生命周期:
  spawn → WORK → 线程结束（一次性）

s11 的队友生命周期:
  spawn → WORK → IDLE → WORK → IDLE → ... → SHUTDOWN（循环复用）
                  ↑                    ↑
                  自动找活              自动找活

关键区别: s11 的队友是"常驻进程"，不是"一次性脚本"
```

---

## 3. 代码逐行详解

### 3.1 文件头和导入

```python
#!/usr/bin/env python3
# Harness: autonomy -- models that find work without being told.
"""
s11_autonomous_agents.py - Autonomous Agents

Idle cycle with task board polling, auto-claiming unclaimed tasks, and
identity re-injection after context compression. Builds on s10's protocols.
"""

import json
import os
import subprocess
import threading
import time
import uuid
from pathlib import Path

from anthropic import Anthropic
from dotenv import load_dotenv
```

和 s10 完全一致，无新增依赖。

### 3.2 环境配置和常量

```python
load_dotenv(override=True)
if os.getenv("ANTHROPIC_BASE_URL"):
    os.environ.pop("ANTHROPIC_AUTH_TOKEN", None)

WORKDIR = Path.cwd()
client = Anthropic(base_url=os.getenv("ANTHROPIC_BASE_URL"))
MODEL = os.environ["MODEL_ID"]
TEAM_DIR = WORKDIR / ".team"
INBOX_DIR = TEAM_DIR / "inbox"
TASKS_DIR = WORKDIR / ".tasks"
```

继承自 s10，目录结构不变。

```python
POLL_INTERVAL = 5   # ← 新增：每 5 秒轮询一次
IDLE_TIMEOUT = 60   # ← 新增：空闲 60 秒后自动关机
```

**这两个常量是 s11 的核心参数：**
- `POLL_INTERVAL = 5`：队友进入 IDLE 后，每 5 秒检查一次有没有新工作
- `IDLE_TIMEOUT = 60`：如果连续 60 秒（= 12 次轮询）都没找到工作，自动关机

```
为什么是 5 秒和 60 秒？

5 秒轮询:
  太短（1秒）→ CPU 浪费，文件 IO 频繁
  太长（30秒）→ 新任务等太久才被认领
  5 秒是合理的折中

60 秒超时:
  太短（10秒）→ 刚创建的任务还没来得及被认领，队友就关机了
  太长（10分钟）→ 空闲线程浪费资源
  60 秒给了足够时间等待新任务
```

### 3.3 System Prompt

```python
SYSTEM = f"You are a team lead at {WORKDIR}. Teammates are autonomous -- they find work themselves."
```

和 s10 相比，领导的 system prompt 增加了"Teammates are autonomous -- they find work themselves"，提示 LLM 队友是自治的，不需要手动分配每个任务。

### 3.4 锁和追踪器

```python
VALID_MSG_TYPES = {
    "message",
    "broadcast",
    "shutdown_request",
    "shutdown_response",
    "plan_approval_response",
}

shutdown_requests = {}    # 继承自 s10
plan_requests = {}        # 继承自 s10
_tracker_lock = threading.Lock()  # 继承自 s10
_claim_lock = threading.Lock()    # ← 新增：任务认领锁
```

**`_claim_lock` 是 s11 新增的**。为什么需要？

```
并发认领问题:
  t1: alice 扫描看板 → 发现任务 #3 未认领
  t2: bob 扫描看板 → 也发现任务 #3 未认领
  t3: alice 开始认领任务 #3（读文件 → 修改 → 写回）
  t4: bob 也开始认领任务 #3（读文件 → 修改 → 写回）
  t5: 结果: task_3.json 的 owner 是 bob（alice 的写入被覆盖）
      但 alice 已经开始做任务 #3 了！
      → 两个人做同一件事，冲突！

加锁后:
  t1: alice 扫描 → 发现任务 #3
  t2: bob 扫描 → 发现任务 #3
  t3: alice 获取 _claim_lock → 认领成功 → 释放锁
  t4: bob 获取 _claim_lock → 读文件发现已有 owner → 认领失败
  t5: bob 继续扫描，找到任务 #4 → 认领任务 #4
  → 没有冲突！
```

### 3.5 MessageBus（继承自 s09/s10，无变化）

```python
class MessageBus:
    def __init__(self, inbox_dir: Path):
        self.dir = inbox_dir
        self.dir.mkdir(parents=True, exist_ok=True)

    def send(self, sender, to, content, msg_type="message", extra=None) -> str:
        # ...JSONL 写入...

    def read_inbox(self, name) -> list:
        # ...读取并清空...

    def broadcast(self, sender, content, teammates) -> str:
        # ...群发...
```

这部分和 s09/s10 完全一致，不展开。

### 3.6 ★ 任务看板扫描（新增）

```python
def scan_unclaimed_tasks() -> list:
    TASKS_DIR.mkdir(exist_ok=True)
    unclaimed = []
    for f in sorted(TASKS_DIR.glob("task_*.json")):
        task = json.loads(f.read_text())
        if (task.get("status") == "pending"
                and not task.get("owner")
                and not task.get("blockedBy")):
            unclaimed.append(task)
    return unclaimed
```

**逐行解读：**

1. `TASKS_DIR.mkdir(exist_ok=True)` -- 确保 `.tasks/` 目录存在
2. `sorted(TASKS_DIR.glob("task_*.json"))` -- 按文件名排序扫描所有任务文件
3. 三个过滤条件（全部满足才算"可认领"）：
   - `status == "pending"` -- 未开始的任务
   - `not owner` -- 没有人认领的
   - `not blockedBy` -- 没有被阻塞的（依赖的前置任务已完成）

```
任务状态过滤示例:

task_1.json: status="pending", owner=null, blockedBy=null     → ✅ 可认领
task_2.json: status="pending", owner=null, blockedBy=[1]       → ❌ 被阻塞
task_3.json: status="in_progress", owner="alice", blockedBy=null → ❌ 已认领
task_4.json: status="completed", owner="bob", blockedBy=null   → ❌ 已完成
task_5.json: status="pending", owner=null, blockedBy=null     → ✅ 可认领

scan_unclaimed_tasks() 返回 [task_1, task_5]
```

### 3.7 ★ 任务认领（新增）

```python
def claim_task(task_id: int, owner: str) -> str:
    with _claim_lock:                               # 加锁防止并发认领
        path = TASKS_DIR / f"task_{task_id}.json"
        if not path.exists():
            return f"Error: Task {task_id} not found"
        task = json.loads(path.read_text())
        task["owner"] = owner                       # 设置归属人
        task["status"] = "in_progress"              # 标记为进行中
        path.write_text(json.dumps(task, indent=2))
    return f"Claimed task #{task_id} for {owner}"
```

**关键设计：**
- 整个读-改-写操作在 `_claim_lock` 内，保证原子性
- 认领同时修改两个字段：`owner`（谁认领的）和 `status`（从 pending 变为 in_progress）

注意：这里没有检查 `task.get("owner")` 是否已有值。这是因为在 `scan_unclaimed_tasks` 阶段已经过滤了有 owner 的任务，而且加锁保证了不会有两个线程同时进入这段代码。但在更严格的实现中，应该加一个二次检查。

### 3.8 ★ 身份重注入（新增）

```python
def make_identity_block(name: str, role: str, team_name: str) -> dict:
    return {
        "role": "user",
        "content": f"<identity>You are '{name}', role: {role}, team: {team_name}. Continue your work.</identity>",
    }
```

这是一个辅助函数，生成身份信息块。使用 `<identity>` 标签包裹，让 LLM 容易识别这是身份提醒而非任务指令。

**身份重注入的触发条件是 `len(messages) <= 3`**（在后面的 `_loop` 中使用），逻辑是：

```
正常情况下 messages 至少有 5-10+ 条:
  [user prompt, assistant reply, user tool_result, assistant reply, ...]

如果 messages <= 3 条，说明发生了上下文压缩:
  压缩前: [msg1, msg2, msg3, ..., msg20]  → 20 条
  压缩后: [summary, recent_reply]          → 2 条  ← 触发重注入

重注入后:
  [identity_block, assistant_ack, summary, recent_reply, new_task]
  → LLM 又知道自己是谁了
```

### 3.9 ★ TeammateManager._loop() -- 核心：双阶段循环

这是 s11 最关键的代码，和 s10 的区别最大。让我们逐段拆解：

#### 3.9.1 初始化

```python
def _loop(self, name: str, role: str, prompt: str):
    team_name = self.config["team_name"]
    sys_prompt = (
        f"You are '{name}', role: {role}, team: {team_name}, at {WORKDIR}. "
        f"Use idle tool when you have no more work. You will auto-claim new tasks."
    )
    messages = [{"role": "user", "content": prompt}]
    tools = self._teammate_tools()
```

和 s10 对比，system prompt 增加了两个关键提示：
- "Use idle tool when you have no more work" -- 告诉 LLM：做完了别直接结束，调用 idle 工具
- "You will auto-claim new tasks" -- 告诉 LLM：你会自动获得新任务

#### 3.9.2 WORK 阶段

```python
    while True:                                 # ← 外层无限循环（WORK-IDLE-WORK-IDLE-...）
        # -- WORK PHASE: standard agent loop --
        for _ in range(50):                     # ← 内层最多 50 轮工具调用
            inbox = BUS.read_inbox(name)         # 每轮先检查收件箱
            for msg in inbox:
                if msg.get("type") == "shutdown_request":
                    self._set_status(name, "shutdown")
                    return                       # 收到关机请求，立即退出
                messages.append({"role": "user", "content": json.dumps(msg)})
            try:
                response = client.messages.create(
                    model=MODEL,
                    system=sys_prompt,
                    messages=messages,
                    tools=tools,
                    max_tokens=8000,
                )
            except Exception:
                self._set_status(name, "idle")
                return                           # API 异常，安全退出
            messages.append({"role": "assistant", "content": response.content})
            if response.stop_reason != "tool_use":
                break                            # LLM 不再调用工具 → 进入 IDLE
            results = []
            idle_requested = False
            for block in response.content:
                if block.type == "tool_use":
                    if block.name == "idle":
                        idle_requested = True    # ← 新增：检测 idle 工具
                        output = "Entering idle phase. Will poll for new tasks."
                    else:
                        output = self._exec(name, block.name, block.input)
                    print(f"  [{name}] {block.name}: {str(output)[:120]}")
                    results.append({
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": str(output),
                    })
            messages.append({"role": "user", "content": results})
            if idle_requested:
                break                            # ← 新增：idle 工具触发，进入 IDLE
```

**和 s10 的 WORK 阶段的关键区别：**

| 区别点 | s10 | s11 |
|--------|-----|-----|
| 外层循环 | 无（执行一次就退出） | `while True`（可反复进入 WORK） |
| idle 工具 | 不存在 | LLM 可调用，触发进入 IDLE |
| 循环退出后 | 线程结束 | 进入 IDLE 阶段 |

#### 3.9.3 ★ IDLE 阶段 -- 空闲轮询

```python
        # -- IDLE PHASE: poll for inbox messages and unclaimed tasks --
        self._set_status(name, "idle")           # 更新状态为 idle
        resume = False
        polls = IDLE_TIMEOUT // max(POLL_INTERVAL, 1)  # 60 // 5 = 12 次轮询
        for _ in range(polls):
            time.sleep(POLL_INTERVAL)            # 等 5 秒
```

**轮询检查 1：收件箱**

```python
            inbox = BUS.read_inbox(name)
            if inbox:
                for msg in inbox:
                    if msg.get("type") == "shutdown_request":
                        self._set_status(name, "shutdown")
                        return                   # 关机请求 → 立即退出
                    messages.append({"role": "user", "content": json.dumps(msg)})
                resume = True
                break                            # 有消息 → 回到 WORK
```

**轮询检查 2：任务看板**

```python
            unclaimed = scan_unclaimed_tasks()
            if unclaimed:
                task = unclaimed[0]              # 取第一个可认领的任务
                claim_task(task["id"], name)      # 原子认领
                task_prompt = (
                    f"<auto-claimed>Task #{task['id']}: {task['subject']}\n"
                    f"{task.get('description', '')}</auto-claimed>"
                )
                # -- 身份重注入 --
                if len(messages) <= 3:           # 上下文被压缩过
                    messages.insert(0, make_identity_block(name, role, team_name))
                    messages.insert(1, {"role": "assistant",
                                       "content": f"I am {name}. Continuing."})
                messages.append({"role": "user", "content": task_prompt})
                messages.append({"role": "assistant",
                                "content": f"Claimed task #{task['id']}. Working on it."})
                resume = True
                break                            # 认领成功 → 回到 WORK
```

**轮询结束后：**

```python
        if not resume:
            self._set_status(name, "shutdown")
            return                               # 12 次轮询都没找到工作 → 关机
        self._set_status(name, "working")        # 找到工作 → 回到 WORK 阶段
```

**IDLE 阶段的完整流程图：**

```
进入 IDLE → 设置状态为 "idle"
     ↓
  ┌─轮询 1 ─────────────────────────────────────┐
  │  sleep(5s)                                    │
  │  检查收件箱 → 有消息？ → Yes → resume WORK   │
  │              → No ↓                           │
  │  扫描任务看板 → 有未认领？ → Yes:            │
  │                  → 认领任务                    │
  │                  → 身份检查（需要重注入？）    │
  │                  → resume WORK                │
  │              → No → 继续下一轮                │
  └───────────────────────────────────────────────┘
  ┌─轮询 2 ─────────────────────────────────────┐
  │  ...同上...                                   │
  └───────────────────────────────────────────────┘
  ... 重复 12 次 ...
  ┌─轮询 12 ────────────────────────────────────┐
  │  ...同上...                                   │
  │  如果仍然没有工作 → resume = False            │
  └───────────────────────────────────────────────┘
     ↓
  resume == False → SHUTDOWN（线程退出）
```

### 3.10 队友工具执行 _exec()

```python
def _exec(self, sender: str, tool_name: str, args: dict) -> str:
    if tool_name == "bash":
        return _run_bash(args["command"])
    if tool_name == "read_file":
        return _run_read(args["path"])
    if tool_name == "write_file":
        return _run_write(args["path"], args["content"])
    if tool_name == "edit_file":
        return _run_edit(args["path"], args["old_text"], args["new_text"])
    if tool_name == "send_message":
        return BUS.send(sender, args["to"], args["content"], args.get("msg_type", "message"))
    if tool_name == "read_inbox":
        return json.dumps(BUS.read_inbox(sender), indent=2)
    if tool_name == "shutdown_response":
        # ...继承自 s10...
    if tool_name == "plan_approval":
        # ...继承自 s10...
    if tool_name == "claim_task":            # ← 新增
        return claim_task(args["task_id"], sender)
    return f"Unknown tool: {tool_name}"
```

和 s10 对比，新增了 `claim_task` 分支。队友可以**手动**认领任务（通过 LLM 调用 `claim_task` 工具），也可以**自动**认领（通过 IDLE 阶段的轮询）。

### 3.11 队友工具列表

```python
def _teammate_tools(self) -> list:
    return [
        # 4 个基础工具（bash, read_file, write_file, edit_file）-- 从 s02 继承
        # 2 个通信工具（send_message, read_inbox）-- 从 s09 继承
        # 2 个协议工具（shutdown_response, plan_approval）-- 从 s10 继承
        # 以下 2 个是 s11 新增:
        {"name": "idle",
         "description": "Signal that you have no more work. Enters idle polling phase.",
         "input_schema": {"type": "object", "properties": {}}},
        {"name": "claim_task",
         "description": "Claim a task from the task board by ID.",
         "input_schema": {"type": "object", "properties": {"task_id": {"type": "integer"}},
                         "required": ["task_id"]}},
    ]
```

**s11 新增的 2 个队友工具：**

| 工具 | 作用 | 使用场景 |
|------|------|----------|
| `idle` | 主动进入空闲状态 | LLM 觉得手头没事做时调用 |
| `claim_task` | 手动认领指定任务 | LLM 看到任务列表后主动认领 |

### 3.12 领导工具表（14 个）

```python
TOOL_HANDLERS = {
    "bash":              lambda **kw: _run_bash(kw["command"]),
    "read_file":         lambda **kw: _run_read(kw["path"], kw.get("limit")),
    "write_file":        lambda **kw: _run_write(kw["path"], kw["content"]),
    "edit_file":         lambda **kw: _run_edit(kw["path"], kw["old_text"], kw["new_text"]),
    "spawn_teammate":    lambda **kw: TEAM.spawn(kw["name"], kw["role"], kw["prompt"]),
    "list_teammates":    lambda **kw: TEAM.list_all(),
    "send_message":      lambda **kw: BUS.send("lead", kw["to"], kw["content"], kw.get("msg_type", "message")),
    "read_inbox":        lambda **kw: json.dumps(BUS.read_inbox("lead"), indent=2),
    "broadcast":         lambda **kw: BUS.broadcast("lead", kw["content"], TEAM.member_names()),
    "shutdown_request":  lambda **kw: handle_shutdown_request(kw["teammate"]),
    "shutdown_response": lambda **kw: _check_shutdown_status(kw.get("request_id", "")),
    "plan_approval":     lambda **kw: handle_plan_review(kw["request_id"], kw["approve"], kw.get("feedback", "")),
    "idle":              lambda **kw: "Lead does not idle.",     # ← 新增（但领导不用）
    "claim_task":        lambda **kw: claim_task(kw["task_id"], "lead"),  # ← 新增
}
```

领导有 14 个工具（s10 是 12 个），新增 `idle` 和 `claim_task`。注意领导的 `idle` 工具只返回 "Lead does not idle." -- 领导不需要空闲轮询，因为领导是由用户驱动的。

### 3.13 领导 Agent Loop（基本继承自 s10）

```python
def agent_loop(messages: list):
    while True:
        inbox = BUS.read_inbox("lead")
        if inbox:
            messages.append({
                "role": "user",
                "content": f"<inbox>{json.dumps(inbox, indent=2)}</inbox>",
            })
            messages.append({
                "role": "assistant",
                "content": "Noted inbox messages.",
            })
        response = client.messages.create(
            model=MODEL, system=SYSTEM, messages=messages, tools=TOOLS, max_tokens=8000,
        )
        messages.append({"role": "assistant", "content": response.content})
        if response.stop_reason != "tool_use":
            return
        results = []
        for block in response.content:
            if block.type == "tool_use":
                handler = TOOL_HANDLERS.get(block.name)
                try:
                    output = handler(**block.input) if handler else f"Unknown tool: {block.name}"
                except Exception as e:
                    output = f"Error: {e}"
                print(f"> {block.name}: {str(output)[:200]}")
                results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": str(output),
                })
        messages.append({"role": "user", "content": results})
```

和 s10 无变化。领导的循环仍然是"用户驱动"的。

### 3.14 REPL 主循环

```python
if __name__ == "__main__":
    history = []
    while True:
        try:
            query = input("\033[36ms11 >> \033[0m")
        except (EOFError, KeyboardInterrupt):
            break
        if query.strip().lower() in ("q", "exit", ""):
            break
        if query.strip() == "/team":
            print(TEAM.list_all())
            continue
        if query.strip() == "/inbox":
            print(json.dumps(BUS.read_inbox("lead"), indent=2))
            continue
        if query.strip() == "/tasks":                # 新增显示方式：带 owner 标记
            TASKS_DIR.mkdir(exist_ok=True)
            for f in sorted(TASKS_DIR.glob("task_*.json")):
                t = json.loads(f.read_text())
                marker = {"pending": "[ ]", "in_progress": "[>]",
                         "completed": "[x]"}.get(t["status"], "[?]")
                owner = f" @{t['owner']}" if t.get("owner") else ""
                print(f"  {marker} #{t['id']}: {t['subject']}{owner}")
            continue
        history.append({"role": "user", "content": query})
        agent_loop(history)
        # ...显示响应...
```

`/tasks` 命令的输出现在包含 `@owner` 标记：

```
  [ ] #1: Setup database
  [>] #2: Write API endpoints @alice
  [>] #3: Frontend login page @bob
  [x] #4: Write tests @alice
```

---

## 4. 完整执行流程示例

### 场景：创建 3 个任务，spawn 2 个队友，观察自动认领

**第 1 轮：用户输入**

```
s11 >> Create 3 tasks on the board, then spawn alice and bob
```

**第 2 轮：领导创建任务 + spawn 队友**

领导 LLM 调用一系列工具：

```
> write_file: Wrote 89 bytes            ← 创建 .tasks/task_1.json
> write_file: Wrote 92 bytes            ← 创建 .tasks/task_2.json
> write_file: Wrote 95 bytes            ← 创建 .tasks/task_3.json
> spawn_teammate: Spawned 'alice' (role: backend)
> spawn_teammate: Spawned 'bob' (role: frontend)
```

此时任务看板：
```
  [ ] #1: Setup database
  [ ] #2: Write API endpoints
  [ ] #3: Create login page
```

**第 3 轮（后台）：alice 和 bob 的 WORK 阶段**

alice 和 bob 各自收到 spawn prompt，开始工作：

```
  [alice] bash: (执行命令...)
  [alice] write_file: Wrote 200 bytes
  [alice] idle: Entering idle phase. Will poll for new tasks.
```

alice 完成了 spawn prompt 的工作，调用 `idle` → 进入 IDLE 阶段

**第 4 轮（后台）：alice 的 IDLE 阶段**

```
alice 进入 IDLE:
  轮询 1 (5s):
    收件箱? → 空
    任务看板? → 发现 task_1 (pending, 无 owner)
    → 认领 task_1!
    → len(messages) > 3，不需要身份重注入
    → messages.append("<auto-claimed>Task #1: Setup database</auto-claimed>")
    → resume = True → 回到 WORK
```

同时 bob 也进入 IDLE：

```
bob 进入 IDLE:
  轮询 1 (5s):
    收件箱? → 空
    任务看板? → task_1 已被 alice 认领，发现 task_2 (pending, 无 owner)
    → 认领 task_2!
    → resume = True → 回到 WORK
```

此时任务看板变为：
```
  [>] #1: Setup database @alice
  [>] #2: Write API endpoints @bob
  [ ] #3: Create login page
```

**第 5 轮（后台）：alice 完成任务 1，再次进入 IDLE**

```
alice WORK: 执行 task_1 的工作...
alice → idle → IDLE

alice 进入 IDLE:
  轮询 1 (5s):
    任务看板? → 发现 task_3 (pending, 无 owner)
    → 认领 task_3!
    → 回到 WORK
```

最终任务看板：
```
  [>] #1: Setup database @alice         (或 completed)
  [>] #2: Write API endpoints @bob      (或 completed)
  [>] #3: Create login page @alice
```

**第 6 轮（后台）：所有任务做完，队友关机**

```
alice WORK: 完成 task_3 → idle → IDLE
alice IDLE:
  轮询 1-12: 收件箱空，任务看板空
  60 秒超时 → SHUTDOWN

bob WORK: 完成 task_2 → idle → IDLE
bob IDLE:
  轮询 1-12: 收件箱空，任务看板空
  60 秒超时 → SHUTDOWN
```

### messages 列表变化追踪（以 alice 为例）

```
=== spawn 时 ===
messages = [
  {role: "user", content: "You are a backend developer. Set up the project."}
]

=== WORK 完成后 ===
messages = [
  {role: "user", content: "You are a backend developer..."},
  {role: "assistant", content: [tool_use: bash(...)]},
  {role: "user", content: [tool_result: "..."]},
  {role: "assistant", content: [tool_use: idle]},
  {role: "user", content: [tool_result: "Entering idle phase..."]},
]

=== 自动认领 task_1 后 ===
messages = [
  ...(上面的 5 条)...,
  {role: "user", content: "<auto-claimed>Task #1: Setup database\n...</auto-claimed>"},
  {role: "assistant", content: "Claimed task #1. Working on it."},
]

=== 如果上下文被压缩过 ===
messages = [
  {role: "user", content: "<identity>You are 'alice', role: backend, team: default...</identity>"},  ← 重注入
  {role: "assistant", content: "I am alice. Continuing."},                                          ← 重注入
  {role: "user", content: "(压缩摘要)"},
  {role: "assistant", content: "(最近回复)"},
  {role: "user", content: "<auto-claimed>Task #3: Create login page</auto-claimed>"},
  {role: "assistant", content: "Claimed task #3. Working on it."},
]
```

---

## 5. 变更总结表

| 组件 | 之前 (s10) | 之后 (s11) |
|------|-----------|-----------|
| Tools | 12 | 14 (+idle, +claim_task) |
| 自治性 | 领导指派 | 自组织 |
| 空闲阶段 | 无 | 轮询收件箱 + 任务看板 |
| 任务认领 | 仅手动 | 自动认领未分配任务 |
| 身份 | 系统提示 | + 压缩后重注入 |
| 超时 | 无 | 60 秒空闲 → 自动关机 |
| 队友生命周期 | 一次性（spawn → end） | 循环复用（WORK → IDLE → WORK → ...） |
| 并发保护 | _tracker_lock | + _claim_lock（任务认领锁） |
| 代码行数 | ~530 行 | ~580 行 |

---

## 6. 关键设计原则

### 原则 1：Pull 优于 Push
队友**主动拉取**任务（pull），而不是等领导**推送**任务（push）。Pull 模式天然平衡负载 -- 做得快的队友自然会认领更多任务。

### 原则 2：空闲不是死亡
做完手头工作不等于线程结束。IDLE 是一个**有限等待状态**，给队友机会发现新工作。只有超时后才真正关机。

### 原则 3：身份是关键上下文
压缩后 LLM 可能忘记自己的角色。身份重注入确保队友始终知道"我是谁、我负责什么"，防止角色漂移。

### 原则 4：先到先得 + 锁保证
任务认领使用"先到先得"策略，`_claim_lock` 保证不会有两个队友抢到同一个任务。这比复杂的任务分配算法更简单、更健壮。

### 原则 5：优雅降级
- API 异常 → 安全退出（不崩溃）
- 没有任务 → 等一等再关机（不浪费）
- 上下文太短 → 重注入身份（不失忆）

### 原则 6：从被动到主动
s09-s10 的队友是"员工"（等指令），s11 的队友是"自由职业者"（自己找活干）。这个思维转变是分布式系统的核心：**去中心化调度比中心化调度更可扩展**。

---

## 7. 试一试

```sh
cd learn-claude-code
python agents/s11_autonomous_agents.py
```

推荐 prompt：

1. `Create 3 tasks on the board, then spawn alice and bob. Watch them auto-claim.`
   → 观察队友如何自动认领任务，用 `/tasks` 查看 owner 变化

2. `Spawn a coder teammate and let it find work from the task board itself`
   → 先创建任务，再 spawn 队友，看它自动进入 IDLE 并认领

3. `Create tasks with dependencies. Watch teammates respect the blocked order.`
   → 创建带 `blockedBy` 的任务，观察队友跳过被阻塞的任务

4. 输入 `/tasks` 查看带 owner 的任务看板

5. 输入 `/team` 监控谁在工作（working）、谁在空闲（idle）、谁已关机（shutdown）
