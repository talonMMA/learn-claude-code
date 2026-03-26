# s03: TodoWrite (待办写入) -- 详细讲解

`s01 > s02 > [ s03 ] s04 > s05 > s06 | s07 > s08 > s09 > s10 > s11 > s12`

> *"没有计划的 agent 走哪算哪"* -- 先列步骤再动手, 完成率翻倍。
>
> **Harness 层**: 规划 -- 让模型不偏航, 但不替它画航线。

这是系列的**第三课**。s01 搭建了 agent 的骨架（循环），s02 扩展了它的能力（多工具 + dispatch map），s03 要回答的核心问题是：**模型怎么在多步任务中不丢失进度？** 答案是给它一个可以自己读写的"待办清单"工具，加上系统级的"催更"提醒。

---

## 1. 问题：为什么需要 TodoWrite？

原文一句话：
> 多步任务中, 模型会丢失进度 -- 重复做过的事、跳步、跑偏。对话越长越严重: 工具结果不断填满上下文, 系统提示的影响力逐渐被稀释。一个 10 步重构可能做完 1-3 步就开始即兴发挥, 因为 4-10 步已经被挤出注意力了。

### 展开理解

想象你让 agent 做一个复杂任务："重构 hello.py：添加 type hints、添加 docstrings、添加 main guard、跑一下测试、修复测试失败"。这是一个 5 步任务。

**没有 TodoWrite 时会发生什么？**

1. 模型看到任务，开始做第 1 步：添加 type hints ✓
2. 工具返回结果，模型继续做第 2 步：添加 docstrings ✓
3. 工具返回结果，现在 messages 列表里已经有了很多内容（原始文件内容、修改后的内容、工具确认信息……）
4. 模型看到第 3 步的结果后，注意力被最近的工具输出占据，忘了还有 4-5 步没做
5. 模型直接回复："Done! I've added type hints and docstrings." -- **跳过了 main guard、测试、修复这三步**

**为什么会这样？**

这是 LLM 的注意力机制导致的固有缺陷：

- **近因效应**：模型对最近的消息赋予更高的注意力权重。当 messages 列表越来越长，最初的用户指令（"做 5 件事"）在注意力分布中的占比越来越低
- **上下文稀释**：每次工具调用都会往 messages 里追加大量内容（代码、输出、确认信息）。5 步任务做到第 3 步时，messages 可能已经有几千 token，原始的 5 步计划被"淹没"了
- **system prompt 被稀释**：虽然 system prompt 每次都发送，但随着 messages 变长，system prompt 的相对影响力下降

**TodoWrite 怎么解决这个问题？**

核心思路是给模型一个**可外化的、结构化的进度跟踪工具**：

1. 模型开始任务前，先用 `todo` 工具创建一份计划清单
2. 每完成一步，更新清单状态（`in_progress` → `completed`）
3. 清单作为 tool_result 回传，**每次调用 todo 工具时模型都能看到完整的进度状态**
4. 如果模型连续 3 轮没有更新清单，harness 自动注入 `<reminder>` 提醒

这相当于给模型配了一个"外部记事本"。人类做复杂任务时也一样 -- 你不会把 10 步计划都记在脑子里，你会写在纸上，做完一步划掉一步。TodoWrite 就是模型的那张纸。

---

## 2. 解决方案的架构

### ASCII 架构图

```
+--------+      +-------+      +---------+
|  User  | ---> |  LLM  | ---> | Tools   |
| prompt |      |       |      | + todo  |
+--------+      +---+---+      +----+----+
                    ^                |
                    |   tool_result  |
                    +----------------+
                          |
              +-----------+-----------+
              | TodoManager state     |
              | [ ] task A            |
              | [>] task B  <- doing  |
              | [x] task C            |
              +-----------------------+
                          |
              if rounds_since_todo >= 3:
                inject <reminder> into tool_result
```

### 架构图逐元素解读

**对比 s02 的架构图：**

s02 的图：
```
User → LLM → Tool Dispatch (字典查找) → tool_result 反馈回 LLM
```

s03 的图：
```
User → LLM → Tool Dispatch (字典查找 + todo 工具) → tool_result 反馈回 LLM
                                                         ↑
                                                   TodoManager 状态
                                                         ↑
                                                   nag reminder 注入
```

s03 在 s02 的基础上增加了**两个新元素**：

**TodoManager state（待办管理器状态）**
- 一个 Python 类的实例，存储在内存中（不是数据库、不是文件）
- 维护一个待办项列表，每项有 `id`、`text`、`status` 三个字段
- `status` 有三种取值：`pending`（待办）、`in_progress`（进行中）、`completed`（已完成）
- **关键约束**：同一时间只能有一个任务处于 `in_progress` 状态
- 模型通过调用 `todo` 工具来读写这个状态

**nag reminder（催更提醒）**
- 一个计数器 `rounds_since_todo`，记录模型连续多少轮没有调用 `todo` 工具
- 当计数达到 3 轮时，harness 在 tool_result 中注入一段 `<reminder>Update your todos.</reminder>` 文本
- 这段文本会被模型看到，起到"催促"的效果
- 一旦模型调用了 `todo`，计数器重置为 0

### 设计哲学

1. **模型自己制定计划，harness 不替它决策** -- harness 只提供一个"记事本"工具和"催更"机制，具体列什么步骤、怎么分步，全由模型自己决定。这就是"让模型不偏航，但不替它画航线"的含义
2. **单任务聚焦** -- 同一时间只允许一个 `in_progress`，强制模型顺序执行而不是并行思考。这减少了"做着 A 又去想 B"导致两个都做不好的情况
3. **问责压力** -- nag reminder 不是"建议"，而是系统级的注入。模型在自然语言层面理解"被催促"的含义，会倾向于回应提醒去更新进度
4. **循环不变性** -- agent_loop 的骨架（while True → API → stop_reason → execute tools → append results）仍然不变，只是在 append results 之前增加了一个 reminder 注入步骤

---

## 3. 代码逐行详解

完整代码文件：`agents/s03_todo_write.py`，共 211 行。下面按功能分段逐一讲解，重点标注与 s02 的增量变化。

### 3.1 文件头和文档字符串（第 1-28 行）

```python
#!/usr/bin/env python3
# Harness: planning -- keeping the model on course without scripting the route.
"""
s03_todo_write.py - TodoWrite

The model tracks its own progress via a TodoManager. A nag reminder
forces it to keep updating when it forgets.

    +----------+      +-------+      +---------+
    |   User   | ---> |  LLM  | ---> | Tools   |
    |  prompt  |      |       |      | + todo  |
    +----------+      +---+---+      +----+----+
                          ^               |
                          |   tool_result |
                          +---------------+
                                |
                    +-----------+-----------+
                    | TodoManager state     |
                    | [ ] task A            |
                    | [>] task B <- doing   |
                    | [x] task C            |
                    +-----------------------+
                                |
                    if rounds_since_todo >= 3:
                      inject <reminder>

Key insight: "The agent can track its own progress -- and I can see it."
"""
```

**解读：**
- 注释从 s02 的 `# Harness: tool dispatch` 变成了 `# Harness: planning`，标明本课的聚焦点从"工具分发"转移到"规划"
- docstring 点明两个核心机制：TodoManager（进度跟踪）和 nag reminder（催更提醒）
- 架构图比 s02 多了底部的 TodoManager 状态框和 reminder 注入条件
- Key insight 一语双关："The agent can track its own progress -- **and I can see it**"。不仅模型能追踪进度，**用户在终端里也能看到进度**（因为 todo 工具的输出会被 print）。这是一个重要的可观测性（observability）特性

### 3.2 导入和环境配置（第 30-44 行）⬜ 从 s02 继承

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

**与 s02 完全一致，无任何变化。**

### 3.3 System Prompt（第 46-48 行）

```python
SYSTEM = f"""You are a coding agent at {WORKDIR}.
Use the todo tool to plan multi-step tasks. Mark in_progress before starting, completed when done.
Prefer tools over prose."""
```

**与 s02 的增量对比：**

| s02 | s03 |
|---|---|
| `You are a coding agent at {WORKDIR}. Use tools to solve tasks. Act, don't explain.` | `You are a coding agent at {WORKDIR}.\nUse the todo tool to plan multi-step tasks. Mark in_progress before starting, completed when done.\nPrefer tools over prose.` |

**关键变化：**

1. **从单行变成三行** -- 信息量增加了，因为需要教会模型如何使用 todo 工具
2. **`Use the todo tool to plan multi-step tasks`** -- 明确告诉模型：多步任务要用 todo 工具来规划。没有这句话，模型可能会忽略 todo 工具，直接动手
3. **`Mark in_progress before starting, completed when done`** -- 教模型使用状态机：开始做之前标记为 `in_progress`，做完后标记为 `completed`。这个工作流指导非常具体
4. **`Prefer tools over prose`** -- 从 s02 的 `Act, don't explain` 变成了更广义的"优先用工具而不是写文字"。含义类似但更通用

**为什么 system prompt 需要教模型用 todo？**
你可能觉得工具定义里的 `description` 已经解释了 todo 的用途，为什么 system prompt 还要重复？因为 `description` 只告诉模型"这个工具能做什么"，但不告诉模型"什么时候该用它"和"怎么用它"。system prompt 中的 `Mark in_progress before starting, completed when done` 是使用策略（policy），不是工具能力描述。

### 3.4 TodoManager 类（第 51-86 行）🆕

这是 s03 最核心的新增代码。逐段讲解。

#### 类定义和初始化（第 52-54 行）

```python
class TodoManager:
    def __init__(self):
        self.items = []
```

- `TodoManager` 是一个简单的 Python 类，不继承任何基类
- `self.items` 是一个列表，存储所有待办项。每项是一个字典：`{"id": "1", "text": "Add type hints", "status": "pending"}`
- 初始化时为空列表 -- 模型通过调用 `todo` 工具来填充它

#### update 方法 -- 核心写入逻辑（第 56-75 行）

```python
    def update(self, items: list) -> str:
        if len(items) > 20:
            raise ValueError("Max 20 todos allowed")
        validated = []
        in_progress_count = 0
        for i, item in enumerate(items):
            text = str(item.get("text", "")).strip()
            status = str(item.get("status", "pending")).lower()
            item_id = str(item.get("id", str(i + 1)))
            if not text:
                raise ValueError(f"Item {item_id}: text required")
            if status not in ("pending", "in_progress", "completed"):
                raise ValueError(f"Item {item_id}: invalid status '{status}'")
            if status == "in_progress":
                in_progress_count += 1
            validated.append({"id": item_id, "text": text, "status": status})
        if in_progress_count > 1:
            raise ValueError("Only one task can be in_progress at a time")
        self.items = validated
        return self.render()
```

**逐行解读：**

| 行 | 代码 | 作用 |
|---|---|---|
| 56 | `def update(self, items: list) -> str:` | 接收一个待办项列表，返回渲染后的文本。**注意**：这是"全量替换"而不是"增量更新" -- 每次调用都传入完整的待办列表 |
| 57-58 | `if len(items) > 20:` | **上限保护**：防止模型创建过多待办项撑爆上下文。20 是一个合理的上限 -- 如果你的任务需要 20 步以上，可能应该拆分成子任务 |
| 59-60 | `validated = []` / `in_progress_count = 0` | 准备两个临时变量：验证后的列表和 in_progress 计数器 |
| 61 | `for i, item in enumerate(items):` | 遍历每个待办项，`enumerate` 提供序号 i 用作默认 id |
| 62 | `text = str(item.get("text", "")).strip()` | 提取 text 字段，转字符串、去空白。`.get("text", "")` 在字段缺失时返回空字符串而不是 None |
| 63 | `status = str(item.get("status", "pending")).lower()` | 提取 status 字段，默认 `"pending"`，转小写（容错：模型可能生成 `"Pending"` 或 `"PENDING"`） |
| 64 | `item_id = str(item.get("id", str(i + 1)))` | 提取 id 字段，默认用序号（1-based）。保证每项都有 id |
| 65-66 | `if not text: raise ValueError(...)` | text 不能为空 -- 一个没有描述的待办项毫无意义 |
| 67-68 | `if status not in ("pending", "in_progress", "completed"):` | 状态必须是三个合法值之一。这是一个**状态机约束** |
| 69-70 | `if status == "in_progress": in_progress_count += 1` | 计数有多少个 in_progress |
| 71 | `validated.append({...})` | 验证通过，加入列表 |
| 72-73 | `if in_progress_count > 1: raise ValueError(...)` | **核心约束**：最多只能有一个 in_progress。这个检查在遍历完所有项之后进行，确保全局视角 |
| 74 | `self.items = validated` | **原子替换**：只有所有验证都通过后才更新 items。如果任何一项验证失败（抛出 ValueError），items 保持不变 |
| 75 | `return self.render()` | 更新成功后立即渲染并返回，模型能看到当前的完整状态 |

**为什么是"全量替换"而不是"增量更新"？**

你可能期望有 `add_item()`、`update_status()` 这样的增量操作 API。但 s03 选择了"每次传入完整列表"的设计。原因：

- **简单性**：只有一个 `update` 方法，工具的 schema 定义也更简单
- **一致性**：模型每次都看到完整的列表，不会出现"我以为还有 3 项但其实只有 2 项"的不一致
- **模型友好**：LLM 生成完整列表比生成"对第 3 项执行 update_status" 这样的精确操作更可靠。模型只需要把当前的理解全部写出来即可
- **原子性**：要么全部更新成功，要么全部不变。没有"更新了一半"的中间状态

**状态机约束的意义：**

```
三种合法状态：
  pending  ──→  in_progress  ──→  completed
   (待做)        (正在做)          (做完了)

约束：同一时间最多 1 个 in_progress
```

"只能有一个 in_progress" 强制模型**顺序聚焦**。想象如果允许多个 in_progress：模型可能同时"开始"了 3 个任务，然后在切换过程中丢失了上下文。单一 in_progress 意味着模型必须做完一件事（标记 completed）才能开始下一件（标记 in_progress）。

#### render 方法 -- 文本渲染（第 77-86 行）

```python
    def render(self) -> str:
        if not self.items:
            return "No todos."
        lines = []
        for item in self.items:
            marker = {"pending": "[ ]", "in_progress": "[>]", "completed": "[x]"}[item["status"]]
            lines.append(f"{marker} #{item['id']}: {item['text']}")
        done = sum(1 for t in self.items if t["status"] == "completed")
        lines.append(f"\n({done}/{len(self.items)} completed)")
        return "\n".join(lines)
```

**逐行解读：**

| 行 | 代码 | 作用 |
|---|---|---|
| 77 | `def render(self) -> str:` | 将当前状态渲染为人类可读的文本 |
| 78-79 | `if not self.items: return "No todos."` | 空列表时返回简短提示 |
| 81-82 | `marker = {...}[item["status"]]` | 用字典映射把状态转为视觉符号：`[ ]` 待做、`[>]` 正在做、`[x]` 已完成。这个字典查找的写法比 if/elif 更简洁 |
| 83 | `lines.append(f"{marker} #{item['id']}: {item['text']}")` | 格式化每一项，如 `[>] #2: Add docstrings` |
| 84 | `done = sum(1 for t in self.items if t["status"] == "completed")` | 统计已完成的数量。生成器表达式 + `sum` 是 Python 惯用的计数写法 |
| 85 | `lines.append(f"\n({done}/{len(self.items)} completed)")` | 添加进度统计，如 `(2/5 completed)` |
| 86 | `return "\n".join(lines)` | 拼接成多行字符串 |

**渲染输出示例：**

```
[ ] #1: Add type hints
[>] #2: Add docstrings
[x] #3: Add main guard

(1/3 completed)
```

**为什么要渲染成文本？**

因为 tool_result 的 content 是字符串。渲染成这种视觉化的格式有两个好处：
1. **模型能理解** -- `[x]` 表示完成、`[>]` 表示进行中，这种标记法对 LLM 来说非常直观
2. **用户在终端能看到** -- 因为 agent_loop 里有 `print(f"> {block.name}: {str(output)[:200]}")`，todo 的渲染结果会直接显示在终端里

### 3.5 TodoManager 实例化（第 89 行）🆕

```python
TODO = TodoManager()
```

**这行很短但很重要：**
- 创建一个**模块级的全局实例**
- 这个实例在整个 REPL 会话期间持续存在（不会每轮重建）
- 模型多次调用 `todo` 工具时，都是操作同一个 `TODO` 对象
- 这意味着待办状态是**有记忆的** -- 上一次调用设置的状态，下一次调用仍然存在

### 3.6 工具实现函数（第 92-138 行）⬜ 从 s02 继承

```python
# -- Tool implementations --
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

**与 s02 基本一致。** 微小差异：

| s02 | s03 |
|---|---|
| `run_read` 先 `read_text()` 再 `splitlines()` 分两行 | 合并为一行 `safe_path(path).read_text().splitlines()` |
| `run_write` 返回 `f"Wrote {len(content)} bytes to {path}"` | 返回 `f"Wrote {len(content)} bytes"`（省略了 path） |

这些是微小的风格差异，逻辑完全等价。

### 3.7 Dispatch Map -- 工具分发字典（第 141-147 行）

```python
TOOL_HANDLERS = {
    "bash":       lambda **kw: run_bash(kw["command"]),
    "read_file":  lambda **kw: run_read(kw["path"], kw.get("limit")),
    "write_file": lambda **kw: run_write(kw["path"], kw["content"]),
    "edit_file":  lambda **kw: run_edit(kw["path"], kw["old_text"], kw["new_text"]),
    "todo":       lambda **kw: TODO.update(kw["items"]),
}
```

**与 s02 的增量对比：**

| s02 | s03 |
|---|---|
| 4 个工具 | 5 个工具（+todo） |

新增的一行：

```python
"todo": lambda **kw: TODO.update(kw["items"]),
```

- 当模型调用 `todo` 工具时，lambda 从参数中提取 `items` 列表，传给全局 `TODO` 实例的 `update` 方法
- `update` 返回 `render()` 的结果（渲染后的文本），这个文本会作为 tool_result 回传给模型
- 模型因此能看到当前的完整待办状态

**这完美体现了 s02 建立的 dispatch map 模式的扩展性：加一个工具 = 加一行字典条目。**

### 3.8 工具定义列表 -- TOOLS（第 149-160 行）

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
    {"name": "todo", "description": "Update task list. Track progress on multi-step tasks.",
     "input_schema": {"type": "object", "properties": {"items": {"type": "array", "items": {"type": "object", "properties": {"id": {"type": "string"}, "text": {"type": "string"}, "status": {"type": "string", "enum": ["pending", "in_progress", "completed"]}}, "required": ["id", "text", "status"]}}}, "required": ["items"]}},
]
```

**与 s02 的增量：新增 todo 工具的 schema 定义。**

逐字段解读 todo 工具的 schema：

| 字段 | 值 | 说明 |
|---|---|---|
| `name` | `"todo"` | 工具名称 |
| `description` | `"Update task list. Track progress on multi-step tasks."` | 告诉模型这个工具的用途：更新任务列表、跟踪多步任务的进度 |
| `input_schema.type` | `"object"` | 输入是一个 JSON 对象 |
| `properties.items` | `"array"` | 主要参数 `items` 是一个数组 |
| `items.items.type` | `"object"` | 数组中每个元素是一个对象 |
| `items.items.properties.id` | `"string"` | 待办项的唯一标识符 |
| `items.items.properties.text` | `"string"` | 待办项的描述文本 |
| `items.items.properties.status` | `"string", enum: [...]` | 状态，限定三个枚举值 |
| `required` (item 级) | `["id", "text", "status"]` | 每项的三个字段都是必填的 |
| `required` (顶层) | `["items"]` | `items` 数组本身是必填的 |

**schema 中的 `enum` 是关键：**
```json
"enum": ["pending", "in_progress", "completed"]
```

这告诉模型 `status` 字段只能取这三个值之一。模型在生成工具调用时会遵守这个约束，避免生成无效状态（如 `"done"` 或 `"working"`）。

### 3.9 核心循环 -- agent_loop 函数（第 163-191 行）🆕 有重要变化

```python
# -- Agent loop with nag reminder injection --
def agent_loop(messages: list):
    rounds_since_todo = 0
    while True:
        # Nag reminder is injected below, alongside tool results
        response = client.messages.create(
            model=MODEL, system=SYSTEM, messages=messages,
            tools=TOOLS, max_tokens=8000,
        )
        messages.append({"role": "assistant", "content": response.content})
        if response.stop_reason != "tool_use":
            return
        results = []
        used_todo = False
        for block in response.content:
            if block.type == "tool_use":
                handler = TOOL_HANDLERS.get(block.name)
                try:
                    output = handler(**block.input) if handler else f"Unknown tool: {block.name}"
                except Exception as e:
                    output = f"Error: {e}"
                print(f"> {block.name}: {str(output)[:200]}")
                results.append({"type": "tool_result", "tool_use_id": block.id, "content": str(output)})
                if block.name == "todo":
                    used_todo = True
        rounds_since_todo = 0 if used_todo else rounds_since_todo + 1
        if rounds_since_todo >= 3:
            results.insert(0, {"type": "text", "text": "<reminder>Update your todos.</reminder>"})
        messages.append({"role": "user", "content": results})
```

**与 s02 的逐行对比：**

循环的整体骨架（while True → API → append assistant → check stop_reason → execute tools → append results）**仍然不变**。s03 的增量变化集中在工具执行部分：

| 变化 | s02 代码 | s03 代码 | 说明 |
|---|---|---|---|
| 计数器初始化 | 无 | `rounds_since_todo = 0` | 新增：跟踪连续未调用 todo 的轮次 |
| todo 标记 | 无 | `used_todo = False` | 新增：标记本轮是否调用了 todo |
| 异常处理 | 无 try/except | `try: ... except Exception as e:` | 新增：捕获工具执行异常 |
| todo 检测 | 无 | `if block.name == "todo": used_todo = True` | 新增：检测是否使用了 todo |
| 计数器更新 | 无 | `rounds_since_todo = 0 if used_todo else rounds_since_todo + 1` | 新增：更新计数器 |
| reminder 注入 | 无 | `if rounds_since_todo >= 3: results.insert(0, {...})` | 新增：注入催更提醒 |

**关键变化详解：**

#### 变化 1：rounds_since_todo 计数器（第 165 行）

```python
rounds_since_todo = 0
```

- 在 `agent_loop` 函数**内部**初始化（不是全局变量）
- 这意味着每次用户输入新内容、触发新的 agent_loop 调用时，计数器重新从 0 开始
- 设计决策：**不跨用户输入保持计数**。每次用户说话都是一个新的上下文，之前的催更状态不应该延续

#### 变化 2：异常处理（第 180-183 行）

```python
try:
    output = handler(**block.input) if handler else f"Unknown tool: {block.name}"
except Exception as e:
    output = f"Error: {e}"
```

**这是 s03 相对 s02 的一个重要改进。** s02 中没有 try/except，如果工具函数抛出异常（比如 TodoManager 的 `raise ValueError`），会导致整个 agent_loop 崩溃。

s03 加上了 try/except 因为 TodoManager 的 `update` 方法会主动 `raise ValueError`（上限检查、空 text、无效 status、多个 in_progress）。这些 ValueError 不应该让 agent 崩溃，而是应该作为错误信息返回给模型，让模型自己修正。

比如模型传了两个 `in_progress`：
1. `update` 抛出 `ValueError("Only one task can be in_progress at a time")`
2. try/except 捕获，output 变成 `"Error: Only one task can be in_progress at a time"`
3. 这个错误信息作为 tool_result 回传给模型
4. 模型看到后理解错误，下次调用时只标记一个 in_progress

**这是"错误即信息"原则的进一步体现。**

#### 变化 3：todo 使用检测（第 186-187 行）

```python
if block.name == "todo":
    used_todo = True
```

简单的布尔标记，放在工具执行循环内部。只要本轮有任何一个 ToolUseBlock 的 name 是 "todo"，就标记为 True。

#### 变化 4：计数器更新（第 188 行）

```python
rounds_since_todo = 0 if used_todo else rounds_since_todo + 1
```

三元表达式：
- 如果本轮用了 todo → 重置为 0
- 如果本轮没用 todo → 计数 +1

#### 变化 5：nag reminder 注入（第 189-190 行）

```python
if rounds_since_todo >= 3:
    results.insert(0, {"type": "text", "text": "<reminder>Update your todos.</reminder>"})
```

**这是 s03 最精妙的设计。** 逐点分析：

**为什么是 3 轮？**
- 太少（1-2 轮）会过于频繁，打断模型的正常工作流。模型可能在做一系列紧密相关的操作（如连续读取多个文件），不需要每步都更新 todo
- 太多（5+ 轮）则提醒太晚，模型可能已经偏航了
- 3 轮是一个平衡点：给模型足够的空间做事，但不至于遗忘太久

**`results.insert(0, ...)` 而不是 `results.append(...)`**
- 用 `insert(0, ...)` 把 reminder 插入到 results 列表的**最前面**
- 这意味着模型在看到本轮的 tool_results 之前，先看到 reminder
- 这种"先提醒，后给结果"的顺序更有效 -- 模型在处理工具结果的同时已经被提醒了

**`{"type": "text", ...}` 而不是 `{"type": "tool_result", ...}`**
- reminder 不是某个工具调用的结果，它是一条额外的文本消息
- 在 Anthropic API 中，`user` 消息的 `content` 可以是一个混合列表，包含 `tool_result` 和 `text` 类型的条目
- 把 `text` 和 `tool_result` 放在同一个 `user` 消息中，模型能在同一轮看到它们

**`<reminder>...</reminder>` 标签**
- 用 XML 风格的标签包裹提醒文本
- 模型对 XML 标签特别敏感（Claude 系列模型在训练中大量使用了 XML 格式的指令）
- `<reminder>` 标签让模型理解"这是一个系统级的提醒，不是用户的对话"

### 3.10 交互式 REPL（第 194-211 行）⬜ 从 s02 继承

```python
if __name__ == "__main__":
    history = []
    while True:
        try:
            query = input("\033[36ms03 >> \033[0m")
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

**与 s02 唯一的区别：** 提示符从 `s02 >> ` 变成了 `s03 >> `。其他完全一致。

---

## 4. 完整执行流程示例

用户输入：`Refactor the file hello.py: add type hints, docstrings, and a main guard`

### 第 1 轮 -- 模型制定计划

**messages 状态（发送前）：**
```
[
  { role: "user", content: "Refactor the file hello.py: add type hints, docstrings, and a main guard" }
]
```

**模型返回**：
```
response.content = [
  ToolUseBlock(id="toolu_01", name="todo", input={
    "items": [
      {"id": "1", "text": "Read hello.py to understand current code", "status": "in_progress"},
      {"id": "2", "text": "Add type hints to all functions", "status": "pending"},
      {"id": "3", "text": "Add docstrings to all functions", "status": "pending"},
      {"id": "4", "text": "Add if __name__ == '__main__' guard", "status": "pending"}
    ]
  }),
  ToolUseBlock(id="toolu_02", name="read_file", input={"path": "hello.py"})
]
response.stop_reason = "tool_use"
```

**执行操作**：
1. 追加 assistant 消息到 messages
2. 检查 stop_reason == "tool_use" → 继续
3. 执行 todo 工具：`TODO.update(items)` → 验证通过 → 渲染返回：
   ```
   [>] #1: Read hello.py to understand current code
   [ ] #2: Add type hints to all functions
   [ ] #3: Add docstrings to all functions
   [ ] #4: Add if __name__ == '__main__' guard

   (0/4 completed)
   ```
4. `used_todo = True` → `rounds_since_todo = 0`
5. 执行 read_file 工具：返回 hello.py 的内容
6. 追加 tool_results 到 messages

**终端输出**：
```
> todo: [>] #1: Read hello.py to understand current code
[ ] #2: Add type hints to all functions
...
> read_file: def greet(name):
    print(f"Hello, {name}!")
...
```

**观察**：模型**先制定了计划，再开始第一步**。第一步和 read_file 在同一轮完成（模型可以在一次回复中调用多个工具）。

### 第 2 轮 -- 执行第 1 步（添加 type hints）

**模型看到了 hello.py 的内容和当前的 todo 状态，开始修改：**

```
response.content = [
  ToolUseBlock(id="toolu_03", name="todo", input={
    "items": [
      {"id": "1", "text": "Read hello.py to understand current code", "status": "completed"},
      {"id": "2", "text": "Add type hints to all functions", "status": "in_progress"},
      {"id": "3", "text": "Add docstrings to all functions", "status": "pending"},
      {"id": "4", "text": "Add if __name__ == '__main__' guard", "status": "pending"}
    ]
  }),
  ToolUseBlock(id="toolu_04", name="edit_file", input={
    "path": "hello.py",
    "old_text": "def greet(name):",
    "new_text": "def greet(name: str) -> None:"
  })
]
response.stop_reason = "tool_use"
```

**执行操作**：
1. todo 工具：#1 变成 completed，#2 变成 in_progress
2. edit_file 工具：添加 type hints
3. `used_todo = True` → `rounds_since_todo = 0`

### 第 3 轮 -- 执行第 2 步（添加 docstrings）

```
response.content = [
  ToolUseBlock(id="toolu_05", name="todo", input={
    "items": [
      {"id": "1", "text": "Read hello.py to understand current code", "status": "completed"},
      {"id": "2", "text": "Add type hints to all functions", "status": "completed"},
      {"id": "3", "text": "Add docstrings to all functions", "status": "in_progress"},
      {"id": "4", "text": "Add if __name__ == '__main__' guard", "status": "pending"}
    ]
  }),
  ToolUseBlock(id="toolu_06", name="edit_file", input={...})
]
response.stop_reason = "tool_use"
```

**执行操作**：
1. todo 工具：#2 completed，#3 in_progress
2. edit_file：添加 docstrings
3. `rounds_since_todo = 0`

### 第 4 轮 -- 执行第 3 步（添加 main guard）

```
response.content = [
  ToolUseBlock(id="toolu_07", name="todo", input={
    "items": [
      {"id": "1", "text": "Read hello.py to understand current code", "status": "completed"},
      {"id": "2", "text": "Add type hints to all functions", "status": "completed"},
      {"id": "3", "text": "Add docstrings to all functions", "status": "completed"},
      {"id": "4", "text": "Add if __name__ == '__main__' guard", "status": "in_progress"}
    ]
  }),
  ToolUseBlock(id="toolu_08", name="edit_file", input={...})
]
response.stop_reason = "tool_use"
```

### 第 5 轮 -- 标记完成并回复

```
response.content = [
  ToolUseBlock(id="toolu_09", name="todo", input={
    "items": [
      {"id": "1", "text": "Read hello.py to understand current code", "status": "completed"},
      {"id": "2", "text": "Add type hints to all functions", "status": "completed"},
      {"id": "3", "text": "Add docstrings to all functions", "status": "completed"},
      {"id": "4", "text": "Add if __name__ == '__main__' guard", "status": "completed"}
    ]
  })
]
response.stop_reason = "tool_use"
```

**todo 工具返回**：
```
[x] #1: Read hello.py to understand current code
[x] #2: Add type hints to all functions
[x] #3: Add docstrings to all functions
[x] #4: Add if __name__ == '__main__' guard

(4/4 completed)
```

### 第 6 轮 -- 最终回复

```
response.content = [
  TextBlock("Done! I've refactored hello.py with all requested changes: ...")
]
response.stop_reason = "end_turn"
```

**循环退出。**

### nag reminder 触发场景（假设模型忘了更新 todo）

假如在第 2 轮之后，模型连续 3 轮只用 edit_file 和 read_file 而不更新 todo：

- 第 3 轮：`rounds_since_todo = 1`（没有催更）
- 第 4 轮：`rounds_since_todo = 2`（没有催更）
- 第 5 轮：`rounds_since_todo = 3` → **触发催更！**

模型在第 5 轮收到的 `user` 消息内容变成：
```json
[
  {"type": "text", "text": "<reminder>Update your todos.</reminder>"},
  {"type": "tool_result", "tool_use_id": "toolu_xx", "content": "Edited hello.py"},
  ...
]
```

模型看到 `<reminder>` 后，通常会在下一轮调用 `todo` 工具更新进度状态。

---

## 5. 消息列表的完整结构

循环结束后的 messages（以简化的 4 步任务为例，假设每轮都更新 todo）：

```
messages[0]:  { role: "user",      content: "Refactor the file hello.py..." }                          ← 用户原始输入
messages[1]:  { role: "assistant", content: [ToolUse(todo), ToolUse(read_file)] }                       ← 模型制定计划 + 读文件
messages[2]:  { role: "user",      content: [tool_result(todo, "[>]#1..."), tool_result(read_file)] }   ← todo 渲染 + 文件内容
messages[3]:  { role: "assistant", content: [ToolUse(todo), ToolUse(edit_file)] }                       ← 更新进度 + 编辑
messages[4]:  { role: "user",      content: [tool_result(todo, "[x]#1,[>]#2..."), tool_result(edit)] }  ← 新进度 + 编辑结果
messages[5]:  { role: "assistant", content: [ToolUse(todo), ToolUse(edit_file)] }                       ← 更新进度 + 编辑
messages[6]:  { role: "user",      content: [tool_result(todo, "[x]#1,[x]#2,[>]#3..."), tool_result] }  ← 新进度 + 编辑结果
messages[7]:  { role: "assistant", content: [ToolUse(todo), ToolUse(edit_file)] }                       ← 更新进度 + 编辑
messages[8]:  { role: "user",      content: [tool_result(todo, "[x]#1,...[>]#4"), tool_result] }        ← 新进度 + 编辑结果
messages[9]:  { role: "assistant", content: [ToolUse(todo)] }                                           ← 全部标记完成
messages[10]: { role: "user",      content: [tool_result(todo, "(4/4 completed)")] }                    ← 最终进度
messages[11]: { role: "assistant", content: [TextBlock("Done! ...")] }                                  ← 最终回答
```

**观察**：
- 每轮的 `user` 消息中都包含 todo 的渲染结果，模型**每次都能看到最新的进度状态**
- todo 的渲染内容从 `(0/4 completed)` 逐步变成 `(4/4 completed)`
- 模型在同一轮中可以同时调用 todo 和其他工具（如 edit_file），这是高效的做法
- 如果某轮的 `user` 消息中还包含 `<reminder>` 文本，它会作为 `{"type": "text", ...}` 出现在 tool_results 之前

---

## 6. 变更总结表

| 组件 | 之前 (s02) | 之后 (s03) |
|---|---|---|
| Tools | 4 (bash, read_file, write_file, edit_file) | 5 (+todo) |
| 规划 | 无 | 带状态的 TodoManager |
| Nag 注入 | 无 | 3 轮后注入 `<reminder>` |
| Agent loop | 简单分发 | + rounds_since_todo 计数器 + used_todo 标记 + reminder 注入 |
| System Prompt | 1 行 | 3 行（增加 todo 使用指导） |
| 异常处理 | 无 try/except | 新增 try/except 捕获工具执行异常 |
| 新增类 | 无 | `TodoManager`（~35 行） |
| 代码行数 | 150 行 | 211 行（+61 行） |

新增的 61 行主要是：TodoManager 类（~35 行）+ todo 工具 schema（~3 行）+ agent_loop 中的计数器/检测/注入逻辑（~10 行）+ system prompt 扩展（~2 行）+ try/except（~3 行）。**循环的骨架仍然不变。**

---

## 7. 关键设计原则

1. **外化记忆（Externalized Memory）** -- 模型的"记忆"（当前进度）不是存在模型的隐式注意力中，而是存在一个显式的、可读写的外部数据结构（TodoManager）中。每次调用 todo 工具都能看到完整状态，不受上下文窗口的稀释影响。

2. **状态机约束（State Machine Constraint）** -- `pending → in_progress → completed` 的状态机，加上"只能有一个 in_progress"的约束，在结构层面强制了顺序执行。harness 不需要理解任务内容，只需要验证状态转换规则。

3. **问责式催更（Accountability Nudge）** -- nag reminder 不是"命令"模型做什么，而是"提醒"模型该做什么。这种间接影响比直接命令更符合 LLM 的工作方式 -- 模型在理解提醒的语义后自主决定如何响应。

4. **全量替换优于增量更新** -- `update` 方法接收完整列表而非增量操作，简化了 API 设计和一致性保证。对 LLM 来说，生成完整列表比精确指定"更新第 N 项的 status"更自然、更不容易出错。

5. **工具即可观测性（Tool as Observability）** -- todo 工具的输出（渲染后的清单）不仅模型能看到，用户在终端里也能看到。这让用户无需猜测"agent 现在在做什么"，直接看进度条。

6. **循环不变性**（延续 s01/s02）-- agent_loop 的骨架（while True → API → stop_reason → tools → results）仍然不变。新增的逻辑（计数器、检测、注入）都是在"追加 results 之前"的可选步骤，不影响核心流程。

---

## 8. 试一试

```sh
cd learn-claude-code
python agents/s03_todo_write.py
```

试试这些 prompt（英文 prompt 对 LLM 效果更好, 也可以用中文）:

1. `Refactor the file hello.py: add type hints, docstrings, and a main guard`
2. `Create a Python package with __init__.py, utils.py, and tests/test_utils.py`
3. `Review all Python files and fix any style issues`

**观察重点：**
- 模型是否在开始工作前先调用了 `todo` 工具制定计划
- 终端中 `> todo: [>] #1: ...` 的输出，观察进度状态如何变化
- 注意 `in_progress` 的标记是否在每步开始前设置、完成后更新
- 如果模型连续做了很多步而没有更新 todo，观察 `<reminder>` 是否被触发
- 对比 s02 中做相同任务时的行为 -- s03 的模型是否更有条理、不跳步

**进阶实验：**
- 给一个 10 步以上的复杂任务，观察模型是否能全部完成而不遗漏
- 手动修改代码，把 `rounds_since_todo >= 3` 改成 `>= 1`，观察过于频繁的催更对模型行为的影响
- 在 `TodoManager.update` 中去掉 `in_progress_count > 1` 的检查，观察模型是否会同时标记多个 in_progress
