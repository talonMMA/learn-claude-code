# s02: Tool Use (工具使用) -- 详细讲解

`s01 > [ s02 ] s03 > s04 > s05 > s06 | s07 > s08 > s09 > s10 > s11 > s12`

> *"加一个工具, 只加一个 handler"* -- 循环不用动, 新工具注册进 dispatch map 就行。
>
> **Harness 层**: 工具分发 -- 扩展模型能触达的边界。

这是系列的**第二课**。s01 搭建了 agent 的骨架（循环 + 一个 bash 工具），s02 要回答的核心问题是：**当模型需要更多能力时，怎么扩展？** 答案出奇简单 -- 循环一行不改，只增加工具定义和对应的处理函数。

---

## 1. 问题：为什么需要多工具 + 工具分发？

原文一句话：
> 只有 `bash` 时, 所有操作都走 shell。`cat` 截断不可预测, `sed` 遇到特殊字符就崩, 每次 bash 调用都是不受约束的安全面。专用工具 (`read_file`, `write_file`) 可以在工具层面做路径沙箱。

### 展开理解

想象你只有一把瑞士军刀（bash），什么都能做，但什么都不太安全：

**问题 1：可靠性差**
- 模型想读一个文件，用 `cat some_file.py`。但如果文件有几万行呢？`cat` 会一股脑全部输出，撑爆上下文窗口
- 模型想改文件里的一行代码，用 `sed -i 's/old/new/' file.py`。但如果 `old` 里包含 `/`、`&`、`*` 这些特殊字符呢？`sed` 直接报错或改错地方
- 模型想创建一个新文件，用 `echo "..." > file.py`。但如果内容里有引号嵌套呢？shell 的引号转义是噩梦

**问题 2：安全面太大**
- 每次 bash 调用，模型可以执行**任意 shell 命令**。虽然 s01 有个简单的危险命令黑名单，但那只是字符串匹配，很容易绕过
- 如果有专用的 `read_file` 工具，你可以在工具层面限制：只能读当前工作目录下的文件，路径不能包含 `..` 逃逸。这比 bash 层面的限制精确得多

**问题 3：代码可维护性**
- 在 s01 中，工具执行是硬编码的：`run_bash(block.input["command"])`。如果有 4 个工具，写 4 个 `if/elif` 分支？10 个工具呢？
- 更好的方式是用一个字典（dispatch map）：`{工具名: 处理函数}`。加新工具就是往字典里加一个键值对

**关键洞察：加工具不需要改循环。** 循环的逻辑是"调用 API → 检查 stop_reason → 执行工具 → 回传结果"。这个逻辑跟你有几个工具、工具是什么没关系。唯一的变化是"执行工具"这一步从直接调用 `run_bash` 变成了查字典找对应的处理函数。

---

## 2. 解决方案的架构

### ASCII 架构图

```
+--------+      +-------+      +------------------+
|  User  | ---> |  LLM  | ---> | Tool Dispatch    |
| prompt |      |       |      | {                |
+--------+      +---+---+      |   bash: run_bash |
                    ^           |   read: run_read |
                    |           |   write: run_wr  |
                    +-----------+   edit: run_edit |
                    tool_result | }                |
                                +------------------+
```

### 架构图逐元素解读

**对比 s01 的架构图：**

s01 的图：
```
User → LLM → Tool execute → tool_result 反馈回 LLM
```

s02 的图：
```
User → LLM → Tool Dispatch (字典查找) → tool_result 反馈回 LLM
```

唯一的结构变化是**右侧从一个单一的 "Tool execute" 变成了一个 "Tool Dispatch" 分发器**。

**Tool Dispatch（工具分发器）**
- 本质就是一个 Python 字典：`{"bash": run_bash, "read_file": run_read, ...}`
- 模型返回 `ToolUseBlock` 里的 `name` 字段作为字典的 key，查找对应的 handler 函数
- 这种设计叫做 **dispatch map（分发映射）** 或 **策略模式**

**四个工具**
| 工具 | 处理函数 | 作用 |
|---|---|---|
| `bash` | `run_bash` | 运行 shell 命令（从 s01 继承） |
| `read_file` | `run_read` | 读取文件内容，支持行数限制 |
| `write_file` | `run_write` | 创建/覆写文件 |
| `edit_file` | `run_edit` | 精确替换文件中的文本片段 |

**tool_result 反馈箭头** -- 和 s01 完全一样，不赘述。

### 设计哲学

1. **开放-封闭原则（Open-Closed Principle）** -- 对扩展开放（加工具只需加 handler + schema），对修改封闭（循环代码不需要改）
2. **每个工具有明确的边界** -- `read_file` 只能读文件，`write_file` 只能写文件，比"什么都能干"的 bash 更可控
3. **路径沙箱** -- 所有文件操作工具共享 `safe_path()` 函数，确保不能逃逸出工作目录
4. **循环不变性** -- s01 的核心循环 `while True → call API → check stop_reason → execute tools` 在 s02 中一字未改

---

## 3. 代码逐行详解

完整代码文件：`agents/s02_tool_use.py`，共 150 行。下面按功能分段逐一讲解，重点标注与 s01 的增量变化。

### 3.1 文件头和文档字符串（第 1-20 行）

```python
#!/usr/bin/env python3
# Harness: tool dispatch -- expanding what the model can reach.
"""
s02_tool_use.py - Tools

The agent loop from s01 didn't change. We just added tools to the array
and a dispatch map to route calls.

    +----------+      +-------+      +------------------+
    |   User   | ---> |  LLM  | ---> | Tool Dispatch    |
    |  prompt  |      |       |      | {                |
    +----------+      +---+---+      |   bash: run_bash |
                          ^          |   read: run_read |
                          |          |   write: run_wr  |
                          +----------+   edit: run_edit |
                          tool_result| }                |
                                     +------------------+

Key insight: "The loop didn't change at all. I just added tools."
"""
```

**解读：**
- 注释从 s01 的 `# Harness: the loop` 变成了 `# Harness: tool dispatch`，标明本课的聚焦点从"循环"转移到"工具分发"
- docstring 第一句就点明核心：**"The agent loop from s01 didn't change"**
- 架构图右侧从 s01 的单个 `Tool execute` 变成了带花括号的 dispatch map
- 最后一句 Key insight 是本课的精髓：循环没变，只是加了工具

### 3.2 导入和环境配置（第 22-36 行）

```python
import os
import subprocess
from pathlib import Path                          # ← 新增

from anthropic import Anthropic
from dotenv import load_dotenv

load_dotenv(override=True)

if os.getenv("ANTHROPIC_BASE_URL"):
    os.environ.pop("ANTHROPIC_AUTH_TOKEN", None)

WORKDIR = Path.cwd()                              # ← 新增
client = Anthropic(base_url=os.getenv("ANTHROPIC_BASE_URL"))
MODEL = os.environ["MODEL_ID"]
```

**与 s01 的增量对比：**

| 变化 | s01 | s02 | 为什么 |
|---|---|---|---|
| 新增 `from pathlib import Path` | 无 | 有 | 文件操作工具需要路径处理。`pathlib` 是 Python 3 的现代路径库，比 `os.path` 更优雅和安全 |
| `os.getcwd()` → `WORKDIR = Path.cwd()` | `os.getcwd()` 返回字符串 | `Path.cwd()` 返回 `Path` 对象 | `Path` 对象支持 `/` 运算符拼接路径、`.resolve()` 解析符号链接、`.is_relative_to()` 检查路径关系。这些是 `safe_path()` 所需要的 |

**其余代码（`load_dotenv`、API 客户端、MODEL）与 s01 完全一致。**

### 3.3 System Prompt（第 38 行）

```python
SYSTEM = f"You are a coding agent at {WORKDIR}. Use tools to solve tasks. Act, don't explain."
```

**与 s01 的增量对比：**

| s01 | s02 |
|---|---|
| `Use bash to solve tasks` | `Use tools to solve tasks` |

唯一的变化：`bash` 变成了 `tools`。因为现在模型有 4 个工具可用，不再只是 bash。这个微妙的措辞变化引导模型优先使用专用工具（`read_file`、`write_file`、`edit_file`）而不是万能的 bash。

### 3.4 路径沙箱函数 -- safe_path（第 41-45 行）🆕

```python
def safe_path(p: str) -> Path:
    path = (WORKDIR / p).resolve()
    if not path.is_relative_to(WORKDIR):
        raise ValueError(f"Path escapes workspace: {p}")
    return path
```

**这是 s02 完全新增的函数。** 逐行解读：

| 行 | 代码 | 作用 |
|---|---|---|
| 41 | `def safe_path(p: str) -> Path:` | 接收一个路径字符串，返回一个安全的 `Path` 对象 |
| 42 | `path = (WORKDIR / p).resolve()` | `WORKDIR / p` 是 pathlib 的路径拼接（如 `/home/user/project` / `src/main.py` = `/home/user/project/src/main.py`）。`.resolve()` 解析所有的 `..`、`.`、符号链接，得到真实的绝对路径 |
| 43 | `if not path.is_relative_to(WORKDIR):` | 检查解析后的路径是否仍然在 WORKDIR 之下。这是**关键的安全检查** |
| 44 | `raise ValueError(...)` | 如果路径逃逸了工作目录（比如 `../../etc/passwd`），直接抛异常 |
| 45 | `return path` | 路径安全，返回 |

**为什么需要 resolve() + is_relative_to() 两步？**

考虑这个攻击路径：
```
p = "../../etc/passwd"
WORKDIR / p = "/home/user/project/../../etc/passwd"
```
不调用 `.resolve()` 的话，路径看起来还在 `project/` 下面。但 `.resolve()` 会把 `..` 解析掉：
```
resolve() → "/etc/passwd"
is_relative_to("/home/user/project") → False → 拦截！
```

**这是一个经典的路径遍历（Path Traversal）防御模式。** 所有文件操作工具（read、write、edit）都通过 `safe_path()` 校验路径，形成统一的安全边界。

### 3.5 bash 工具函数 -- run_bash（第 48-58 行）⬜ 从 s01 继承

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

**与 s01 几乎完全一致。** 唯一的微小变化：

| s01 | s02 |
|---|---|
| `cwd=os.getcwd()` | `cwd=WORKDIR` |

`os.getcwd()` 和 `WORKDIR`（即 `Path.cwd()`）在这里语义等价，只是统一使用了 `WORKDIR` 变量。

### 3.6 读文件工具 -- run_read（第 61-69 行）🆕

```python
def run_read(path: str, limit: int = None) -> str:
    try:
        text = safe_path(path).read_text()
        lines = text.splitlines()
        if limit and limit < len(lines):
            lines = lines[:limit] + [f"... ({len(lines) - limit} more lines)"]
        return "\n".join(lines)[:50000]
    except Exception as e:
        return f"Error: {e}"
```

**逐行解读：**

| 行 | 代码 | 作用 |
|---|---|---|
| 61 | `def run_read(path: str, limit: int = None)` | 接收文件路径和可选的行数限制 |
| 63 | `text = safe_path(path).read_text()` | 先通过 `safe_path` 校验路径安全，然后读取文件全文 |
| 64 | `lines = text.splitlines()` | 按行拆分。`.splitlines()` 比 `.split('\n')` 更可靠（能处理 `\r\n` 等不同换行符） |
| 65-66 | `if limit and limit < len(lines):` | 如果设置了行数限制且文件超长，截断并添加提示 `"... (N more lines)"` |
| 67 | `return "\n".join(lines)[:50000]` | 重新拼接成字符串，截断到 50000 字符（和 s01 的 bash 一样的保护措施） |
| 68-69 | `except Exception as e:` | 捕获所有异常（文件不存在、权限不足、路径逃逸等），返回错误信息字符串 |

**为什么不直接用 bash 的 `cat`？**
- `cat` 没有行数限制功能（`head -n` 可以，但需要管道组合）
- `cat` 不会告诉你"还有多少行没显示"
- `cat` 没有路径沙箱保护
- `run_read` 的异常处理更优雅：返回错误字符串而不是让 shell 报错

### 3.7 写文件工具 -- run_write（第 72-79 行）🆕

```python
def run_write(path: str, content: str) -> str:
    try:
        fp = safe_path(path)
        fp.parent.mkdir(parents=True, exist_ok=True)
        fp.write_text(content)
        return f"Wrote {len(content)} bytes to {path}"
    except Exception as e:
        return f"Error: {e}"
```

**逐行解读：**

| 行 | 代码 | 作用 |
|---|---|---|
| 73 | `fp = safe_path(path)` | 路径安全校验 |
| 74 | `fp.parent.mkdir(parents=True, exist_ok=True)` | **自动创建父目录**。`parents=True` 递归创建（如 `a/b/c/`），`exist_ok=True` 目录已存在不报错。这意味着模型可以直接写 `src/utils/helper.py` 而不用先 `mkdir -p src/utils/` |
| 75 | `fp.write_text(content)` | 写入文件内容。如果文件已存在会**覆写** |
| 76 | `return f"Wrote {len(content)} bytes to {path}"` | 返回确认信息，包含写入的字节数，让模型知道操作成功了 |

**为什么不直接用 bash 的 `echo > file`？**
- `echo` 处理多行内容和特殊字符很麻烦（引号嵌套、`$` 变量展开等）
- `write_file` 接收纯字符串参数，没有 shell 转义问题
- 自动创建父目录（bash 需要先 `mkdir -p`）
- 路径沙箱保护

### 3.8 编辑文件工具 -- run_edit（第 82-91 行）🆕

```python
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

**逐行解读：**

| 行 | 代码 | 作用 |
|---|---|---|
| 84 | `fp = safe_path(path)` | 路径安全校验 |
| 85 | `content = fp.read_text()` | 读取文件全文 |
| 86-87 | `if old_text not in content:` | **先检查再替换**。如果要替换的文本不存在，返回错误而不是静默成功。这让模型知道它的 `old_text` 找错了，可以调整 |
| 88 | `content.replace(old_text, new_text, 1)` | 第三个参数 `1` 表示**只替换第一次出现**。这很重要 -- 如果文件中有多处相同的文本，只改第一处，避免意外改动其他地方 |
| 89 | `return f"Edited {path}"` | 返回确认信息 |

**为什么不直接用 bash 的 `sed`？**
- `sed` 使用正则表达式，特殊字符需要转义（`/`、`&`、`*`、`.`、`[` 等）
- 模型生成的替换内容经常包含这些字符，导致 `sed` 命令出错
- `run_edit` 做的是**精确字符串匹配**（`str.replace`），没有正则转义问题
- 路径沙箱保护
- 找不到目标文本时有明确的错误提示

**设计决策：`replace` 而不是正则**
这是一个刻意的简化。精确匹配意味着模型必须提供文件中**确切存在**的文本片段作为 `old_text`。这迫使模型先 `read_file` 看清楚文件内容，再做精确替换，减少了"猜测"带来的错误。

### 3.9 Dispatch Map -- 工具分发字典（第 94-100 行）🆕

```python
# -- The dispatch map: {tool_name: handler} --
TOOL_HANDLERS = {
    "bash":       lambda **kw: run_bash(kw["command"]),
    "read_file":  lambda **kw: run_read(kw["path"], kw.get("limit")),
    "write_file": lambda **kw: run_write(kw["path"], kw["content"]),
    "edit_file":  lambda **kw: run_edit(kw["path"], kw["old_text"], kw["new_text"]),
}
```

**这是 s02 最核心的新增概念。** 逐行解读：

| 工具名 | lambda 表达式 | 说明 |
|---|---|---|
| `"bash"` | `lambda **kw: run_bash(kw["command"])` | 从关键字参数中提取 `command`，传给 `run_bash` |
| `"read_file"` | `lambda **kw: run_read(kw["path"], kw.get("limit"))` | 提取 `path` 和可选的 `limit`。`kw.get("limit")` 在没有 `limit` 时返回 `None` |
| `"write_file"` | `lambda **kw: run_write(kw["path"], kw["content"])` | 提取 `path` 和 `content` |
| `"edit_file"` | `lambda **kw: run_edit(kw["path"], kw["old_text"], kw["new_text"])` | 提取三个参数 |

**为什么用 `lambda **kw` 而不是直接映射函数？**

模型返回的 `block.input` 是一个字典，比如 `{"path": "hello.py", "content": "print('hi')"}`。在循环中我们用 `handler(**block.input)` 调用，也就是把字典解包为关键字参数。

但每个工具的参数名和数量不同，直接映射函数会导致参数不匹配。lambda 层起到了**参数适配**的作用：从统一的 `**kw` 中提取出每个工具需要的具体参数。

**如何扩展？**

假设你想加一个 `grep` 工具：
```python
TOOL_HANDLERS["grep"] = lambda **kw: run_grep(kw["pattern"], kw.get("path", "."))
```
然后在 `TOOLS` 列表中加上对应的 schema 定义。就这么简单，循环代码不需要任何修改。

### 3.10 工具定义列表 -- TOOLS（第 102-111 行）

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
]
```

**与 s01 的增量对比：**
- s01：`TOOLS` 列表只有 1 个工具（bash）
- s02：`TOOLS` 列表有 4 个工具

**逐工具解读：**

| 工具 | description | properties | required | 说明 |
|---|---|---|---|---|
| `bash` | "Run a shell command." | `command: string` | `["command"]` | 与 s01 完全一致 |
| `read_file` | "Read file contents." | `path: string`, `limit: integer` | `["path"]` | `limit` 是可选参数（不在 required 中），模型可以不传 |
| `write_file` | "Write content to file." | `path: string`, `content: string` | `["path", "content"]` | 两个都是必填 |
| `edit_file` | "Replace exact text in file." | `path: string`, `old_text: string`, `new_text: string` | `["path", "old_text", "new_text"]` | 三个都是必填。description 中 "Replace exact text" 告诉模型这是精确匹配而非正则 |

**description 的重要性：**
模型根据 `description` 决定何时使用哪个工具。比如：
- 用户说"读取 config.py" → 模型看到 "Read file contents." → 选择 `read_file`
- 用户说"运行测试" → 模型看到 "Run a shell command." → 选择 `bash`
- 用户说"把第 5 行的 foo 改成 bar" → 模型看到 "Replace exact text in file." → 选择 `edit_file`

description 不需要很长，但必须**准确描述工具的能力**。

### 3.11 核心循环 -- agent_loop 函数（第 114-130 行）

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
                handler = TOOL_HANDLERS.get(block.name)                        # ← 变化
                output = handler(**block.input) if handler \                    # ← 变化
                    else f"Unknown tool: {block.name}"                         # ← 变化
                print(f"> {block.name}: {output[:200]}")                       # ← 变化
                results.append({"type": "tool_result", "tool_use_id": block.id, "content": output})
        messages.append({"role": "user", "content": results})
```

**与 s01 的逐行对比：**

循环的整体结构（while True → API 调用 → append assistant → 检查 stop_reason → 执行工具 → append tool_result）**完全一致**。

变化只在"执行工具"的 4 行：

| 行 | s01 代码 | s02 代码 | 变化说明 |
|---|---|---|---|
| 执行 | `run_bash(block.input["command"])` | `handler = TOOL_HANDLERS.get(block.name)` | 从硬编码变成字典查找 |
| 调用 | （直接调用） | `handler(**block.input) if handler else f"Unknown tool: {block.name}"` | 通用调用 + 未知工具兜底 |
| 打印 | `print(f"\033[33m$ {block.input['command']}\033[0m")` | `print(f"> {block.name}: {output[:200]}")` | 从打印 bash 命令变成打印工具名 + 输出 |
| 打印 | `print(output[:200])` | （合并到上一行） | — |

**关键变化详解：**

#### 字典查找替代硬编码
```python
# s01: 硬编码
output = run_bash(block.input["command"])

# s02: dispatch map
handler = TOOL_HANDLERS.get(block.name)
output = handler(**block.input) if handler else f"Unknown tool: {block.name}"
```

- `TOOL_HANDLERS.get(block.name)` 用 `.get()` 而不是 `[]`，当工具名不在字典里时返回 `None` 而不是抛 `KeyError`
- `handler(**block.input) if handler else ...` 是一个三元表达式：如果找到了 handler 就调用它，否则返回 "Unknown tool" 错误信息
- `**block.input` 把字典解包为关键字参数，比如 `{"path": "hello.py", "content": "..."}` 变成 `path="hello.py", content="..."`

#### 打印格式变化
```python
# s01: 只打印 bash 命令
print(f"\033[33m$ {block.input['command']}\033[0m")  # 黄色 $ command
print(output[:200])

# s02: 打印工具名 + 输出
print(f"> {block.name}: {output[:200]}")
```

s02 的打印格式是通用的（`> 工具名: 输出`），适用于所有工具，而不是只针对 bash。

### 3.12 交互式 REPL（第 133-149 行）⬜ 从 s01 继承

```python
if __name__ == "__main__":
    history = []
    while True:
        try:
            query = input("\033[36ms02 >> \033[0m")
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

**与 s01 唯一的区别：** 提示符从 `s01 >> ` 变成了 `s02 >> `。其他完全一致。

---

## 4. 完整执行流程示例

用户输入：`Create a file called greet.py with a greet(name) function, then add a docstring to it`

### 第 1 轮

**messages 状态（发送前）：**
```
[
  { role: "user", content: "Create a file called greet.py with a greet(name) function, then add a docstring to it" }
]
```

**API 调用**：发送 messages + tools（4 个工具）+ system

**模型返回**：
```
response.content = [
  ToolUseBlock(id="toolu_01", name="write_file", input={
    "path": "greet.py",
    "content": "def greet(name):\n    print(f\"Hello, {name}!\")\n"
  })
]
response.stop_reason = "tool_use"
```

**执行操作**：
1. 追加 assistant 消息到 messages
2. 检查 stop_reason == "tool_use" → 继续循环
3. 查找 handler：`TOOL_HANDLERS.get("write_file")` → 找到 lambda
4. 调用：`run_write("greet.py", "def greet(name):\n    print(f\"Hello, {name}!\")\n")`
5. `safe_path("greet.py")` → 校验通过
6. 写入文件，返回 `"Wrote 42 bytes to greet.py"`
7. 终端打印：`> write_file: Wrote 42 bytes to greet.py`
8. 追加 tool_result 到 messages

### 第 2 轮

**messages 状态（发送前）：**
```
[
  { role: "user",      content: "Create a file called greet.py ..." },
  { role: "assistant", content: [ToolUseBlock(id="toolu_01", name="write_file", ...)] },
  { role: "user",      content: [{ type: "tool_result", tool_use_id: "toolu_01", content: "Wrote 42 bytes to greet.py" }] }
]
```

**模型看到文件创建成功，现在需要读取文件来获取精确的文本内容以便编辑：**

```
response.content = [
  ToolUseBlock(id="toolu_02", name="read_file", input={"path": "greet.py"})
]
response.stop_reason = "tool_use"
```

**执行操作**：
1. 追加 assistant 消息
2. stop_reason == "tool_use" → 继续
3. 查找 handler：`TOOL_HANDLERS.get("read_file")` → 找到 lambda
4. 调用：`run_read("greet.py", None)`
5. 读取文件，返回文件内容
6. 终端打印：`> read_file: def greet(name):\n    print(f"Hello, {name}!")`
7. 追加 tool_result

### 第 3 轮

**模型现在有了文件的精确内容，可以做精确替换：**

```
response.content = [
  ToolUseBlock(id="toolu_03", name="edit_file", input={
    "path": "greet.py",
    "old_text": "def greet(name):",
    "new_text": "def greet(name):\n    \"\"\"Greet a person by name.\"\"\""
  })
]
response.stop_reason = "tool_use"
```

**执行操作**：
1. 追加 assistant 消息
2. stop_reason == "tool_use" → 继续
3. 查找 handler：`TOOL_HANDLERS.get("edit_file")` → 找到 lambda
4. 调用：`run_edit("greet.py", "def greet(name):", "def greet(name):\n    \"\"\"Greet a person by name.\"\"\"")`
5. `safe_path` 校验 → 读取文件 → `old_text in content` → 替换 → 写回
6. 返回 `"Edited greet.py"`
7. 追加 tool_result

### 第 4 轮

**模型验证编辑结果：**

```
response.content = [
  ToolUseBlock(id="toolu_04", name="read_file", input={"path": "greet.py"})
]
response.stop_reason = "tool_use"
```

执行后返回文件新内容。

### 第 5 轮

**模型确认一切正常，生成最终回复：**

```
response.content = [
  TextBlock("Done! I've created greet.py with a greet(name) function and added a docstring to it.")
]
response.stop_reason = "end_turn"
```

**循环退出。**

### 观察要点

在这个 5 轮示例中，模型使用了 **3 种不同的工具**：
1. `write_file` -- 创建文件
2. `read_file` -- 读取文件内容（为编辑做准备 + 验证结果）
3. `edit_file` -- 精确修改文件

注意模型的行为模式：**先写 → 读 → 改 → 读验证**。这和真实的编程工作流一致。特别是"先 read 再 edit"这个模式 -- 模型需要看到文件的精确内容，才能给 `edit_file` 提供正确的 `old_text` 参数。

如果只有 `bash` 工具（s01），模型可能会用 `echo > file` 创建、`cat` 读取、`sed` 修改。这些操作在简单情况下能工作，但遇到特殊字符、多行内容、大文件时就容易出问题。

---

## 5. 消息列表的完整结构

循环结束后的 messages（10 条消息）：

```
messages[0]:  { role: "user",      content: "Create a file called greet.py..." }              ← 用户原始输入
messages[1]:  { role: "assistant", content: [ToolUse("toolu_01", write_file)] }                ← 模型创建文件
messages[2]:  { role: "user",      content: [tool_result("toolu_01", "Wrote 42 bytes...")] }   ← 写入结果
messages[3]:  { role: "assistant", content: [ToolUse("toolu_02", read_file)] }                 ← 模型读取文件
messages[4]:  { role: "user",      content: [tool_result("toolu_02", "def greet...")] }        ← 文件内容
messages[5]:  { role: "assistant", content: [ToolUse("toolu_03", edit_file)] }                 ← 模型编辑文件
messages[6]:  { role: "user",      content: [tool_result("toolu_03", "Edited greet.py")] }     ← 编辑结果
messages[7]:  { role: "assistant", content: [ToolUse("toolu_04", read_file)] }                 ← 模型验证
messages[8]:  { role: "user",      content: [tool_result("toolu_04", "def greet...")] }        ← 验证内容
messages[9]:  { role: "assistant", content: [TextBlock("Done! ...")] }                         ← 最终回答
```

**观察**：
- 消息严格交替：user → assistant → user → assistant → ...
- 每条 tool_result 的 `tool_use_id` 都与对应 ToolUseBlock 的 `id` 配对
- 不同轮次使用了不同的工具（write_file → read_file → edit_file → read_file）
- 循环中工具的选择完全由模型决定，harness 只是忠实执行

---

## 6. 变更总结表

| 组件 | 之前 (s01) | 之后 (s02) |
|---|---|---|
| Tools | 1 (仅 bash) | 4 (bash, read_file, write_file, edit_file) |
| Dispatch | 硬编码 `run_bash()` 调用 | `TOOL_HANDLERS` 字典查找 |
| 路径安全 | 无 | `safe_path()` 沙箱 |
| 新增导入 | 无 | `from pathlib import Path` |
| WORKDIR | `os.getcwd()` 字符串 | `Path.cwd()` Path 对象 |
| System Prompt | "Use bash to solve tasks" | "Use tools to solve tasks" |
| Agent loop | `while True` + `stop_reason` | **不变** |
| REPL | 交互式输入 | **不变**（仅提示符 s01→s02） |
| 代码行数 | 108 行 | 150 行（+42 行） |

新增的 42 行全部是工具相关代码（safe_path + 3 个新工具函数 + dispatch map + 3 个新工具 schema）。循环和 REPL 的核心逻辑一行未改。

---

## 7. 关键设计原则

1. **Dispatch Map 模式** -- 用字典替代 if/elif 链实现工具路由。加工具 = 加一个字典条目 + 一个 schema 定义，循环代码零修改。这是经典的策略模式（Strategy Pattern）的轻量实现。

2. **路径沙箱（Path Sandbox）** -- `safe_path()` 函数通过 `resolve()` + `is_relative_to()` 两步验证，确保所有文件操作不能逃逸出工作目录。这是深度防御（Defense in Depth）的体现：即使模型被诱导生成了恶意路径，工具层也能拦截。

3. **专用工具优于通用工具** -- `read_file` 比 `cat` 更可靠（行数限制、截断提示），`edit_file` 比 `sed` 更安全（精确字符串匹配、无正则转义问题），`write_file` 比 `echo >` 更省心（自动创建目录、无 shell 转义）。每个专用工具都在其职责范围内做到最好。

4. **错误即信息，不是异常** -- 所有工具函数都 catch 异常并返回错误字符串（`f"Error: {e}"`），永不抛异常导致 agent 崩溃。模型看到错误信息后可以自行调整策略（比如换个路径、换个工具）。

5. **循环不变性**（延续 s01）-- 这是整个系列最重要的原则。s02 加了 3 个工具和一个 dispatch map，但 agent_loop 的骨架没有任何变化。后续课程（s03 加 TodoWrite、s04 加子 agent、s05 加 Skill……）都遵循这个原则。

6. **先读后改** -- `edit_file` 要求提供文件中**确切存在**的文本片段作为 `old_text`，这迫使模型养成"先 read_file 看清内容，再 edit_file 精确修改"的习惯。这比"盲改"（不看就用 sed 替换）可靠得多。

---

## 8. 试一试

```sh
cd learn-claude-code
python agents/s02_tool_use.py
```

试试这些 prompt（英文 prompt 对 LLM 效果更好, 也可以用中文）:

1. `Read the file requirements.txt`
2. `Create a file called greet.py with a greet(name) function`
3. `Edit greet.py to add a docstring to the function`
4. `Read greet.py to verify the edit worked`

**观察重点：**
- 注意模型选择了哪个工具（是用 `read_file` 还是 `bash cat`？）
- 注意 `edit_file` 之前模型是否先 `read_file`（先读后改模式）
- 注意终端输出 `> tool_name: output` 的格式
- 尝试让模型做一个复杂任务（如"创建一个 calculator.py 并添加 add、subtract 函数"），观察它如何组合使用多个工具

**进阶实验：**
- 尝试输入一个带 `../` 的路径（如 `Read the file ../etc/passwd`），观察 `safe_path` 的拦截效果
- 在 `TOOL_HANDLERS` 和 `TOOLS` 中添加一个新工具（如 `list_files`），验证"加工具不需要改循环"的原则
