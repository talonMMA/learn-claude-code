# s07: Task System (任务系统) -- 详细讲解

`s01 > s02 > s03 > s04 > s05 > s06 | [ s07 ] s08 > s09 > s10 > s11 > s12`

> *"大目标要拆成小任务, 排好序, 记在磁盘上"* -- 文件持久化的任务图, 为多 agent 协作打基础。
>
> **Harness 层**: 持久化任务 -- 比任何一次对话都长命的目标。

这是系列的**第七课**，也是**第三阶段（持久化与后台）**的起点。s01 搭建了 agent 的骨架（循环），s02 扩展了能力（多工具 + dispatch map），s03 教会了它自我规划（TodoManager + nag reminder），s04 引入子智能体隔离上下文，s05 实现了按需技能加载，s06 用三层压缩让会话可以无限进行。s07 要回答的核心问题是：**当内存中的待办清单（s03 的 TodoManager）在上下文压缩后就消失了，我们怎么让任务状态比对话更长命？又怎么表达任务之间的先后依赖？** 答案是：**把扁平清单升级为持久化到磁盘的任务图（DAG）**。

---

## 1. 问题：为什么需要任务系统？

原文一句话：
> s03 的 TodoManager 只是内存中的扁平清单: 没有顺序、没有依赖、状态只有做完没做完。真实目标是有结构的。

### 展开理解

回忆 s03 的 TodoManager 是怎么工作的：

```
messages[] 中的一条 user 消息:
{
  "role": "user",
  "content": "[TASKS]\n[ ] 设置数据库\n[ ] 写代码\n[ ] 写测试\n[!] 还有未完成的任务，请继续。"
}
```

TodoManager 把任务清单**注入到 messages 列表**中。这意味着：

**痛点 1：压缩即丢失**

```
场景: 你让 agent 做一个有 10 个步骤的大型重构
步骤 1-3: 顺利完成
s06 auto_compact 触发: messages[] 被压缩为摘要
步骤 4: agent 说"让我继续之前的工作"
问题: Todo 清单在 messages[] 里，压缩后没了

之前: messages 中有 [TASKS] 注入，agent 知道还有 7 个任务
之后: messages 只有一条摘要，agent 不知道还要做什么
```

**痛点 2：没有依赖关系**

```
用户: "帮我搭一个 REST API：先建 schema，然后写 model，最后写 endpoint"

TodoManager 的视角:
  [ ] 建 schema
  [ ] 写 model
  [ ] 写 endpoint

→ 三个任务是"平的"，agent 不知道它们之间有先后关系
→ 可能先去写 endpoint，但 schema 还没建好
→ 甚至在多 agent 场景下，两个 agent 可能同时拿同一个任务
```

**痛点 3：状态太简单**

```
TodoManager 的状态: 做了 / 没做（checkbox）
缺少的状态:
  - in_progress: 正在做（但还没完成）→ 防止别的 agent 重复认领
  - blocked: 被卡住了（等前置任务完成）→ 表达依赖
  - owner: 谁在做这个任务 → 为多 agent 协作做准备
```

### 核心矛盾

```
需求 1: 任务状态必须在上下文压缩后存活
需求 2: 任务之间有先后依赖关系需要表达
需求 3: 多个 agent 需要共享同一份任务状态（为 s09+ 做准备）

TodoManager 的局限:
  ✗ 存活于 messages[] → 压缩后丢失
  ✗ 扁平列表 → 无法表达依赖
  ✗ 单进程内存 → 无法跨 agent 共享
```

### 解决思路

把任务从"内存中的字符串"搬到"磁盘上的文件"：

```
TodoManager (s03)              TaskManager (s07)
─────────────────             ──────────────────
存储: messages[] 中            存储: .tasks/ 目录中的 JSON 文件
生命周期: 随对话存亡            生命周期: 比对话更长，比进程更长
结构: 扁平列表                 结构: 带 blockedBy/blocks 的有向无环图 (DAG)
状态: done / not done          状态: pending → in_progress → completed
并发: 单 agent                 并发: 多 agent 可读写同一目录
```

---

## 2. 解决方案的架构

### 任务图（DAG）

```
.tasks/
  task_1.json  {"id":1, "status":"completed"}
  task_2.json  {"id":2, "blockedBy":[1], "status":"pending"}
  task_3.json  {"id":3, "blockedBy":[1], "status":"pending"}
  task_4.json  {"id":4, "blockedBy":[2,3], "status":"pending"}

任务图 (DAG):
                 +----------+
            +--> | task 2   | --+
            |    | pending  |   |
+----------+     +----------+    +--> +----------+
| task 1   |                          | task 4   |
| completed| --> +----------+    +--> | blocked  |
+----------+     | task 3   | --+     +----------+
                 | pending  |
                 +----------+

顺序:   task 1 必须先完成, 才能开始 2 和 3
并行:   task 2 和 3 可以同时执行
依赖:   task 4 要等 2 和 3 都完成
状态:   pending -> in_progress -> completed
```

### 架构图逐元素解读

**`.tasks/` 目录**
- 每个任务一个 JSON 文件，文件名即身份：`task_1.json`、`task_2.json`
- 文件系统就是数据库——零依赖，人类可直接用文本编辑器查看/修改
- 多个进程可以读写同一目录，为后续多 agent 协作（s09+）打基础

**JSON 文件结构**

```json
{
  "id": 2,
  "subject": "Implement user model",
  "description": "Create User class with SQLAlchemy",
  "status": "pending",
  "blockedBy": [1],
  "blocks": [3, 4],
  "owner": ""
}
```

| 字段 | 类型 | 含义 |
|---|---|---|
| `id` | int | 全局唯一 ID，单调递增 |
| `subject` | string | 任务标题（一句话描述） |
| `description` | string | 详细描述（可选） |
| `status` | string | `pending` / `in_progress` / `completed` |
| `blockedBy` | int[] | 前置依赖：这些任务没完成前，本任务不应开始 |
| `blocks` | int[] | 后置依赖：本任务完成后，会解锁这些任务 |
| `owner` | string | 谁在做这个任务（为多 agent 预留） |

**状态机**

```
         create
           |
           v
      +---------+     start      +-------------+     done      +-----------+
      | pending | ------------> | in_progress | -----------> | completed |
      +---------+               +-------------+              +-----------+
           ^                                                       |
           |                                                       v
           |                                              _clear_dependency()
           |                                              从其他任务的 blockedBy
           +--- 被前置任务阻塞时停留在这里                    中移除自己的 ID
```

**依赖解除机制**

这是 s07 最关键的自动化逻辑：当一个任务标记为 `completed` 时，自动扫描所有其他任务，把自己的 ID 从它们的 `blockedBy` 列表中移除。

```
完成 task 1 前:
  task_2.json: {"blockedBy": [1]}   ← task 2 被 task 1 阻塞
  task_3.json: {"blockedBy": [1]}   ← task 3 被 task 1 阻塞

完成 task 1 后 (_clear_dependency 执行):
  task_2.json: {"blockedBy": []}    ← task 2 解锁，可以开始了
  task_3.json: {"blockedBy": []}    ← task 3 解锁，可以开始了
```

### 设计哲学

1. **文件即数据库**：不用 SQLite、不用 Redis，JSON 文件是最简单的零依赖持久化方案。人类可读、可编辑、可 git 版本控制
2. **声明式依赖**：agent 不需要一个"协调器"来告诉它做什么，只需读 `blockedBy` 字段——为空就可以做，有值就等着
3. **双向链接**：`blockedBy` 和 `blocks` 是同一条边的两个方向。`blockedBy` 用于判断"我能不能开始"，`blocks` 用于记录"我完成后谁会被解锁"
4. **状态比对话更长命**：任务文件在磁盘上，不管对话怎么压缩、进程怎么重启，任务板始终在那里

---

## 3. 代码逐行详解

### 3.1 文件头和导入

```python
#!/usr/bin/env python3
# Harness: persistent tasks -- goals that outlive any single conversation.
"""
s07_task_system.py - Tasks

Tasks persist as JSON files in .tasks/ so they survive context compression.
Each task has a dependency graph (blockedBy/blocks).

    .tasks/
      task_1.json  {"id":1, "subject":"...", "status":"completed", ...}
      task_2.json  {"id":2, "blockedBy":[1], "status":"pending", ...}
      task_3.json  {"id":3, "blockedBy":[2], "blocks":[], ...}

    Dependency resolution:
    +----------+     +----------+     +----------+
    | task 1   | --> | task 2   | --> | task 3   |
    | complete |     | blocked  |     | blocked  |
    +----------+     +----------+     +----------+
         |                ^
         +--- completing task 1 removes it from task 2's blockedBy

Key insight: "State that survives compression -- because it's outside the conversation."
"""
```

核心洞见：**"状态活在对话之外，所以能在压缩后存活"**。这一句话概括了 s07 相对于 s03 的根本转变——数据的生命周期从"随 messages 存亡"变为"随文件系统存亡"。

```python
import json
import os
import subprocess
from pathlib import Path

from anthropic import Anthropic
from dotenv import load_dotenv
```

与 s06 对比：
- **移除**了 `time` —— s06 用它给 transcript 文件名加时间戳，s07 不需要
- 其余导入完全一致
- 注意 s07 没有 s06 的压缩机制——这不是因为不需要，而是因为每课只聚焦一个主题。真实系统中 s06 和 s07 会共存

### 3.2 环境配置

```python
load_dotenv(override=True)

if os.getenv("ANTHROPIC_BASE_URL"):
    os.environ.pop("ANTHROPIC_AUTH_TOKEN", None)

WORKDIR = Path.cwd()
client = Anthropic(base_url=os.getenv("ANTHROPIC_BASE_URL"))
MODEL = os.environ["MODEL_ID"]
TASKS_DIR = WORKDIR / ".tasks"
```

前面的配置与 s06 完全一致。**新增**的是：

| 常量 | 值 | 含义 |
|---|---|---|
| `TASKS_DIR` | `WORKDIR / ".tasks"` | 任务文件的存储目录 |

**为什么用 `.tasks/`（带点号前缀）？**

- 点号前缀在 Unix 中表示"隐藏目录"，不会在 `ls` 中默认显示
- 这是工具/框架数据的惯例：`.git/`、`.vscode/`、`.transcripts/`（s06）
- 不污染用户项目的目录结构

### 3.3 System Prompt

```python
SYSTEM = f"You are a coding agent at {WORKDIR}. Use task tools to plan and track work."
```

与 s06 对比的变化：

```
s06: "You are a coding agent at {WORKDIR}. Use tools to solve tasks."
s07: "You are a coding agent at {WORKDIR}. Use task tools to plan and track work."
                                              ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
```

微妙但重要的变化：从"用工具解决任务"变为"用任务工具来**规划和追踪**工作"。这引导模型更主动地使用任务系统来组织多步骤工作。

### 3.4 TaskManager 类（新增核心）

这是 s07 的核心新增——完整的 TaskManager 类。

#### 3.4.1 初始化和辅助方法

```python
class TaskManager:
    def __init__(self, tasks_dir: Path):
        self.dir = tasks_dir
        self.dir.mkdir(exist_ok=True)
        self._next_id = self._max_id() + 1
```

- `mkdir(exist_ok=True)`：确保 `.tasks/` 目录存在，首次运行时自动创建
- `_next_id`：基于已有文件中的最大 ID + 1，保证 ID 不重复
- 即使进程重启，通过扫描磁盘上的文件也能恢复正确的 ID 序列

```python
    def _max_id(self) -> int:
        ids = [int(f.stem.split("_")[1]) for f in self.dir.glob("task_*.json")]
        return max(ids) if ids else 0
```

- `self.dir.glob("task_*.json")`：找到所有任务文件
- `f.stem.split("_")[1]`：从文件名 `task_3` 中提取数字 `3`
- 如果没有任何任务文件，返回 0（下一个 ID 就是 1）

**巧妙之处**：ID 恢复完全依赖文件名，不需要额外的元数据文件或数据库。

```python
    def _load(self, task_id: int) -> dict:
        path = self.dir / f"task_{task_id}.json"
        if not path.exists():
            raise ValueError(f"Task {task_id} not found")
        return json.loads(path.read_text())

    def _save(self, task: dict):
        path = self.dir / f"task_{task['id']}.json"
        path.write_text(json.dumps(task, indent=2))
```

- `_load`：从磁盘读取 → 反序列化为 dict
- `_save`：序列化为 JSON → 写入磁盘
- `indent=2`：人类可读格式，方便调试时直接查看文件内容

**注意**：每次 `_load` 都从磁盘重新读取，而不是缓存在内存中。这看似效率低，但有一个关键好处：如果另一个进程（另一个 agent）修改了同一个文件，下次读取就能看到最新状态。

#### 3.4.2 create 方法

```python
    def create(self, subject: str, description: str = "") -> str:
        task = {
            "id": self._next_id, "subject": subject, "description": description,
            "status": "pending", "blockedBy": [], "blocks": [], "owner": "",
        }
        self._save(task)
        self._next_id += 1
        return json.dumps(task, indent=2)
```

- 新任务默认 `status: "pending"`、`blockedBy: []`（无依赖）、`owner: ""`（无人认领）
- `_next_id` 在内存中递增，同时任务文件写入磁盘
- 返回 JSON 字符串给模型，让模型看到创建的结果

**为什么返回 JSON 字符串而不是 dict？** 因为返回值会作为 `tool_result` 的 content 送回 API，content 必须是字符串。

#### 3.4.3 update 方法（核心逻辑）

```python
    def update(self, task_id: int, status: str = None,
               add_blocked_by: list = None, add_blocks: list = None) -> str:
        task = self._load(task_id)
```

**参数设计**：所有参数都是可选的（除了 `task_id`），这让一次调用可以只改状态、只加依赖、或者两者都改。

```python
        if status:
            if status not in ("pending", "in_progress", "completed"):
                raise ValueError(f"Invalid status: {status}")
            task["status"] = status
            # When a task is completed, remove it from all other tasks' blockedBy
            if status == "completed":
                self._clear_dependency(task_id)
```

- 状态校验：只允许三个合法值
- **关键逻辑**：当状态变为 `completed` 时，自动触发 `_clear_dependency`——这就是"完成即解锁"的机制

```python
        if add_blocked_by:
            task["blockedBy"] = list(set(task["blockedBy"] + add_blocked_by))
        if add_blocks:
            task["blocks"] = list(set(task["blocks"] + add_blocks))
            # Bidirectional: also update the blocked tasks' blockedBy lists
            for blocked_id in add_blocks:
                try:
                    blocked = self._load(blocked_id)
                    if task_id not in blocked["blockedBy"]:
                        blocked["blockedBy"].append(task_id)
                        self._save(blocked)
                except ValueError:
                    pass
        self._save(task)
        return json.dumps(task, indent=2)
```

**双向链接维护**：

当你说"task 1 blocks task 2"时：
1. task 1 的 `blocks` 列表加入 2
2. task 2 的 `blockedBy` 列表加入 1

这两步是原子性地在同一个方法中完成的，保证了双向一致性。

`list(set(...))`：用 set 去重后转回 list，防止重复添加同一个依赖。

`except ValueError: pass`：如果被阻塞的任务不存在（可能还没创建），静默跳过而不是报错。这允许"先声明依赖，后创建任务"的灵活用法。

#### 3.4.4 _clear_dependency 方法

```python
    def _clear_dependency(self, completed_id: int):
        """Remove completed_id from all other tasks' blockedBy lists."""
        for f in self.dir.glob("task_*.json"):
            task = json.loads(f.read_text())
            if completed_id in task.get("blockedBy", []):
                task["blockedBy"].remove(completed_id)
                self._save(task)
```

这是任务系统的**核心自动化逻辑**：

1. 遍历 `.tasks/` 中的**所有**任务文件
2. 检查每个任务的 `blockedBy` 列表
3. 如果包含刚完成的任务 ID，将其移除
4. 保存修改后的任务

**效果示例**：

```
完成 task 1 之前:
  task_2.json: {"blockedBy": [1, 5]}  ← 被 task 1 和 task 5 阻塞
  task_3.json: {"blockedBy": [1]}     ← 只被 task 1 阻塞

_clear_dependency(1) 执行后:
  task_2.json: {"blockedBy": [5]}     ← 还被 task 5 阻塞，不能开始
  task_3.json: {"blockedBy": []}      ← 完全解锁！可以开始了
```

**设计选择**：扫描所有文件而不是只看 `blocks` 列表。这比只扫描 `task["blocks"]` 中列出的任务更安全——即使双向链接不完整，也能正确解锁。

#### 3.4.5 list_all 方法

```python
    def list_all(self) -> str:
        tasks = []
        for f in sorted(self.dir.glob("task_*.json")):
            tasks.append(json.loads(f.read_text()))
        if not tasks:
            return "No tasks."
        lines = []
        for t in tasks:
            marker = {"pending": "[ ]", "in_progress": "[>]", "completed": "[x]"}.get(t["status"], "[?]")
            blocked = f" (blocked by: {t['blockedBy']})" if t.get("blockedBy") else ""
            lines.append(f"{marker} #{t['id']}: {t['subject']}{blocked}")
        return "\n".join(lines)
```

- `sorted(...)`：按文件名排序，即按 ID 排序
- 状态标记映射：`[ ]` 待做、`[>]` 进行中、`[x]` 完成、`[?]` 未知状态
- 如果有 `blockedBy`，在行尾标注依赖关系

**输出示例**：

```
[x] #1: Setup database schema
[>] #2: Implement user model (blocked by: [])
[ ] #3: Add auth endpoints (blocked by: [2])
[ ] #4: Write deployment config (blocked by: [2, 3])
```

这个输出格式让模型一眼就能看清：什么完成了、什么在做、什么被卡住。

### 3.5 TaskManager 实例化

```python
TASKS = TaskManager(TASKS_DIR)
```

模块级别创建全局实例。在 `__init__` 中会自动创建 `.tasks/` 目录并恢复 ID 序列。

### 3.6 基础工具实现（从 s06 继承）

```python
def safe_path(p: str) -> Path:
    path = (WORKDIR / p).resolve()
    if not path.is_relative_to(WORKDIR):
        raise ValueError(f"Path escapes workspace: {p}")
    return path
```

路径安全校验，防止 `../../etc/passwd` 这样的路径逃逸。**从 s02 开始一直沿用**。

```python
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
```

Bash 执行函数，**从 s02 继承**。

```python
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
        c = fp.read_text()
        if old_text not in c:
            return f"Error: Text not found in {path}"
        fp.write_text(c.replace(old_text, new_text, 1))
        return f"Edited {path}"
    except Exception as e:
        return f"Error: {e}"
```

三个文件操作工具，**从 s02 开始一直沿用**。注意 s07 **没有** s06 的 `compact` 工具和三层压缩函数——每课聚焦一个主题。

### 3.7 Dispatch Map（新增任务工具）

```python
TOOL_HANDLERS = {
    "bash":        lambda **kw: run_bash(kw["command"]),
    "read_file":   lambda **kw: run_read(kw["path"], kw.get("limit")),
    "write_file":  lambda **kw: run_write(kw["path"], kw["content"]),
    "edit_file":   lambda **kw: run_edit(kw["path"], kw["old_text"], kw["new_text"]),
    "task_create": lambda **kw: TASKS.create(kw["subject"], kw.get("description", "")),
    "task_update": lambda **kw: TASKS.update(kw["task_id"], kw.get("status"), kw.get("addBlockedBy"), kw.get("addBlocks")),
    "task_list":   lambda **kw: TASKS.list_all(),
    "task_get":    lambda **kw: TASKS.get(kw["task_id"]),
}
```

相比 s06，dispatch map 的变化：

| 变化 | 说明 |
|---|---|
| **移除** `compact` | s06 的压缩工具在本课中不包含 |
| **新增** `task_create` | 创建新任务 |
| **新增** `task_update` | 更新任务状态/依赖 |
| **新增** `task_list` | 列出所有任务 |
| **新增** `task_get` | 获取单个任务详情 |

四个任务工具都是对 `TASKS`（TaskManager 实例）方法的薄包装。

### 3.8 工具定义（JSON Schema）

```python
TOOLS = [
    {"name": "bash", "description": "Run a shell command.",
     "input_schema": {"type": "object", "properties": {"command": {"type": "string"}}, "required": ["command"]}},
    {"name": "read_file", "description": "Read file contents.",
     "input_schema": {"type": "object", "properties": {"path": {"type": "string"}, "limit": {"type": "integer"}}, "required": ["path"]}},
    {"name": "write_file", "description": "Write content to file.",
     "input_schema": {"type": "object", "properties": {"path": {"type": "string"}, "content": {"type": "string"}}, "required": ["path", "content"]}},
    {"name": "edit_file", "description": "Replace exact text in file.",
     "input_schema": {"type": "object", "properties": {"path": {"type": "string"}, "old_text": {"type": "string"}, "new_text": {"type": "string"}}, "required": ["path", "old_text", "new_text"]}},
```

以上四个是继承的基础工具定义。下面是 **s07 新增**的四个：

```python
    {"name": "task_create", "description": "Create a new task.",
     "input_schema": {"type": "object", "properties": {"subject": {"type": "string"}, "description": {"type": "string"}}, "required": ["subject"]}},
```

- `subject` 必填，`description` 可选
- 简单的 schema——不在创建时指定依赖，依赖通过 `task_update` 后续添加

```python
    {"name": "task_update", "description": "Update a task's status or dependencies.",
     "input_schema": {"type": "object", "properties": {"task_id": {"type": "integer"}, "status": {"type": "string", "enum": ["pending", "in_progress", "completed"]}, "addBlockedBy": {"type": "array", "items": {"type": "integer"}}, "addBlocks": {"type": "array", "items": {"type": "integer"}}}, "required": ["task_id"]}},
```

- 只有 `task_id` 必填，其余都是可选的——灵活的增量更新
- `enum` 约束：模型只能传入三个合法状态值
- `addBlockedBy`/`addBlocks`：**增量**添加依赖，不是覆盖

```python
    {"name": "task_list", "description": "List all tasks with status summary.",
     "input_schema": {"type": "object", "properties": {}}},
```

- 无参数——列出所有任务

```python
    {"name": "task_get", "description": "Get full details of a task by ID.",
     "input_schema": {"type": "object", "properties": {"task_id": {"type": "integer"}}, "required": ["task_id"]}},
]
```

- 获取单个任务的完整 JSON，包括所有字段

### 3.9 Agent Loop（从 s06 继承）

```python
def agent_loop(messages: list):
    while True:
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

与 s06 对比的变化：**完全没有变化**（但移除了 s06 的 micro_compact 和 auto_compact 调用）。

核心循环从 s01 就定型了：
1. 调 API
2. 存 assistant 消息
3. 如果 `stop_reason != "tool_use"`，结束
4. 否则执行工具，把结果作为 user 消息存入
5. 回到步骤 1

**s07 的变化完全在工具层**——新增了 4 个工具，循环本身零修改。这体现了 s02 dispatch map 设计的威力：扩展功能只需要加工具，不需要改循环。

### 3.10 REPL 入口（从 s06 继承）

```python
if __name__ == "__main__":
    history = []
    while True:
        try:
            query = input("\033[36ms07 >> \033[0m")
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

唯一的视觉变化：提示符从 `s06 >>` 变成 `s07 >>`。逻辑完全一致。

---

## 4. 完整执行流程示例

用户输入：`Create 3 tasks: "Setup DB", "Write API", "Write Tests". Make Write API depend on Setup DB, and Write Tests depend on Write API.`

### 第 1 轮：模型创建三个任务

```
用户消息: "Create 3 tasks..."

API 响应 (stop_reason: "tool_use"):
  [tool_use: task_create(subject="Setup DB")]
  [tool_use: task_create(subject="Write API")]
  [tool_use: task_create(subject="Write Tests")]

Harness 执行:
  > task_create: 创建 .tasks/task_1.json  → {"id":1, "status":"pending", "blockedBy":[]}
  > task_create: 创建 .tasks/task_2.json  → {"id":2, "status":"pending", "blockedBy":[]}
  > task_create: 创建 .tasks/task_3.json  → {"id":3, "status":"pending", "blockedBy":[]}

messages[] 增长:
  [0] user: "Create 3 tasks..."
  [1] assistant: [tool_use × 3]
  [2] user: [tool_result × 3]
```

### 第 2 轮：模型设置依赖关系

```
API 响应 (stop_reason: "tool_use"):
  [tool_use: task_update(task_id=2, addBlockedBy=[1])]
  [tool_use: task_update(task_id=3, addBlockedBy=[2])]

Harness 执行:
  > task_update: task_2.json → {"blockedBy": [1]}
  > task_update: task_3.json → {"blockedBy": [2]}

  同时双向链接自动更新:
  task_1.json → {"blocks": [2]}  (因为 task 2 声明了 blockedBy=[1])
  task_2.json → {"blocks": [3]}  (因为 task 3 声明了 blockedBy=[2])

messages[] 增长:
  [3] assistant: [tool_use × 2]
  [4] user: [tool_result × 2]
```

注意：`addBlockedBy` 的双向更新在 `update` 方法中并没有实现（只有 `addBlocks` 会更新对方的 `blockedBy`）。所以上面的 task_1 blocks 不会自动填充。这是一个简化设计——模型如果需要完整的双向信息，可以分别调用。

### 第 3 轮：模型确认并列出任务

```
API 响应 (stop_reason: "tool_use"):
  [tool_use: task_list()]

Harness 执行:
  > task_list:
    [ ] #1: Setup DB
    [ ] #2: Write API (blocked by: [1])
    [ ] #3: Write Tests (blocked by: [2])

messages[] 增长:
  [5] assistant: [tool_use × 1]
  [6] user: [tool_result × 1]
```

### 第 4 轮：模型回复用户

```
API 响应 (stop_reason: "end_turn"):
  [text: "Created 3 tasks with dependencies:
   1. Setup DB (ready to start)
   2. Write API (depends on Setup DB)
   3. Write Tests (depends on Write API)"]

messages[] 最终状态:
  [7] assistant: [text]
```

### 后续：用户完成 task 1

```
用户: "Complete task 1"

模型调用: task_update(task_id=1, status="completed")

_clear_dependency(1) 执行:
  扫描所有任务文件
  task_2.json: blockedBy 从 [1] 变为 []  ← 自动解锁！
  task_3.json: blockedBy 仍为 [2]        ← 还被 task 2 阻塞

模型调用: task_list()
输出:
  [x] #1: Setup DB
  [ ] #2: Write API              ← 之前被阻塞，现在可以开始了
  [ ] #3: Write Tests (blocked by: [2])
```

---

## 5. 消息列表的完整结构

第 4 轮结束后的 messages[] 全貌：

```
messages[0] = {
  "role": "user",
  "content": "Create 3 tasks: \"Setup DB\", \"Write API\", \"Write Tests\". Make..."
}

messages[1] = {
  "role": "assistant",
  "content": [
    TextBlock(text="I'll create the tasks and set up dependencies."),
    ToolUseBlock(type="tool_use", id="toolu_01A", name="task_create",
                 input={"subject": "Setup DB"}),
    ToolUseBlock(type="tool_use", id="toolu_01B", name="task_create",
                 input={"subject": "Write API"}),
    ToolUseBlock(type="tool_use", id="toolu_01C", name="task_create",
                 input={"subject": "Write Tests"}),
  ]
}

messages[2] = {
  "role": "user",
  "content": [
    {"type": "tool_result", "tool_use_id": "toolu_01A",
     "content": '{"id": 1, "subject": "Setup DB", "status": "pending", ...}'},
    {"type": "tool_result", "tool_use_id": "toolu_01B",
     "content": '{"id": 2, "subject": "Write API", "status": "pending", ...}'},
    {"type": "tool_result", "tool_use_id": "toolu_01C",
     "content": '{"id": 3, "subject": "Write Tests", "status": "pending", ...}'},
  ]
}

messages[3] = {
  "role": "assistant",
  "content": [
    ToolUseBlock(type="tool_use", id="toolu_02A", name="task_update",
                 input={"task_id": 2, "addBlockedBy": [1]}),
    ToolUseBlock(type="tool_use", id="toolu_02B", name="task_update",
                 input={"task_id": 3, "addBlockedBy": [2]}),
  ]
}

messages[4] = {
  "role": "user",
  "content": [
    {"type": "tool_result", "tool_use_id": "toolu_02A",
     "content": '{"id": 2, "blockedBy": [1], ...}'},
    {"type": "tool_result", "tool_use_id": "toolu_02B",
     "content": '{"id": 3, "blockedBy": [2], ...}'},
  ]
}

messages[5] = {
  "role": "assistant",
  "content": [
    ToolUseBlock(type="tool_use", id="toolu_03A", name="task_list", input={}),
  ]
}

messages[6] = {
  "role": "user",
  "content": [
    {"type": "tool_result", "tool_use_id": "toolu_03A",
     "content": "[ ] #1: Setup DB\n[ ] #2: Write API (blocked by: [1])\n[ ] #3: Write Tests (blocked by: [2])"},
  ]
}

messages[7] = {
  "role": "assistant",
  "content": [
    TextBlock(text="Created 3 tasks with dependencies: ..."),
  ]
}
```

**磁盘上的状态**（与 messages 并行存在）：

```
.tasks/
  task_1.json: {"id":1, "subject":"Setup DB",     "status":"pending", "blockedBy":[], "blocks":[], ...}
  task_2.json: {"id":2, "subject":"Write API",    "status":"pending", "blockedBy":[1], "blocks":[], ...}
  task_3.json: {"id":3, "subject":"Write Tests",  "status":"pending", "blockedBy":[2], "blocks":[], ...}
```

关键观察：**即使 messages[] 被压缩清空，.tasks/ 目录中的文件不受影响**。这就是"状态活在对话之外"的意义。

---

## 6. 变更总结表

| 组件 | 之前 (s06) | 之后 (s07) |
|---|---|---|
| Tools | 5 (bash, read, write, edit, compact) | 8 (bash, read, write, edit + task_create/update/list/get) |
| 规划模型 | 扁平清单 (仅内存, s03 的 TodoManager) | 带依赖关系的任务图 (磁盘持久化) |
| 关系 | 无 | `blockedBy` + `blocks` 双向边 |
| 状态追踪 | 做完/没做完 (checkbox) | `pending` → `in_progress` → `completed` |
| 持久化 | 压缩后丢失 | 压缩和重启后存活 |
| 新增类 | 无 | `TaskManager` (~80 行) |
| 新增文件 | 无 | `.tasks/task_*.json` |
| 移除 | compact 工具 + 三层压缩 | （本课聚焦任务系统，压缩在真实系统中会共存） |
| 循环变化 | 无 | 无（只新增工具，循环零修改） |

---

## 7. 关键设计原则

### 原则 1：状态外置于对话

> 重要的状态不应该只活在 messages[] 里。

messages 是临时的——会被压缩、会被截断。任务文件在磁盘上，生命周期与进程、对话无关。这是从 s03（内存清单）到 s07（磁盘文件）的根本跃迁。

### 原则 2：文件即数据库

> 不需要数据库，文件系统就够了。

JSON 文件是最简单的零依赖持久化方案：人类可读、可 git 追踪、可跨进程共享。代价是没有 ACID 事务保证和并发安全——但对于教学和轻量场景已经足够。

### 原则 3：声明式依赖优于命令式编排

> 让每个任务声明自己的依赖，而不是让一个调度器决定执行顺序。

`blockedBy: [1, 3]` 是**声明式**的——"我需要 task 1 和 3 完成才能开始"。任何 agent 只要读到这个字段就知道该不该动这个任务。不需要中央调度器，每个 agent 可以独立决策。

### 原则 4：完成即解锁

> 标记完成不只是改自己的状态，还要解放后续任务。

`_clear_dependency` 是一个"副作用"：完成一个任务时，自动扫描并解锁所有等待它的任务。这让任务图像多米诺骨牌一样自动推进，而不需要人工干预。

### 原则 5：工具扩展不改循环

> 新增能力 = 新增工具 + dispatch map 条目，循环零修改。

从 s02 确立的 dispatch map 模式在此再次验证：agent_loop 函数从 s01 到 s07 几乎没变，所有新能力都通过工具层扩展。这是良好架构的标志——核心稳定，边缘灵活。

---

## 8. 与 s03 TodoManager 的对比

| 维度 | s03 TodoManager | s07 TaskManager |
|---|---|---|
| 存储位置 | messages[] 中（nag 注入） | .tasks/ 目录（JSON 文件） |
| 生命周期 | 随对话/压缩消亡 | 跨对话/跨进程持久 |
| 数据结构 | 扁平列表 | 有向无环图 (DAG) |
| 依赖关系 | 无 | blockedBy + blocks |
| 状态 | pending / completed | pending / in_progress / completed |
| 多 agent | 不支持 | 支持（共享文件目录） |
| 适用场景 | 单次会话的轻量清单 | 跨会话的复杂项目 |
| 与模型交互 | 注入到 system/user 消息 | 通过工具调用读写 |

**两者共存**：s03 适合"帮我改这三个 bug"这样的快速清单；s07 适合"帮我搭建整个项目的后端"这样的多步骤、有依赖的复杂任务。

---

## 9. 为后续课程奠基

s07 的任务图是后续所有机制的**协调骨架**：

```
s07 任务系统 (本课)
  │
  ├── s08 后台任务: 把 in_progress 的任务丢到后台线程执行
  │
  ├── s09 智能体团队: 多个 agent 读写同一个 .tasks/ 目录
  │                   通过 owner 字段分配任务
  │
  ├── s10 团队协议: 基于任务状态的请求-响应协议
  │
  ├── s11 自治智能体: 空闲时自动扫描 .tasks/ 找到可做的任务
  │
  └── s12 Worktree 隔离: 每个任务在独立的 git worktree 中执行
```

理解 s07 的任务文件结构和依赖机制，是理解 s08-s12 的前提。

---

## 10. 试一试

```sh
cd learn-claude-code
python agents/s07_task_system.py
```

### 推荐 prompt

1. **基础创建和依赖**
   ```
   Create 3 tasks: "Setup project", "Write code", "Write tests".
   Make them depend on each other in order.
   ```

2. **查看依赖图**
   ```
   List all tasks and show the dependency graph
   ```

3. **完成任务并观察解锁**
   ```
   Complete task 1 and then list tasks to see task 2 unblocked
   ```

4. **并行依赖场景**
   ```
   Create a task board for refactoring: parse -> transform -> emit -> test,
   where transform and emit can run in parallel after parse
   ```

### 动手实验建议

1. **手动查看任务文件**：运行完上面的 prompt 后，去 `.tasks/` 目录看看 JSON 文件的内容
2. **模拟进程重启**：创建几个任务，退出程序，重新启动，用 `task_list` 验证任务还在
3. **手动编辑任务**：用文本编辑器直接修改 `.tasks/task_1.json`，把 status 改成 completed，重新运行看看依赖是否正确解锁
4. **思考并发问题**：如果两个 agent 同时读取同一个任务文件并修改，会发生什么？（提示：后写入的会覆盖先写入的——这就是为什么注解中提到"持久化仍需要写入纪律"）
