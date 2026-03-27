# s04: Subagent (子智能体) -- 详细讲解

`s01 > s02 > s03 > [ s04 ] s05 > s06 | s07 > s08 > s09 > s10 > s11 > s12`

> *"大任务拆小, 每个小任务干净的上下文"* -- 子智能体用独立 messages[], 不污染主对话。
>
> **Harness 层**: 上下文隔离 -- 守护模型的思维清晰度。

这是系列的**第四课**。s01 搭建了 agent 的骨架（循环），s02 扩展了它的能力（多工具 + dispatch map），s03 教会了它自我规划（TodoManager + nag reminder）。s04 要回答的核心问题是：**当一个探索性的子任务需要大量工具调用时，怎么防止这些中间过程污染主对话的上下文？** 答案是启动一个拥有独立 `messages=[]` 的子智能体，让它完成工作后只把摘要文本返回给父智能体。

---

## 1. 问题：为什么需要子智能体？

原文一句话：
> 智能体工作越久, messages 数组越胖。每次读文件、跑命令的输出都永久留在上下文里。"这个项目用什么测试框架?" 可能要读 5 个文件, 但父智能体只需要一个词: "pytest。"

### 展开理解

想象你让 agent 做一个任务："先调查一下这个项目用什么测试框架，然后给 utils.py 写单元测试"。

**没有子智能体时会发生什么？**

1. 模型收到指令，开始调查测试框架
2. 读 `requirements.txt`（假设 100 行），输出追加到 messages ✓
3. 读 `setup.py`（假设 50 行），输出追加到 messages ✓
4. 读 `tox.ini`（假设 30 行），输出追加到 messages ✓
5. 读 `conftest.py`（假设 80 行），输出追加到 messages ✓
6. 读 `.github/workflows/test.yml`（假设 40 行），输出追加到 messages ✓
7. 模型终于确定："这个项目用 pytest"
8. 现在要开始写测试了，但 messages 里已经堆了 300+ 行的文件内容，**全是"调查阶段"的中间产物**
9. 模型的上下文窗口被调查结果占满，写测试时能利用的有效上下文变少了

**问题的本质是什么？**

这是**信息粒度不匹配**的问题：

- **探索阶段**需要大量细节（读文件、跑命令、看输出），这些细节必须留在上下文中，模型才能做出判断
- **使用结果**时只需要一个结论（"pytest"），之前的所有细节都成了"垃圾"
- 但 messages 列表是**只增不删**的——API 没有"删除某条消息"的接口，所有历史内容永久存在
- 随着对话越来越长，有用信息的密度越来越低，模型的"信噪比"不断恶化

这跟人类的工作方式很像。如果你让一个同事去调查一件事，你希望他回来告诉你"结论是 pytest"，而不是把他查看的每个文件内容都念给你听。**你要的是结论，不是过程。**

**与 s03 的 TodoManager 有什么区别？**

s03 解决的是"模型忘了自己的计划"——通过外化记忆防止跳步。s04 解决的是一个更根本的问题——**上下文污染**。即使模型没有忘记计划，它的上下文窗口也会被无关的中间结果填满，降低后续工作的质量。

TodoManager 不能解决这个问题。即使你用 todo 跟踪了"调查测试框架"这一步，读的 5 个文件内容仍然永久留在 messages 里。TodoManager 管"做什么"，子智能体管"在哪做"。

**子智能体怎么解决这个问题？**

核心思路是**进程隔离**（process isolation）：

1. 父智能体遇到探索性子任务时，调用 `task` 工具，传入一个 prompt（如 "调查这个项目的测试框架"）
2. Harness 启动一个子智能体，它拥有**全新的 `messages=[]`**，独立运行自己的 agent loop
3. 子智能体自由地读文件、跑命令，所有中间结果只存在于**它自己的 messages 列表**中
4. 子智能体完成后，只返回最终的文本摘要（如 "This project uses pytest for testing"）
5. 父智能体收到这段摘要作为 `tool_result`，**仅占几十个 token**
6. 子智能体的 messages 列表被完整丢弃，不留任何痕迹

这相当于给模型配了一个"临时工"。临时工去做脏活累活（读 5 个文件），做完后给你一份简报，然后走人。你的办公桌（上下文窗口）保持干净。

---

## 2. 解决方案的架构

### ASCII 架构图

```
Parent agent                     Subagent
+------------------+             +------------------+
| messages=[...]   |             | messages=[]      | <-- fresh
|                  |  dispatch   |                  |
| tool: task       | ----------> | while tool_use:  |
|   prompt="..."   |             |   call tools     |
|                  |  summary    |   append results |
|   result = "..." | <---------- | return last text |
+------------------+             +------------------+

Parent context stays clean. Subagent context is discarded.
```

### 架构图逐元素解读

**对比 s03 的架构图：**

s03 的图：
```
User → LLM → Tool Dispatch (字典查找 + todo 工具) → tool_result 反馈回 LLM
                                                         ↑
                                                   TodoManager 状态
                                                         ↑
                                                   nag reminder 注入
```

s04 的图：
```
User → Parent LLM → Tool Dispatch → tool_result 反馈回 Parent LLM
                         |
                   tool == "task" ?
                     ↓ Yes
               启动 Subagent LLM (独立 messages=[])
                     ↓
               Subagent 自己的 Tool Dispatch → 循环执行工具
                     ↓
               返回最终文本摘要 → 作为 tool_result 回到 Parent
```

s04 在之前的基础上增加了**一个全新的维度**：

**Parent agent（父智能体）**
- 拥有完整的 messages 历史，维护与用户的对话
- 拥有所有基础工具 + 一个额外的 `task` 工具
- `task` 工具的作用就是启动子智能体
- 收到子智能体的摘要后，把它当作普通的 `tool_result` 追加到自己的 messages 里

**Subagent（子智能体）**
- 以 `messages=[]` 启动——**完全空白的上下文**
- 只有基础工具（bash, read_file, write_file, edit_file），**没有 `task` 工具**——不能递归生成子智能体
- 运行自己独立的 agent loop：调用 API → 执行工具 → 追加结果 → 继续循环
- 当 `stop_reason != "tool_use"` 时停止，提取最终文本作为返回值
- 循环结束后，它的 messages 列表被**完整丢弃**

**关键箭头：dispatch 和 summary**
- `dispatch`：父智能体把用户的子任务描述传给子智能体，作为子智能体的第一条 user 消息
- `summary`：子智能体返回最终文本摘要，父智能体只收到这段文字

### 设计哲学

1. **进程隔离给上下文隔离（Process isolation gives context isolation for free）** -- 子智能体有自己独立的 `messages=[]`，这意味着它的所有中间工具调用结果不会出现在父智能体的上下文中。不需要复杂的"消息过滤"或"选择性遗忘"机制，直接用独立的消息列表实现了天然隔离

2. **共享文件系统，不共享对话历史** -- 父子智能体操作的是同一个文件系统（`WORKDIR`），所以子智能体的文件读写操作是真实生效的。但它们的 messages 列表完全独立。这就像两个同事共用一间办公室，但各自有自己的笔记本

3. **摘要即压缩** -- 子智能体可能执行了 30 次工具调用，产生了数万 token 的中间结果，但最终返回给父智能体的可能只有 50 个 token 的摘要。这是一种极端的信息压缩，压缩比可能高达 100:1

4. **禁止递归生成** -- 子智能体没有 `task` 工具，不能再启动子子智能体。这是一个重要的安全约束，防止无限递归消耗资源

5. **循环不变性**（延续 s01/s02/s03） -- 父智能体的 agent_loop 骨架仍然不变（while True → API → stop_reason → tools → results）。`task` 只是 dispatch map 中的又一个条目。子智能体内部也是同样的 agent loop 模式

---

## 3. 代码逐行详解

完整代码文件：`agents/s04_subagent.py`，共 185 行。下面按功能分段逐一讲解，重点标注与 s03 的增量变化。

### 3.1 文件头和文档字符串（第 1-24 行）

```python
#!/usr/bin/env python3
# Harness: context isolation -- protecting the model's clarity of thought.
"""
s04_subagent.py - Subagents

Spawn a child agent with fresh messages=[]. The child works in its own
context, sharing the filesystem, then returns only a summary to the parent.

    Parent agent                     Subagent
    +------------------+             +------------------+
    | messages=[...]   |             | messages=[]      |  <-- fresh
    |                  |  dispatch   |                  |
    | tool: task       | ---------->| while tool_use:  |
    |   prompt="..."   |            |   call tools     |
    |   description="" |            |   append results |
    |                  |  summary   |                  |
    |   result = "..." | <--------- | return last text |
    +------------------+             +------------------+
              |
    Parent context stays clean.
    Subagent context is discarded.

Key insight: "Process isolation gives context isolation for free."
"""
```

**解读：**
- 注释从 s03 的 `# Harness: planning` 变成了 `# Harness: context isolation`，标明本课的聚焦点从"规划"转移到"上下文隔离"
- docstring 的架构图是全新的——不再是单一循环中的状态管理，而是**双循环**（Parent 和 Subagent 各有自己的循环）
- Key insight 精确概括了核心设计："进程隔离天然带来上下文隔离"——不需要发明复杂的消息过滤机制，只需要用一个新的 `messages=[]` 列表

**与 s03 的对比：**

| s03 | s04 |
|---|---|
| `# Harness: planning` | `# Harness: context isolation` |
| 架构图：单循环 + TodoManager | 架构图：双循环 Parent ↔ Subagent |
| Key insight: "agent 能追踪进度——我也能看到" | Key insight: "进程隔离免费带来上下文隔离" |

### 3.2 导入和环境配置（第 26-40 行）⬜ 从 s03 继承

```python
import os
import subprocess
from pathlib import Path

from anthropic import Anthropic
from dotenv import load_dotenv

load_dotenv(override=True)

if os.getenv("ANTHROPIC_BASE_URL"):
    os.environ.pop("ANTHROPIC_AUTH_TOKEN", None)

WORKDIR = Path.cwd()
client = Anthropic(base_url=os.getenv("ANTHROPIC_BASE_URL"))
MODEL = os.environ["MODEL_ID"]
```

**与 s03 完全一致，无任何变化。**

### 3.3 System Prompt（第 42-43 行）🔄 有变化

```python
SYSTEM = f"You are a coding agent at {WORKDIR}. Use the task tool to delegate exploration or subtasks."
SUBAGENT_SYSTEM = f"You are a coding subagent at {WORKDIR}. Complete the given task, then summarize your findings."
```

**与 s03 的增量对比：**

| s03 | s04 |
|---|---|
| `SYSTEM` = 3 行，教模型用 todo 工具 | `SYSTEM` = 1 行，教模型用 task 工具委派子任务 |
| 无 `SUBAGENT_SYSTEM` | 🆕 `SUBAGENT_SYSTEM` = 1 行，告诉子智能体"完成任务后总结发现" |

**关键变化：**

1. **父智能体的 SYSTEM 变短了** -- s03 有 3 行来教模型用 todo，s04 只有 1 行。因为 `task` 工具的使用比 `todo` 简单——只需要传入一个 prompt，不需要管理状态机
2. **`Use the task tool to delegate exploration or subtasks`** -- 明确告诉模型两个适用场景："探索"（如调查项目结构）和"子任务"（如在隔离环境中执行操作）
3. **🆕 新增了 `SUBAGENT_SYSTEM`** -- 子智能体有自己独立的 system prompt！这是 s04 的重要设计：父和子有不同的"人格"和工作指令
4. **`Complete the given task, then summarize your findings`** -- 关键指令是 **summarize**。如果没有这个指令，子智能体可能只执行工具但不返回文本摘要，那父智能体就收不到有用的信息

**为什么不复用 s03 的 todo 相关 prompt？**

注意 s04 的代码里**没有 TodoManager**。这不是说 todo 不好用，而是 s04 聚焦于演示子智能体这一个新机制。在实际产品中，父智能体可以同时拥有 todo 和 task 工具（s_full.py 总纲就是这样做的）。每课只增加一个新概念，保持学习曲线平缓。

### 3.4 工具实现函数（第 46-100 行）⬜ 从 s03 继承

```python
# -- Tool implementations shared by parent and child --
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

def run_read(path: str, limit: int = None) -> str:
    try:
        lines = safe_path(path).read_text().splitlines()
        if limit and limit < len(lines):
            lines = lines[:limit] + [f"... ({len(lines) - limit} more)"]
        return "\n".join(lines)[:50000]
    except Exception as e:
        return f"Error: {e}"

def run_write(path: str, content: str) -> str:
    try:
        fp = safe_path(path)
        fp.parent.mkdir(parents=True, exist_ok=True)
        fp.write_text(content)
        return f"Wrote {len(content)} bytes"
    except Exception as e:
        return f"Error: {e}"

def run_edit(path: str, old_text: str, new_text: str) -> str:
    try:
        fp = safe_path(path)
        content = fp.read_text()
        if old_text not in content:
            return f"Error: Text not found in {path}"
        fp.write_text(content.replace(old_text, new_text, 1))
        return f"Edited {path}"
    except Exception as e:
        return f"Error: {e}"
```

**解读：**

注释 `# -- Tool implementations shared by parent and child --` 是 s04 的特别之处。在 s03 里这些函数没有这个注释，因为只有一个 agent 在使用它们。现在有了父和子两个智能体，注释明确标出：**这些工具函数是父子共享的**。

实现代码与 s03 完全一致。这体现了"共享文件系统"的设计——父子智能体调用的是同一套工具函数，操作同一个 `WORKDIR`。子智能体用 `write_file` 创建的文件，父智能体可以立即读到。

### 3.5 Dispatch Map（第 95-100 行）⬜ 从 s03 继承

```python
TOOL_HANDLERS = {
    "bash":       lambda **kw: run_bash(kw["command"]),
    "read_file":  lambda **kw: run_read(kw["path"], kw.get("limit")),
    "write_file": lambda **kw: run_write(kw["path"], kw["content"]),
    "edit_file":  lambda **kw: run_edit(kw["path"], kw["old_text"], kw["new_text"]),
}
```

**与 s03 的对比：** s03 有 5 个条目（包含 `todo`），s04 只有 4 个基础条目。`todo` 被移除了（本课不演示 TodoManager），`task` 工具没有放在 dispatch map 里（它在 agent_loop 中单独处理）。

### 3.6 工具定义（第 102-112 行）🔄 有变化

```python
# Child gets all base tools except task (no recursive spawning)
CHILD_TOOLS = [
    {"name": "bash", "description": "Run a shell command.",
     "input_schema": {"type": "object", "properties": {"command": {"type": "string"}}, "required": ["command"]}},
    {"name": "read_file", "description": "Read file contents.",
     "input_schema": {"type": "object", "properties": {"path": {"type": "string"}, "limit": {"type": "integer"}}, "required": ["path"]}},
    {"name": "write_file", "description": "Write content to file.",
     "input_schema": {"type": "object", "properties": {"path": {"type": "string"}, "content": {"type": "string"}}, "required": ["path", "content"]}},
    {"name": "edit_file", "description": "Replace exact text in file.",
     "input_schema": {"type": "object", "properties": {"path": {"type": "string"}, "old_text": {"type": "string"}, "new_text": {"type": "string"}}, "required": ["path", "old_text", "new_text"]}},
]
```

**关键设计：子智能体的工具列表**

注释 `# Child gets all base tools except task (no recursive spawning)` 揭示了核心约束：

- `CHILD_TOOLS` 包含 4 个基础工具：bash, read_file, write_file, edit_file
- **没有 `task` 工具** -- 子智能体不能启动自己的子智能体
- **没有 `todo` 工具** -- 本课不使用 TodoManager（简化演示）

**为什么叫 `CHILD_TOOLS` 而不直接叫 `TOOLS`？**

因为 s04 需要区分两套工具列表：子智能体的 `CHILD_TOOLS` 和父智能体的 `PARENT_TOOLS`。命名上用 CHILD/PARENT 前缀让关系一目了然。

### 3.7 🆕 子智能体函数 `run_subagent()`（第 115-134 行）

这是 s04 的**核心新增代码**。逐段讲解。

```python
# -- Subagent: fresh context, filtered tools, summary-only return --
def run_subagent(prompt: str) -> str:
    sub_messages = [{"role": "user", "content": prompt}]  # fresh context
```

**第 116-117 行：启动子智能体**

- `sub_messages = [{"role": "user", "content": prompt}]` -- 这是关键所在！**全新的 messages 列表**，只包含一条 user 消息（父智能体传来的子任务描述）
- 注释 `# fresh context` 强调：这个 messages 列表是干净的，没有任何历史包袱
- 父智能体的 messages 可能有几十条消息、数千 token，但子智能体从零开始

```python
    for _ in range(30):  # safety limit
        response = client.messages.create(
            model=MODEL, system=SUBAGENT_SYSTEM, messages=sub_messages,
            tools=CHILD_TOOLS, max_tokens=8000,
        )
```

**第 118-122 行：子智能体的 agent loop**

- `for _ in range(30)` -- 安全限制，最多循环 30 次。防止子智能体陷入无限循环（比如反复读同一个文件）
- `system=SUBAGENT_SYSTEM` -- 使用子智能体专用的 system prompt，**不是父智能体的 SYSTEM**
- `messages=sub_messages` -- 使用子智能体自己的 messages 列表
- `tools=CHILD_TOOLS` -- 使用子智能体的工具列表（没有 `task`）
- `max_tokens=8000` -- 与父智能体相同的 token 限制

**注意：** 这里用的是 `for` 循环而不是 `while True`。这是一个**有界循环**，最多 30 轮。父智能体的 `agent_loop` 用的是 `while True`（无界），因为父智能体的生命周期由用户控制。但子智能体是自动任务，必须有上界，否则一个失控的子任务可能永远跑不完。

```python
        sub_messages.append({"role": "assistant", "content": response.content})
        if response.stop_reason != "tool_use":
            break
```

**第 123-124 行：追加 + 退出判断**

- 与 s01 以来的标准模式一致：先追加 assistant 消息，再检查 stop_reason
- `stop_reason != "tool_use"` 时退出循环（通常是 `"end_turn"`，表示子智能体认为任务已完成）

```python
        results = []
        for block in response.content:
            if block.type == "tool_use":
                handler = TOOL_HANDLERS.get(block.name)
                output = handler(**block.input) if handler else f"Unknown tool: {block.name}"
                results.append({"type": "tool_result", "tool_use_id": block.id, "content": str(output)[:50000]})
        sub_messages.append({"role": "user", "content": results})
```

**第 125-132 行：工具执行**

- 标准的工具执行流程，与 s02/s03 的 agent_loop 中的逻辑几乎一模一样
- 从 `TOOL_HANDLERS` dispatch map 中查找处理函数
- 添加了 `f"Unknown tool: {block.name}"` 的容错处理（s03 中也有类似逻辑）
- 结果追加到 `sub_messages`（子智能体自己的 messages 列表）

**关键区别**：这些 tool_result 只会追加到 `sub_messages` 里，**永远不会出现在父智能体的 messages 中**。这就是上下文隔离的实现！

```python
    # Only the final text returns to the parent -- child context is discarded
    return "".join(b.text for b in response.content if hasattr(b, "text")) or "(no summary)"
```

**第 133-134 行：摘要提取和返回**

- `response.content` 是子智能体最后一次 API 回复的 content（循环退出时的那一轮）
- `b.text for b in response.content if hasattr(b, "text")` -- 从 response 中提取所有 TextBlock 的文本，拼接成一个字符串
- `or "(no summary)"` -- 容错处理：如果子智能体最后一轮没有返回任何文本（比如只调用了工具），返回一个占位符
- 注释 `# Only the final text returns to the parent -- child context is discarded` 点明：**只有这一行文本返回给父智能体，sub_messages 中积累的所有中间结果全部丢弃**

**这行代码是 s04 最精髓的一行。** 子智能体可能读了 10 个文件、执行了 20 次工具调用、在 sub_messages 中积累了数万 token，但最终返回的只是一段短文本。信息压缩在这里发生。

### 3.8 父智能体工具列表（第 137-141 行）🆕

```python
# -- Parent tools: base tools + task dispatcher --
PARENT_TOOLS = CHILD_TOOLS + [
    {"name": "task", "description": "Spawn a subagent with fresh context. It shares the filesystem but not conversation history.",
     "input_schema": {"type": "object", "properties": {"prompt": {"type": "string"}, "description": {"type": "string", "description": "Short description of the task"}}, "required": ["prompt"]}},
]
```

**解读：**

- `PARENT_TOOLS = CHILD_TOOLS + [...]` -- 父智能体的工具 = 子智能体的所有工具 + `task` 工具。这是一个**超集关系**
- `task` 工具有两个参数：
  - `prompt`（必填）：传给子智能体的完整任务描述
  - `description`（可选）：任务的简短描述，仅用于终端输出显示
- `description` 字段说明 `"Spawn a subagent with fresh context. It shares the filesystem but not conversation history."` -- 这句话精确告诉模型 task 工具的语义：**文件系统共享，对话历史不共享**

**工具列表的层级关系：**

```
CHILD_TOOLS  = [bash, read_file, write_file, edit_file]          ← 4 个
PARENT_TOOLS = [bash, read_file, write_file, edit_file, task]    ← 5 个
```

子智能体看到的工具列表里没有 `task`，所以它**不知道** `task` 的存在，自然也不会尝试调用它。这是通过**工具可见性**来实现的递归禁止，而不是运行时检查。

### 3.9 父智能体的 agent_loop（第 144-165 行）🔄 有变化

```python
def agent_loop(messages: list):
    while True:
        response = client.messages.create(
            model=MODEL, system=SYSTEM, messages=messages,
            tools=PARENT_TOOLS, max_tokens=8000,
        )
        messages.append({"role": "assistant", "content": response.content})
        if response.stop_reason != "tool_use":
            return
```

**第 144-152 行：标准循环骨架**

- 与 s01/s02/s03 完全一致的骨架：`while True` → API 调用 → 追加 assistant → 检查 stop_reason
- 使用 `PARENT_TOOLS`（比 CHILD_TOOLS 多了 `task`）
- 使用 `SYSTEM`（父智能体的 system prompt）

```python
        results = []
        for block in response.content:
            if block.type == "tool_use":
                if block.name == "task":
                    desc = block.input.get("description", "subtask")
                    print(f"> task ({desc}): {block.input['prompt'][:80]}")
                    output = run_subagent(block.input["prompt"])
                else:
                    handler = TOOL_HANDLERS.get(block.name)
                    output = handler(**block.input) if handler else f"Unknown tool: {block.name}"
                print(f"  {str(output)[:200]}")
                results.append({"type": "tool_result", "tool_use_id": block.id, "content": str(output)})
        messages.append({"role": "user", "content": results})
```

**第 153-165 行：工具分发——核心变化所在**

与 s03 的 agent_loop 对比，这里的关键变化是 **`task` 工具的特殊处理**：

```python
if block.name == "task":
    desc = block.input.get("description", "subtask")
    print(f"> task ({desc}): {block.input['prompt'][:80]}")
    output = run_subagent(block.input["prompt"])
```

- 当遇到 `task` 工具时，不走 `TOOL_HANDLERS` dispatch map，而是直接调用 `run_subagent()`
- `desc` 提取可选的 `description` 参数，用于终端显示
- `print(f"> task ({desc}): {block.input['prompt'][:80]}")` -- 终端输出子任务信息，只显示 prompt 的前 80 个字符（避免长 prompt 占满屏幕）
- `output = run_subagent(block.input["prompt"])` -- **这里发生了所有魔法**：启动子智能体，等待它完成，收到摘要

**为什么 `task` 不放在 `TOOL_HANDLERS` 里？**

因为 `TOOL_HANDLERS` 是父子共享的 dispatch map（子智能体也用它来执行工具）。如果把 `task` 的处理函数放在 `TOOL_HANDLERS` 里，虽然子智能体的工具列表中没有 `task`，但代码上会造成混淆。把 `task` 的处理逻辑直接写在 `agent_loop` 的 `if` 分支里，既清晰又安全。

**与 s03 的 agent_loop 对比：**

| 特性 | s03 | s04 |
|---|---|---|
| TodoManager | 有 rounds_since_todo 计数器 | 无（移除了 todo 机制） |
| nag reminder | 有 | 无 |
| task 分发 | 无 | 有 `if block.name == "task"` 分支 |
| 异常处理 | 有 try/except | 无（简化了） |
| 循环骨架 | while True → API → stop_reason | while True → API → stop_reason（不变） |

**注意 s04 移除了什么：** s04 没有 TodoManager、没有 nag reminder、没有 `try/except`。这不是退步，而是每课**聚焦于一个新概念**的教学设计。实际产品会把所有机制组合在一起（参见 s_full.py）。

### 3.10 交互式 REPL（第 168-184 行）⬜ 从 s03 继承

```python
if __name__ == "__main__":
    history = []
    while True:
        try:
            query = input("\033[36ms04 >> \033[0m")
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

**解读：**

- 提示符从 `\033[36ms03 >> \033[0m` 变成 `\033[36ms04 >> \033[0m`（仅版本号改变）
- `history = []` -- 这是**父智能体**的 messages 列表，在整个会话期间持续增长
- 每次用户输入后调用 `agent_loop(history)`，父智能体的所有历史都保留

**思考：** 注意 `history` 列表只增不减。但正因为有了子智能体机制，当模型选择用 `task` 来处理探索性子任务时，`history` 中不会出现子智能体的中间工具调用结果，只有摘要。这正是上下文隔离的价值。

---

## 4. 完整执行流程示例

用户输入：`Use a subtask to find what testing framework this project uses, then tell me`

### 第 1 轮 -- 父智能体决定委派子任务

**messages 状态（发送前）：**
```
[
  { role: "user", content: "Use a subtask to find what testing framework this project uses, then tell me" }
]
```

**父智能体返回**：
```
response.content = [
  ToolUseBlock(id="toolu_01", name="task", input={
    "prompt": "Investigate what testing framework this project uses. Check requirements.txt, setup.py, pyproject.toml, and any config files.",
    "description": "find testing framework"
  })
]
response.stop_reason = "tool_use"
```

**执行操作**：
1. 追加 assistant 消息到 messages
2. 检查 stop_reason == "tool_use" → 继续
3. 检测到 `block.name == "task"` → 调用 `run_subagent()`
4. 终端输出：`> task (find testing framework): Investigate what testing framework this project uses. Check re...`

### 子智能体开始工作（独立上下文）

子智能体的 `sub_messages` 初始化为：
```
[
  { role: "user", content: "Investigate what testing framework this project uses. Check requirements.txt, setup.py, pyproject.toml, and any config files." }
]
```

#### 子智能体第 1 轮 -- 读 requirements.txt

```
response.content = [
  ToolUseBlock(id="sub_01", name="read_file", input={"path": "requirements.txt"})
]
response.stop_reason = "tool_use"
```

工具返回：
```
anthropic>=0.42.0
python-dotenv>=1.0
```

sub_messages 变成 3 条消息。

#### 子智能体第 2 轮 -- 读 pyproject.toml

```
response.content = [
  ToolUseBlock(id="sub_02", name="bash", input={"command": "find . -name 'pytest.ini' -o -name 'conftest.py' -o -name 'pyproject.toml' 2>/dev/null"})
]
response.stop_reason = "tool_use"
```

工具返回文件路径列表。sub_messages 变成 5 条消息。

#### 子智能体第 3 轮 -- 检查 test 目录

```
response.content = [
  ToolUseBlock(id="sub_03", name="bash", input={"command": "ls tests/ 2>/dev/null || echo 'no tests dir'"})
]
response.stop_reason = "tool_use"
```

sub_messages 变成 7 条消息。

#### 子智能体第 4 轮 -- 返回摘要

```
response.content = [
  TextBlock("This project does not appear to have a specific testing framework configured. The requirements.txt only lists anthropic and python-dotenv. There are no pytest.ini, conftest.py, or test directories present. This is primarily a learning/demo project without test infrastructure.")
]
response.stop_reason = "end_turn"
```

**子智能体退出循环。** 返回值 = 上面这段文本。

**此时 sub_messages 有 8 条消息**（包含了 3 次工具调用的所有输入输出），但这些全部被丢弃。

### 回到父智能体

`run_subagent()` 返回摘要文本，父智能体收到 tool_result：

```python
results.append({
    "type": "tool_result",
    "tool_use_id": "toolu_01",
    "content": "This project does not appear to have a specific testing framework configured. The requirements.txt only lists anthropic and python-dotenv. There are no pytest.ini, conftest.py, or test directories present. This is primarily a learning/demo project without test infrastructure."
})
```

**终端输出**：
```
  This project does not appear to have a specific testing framework configured...
```

### 第 2 轮 -- 父智能体回复用户

**messages 状态（发送前）：**
```
[
  { role: "user", content: "Use a subtask to find what testing framework..." },
  { role: "assistant", content: [ToolUseBlock(task)] },
  { role: "user", content: [tool_result("This project does not appear to...")] }
]
```

**父智能体返回**：
```
response.content = [
  TextBlock("Based on the investigation, this project doesn't use a specific testing framework. It's a learning/demo project with only anthropic and python-dotenv as dependencies, and no test infrastructure.")
]
response.stop_reason = "end_turn"
```

**循环退出。**

### 对比：有无子智能体的 messages 差异

**没有子智能体**（假设父智能体自己做调查）：
```
messages[0]: { role: "user", content: "...find testing framework..." }
messages[1]: { role: "assistant", content: [ToolUse(read_file, "requirements.txt")] }
messages[2]: { role: "user", content: [tool_result("anthropic>=0.42.0\npython-dotenv...")] }     ← 文件内容
messages[3]: { role: "assistant", content: [ToolUse(bash, "find . -name ...")] }
messages[4]: { role: "user", content: [tool_result("./pyproject.toml\n...")] }                    ← 命令输出
messages[5]: { role: "assistant", content: [ToolUse(bash, "ls tests/...")] }
messages[6]: { role: "user", content: [tool_result("no tests dir")] }                             ← 命令输出
messages[7]: { role: "assistant", content: [TextBlock("Based on my investigation...")] }
→ 共 8 条消息，包含所有中间文件内容和命令输出
```

**有子智能体**：
```
messages[0]: { role: "user", content: "...find testing framework..." }
messages[1]: { role: "assistant", content: [ToolUse(task, prompt="Investigate...")] }
messages[2]: { role: "user", content: [tool_result("This project does not appear...")] }          ← 仅摘要！
messages[3]: { role: "assistant", content: [TextBlock("Based on the investigation...")] }
→ 共 4 条消息，中间过程全部被隔离
```

**消息数量从 8 条减少到 4 条。** 更重要的是，tool_result 中的内容从数百 token 的文件内容变成了几十 token 的摘要。如果后续还有其他任务要做，父智能体的上下文会更加干净。

---

## 5. 消息列表的完整结构

### 父智能体的 messages（循环结束后）

```
messages[0]:  { role: "user",      content: "Use a subtask to find what testing framework..." }             ← 用户输入
messages[1]:  { role: "assistant", content: [ToolUseBlock(name="task", prompt="Investigate...")] }          ← 模型委派子任务
messages[2]:  { role: "user",      content: [tool_result(id="toolu_01", "This project does not...")] }     ← 子智能体摘要
messages[3]:  { role: "assistant", content: [TextBlock("Based on the investigation...")] }                  ← 最终回答
```

### 子智能体的 sub_messages（循环结束后，即将被丢弃）

```
sub_messages[0]: { role: "user",      content: "Investigate what testing framework..." }                     ← 父智能体的 prompt
sub_messages[1]: { role: "assistant", content: [ToolUseBlock(read_file, "requirements.txt")] }               ← 子智能体决定读文件
sub_messages[2]: { role: "user",      content: [tool_result("anthropic>=0.42.0\npython-dotenv...")] }        ← 文件内容
sub_messages[3]: { role: "assistant", content: [ToolUseBlock(bash, "find . -name ...")] }                    ← 子智能体搜索配置文件
sub_messages[4]: { role: "user",      content: [tool_result("./pyproject.toml")] }                           ← 搜索结果
sub_messages[5]: { role: "assistant", content: [ToolUseBlock(bash, "ls tests/ ...")] }                       ← 子智能体检查 tests 目录
sub_messages[6]: { role: "user",      content: [tool_result("no tests dir")] }                               ← 目录不存在
sub_messages[7]: { role: "assistant", content: [TextBlock("This project does not appear...")] }              ← 摘要（返回给父智能体）
```

**观察**：
- 父智能体的 messages **只有 4 条**，其中 `messages[2]` 的 tool_result 内容是子智能体的摘要文本
- 子智能体的 sub_messages **有 8 条**，包含了 3 次工具调用的全部输入和输出
- 当 `run_subagent()` 返回后，`sub_messages` 作为局部变量被 Python 的垃圾回收机制自动清理
- 父智能体**看不到** sub_messages 中的任何内容，它只知道"子任务返回了一段摘要"
- 如果后续用户继续对话，父智能体的干净上下文让它能更高效地处理新任务

---

## 6. 变更总结表

| 组件 | 之前 (s03) | 之后 (s04) |
|---|---|---|
| Tools | 5 (bash, read_file, write_file, edit_file, todo) | 5 基础 (bash, read_file, write_file, edit_file) + task (仅父端) |
| 上下文 | 单一共享 | 父 + 子隔离 |
| Subagent | 无 | `run_subagent()` 函数 (~19 行) |
| 返回值 | 不适用 | 仅摘要文本 |
| TodoManager | 有（~35 行） | 无（本课移除） |
| Nag reminder | 有 | 无（本课移除） |
| System Prompt | 3 行（含 todo 使用指导） | 2 行（SYSTEM + SUBAGENT_SYSTEM） |
| 工具列表 | 1 套（TOOLS） | 2 套（CHILD_TOOLS + PARENT_TOOLS） |
| Dispatch map | 5 条目 | 4 条目（task 在 agent_loop 中特殊处理） |
| 安全限制 | 无循环限制 | 子智能体 30 轮上限 |
| 代码行数 | 211 行 | 185 行（-26 行） |

s04 的代码比 s03 **短了 26 行**，因为移除了 TodoManager（~35 行）和 nag reminder 逻辑，但新增了 `run_subagent()`（~19 行）和 `PARENT_TOOLS` 定义（~4 行）。**循环的骨架仍然不变。**

---

## 7. 关键设计原则

1. **进程隔离即上下文隔离（Process Isolation = Context Isolation）** -- 子智能体用独立的 `messages=[]` 运行，所有中间结果只存在于子智能体的上下文中。这是最简单、最可靠的隔离方式——不需要消息过滤、不需要选择性删除，只需要一个新的列表。

2. **共享执行环境，不共享思维过程（Shared Filesystem, Independent Minds）** -- 父子智能体操作同一个文件系统，子智能体的文件操作（写入、编辑）是真实生效的。但它们的对话历史完全独立，互不影响。这就像两个同事可以在同一台服务器上工作，但各自有自己的 terminal 窗口。

3. **摘要即压缩（Summary as Compression）** -- 子智能体可能执行了数十次工具调用，产生了数万 token 的中间结果，但返回给父智能体的只有几十 token 的摘要。这是一种高效的信息压缩机制，保留了结论，丢弃了过程。

4. **工具可见性控制递归（Tool Visibility Controls Recursion）** -- 子智能体的工具列表中没有 `task`，它**看不到**这个工具的存在，自然也不会尝试调用它。这比运行时检查（"如果子智能体调用了 task 就报错"）更优雅——问题从源头消除。

5. **有界子循环（Bounded Subloop）** -- 父智能体用 `while True`（生命周期由用户控制），子智能体用 `for _ in range(30)`（有明确上界）。这确保了即使子智能体出现异常行为（比如无限循环读同一个文件），也会在 30 轮后被强制停止。

6. **循环不变性**（延续 s01/s02/s03）-- agent_loop 的骨架（while True → API → stop_reason → tools → results）仍然不变。`task` 只是 dispatch 中的又一个分支，`run_subagent` 只是又一个工具处理函数。新的机制总是在不改变循环结构的前提下叠加进来。

---

## 8. 试一试

```sh
cd learn-claude-code
python agents/s04_subagent.py
```

试试这些 prompt（英文 prompt 对 LLM 效果更好, 也可以用中文）:

1. `Use a subtask to find what testing framework this project uses`
2. `Delegate: read all .py files and summarize what each one does`
3. `Use a task to create a new module, then verify it from here`

**观察重点：**
- 终端中 `> task (...)` 的输出，注意子任务的 prompt 描述
- 子智能体返回的摘要长度——它可能读了很多文件，但返回的只有几句话
- 父智能体收到摘要后的行为——它不需要自己再去读文件，直接基于摘要回答
- 对比直接问 agent 同一个问题（不要求用 subtask）和要求用 subtask 的区别：前者的 messages 会更长，后者更短

**进阶实验：**
- 试试让父智能体在一次对话中多次委派子任务，观察父智能体的 messages 是否保持精简
- 试试故意给子智能体一个无法完成的任务（如"读一个不存在的文件 abc.xyz"），观察子智能体如何处理错误并返回摘要
- 思考：如果允许子智能体也启动子子智能体，会发生什么？为什么代码中要禁止这种递归？
