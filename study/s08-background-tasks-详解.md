# s08: Background Tasks (后台任务) -- 详细讲解

`s01 > s02 > s03 > s04 > s05 > s06 | s07 > [ s08 ] s09 > s10 > s11 > s12`

> *"慢操作丢后台, agent 继续想下一步"* -- 后台线程跑命令, 完成后注入通知。
>
> **Harness 层**: 后台执行 -- 模型继续思考, harness 负责等待。

这是系列的**第八课**，也是**第三阶段（持久化与后台）**的第二课。s07 解决了"任务状态比对话更长命"的问题，把扁平清单升级为磁盘上的任务图（DAG）。s08 要回答的核心问题是：**当 agent 需要跑一个耗时几分钟的命令（`npm install`、`pytest`、`docker build`）时，能不能不干等，而是继续做别的事？** 答案是：**用守护线程跑慢操作，主线程继续 LLM 循环，完成后通过通知队列把结果注入回来**。

---

## 1. 问题：为什么需要后台任务？

原文一句话：
> 有些命令要跑好几分钟: `npm install`、`pytest`、`docker build`。阻塞式循环下模型只能干等。

### 展开理解

回忆 s02 的 `run_bash` 是怎么工作的：

```python
def run_bash(command: str) -> str:
    r = subprocess.run(command, shell=True, capture_output=True, text=True, timeout=120)
    return (r.stdout + r.stderr).strip()
```

`subprocess.run` 是**阻塞调用**。整个 agent loop 停在这里，等子进程跑完才继续。

**痛点 1：串行瓶颈**

```
用户: "安装依赖，顺便帮我建一个配置文件"

理想的执行方式:
  Agent ──[spawn npm install]──[写配置文件]──[npm 完成]── done
              |                    |
              v                    v
           [npm 在后台跑]      [同时写文件]     ← 并行

实际的执行方式 (s07 及之前):
  Agent ──[npm install ... 等 60 秒 ...]──[写配置文件]── done
              |                                |
              v                                v
           [整个循环卡住]                  [60秒后才开始]   ← 串行
```

用户说的是"顺便"，暗示两件事可以并行。但阻塞式的 `subprocess.run` 让 agent 只能一个一个来。

**痛点 2：浪费 LLM 算力**

```
一轮 LLM 调用的成本:
  - API 调用费用 (按 token 计费)
  - 网络延迟 (~100-500ms)
  - 推理时间 (~1-5s)

阻塞等待 npm install 的 60 秒里:
  - LLM 什么都没做
  - 用户什么都看不到
  - 白白浪费了 60 秒可以做其他事的时间

理想情况:
  - 慢命令丢后台
  - LLM 继续处理下一个工具调用或回复用户
  - 后台完成后, 结果在下一轮注入
```

**痛点 3：超时风险**

```
s02 的 bash 工具: timeout=120 (2分钟)
但真实场景中:
  - docker build: 可能 5-10 分钟
  - npm install (大项目): 可能 3-5 分钟
  - pytest (完整测试套件): 可能 10+ 分钟

如果提高 timeout:
  → 整个 agent loop 被卡更久
  → 用户体验极差

如果不提高 timeout:
  → 命令被强制杀死, 任务失败
```

### 核心矛盾

```
需求 1: 慢命令需要足够的时间完成 (不能超时杀死)
需求 2: agent 不能因为等命令而停止思考
需求 3: 命令完成后, 结果必须回到 agent 的上下文中

s07 及之前的局限:
  ✗ subprocess.run 是阻塞的 → agent 只能干等
  ✗ 没有并行能力 → 两个独立命令只能串行
  ✗ 超时短 → 长命令会被杀
  ✗ 没有异步通知 → 不知道后台完成了
```

### 解决思路

把"阻塞执行"拆成"启动 + 通知"两步：

```
subprocess.run (s07)            BackgroundManager (s08)
────────────────────           ──────────────────────
执行方式: 阻塞, 等到完成       执行方式: 守护线程, 立即返回
并发:     不支持               并发:     多个后台任务同时跑
通知:     直接返回结果         通知:     完成后进入通知队列
超时:     120s                 超时:     300s (更宽容)
适合:     快命令 (<10s)        适合:     慢命令 (>10s)
```

---

## 2. 解决方案的架构

### 主线程 + 后台线程模型

```
Main thread                Background thread
+-----------------+        +-----------------+
| agent loop      |        | subprocess runs |
| ...             |        | ...             |
| [LLM call] <---+------- | enqueue(result) |
|  ^drain queue   |        +-----------------+
+-----------------+

Timeline:
Agent --[spawn A]--[spawn B]--[other work]----
             |          |
             v          v
          [A runs]   [B runs]      (parallel)
             |          |
             +-- results injected before next LLM call --+
```

### 架构图逐元素解读

**Main thread (主线程)**
- 就是 s01 以来的 agent loop：`while True → LLM call → tool use → ...`
- **始终是单线程**。没有引入多线程并发到主循环
- 唯一的变化：每次 LLM 调用前，先排空通知队列（drain）

**Background thread (后台线程)**
- 每个 `background_run` 调用启动一个 Python 守护线程（`daemon=True`）
- 守护线程内部跑 `subprocess.run`（这是阻塞的，但在后台线程中阻塞不影响主线程）
- 完成后，把结果放进通知队列

**通知队列 (Notification Queue)**
- 连接主线程和后台线程的桥梁
- 线程安全：用 `threading.Lock` 保护
- 生产者：后台线程的 `_execute` 方法
- 消费者：主循环的 `drain_notifications` 方法

**关键设计决策：为什么用线程而不是 asyncio？**

```
方案对比:
  asyncio:  需要把整个 agent loop 改成 async/await
            所有工具函数都要改成 async
            Anthropic SDK 需要用异步版本
            改动巨大, 不兼容 s01-s07 的同步模式

  threading: 主线程保持同步, 一行代码不用改
             只有后台任务跑在新线程里
             通知队列做桥梁
             增量改动最小 ← 这是选它的原因

  multiprocessing: 跨进程通信更复杂
                   序列化/反序列化开销
                   杀伤力过大
```

**选线程的理由：最小增量原则**。主循环保持单线程、同步、与 s01-s07 完全一致。只有"等子进程"这个 I/O 操作被并行化了。

### 数据流

```
1. 用户: "npm install 后台跑, 顺便帮我写个 config"

2. LLM 响应: tool_use = background_run(command="npm install")
   → BG.run("npm install")
   → 启动守护线程, 返回 "Background task abc123 started"
   → tool_result 返回给 LLM

3. LLM 继续响应: tool_use = write_file(path="config.json", content="{...}")
   → 写文件, 返回 "Wrote 42 bytes"
   → 同时后台线程在跑 npm install (完全并行)

4. LLM 回复用户: "配置文件已创建, npm install 正在后台运行"

5. 用户: "好的, 装完了吗?"

6. 下一轮 agent_loop 进入前:
   → drain_notifications() 检查通知队列
   → 如果 npm install 完成了: 把结果注入 messages[]
   → LLM 看到: <background-results> [bg:abc123] completed: ... </background-results>
   → LLM 回复: "npm install 已完成, 以下是输出..."
```

---

## 3. 代码逐行详解

### 3.1 文件头和导入

```python
#!/usr/bin/env python3
# Harness: background execution -- the model thinks while the harness waits.
"""
s08_background_tasks.py - Background Tasks

Run commands in background threads. A notification queue is drained
before each LLM call to deliver results.
"""

import os
import subprocess
import threading       # ← 新增: Python 标准库线程模块
import uuid            # ← 新增: 生成唯一任务 ID
from pathlib import Path

from anthropic import Anthropic
from dotenv import load_dotenv
```

**vs s07**: s07 导入了 `json`（文件持久化）和 `glob`（扫描 .tasks/ 目录）。s08 不需要这些，因为后台任务是内存中的、会话级的。取而代之的是 `threading` 和 `uuid`。

**`threading`**: Python 标准库。虽然 Python 有 GIL（全局解释器锁），但 GIL 在 I/O 等待（如 `subprocess.run`）时会自动释放，所以线程非常适合"等子进程"这种 I/O 密集型场景。

**`uuid`**: 生成全局唯一的任务 ID。用 `uuid4()[:8]` 截取前 8 位作为短 ID（如 `"a3f7b2c1"`），既唯一又方便 LLM 和用户引用。

### 3.2 环境配置

```python
load_dotenv(override=True)

if os.getenv("ANTHROPIC_BASE_URL"):
    os.environ.pop("ANTHROPIC_AUTH_TOKEN", None)

WORKDIR = Path.cwd()
client = Anthropic(base_url=os.getenv("ANTHROPIC_BASE_URL"))
MODEL = os.environ["MODEL_ID"]
```

**与 s07 完全相同**。环境配置是整个系列的"稳定基座"，从 s01 到 s08 几乎不变。

### 3.3 System Prompt

```python
SYSTEM = f"You are a coding agent at {WORKDIR}. Use background_run for long-running commands."
```

**vs s07**: s07 的 system prompt 包含任务管理的详细指令（何时拆任务、如何管理依赖）。s08 回到极简风格——只添一句"用 `background_run` 跑长命令"。

**设计考量**: 这不是说 s08 不需要复杂提示，而是本课聚焦后台机制本身。在真实产品中，你可以把 s07 的任务管理 + s08 的后台执行结合在一个 system prompt 中。

### 3.4 BackgroundManager 类 —— 本课核心

这是 s08 的全部新增核心。我们逐方法拆解。

#### 3.4.1 `__init__`: 初始化

```python
class BackgroundManager:
    def __init__(self):
        self.tasks = {}               # task_id -> {status, result, command}
        self._notification_queue = []  # completed task results
        self._lock = threading.Lock()  # 保护 _notification_queue
```

| 属性 | 类型 | 作用 |
|---|---|---|
| `tasks` | `dict` | 所有任务的注册表。key 是 task_id (str)，value 是包含 status/result/command 的字典 |
| `_notification_queue` | `list` | 已完成任务的通知队列。后台线程往里 append，主线程 drain 清空 |
| `_lock` | `threading.Lock` | 互斥锁，保护 `_notification_queue` 的读写 |

**为什么 `tasks` 不需要锁？**

```
tasks 的写入发生在两个地方:
  1. run() 方法: 主线程中, 创建初始条目 → tasks[id] = {status: "running"}
  2. _execute() 方法: 后台线程中, 更新状态 → tasks[id]["status"] = "completed"

这两个写入不会同时发生:
  - run() 先执行, 创建条目
  - _execute() 后执行, 修改已有条目
  - Python 的 dict 赋值是原子的 (CPython 实现细节)

而 _notification_queue 的 append 和 drain 可能同时发生:
  - 后台线程 A 在 append
  - 主线程在 drain (读取 + 清空)
  → 需要锁保护
```

#### 3.4.2 `run()`: 启动后台任务

```python
def run(self, command: str) -> str:
    """Start a background thread, return task_id immediately."""
    task_id = str(uuid.uuid4())[:8]
    self.tasks[task_id] = {"status": "running", "result": None, "command": command}
    thread = threading.Thread(
        target=self._execute, args=(task_id, command), daemon=True
    )
    thread.start()
    return f"Background task {task_id} started: {command[:80]}"
```

**逐行解析**:

1. **`task_id = str(uuid.uuid4())[:8]`**
   - `uuid4()` 生成一个随机 UUID，如 `"a3f7b2c1-...""`
   - 截取前 8 位作为短 ID：`"a3f7b2c1"`
   - 8 位十六进制 = 4,294,967,296 种可能，单次会话不可能冲突

2. **`self.tasks[task_id] = {...}`**
   - 在注册表中记录这个任务
   - `status: "running"` 表示正在执行
   - `result: None` 表示尚无结果

3. **`threading.Thread(target=..., daemon=True)`**
   - 创建一个新线程，目标函数是 `self._execute`
   - **`daemon=True`**: 守护线程。主程序退出时，守护线程自动被杀。不会阻止程序退出
   - 如果不设 daemon，即使用户按 Ctrl+C，后台线程可能导致进程挂起

4. **`thread.start()`**
   - 线程开始执行，`run()` 方法**立即返回**
   - 这是"火并遗忘 (fire and forget)"的关键：调用方不等结果

5. **返回值**: `"Background task abc123 started: npm install"`
   - 这个字符串作为 tool_result 返回给 LLM
   - LLM 拿到 task_id 后可以用 `check_background` 查询状态

#### 3.4.3 `_execute()`: 后台线程的执行体

```python
def _execute(self, task_id: str, command: str):
    """Thread target: run subprocess, capture output, push to queue."""
    try:
        r = subprocess.run(
            command, shell=True, cwd=WORKDIR,
            capture_output=True, text=True, timeout=300
        )
        output = (r.stdout + r.stderr).strip()[:50000]
        status = "completed"
    except subprocess.TimeoutExpired:
        output = "Error: Timeout (300s)"
        status = "timeout"
    except Exception as e:
        output = f"Error: {e}"
        status = "error"
    self.tasks[task_id]["status"] = status
    self.tasks[task_id]["result"] = output or "(no output)"
    with self._lock:
        self._notification_queue.append({
            "task_id": task_id,
            "status": status,
            "command": command[:80],
            "result": (output or "(no output)")[:500],
        })
```

**关键点**:

| 要素 | 说明 |
|---|---|
| **timeout=300** | 后台任务给 5 分钟（vs 阻塞式 bash 的 120s）。因为不阻塞主线程，可以更宽容 |
| **output[:50000]** | 保存在 tasks 中的完整输出，截断到 50KB |
| **result[:500]** | 通知队列中的摘要输出，只截取 500 字符。够 LLM 判断成功/失败，不必看完整日志 |
| **with self._lock** | 互斥锁保护队列写入。确保多个后台线程同时完成时不会丢通知 |

**两层输出截断的设计**:

```
tasks[task_id]["result"]     = output[:50000]    # 完整日志, 供 check_background 查询
_notification_queue.result   = output[:500]      # 摘要, 自动注入到 messages[]

为什么区分?
  - 通知队列的内容会被注入到 messages[]
  - messages[] 中的内容算 token, 太长会浪费上下文窗口
  - 所以通知只给摘要 (500 字符)
  - 如果 LLM 需要看完整输出, 可以调用 check_background(task_id)
```

#### 3.4.4 `check()`: 查询任务状态

```python
def check(self, task_id: str = None) -> str:
    """Check status of one task or list all."""
    if task_id:
        t = self.tasks.get(task_id)
        if not t:
            return f"Error: Unknown task {task_id}"
        return f"[{t['status']}] {t['command'][:60]}\n{t.get('result') or '(running)'}"
    lines = []
    for tid, t in self.tasks.items():
        lines.append(f"{tid}: [{t['status']}] {t['command'][:60]}")
    return "\n".join(lines) if lines else "No background tasks."
```

**两种用法**:
- `check(task_id="abc123")`: 查看特定任务的状态和输出
- `check()`: 列出所有后台任务的概览

LLM 可以主动调用这个工具来检查任务进度，不必等通知。

#### 3.4.5 `drain_notifications()`: 排空通知队列

```python
def drain_notifications(self) -> list:
    """Return and clear all pending completion notifications."""
    with self._lock:
        notifs = list(self._notification_queue)
        self._notification_queue.clear()
    return notifs
```

**这是连接主线程和后台线程的桥梁**:

```
操作:
  1. 加锁 (防止后台线程同时写入)
  2. 复制整个队列到 notifs
  3. 清空队列
  4. 解锁
  5. 返回 notifs

为什么要 list() 复制?
  如果直接 notifs = self._notification_queue
  然后 clear()
  → notifs 也会变空 (引用同一个列表对象)
  所以用 list() 做浅拷贝
```

### 3.5 全局实例

```python
BG = BackgroundManager()
```

整个进程只有一个 BackgroundManager 实例。所有后台任务共享同一个注册表和通知队列。

### 3.6 工具函数 (从 s07 继承)

```python
def safe_path(p: str) -> Path:
    path = (WORKDIR / p).resolve()
    if not path.is_relative_to(WORKDIR):
        raise ValueError(f"Path escapes workspace: {p}")
    return path

def run_bash(command: str) -> str:
    dangerous = ["rm -rf /", "sudo", "shutdown", "reboot", "> /dev/"]
    if any(d in command for d in dangerous):
        return "Error: Dangerous command blocked"
    try:
        r = subprocess.run(command, shell=True, cwd=WORKDIR,
                           capture_output=True, text=True, timeout=120)
        out = (r.stdout + r.stderr).strip()
        return out[:50000] if out else "(no output)"
    except subprocess.TimeoutExpired:
        return "Error: Timeout (120s)"

def run_read(path: str, limit: int = None) -> str: ...
def run_write(path: str, content: str) -> str: ...
def run_edit(path: str, old_text: str, new_text: str) -> str: ...
```

**与 s07 的区别**: s07 有 8 个工具（包括 create_task、update_task、list_tasks、get_task）。s08 砍掉了任务管理工具，回到 4 个基础工具 + 2 个后台工具 = 6 个工具。

**阻塞式 bash 仍然保留**:

```
bash (阻塞):          适合快命令 — ls, cat, echo, git status
background_run (异步): 适合慢命令 — npm install, pytest, docker build

两种执行方式共存, LLM 自己选择:
  - system prompt 说 "Use background_run for long-running commands"
  - LLM 根据命令预估耗时, 决定用哪个
```

### 3.7 工具注册 (Dispatch Map + Schema)

```python
TOOL_HANDLERS = {
    "bash":             lambda **kw: run_bash(kw["command"]),
    "read_file":        lambda **kw: run_read(kw["path"], kw.get("limit")),
    "write_file":       lambda **kw: run_write(kw["path"], kw["content"]),
    "edit_file":        lambda **kw: run_edit(kw["path"], kw["old_text"], kw["new_text"]),
    "background_run":   lambda **kw: BG.run(kw["command"]),         # ← 新增
    "check_background": lambda **kw: BG.check(kw.get("task_id")),   # ← 新增
}
```

**新增的两个工具**:

| 工具名 | 映射到 | 说明 |
|---|---|---|
| `background_run` | `BG.run(command)` | 启动后台任务，立即返回 task_id |
| `check_background` | `BG.check(task_id)` | 查询任务状态，task_id 可选 |

```python
TOOLS = [
    # ... 4 个基础工具 (bash, read_file, write_file, edit_file) ...
    {"name": "background_run", "description": "Run command in background thread. Returns task_id immediately.",
     "input_schema": {"type": "object", "properties": {"command": {"type": "string"}}, "required": ["command"]}},
    {"name": "check_background", "description": "Check background task status. Omit task_id to list all.",
     "input_schema": {"type": "object", "properties": {"task_id": {"type": "string"}}}},
]
```

注意 `check_background` 的 `task_id` **不是 required**。省略时列出全部任务。

### 3.8 Agent Loop —— 核心变更

```python
def agent_loop(messages: list):
    while True:
        # ========== 新增: 排空后台通知 ==========
        notifs = BG.drain_notifications()
        if notifs and messages:
            notif_text = "\n".join(
                f"[bg:{n['task_id']}] {n['status']}: {n['result']}" for n in notifs
            )
            messages.append({"role": "user", "content": f"<background-results>\n{notif_text}\n</background-results>"})
            messages.append({"role": "assistant", "content": "Noted background results."})
        # ========== 以上是新增部分 ==========

        response = client.messages.create(
            model=MODEL, system=SYSTEM, messages=messages,
            tools=TOOLS, max_tokens=8000,
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
                results.append({"type": "tool_result", "tool_use_id": block.id, "content": str(output)})
        messages.append({"role": "user", "content": results})
```

**唯一变更是循环顶部的通知排空逻辑**。主循环的其余部分与 s02 以来完全一致。

让我们仔细看通知注入的过程：

```python
# 1. 排空通知队列
notifs = BG.drain_notifications()

# 2. 如果有通知 (且 messages 非空)
if notifs and messages:
    # 3. 格式化通知文本
    notif_text = "\n".join(
        f"[bg:{n['task_id']}] {n['status']}: {n['result']}" for n in notifs
    )
    # 4. 注入为 user 消息 (用 XML 标签包裹)
    messages.append({
        "role": "user",
        "content": f"<background-results>\n{notif_text}\n</background-results>"
    })
    # 5. 补一条 assistant 消息 (保持交替)
    messages.append({
        "role": "assistant",
        "content": "Noted background results."
    })
```

**为什么要用 `<background-results>` 标签？**

```
LLM 看到的效果:
  user: <background-results>
        [bg:abc123] completed: Successfully installed 42 packages
        </background-results>
  assistant: Noted background results.
  (然后 LLM 生成新的 response)

标签的作用:
  - 让 LLM 明确区分"用户说的话"和"系统通知"
  - XML 标签是 Claude 系列模型天然理解的结构
  - 避免 LLM 把通知误解为用户在问新问题
```

**为什么要补一条 "Noted background results." ？**

```
Claude API 要求 messages 交替: user → assistant → user → assistant → ...

如果只 append user 消息, 然后 LLM 生成新 response:
  messages[-2] = user (background results)
  messages[-1] = user (LLM的新 response 会报错, 因为连续两个 user)

所以在通知后补一条 assistant 消息, 保持交替:
  messages[-2] = user  (background results)
  messages[-1] = assistant ("Noted background results.")
  → 然后 LLM call 正常返回新的 assistant response
```

**通知注入的时机**:

```
为什么在 LLM 调用前, 而不是调用后?

Before LLM call:
  drain → inject → LLM sees results → LLM can respond to them
  ✓ LLM 能看到并处理后台结果

After LLM call:
  LLM call → drain → inject → ???
  ✗ LLM 已经回复了, 看不到结果
  ✗ 要等下一轮用户输入才能处理
```

### 3.9 交互式 REPL

```python
if __name__ == "__main__":
    history = []
    while True:
        try:
            query = input("\033[36ms08 >> \033[0m")
        except (EOFError, KeyboardInterrupt):
            break
        if query.strip().lower() in ("q", "exit", ""):
            break
        history.append({"role": "user", "content": query})
        agent_loop(history)
        response_content = history[-1]["content"]
        if isinstance(response_content, list):
            for block in response_content:
                if hasattr(block, "text"):
                    print(block.text)
        print()
```

与之前的课程格式一致。提示符变为 `s08 >>` （青色）。

---

## 4. 完整执行流程示例

### 场景: "后台安装依赖, 同时创建配置文件"

用户输入: `Run "sleep 5 && echo done" in the background, then create a file greeting.txt with "hello"`

#### 第一轮: 用户输入

```
messages = [
  { role: "user", content: 'Run "sleep 5 && echo done" in the background, then create a file greeting.txt with "hello"' }
]
```

进入 agent_loop:
- `drain_notifications()` → 空列表 (没有后台任务)
- LLM 调用 → response 包含两个 tool_use:

```
response.content = [
  TextBlock("I'll run the command in the background and create the file."),
  ToolUseBlock(id="tu1", name="background_run", input={"command": "sleep 5 && echo done"}),
  ToolUseBlock(id="tu2", name="write_file", input={"path": "greeting.txt", "content": "hello"}),
]
response.stop_reason = "tool_use"
```

执行两个工具:
- `background_run`: BG.run("sleep 5 && echo done") → "Background task a3f7b2c1 started: sleep 5 && echo done"
  - 同时，守护线程开始在后台跑 `sleep 5 && echo done`
- `write_file`: run_write("greeting.txt", "hello") → "Wrote 5 bytes"

```
messages = [
  { role: "user", content: '...' },
  { role: "assistant", content: [TextBlock, ToolUseBlock(tu1), ToolUseBlock(tu2)] },
  { role: "user", content: [
      { type: "tool_result", tool_use_id: "tu1", content: "Background task a3f7b2c1 started: sleep 5 && echo done" },
      { type: "tool_result", tool_use_id: "tu2", content: "Wrote 5 bytes" },
  ]}
]
```

#### 第二轮: LLM 总结

进入下一轮循环:
- `drain_notifications()` → 空列表 (sleep 5 还在跑)
- LLM 调用 → response 是纯文本回复:

```
response.content = [
  TextBlock("I've started the command in the background (task a3f7b2c1) and created greeting.txt. The background task is still running."),
]
response.stop_reason = "end_turn"
```

`stop_reason == "end_turn"` → 退出 agent_loop，回到 REPL 等待用户输入。

#### 第三轮: 用户查询 (5秒后)

用户输入: `Is the background task done?`

```
messages = [
  ...之前的消息...,
  { role: "user", content: "Is the background task done?" }
]
```

进入 agent_loop:
- `drain_notifications()` → **这次有结果了！**
  - 后台线程在 5 秒后完成了 sleep，往通知队列 append 了结果
  - drain 返回: `[{"task_id": "a3f7b2c1", "status": "completed", "command": "sleep 5 && echo done", "result": "done"}]`

通知注入:

```
messages = [
  ...之前的消息...,
  { role: "user", content: "<background-results>\n[bg:a3f7b2c1] completed: done\n</background-results>" },
  { role: "assistant", content: "Noted background results." },
  { role: "user", content: "Is the background task done?" }
]
```

**等等，这里有个问题**: 通知注入后，又 append 了用户的新消息。但看代码逻辑，通知注入发生在 agent_loop 内部，而用户消息在 REPL 中已经 append 到 history 了。让我重新梳理：

实际执行顺序:
1. REPL: `history.append({"role": "user", "content": "Is the background task done?"})` → 用户消息已在 messages 末尾
2. 进入 agent_loop
3. `drain_notifications()` → 有通知
4. 通知被 insert 到 messages 末尾... 但用户消息已经在末尾了

看代码: `messages.append(...)` 是 append 到末尾。所以实际 messages 变成:

```
messages = [
  ...之前的消息...,
  { role: "user", content: "Is the background task done?" },     ← REPL append 的
  { role: "user", content: "<background-results>..." },           ← drain append 的
  { role: "assistant", content: "Noted background results." },   ← drain append 的
]
```

这意味着用户消息和通知之间有两个连续的 user 消息——但通知后面紧跟一个 assistant 消息，所以 LLM 看到的最后三条消息的 role 交替是 `user → user → assistant`，这**违反了交替规则**。

**实际上这是一个已知的简化**：在 LLM API 调用时，连续的同 role 消息会被自动合并（Anthropic API 会处理），或者说这段代码在通知和用户消息恰好同时存在时可能需要更细致的插入逻辑。但在实践中，由于后台任务通常在 agent 回复后的"等待用户输入"期间完成，通知注入通常发生在新的用户消息**之前**（新轮对话中），所以这个边界情况很少触发。

不管怎样，LLM 最终看到了后台结果 + 用户问题，会回复:

```
response.content = [
  TextBlock("Yes! The background task completed successfully. Output: 'done'"),
]
response.stop_reason = "end_turn"
```

---

## 5. 消息列表的完整结构 (循环结束后)

```
messages[] 在完整对话后的状态:

[0] user:      "Run sleep 5 in background, then create greeting.txt"
[1] assistant: [TextBlock, ToolUseBlock(background_run), ToolUseBlock(write_file)]
[2] user:      [tool_result(task started), tool_result(wrote 5 bytes)]
[3] assistant: [TextBlock("Started and created...")]
[4] user:      "Is the background task done?"
[5] user:      "<background-results>[bg:a3f7b2c1] completed: done</background-results>"
[6] assistant: "Noted background results."
[7] user:      (LLM 调用时, messages 到 [6] 为止)
    → LLM 看到通知 + 用户问题, 综合回复
[7] assistant: [TextBlock("Yes! Completed.")]
```

**BackgroundManager 内部状态**:

```
BG.tasks = {
    "a3f7b2c1": {
        "status": "completed",
        "result": "done",
        "command": "sleep 5 && echo done"
    }
}
BG._notification_queue = []   # 已经被 drain 清空了
```

---

## 6. 变更总结表

| 组件           | 之前 (s07)           | 之后 (s08)                           |
|----------------|---------------------|--------------------------------------|
| Tools          | 8 (基础4 + 任务管理4) | 6 (基础4 + background_run + check)  |
| 核心新增类     | TaskManager          | BackgroundManager                    |
| 执行方式       | 仅阻塞              | 阻塞 + 后台线程                      |
| 通知机制       | 无                  | 每轮排空的通知队列                    |
| 并发           | 无                  | 守护线程 (daemon threads)            |
| 持久化         | .tasks/ 目录 JSON    | 无 (内存中, 会话级)                  |
| 数据安全       | 文件锁              | threading.Lock                       |
| 状态存储       | 磁盘                | 内存                                 |
| System Prompt  | 详细任务管理指令      | 一句话 "use background_run"          |
| 代码行数       | ~350行               | ~235行                               |

**注意**: s08 比 s07 更简洁，因为它**移除了任务管理系统**，只聚焦后台执行机制。在真实产品中，s07 的持久化任务 + s08 的后台执行是互补的，应该结合使用。

---

## 7. 关键设计原则

### 原则 1: 最小增量改动

**一句话**: 只把"等子进程"这一个环节并行化，主循环保持单线程不变。

**为什么**: 引入完整的异步框架（asyncio）会要求重写所有工具函数和 API 调用。线程只需要在 "subprocess.run 太慢" 这个点上做局部优化，改动量极小。

### 原则 2: 生产者-消费者解耦

**一句话**: 后台线程只管"生产结果"放入队列，主线程只管"消费结果"从队列取出。

**为什么**: 两个线程通过队列通信，不需要互相等待。后台线程不关心 LLM 在做什么，主线程不关心子进程跑到哪了。解耦带来简单性。

### 原则 3: 摘要 vs 完整日志的分层

**一句话**: 通知队列只传 500 字符摘要，完整输出留在 tasks 字典里按需查。

**为什么**: messages[] 中的每个字符都是 token，都有成本。自动注入的通知应该尽可能短——够判断成功/失败就行。如果 LLM 需要详细信息，它可以主动调用 `check_background`。

### 原则 4: 保留阻塞式工具

**一句话**: `bash` (阻塞) 和 `background_run` (异步) 共存，LLM 自己选择。

**为什么**: 对于 `ls`、`cat`、`echo` 这类瞬时命令，阻塞式执行更简单直接。不是所有命令都需要后台执行。给 LLM 选择权比强制所有命令走后台更实用。

### 原则 5: 守护线程保证可退出

**一句话**: `daemon=True` 确保主程序退出时后台线程自动终止。

**为什么**: 如果不设 daemon，用户按 Ctrl+C 后，后台线程可能继续运行，导致进程挂起。守护线程自动跟随主线程生死。

---

## 8. 与 Claude Code 的对应

在实际的 Claude Code 中，后台任务机制对应：

| s08 概念 | Claude Code 实际功能 |
|---|---|
| `background_run` | Bash 工具的 `run_in_background` 参数 |
| `check_background` | `TaskOutput` 工具 (读取后台任务输出) |
| 通知队列 | `<task-notification>` 系统消息 |
| `drain_notifications` | 任务完成时自动通知当前会话 |
| `BG.tasks` 注册表 | `/tasks` 命令列出所有后台任务 |

---

## 9. 试一试

```sh
cd learn-claude-code
python agents/s08_background_tasks.py
```

试试这些 prompt (英文 prompt 对 LLM 效果更好, 也可以用中文):

1. **基础后台任务**: `Run "sleep 5 && echo done" in the background, then create a file while it runs`
   - 观察: background_run 立即返回，write_file 不等 sleep
   - 5 秒后再问一个问题，看通知是否注入

2. **多个并发任务**: `Start 3 background tasks: "sleep 2", "sleep 4", "sleep 6". Check their status.`
   - 观察: 三个任务同时启动
   - check_background 列出所有任务的状态

3. **阻塞 vs 后台的选择**: `Run pytest in the background and keep working on other things`
   - 观察: LLM 是否自主选择 background_run

4. **主动查询**: 启动一个后台任务后，手动输入 `Check the background tasks` 看 LLM 如何调用 check_background
