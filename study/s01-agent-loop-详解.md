# s01: The Agent Loop (智能体循环) -- 详细讲解

`[ s01 ] s02 > s03 > s04 > s05 > s06 | s07 > s08 > s09 > s10 > s11 > s12`

> *"One loop & Bash is all you need"* -- 一个工具 + 一个循环 = 一个智能体。
>
> **Harness 层**: 循环 -- 模型与真实世界的第一道连接。

这是整个系列 12 课的**第一课**，也是最重要的一课。后续所有课程（工具分发、子智能体、上下文压缩、任务系统、团队协作……）都是在这个循环之上叠加机制。**循环本身从始至终不变。**

---

## 1. 问题：为什么需要 Agent Loop？

原文一句话：
> 语言模型能推理代码, 但碰不到真实世界 -- 不能读文件、跑测试、看报错。没有循环, 每次工具调用你都得手动把结果粘回去。你自己就是那个循环。

### 展开理解

想象一下你在 ChatGPT / Claude 网页版里工作的场景：

1. 你说："帮我看看当前目录有哪些文件"
2. 模型回答："我无法访问你的文件系统，但你可以运行 `ls` 命令……"
3. 你手动跑 `ls`，把结果粘回聊天框
4. 模型看到结果，说："你有 3 个 Python 文件，我建议修改 main.py……"
5. 你说："好的，帮我改"
6. 模型给出代码，你手动粘贴到文件里
7. 你跑一下测试，发现报错，再把报错粘回去……

**你自己就是那个循环** -- 你在模型和真实世界之间来回传递信息。这个过程：
- **慢**：每次都要手动复制粘贴
- **容易出错**：你可能粘错了、漏了一段输出
- **无法自动化**：你必须一直盯着

Agent Loop 要解决的核心问题就是：**把"你"从这个循环中替换掉，让程序自动完成"调用工具 → 拿到结果 → 喂回模型 → 再调用工具"的过程。**

关键洞察：模型自己知道什么时候该调用工具、什么时候该停止。它通过 `stop_reason` 这个字段告诉你："我还需要用工具"（`stop_reason == "tool_use"`）或者"我说完了"（`stop_reason == "end_turn"`）。所以你只需要一个 `while` 循环，检查这个字段就行了。

---

## 2. 解决方案的架构

### ASCII 架构图

```
+--------+      +-------+      +---------+
|  User  | ---> |  LLM  | ---> |  Tool   |
| prompt |      |       |      | execute |
+--------+      +---+---+      +----+----+
                    ^                |
                    |   tool_result  |
                    +----------------+
                    (loop until stop_reason != "tool_use")
```

### 架构图逐元素解读

**User prompt（用户输入）**
- 这是循环的起点。用户输入一条自然语言指令，比如"帮我创建一个 hello.py 文件"
- 它被封装成 `{"role": "user", "content": "..."}` 格式，追加到 `messages` 列表中

**LLM（大语言模型）**
- 接收完整的 `messages` 列表 + `tools` 定义，返回一个 `response`
- response 里包含两种可能的内容块：
  - `TextBlock`：模型的文字回答
  - `ToolUseBlock`：模型想要调用某个工具，包含工具名和参数
- response 还有一个关键字段 `stop_reason`，决定循环是否继续

**Tool execute（工具执行）**
- 当模型的 response 包含 `ToolUseBlock` 时，harness（你写的 Python 代码）负责实际执行这个工具
- 在 s01 中，唯一的工具就是 `bash`，即运行一条 shell 命令
- 执行结果被封装成 `tool_result` 格式，追加回 `messages` 列表

**tool_result 反馈箭头（循环的关键）**
- 工具的执行结果作为 `user` 角色的消息追加到 `messages`（这是 API 的要求：tool_result 必须在 user 消息中）
- 然后再次调用 LLM，让模型看到工具的输出
- 模型可能决定继续调用工具（比如第一个命令的输出显示还需要做更多事），也可能决定停止

**loop until stop_reason != "tool_use"（退出条件）**
- 这是整个架构的灵魂。只有一个判断条件：`stop_reason != "tool_use"`
- 当模型认为任务完成了，它会直接生成文本回答而不调用工具，此时 `stop_reason` 变成 `"end_turn"`，循环结束

### 设计哲学

1. **模型决定何时停止，不是代码决定** -- 代码不需要判断"任务完成了吗？"，它只看 `stop_reason`
2. **最小 harness 原则** -- 循环代码本身极简，所有"智能"都在模型里
3. **累积式上下文** -- `messages` 列表只增不减，每轮对话（assistant 回复 + tool 结果）都追加进去，模型能看到完整的历史

---

## 3. 代码逐行详解

完整代码文件：`agents/s01_agent_loop.py`，共 108 行。下面按功能分段逐一讲解。

### 3.1 文件头和文档字符串（第 1-25 行）

```python
#!/usr/bin/env python3
# Harness: the loop -- the model's first connection to the real world.
"""
s01_agent_loop.py - The Agent Loop

The entire secret of an AI coding agent in one pattern:

    while stop_reason == "tool_use":
        response = LLM(messages, tools)
        execute tools
        append results

    +----------+      +-------+      +---------+
    |   User   | ---> |  LLM  | ---> |  Tool   |
    |  prompt  |      |       |      | execute |
    +----------+      +---+---+      +----+----+
                          ^               |
                          |   tool_result |
                          +---------------+
                          (loop continues)

This is the core loop: feed tool results back to the model
until the model decides to stop. Production agents layer
policy, hooks, and lifecycle controls on top.
"""
```

**解读：**
- `#!/usr/bin/env python3`：shebang 行，让文件可以直接 `./s01_agent_loop.py` 运行
- 注释 `# Harness: the loop` 点明了这个文件在整个系列中的定位：harness（马具/框架）的循环部分
- docstring 里的伪代码 `while stop_reason == "tool_use"` 是整个 agent 的核心模式，后面的实际代码就是这个伪代码的展开
- ASCII 图和架构文档中的一致，方便对照

### 3.2 导入和环境配置（第 27-39 行）

```python
import os
import subprocess

from anthropic import Anthropic
from dotenv import load_dotenv

load_dotenv(override=True)

if os.getenv("ANTHROPIC_BASE_URL"):
    os.environ.pop("ANTHROPIC_AUTH_TOKEN", None)

client = Anthropic(base_url=os.getenv("ANTHROPIC_BASE_URL"))
MODEL = os.environ["MODEL_ID"]
```

**逐行解读：**

| 行 | 代码 | 作用 |
|---|---|---|
| 27 | `import os` | 读取环境变量、获取当前目录 |
| 28 | `import subprocess` | 执行 shell 命令的标准库 |
| 30 | `from anthropic import Anthropic` | 导入 Anthropic 官方 Python SDK 的客户端类 |
| 31 | `from dotenv import load_dotenv` | 从 `.env` 文件加载环境变量 |
| 33 | `load_dotenv(override=True)` | 读取 `.env` 文件，`override=True` 表示 `.env` 中的值会覆盖已有的同名环境变量。这样修改 `.env` 后不用重启终端 |
| 35-36 | `if os.getenv("ANTHROPIC_BASE_URL"): ...pop("ANTHROPIC_AUTH_TOKEN")` | **兼容性处理**：如果配置了自定义 API 地址（比如代理服务器），就移除可能冲突的 `ANTHROPIC_AUTH_TOKEN`。这是为了支持非官方 API 端点 |
| 38 | `client = Anthropic(base_url=...)` | 创建 API 客户端实例。`base_url` 可以是 `None`（使用官方默认地址）或自定义地址 |
| 39 | `MODEL = os.environ["MODEL_ID"]` | 从环境变量读取模型 ID。用 `os.environ[]` 而不是 `os.getenv()` 是因为这个变量**必须存在**，不存在时应该立即报错（KeyError），而不是静默返回 None |

**为什么这么设计：**
- 所有配置都通过环境变量，不硬编码。这样你可以轻松切换模型（Claude Sonnet → Opus）或 API 端点
- `.env` 文件不会被提交到 git（在 `.gitignore` 中），保护 API key 安全

### 3.3 System Prompt（第 41 行）

```python
SYSTEM = f"You are a coding agent at {os.getcwd()}. Use bash to solve tasks. Act, don't explain."
```

**解读：**

这一行虽然短，但信息量很大：

- **`You are a coding agent`**：告诉模型它的角色定位。角色定位会显著影响模型的行为模式
- **`at {os.getcwd()}`**：用 f-string 把当前工作目录注入到 system prompt 里。这样模型知道自己"在哪里"工作，跑 bash 命令时能给出正确的相对路径
- **`Use bash to solve tasks`**：明确告诉模型应该用 bash 工具来完成任务，而不是只给出文字建议
- **`Act, don't explain`**：这句是关键指令。没有这句话，模型倾向于解释"你可以这样做……"而不是直接动手。加上这句后，模型会直接调用 bash 工具执行命令

**为什么 system prompt 很重要：**
System prompt 是模型每次被调用时都会看到的"上帝视角"指令。它设定了整个对话的基调。在 agent 场景中，system prompt 的核心作用是让模型从"被动回答者"变成"主动执行者"。

### 3.4 工具定义（第 43-51 行）

```python
TOOLS = [{
    "name": "bash",
    "description": "Run a shell command.",
    "input_schema": {
        "type": "object",
        "properties": {"command": {"type": "string"}},
        "required": ["command"],
    },
}]
```

**逐字段解读：**

| 字段 | 值 | 说明 |
|---|---|---|
| `name` | `"bash"` | 工具名称。模型在 response 中会引用这个名字来调用工具 |
| `description` | `"Run a shell command."` | 工具描述。模型根据这个描述来决定什么时候该用这个工具。描述越清晰，模型使用越准确 |
| `input_schema` | JSON Schema 对象 | 定义工具的输入参数格式，遵循 JSON Schema 标准 |
| `type: "object"` | - | 输入必须是一个 JSON 对象 |
| `properties.command` | `{"type": "string"}` | 有一个 `command` 属性，类型是字符串 |
| `required: ["command"]` | - | `command` 是必填参数 |

**TOOLS 是一个列表**，在 s01 中只有一个工具 `bash`。在后续课程（s02 Tool Use）中，会往这个列表里加更多工具（如 `read_file`、`write_file`、`grep` 等），但循环代码本身不需要任何修改。

**这个工具定义会被发给 API**，模型看到后就知道它有一个叫 `bash` 的工具可以用，需要提供一个 `command` 字符串参数。当模型决定使用这个工具时，它会在 response 中生成一个 `ToolUseBlock`，里面包含 `name: "bash"` 和 `input: {"command": "ls -la"}` 这样的内容。

### 3.5 工具执行函数（第 54-64 行）

```python
def run_bash(command: str) -> str:
    dangerous = ["rm -rf /", "sudo", "shutdown", "reboot", "> /dev/"]
    if any(d in command for d in dangerous):
        return "Error: Dangerous command blocked"
    try:
        r = subprocess.run(command, shell=True, cwd=os.getcwd(),
                           capture_output=True, text=True, timeout=120)
        out = (r.stdout + r.stderr).strip()
        return out[:50000] if out else "(no output)"
    except subprocess.TimeoutExpired:
        return "Error: Timeout (120s)"
```

**逐行解读：**

| 行 | 代码 | 作用 |
|---|---|---|
| 54 | `def run_bash(command: str) -> str:` | 函数签名：接收一个命令字符串，返回执行结果字符串。**注意返回的是字符串而不是抛异常** -- 无论成功还是失败，都返回文本，让模型自己判断 |
| 55 | `dangerous = [...]` | 危险命令黑名单。这是一个**最基本的安全措施**，不是完美的（可以绕过），但能防止最明显的危险操作 |
| 56-57 | `if any(d in command for d in dangerous)` | 如果命令包含任何危险关键字，直接返回错误信息，不执行。`any()` + 生成器表达式是 Python 的惯用写法 |
| 59 | `subprocess.run(command, shell=True, ...)` | 执行 shell 命令。关键参数解释见下表 |
| 61 | `out = (r.stdout + r.stderr).strip()` | 合并标准输出和标准错误。**为什么要合并？** 因为模型需要看到完整的输出，包括错误信息。`.strip()` 去掉首尾空白 |
| 62 | `return out[:50000] if out else "(no output)"` | 截断到 50000 字符。防止一个命令输出太多内容撑爆 API 的上下文窗口。如果没有输出，返回 `"(no output)"` 而不是空字符串，让模型知道命令执行了但没有输出 |
| 63-64 | `except subprocess.TimeoutExpired` | 120 秒超时保护。防止模型运行了一个死循环或者极慢的命令 |

**`subprocess.run` 关键参数：**

| 参数 | 值 | 说明 |
|---|---|---|
| `shell=True` | - | 通过 shell 执行命令，支持管道、通配符等 shell 特性。注意：这有安全风险（命令注入），但对于 agent 场景是必要的 |
| `cwd=os.getcwd()` | - | 在当前工作目录执行命令。确保与 system prompt 中告知模型的目录一致 |
| `capture_output=True` | - | 捕获 stdout 和 stderr，而不是打印到终端 |
| `text=True` | - | 以文本（字符串）模式处理输出，而不是字节 |
| `timeout=120` | - | 120 秒超时，防止长时间阻塞 |

**设计决策解析：**
- 返回值永远是字符串，永远不抛异常。这是因为工具的执行结果要作为 `tool_result` 发回给模型，模型需要看到"发生了什么"（包括错误），然后自己决定下一步。如果函数抛异常导致程序崩溃，agent 就死了。
- 危险命令检查是**简单的字符串匹配**，不是沙箱。生产环境需要更强的安全措施（容器隔离、权限控制等）。

### 3.6 核心循环 -- agent_loop 函数（第 67-88 行）

```python
# -- The core pattern: a while loop that calls tools until the model stops --
def agent_loop(messages: list):
    while True:
        response = client.messages.create(
            model=MODEL, system=SYSTEM, messages=messages,
            tools=TOOLS, max_tokens=8000,
        )
        # Append assistant turn
        messages.append({"role": "assistant", "content": response.content})
        # If the model didn't call a tool, we're done
        if response.stop_reason != "tool_use":
            return
        # Execute each tool call, collect results
        results = []
        for block in response.content:
            if block.type == "tool_use":
                print(f"\033[33m$ {block.input['command']}\033[0m")
                output = run_bash(block.input["command"])
                print(output[:200])
                results.append({"type": "tool_result", "tool_use_id": block.id,
                                "content": output})
        messages.append({"role": "user", "content": results})
```

**这是整个文件最重要的部分。** 逐行拆解：

#### 第 68 行：函数签名
```python
def agent_loop(messages: list):
```
- 接收一个 `messages` 列表的**引用**（不是副本）
- 这意味着函数内部对 `messages` 的修改（append）会直接影响外部的列表
- 这是故意的设计：REPL 层需要看到更新后的 messages（包含 assistant 回复）来显示最终输出

#### 第 69 行：无限循环
```python
    while True:
```
- 用 `while True` 而不是 `while stop_reason == "tool_use"` 是因为第一次进循环时还没有 `stop_reason`
- 退出由内部 `if ... return` 控制

#### 第 70-73 行：调用 LLM API
```python
        response = client.messages.create(
            model=MODEL, system=SYSTEM, messages=messages,
            tools=TOOLS, max_tokens=8000,
        )
```
- `client.messages.create()`：调用 Anthropic Messages API
- `model=MODEL`：使用哪个模型
- `system=SYSTEM`：system prompt，每次调用都会发送
- `messages=messages`：完整的对话历史，累积式增长
- `tools=TOOLS`：工具定义列表，告诉模型可以用什么工具
- `max_tokens=8000`：限制模型单次回复的最大 token 数

**注意**：`system` 参数是**独立于 messages 的**，它不是 messages 列表的一部分。每次 API 调用都会自动包含 system prompt，不需要手动管理。

#### 第 75 行：追加 assistant 回复
```python
        messages.append({"role": "assistant", "content": response.content})
```
- `response.content` 是一个列表，可能包含 `TextBlock` 和/或 `ToolUseBlock`
- 无论模型是否调用了工具，都先把回复追加到 messages
- 这保证了消息历史的完整性：下次调用 API 时，模型能看到自己之前说了什么

#### 第 77-78 行：退出条件
```python
        if response.stop_reason != "tool_use":
            return
```
- **这两行是整个 agent 的灵魂**
- `stop_reason` 的可能取值：
  - `"tool_use"`：模型想要调用工具，循环继续
  - `"end_turn"`：模型认为回合结束，循环退出
  - `"max_tokens"`：达到 token 上限被截断，循环也退出
- 只有 `"tool_use"` 才继续循环，其他所有情况都退出
- 模型自己决定何时停止 -- 这是 agent 的核心设计

#### 第 80-88 行：执行工具并收集结果
```python
        results = []
        for block in response.content:
            if block.type == "tool_use":
                print(f"\033[33m$ {block.input['command']}\033[0m")
                output = run_bash(block.input["command"])
                print(output[:200])
                results.append({"type": "tool_result", "tool_use_id": block.id,
                                "content": output})
        messages.append({"role": "user", "content": results})
```

**逐行解读：**

| 行 | 代码 | 说明 |
|---|---|---|
| 80 | `results = []` | 收集本轮所有工具调用的结果（模型可能一次调用多个工具） |
| 81 | `for block in response.content:` | 遍历 response 的所有内容块 |
| 82 | `if block.type == "tool_use":` | 只处理 `ToolUseBlock` 类型的块（跳过 `TextBlock`） |
| 83 | `print(f"\033[33m$ {block.input['command']}\033[0m")` | **终端输出**：用黄色（`\033[33m`）显示要执行的命令，让用户在终端里看到 agent 在干什么 |
| 84 | `output = run_bash(block.input["command"])` | 实际执行命令，拿到输出 |
| 85 | `print(output[:200])` | 在终端打印前 200 字符的输出，让用户知道发生了什么 |
| 86-87 | `results.append({...})` | 把结果封装成 API 要求的 `tool_result` 格式 |
| 88 | `messages.append({"role": "user", "content": results})` | 以 `user` 角色追加到消息列表 |

**`tool_result` 的关键字段：**

| 字段 | 说明 |
|---|---|
| `type: "tool_result"` | 标识这是一个工具结果 |
| `tool_use_id: block.id` | **配对机制**：每个 `ToolUseBlock` 都有一个唯一 `id`，`tool_result` 的 `tool_use_id` 必须与之匹配。这让模型知道"这个结果是哪个工具调用的输出"。如果不匹配，API 会报错 |
| `content: output` | 工具的实际输出内容（字符串） |

**为什么 tool_result 要放在 `user` 消息里？**
这是 Anthropic Messages API 的设计约定：对话交替为 `user → assistant → user → assistant → ...`。工具结果本质上是"真实世界的反馈"，在 API 层面它属于 `user` 角色的消息。这样做还有一个好处：模型能自然地区分"用户说的话"和"工具返回的结果"。

### 3.7 交互式 REPL（第 91-107 行）

```python
if __name__ == "__main__":
    history = []
    while True:
        try:
            query = input("\033[36ms01 >> \033[0m")
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

**逐行解读：**

| 行 | 代码 | 说明 |
|---|---|---|
| 91 | `if __name__ == "__main__":` | Python 标准入口点，直接运行此文件时执行 |
| 92 | `history = []` | 创建一个空的消息历史列表。这个列表在整个 REPL 生命周期内持续增长，实现多轮对话 |
| 93 | `while True:` | REPL 的外层循环（Read-Eval-Print Loop） |
| 95 | `input("\033[36ms01 >> \033[0m")` | 显示青色（`\033[36m`）的提示符 `s01 >> `，等待用户输入 |
| 96-97 | `except (EOFError, KeyboardInterrupt)` | 处理 Ctrl+D（EOF）和 Ctrl+C（中断），优雅退出 |
| 98-99 | `if query.strip().lower() in (...)` | 输入 `q`、`exit` 或空行时退出 |
| 100 | `history.append({"role": "user", "content": query})` | 把用户输入追加到历史 |
| 101 | `agent_loop(history)` | **调用核心循环！** 注意传入的是 `history` 的引用，循环内部会直接修改这个列表 |
| 102 | `response_content = history[-1]["content"]` | 拿到历史中最后一条消息的内容（即 assistant 的最终回复） |
| 103-106 | `if isinstance(response_content, list): ...` | 打印助手回复中的文本内容。`response.content` 是一个对象列表，需要遍历提取 `text` 属性 |
| 107 | `print()` | 空行分隔，美观 |

**REPL 和 agent_loop 的关系：**
- REPL 是"外层循环"，负责与用户交互（读输入、显示输出）
- agent_loop 是"内层循环"，负责与模型交互（调用 API、执行工具）
- 它们通过共享的 `history` 列表连接在一起
- 每次用户输入后，agent_loop 可能运行多轮（多次工具调用），但对用户来说只是"一次交互"

---

## 4. 完整执行流程示例

用户输入：`Create a file called hello.py that prints "Hello, World!"`

### 第 1 轮

**messages 状态（发送前）：**
```
[
  { role: "user", content: "Create a file called hello.py that prints \"Hello, World!\"" }
]
```

**API 调用**：发送 messages + tools + system

**模型返回**：
```
response.content = [
  TextBlock("I'll create that file for you."),
  ToolUseBlock(id="toolu_01", name="bash", input={"command": "echo 'print(\"Hello, World!\")' > hello.py"})
]
response.stop_reason = "tool_use"
```

**执行操作**：
1. 追加 assistant 消息到 messages
2. 检查 stop_reason == "tool_use" → 继续循环
3. 执行 bash 命令 `echo 'print("Hello, World!")' > hello.py`
4. 输出："(no output)"
5. 追加 tool_result 到 messages

### 第 2 轮

**messages 状态（发送前）：**
```
[
  { role: "user",      content: "Create a file called hello.py ..." },
  { role: "assistant", content: [TextBlock(...), ToolUseBlock(id="toolu_01", ...)] },
  { role: "user",      content: [{ type: "tool_result", tool_use_id: "toolu_01", content: "(no output)" }] }
]
```

**模型返回**：
```
response.content = [
  ToolUseBlock(id="toolu_02", name="bash", input={"command": "python hello.py"})
]
response.stop_reason = "tool_use"
```

**执行操作**：
1. 追加 assistant 消息
2. stop_reason == "tool_use" → 继续
3. 执行 `python hello.py`
4. 输出："Hello, World!"
5. 追加 tool_result

### 第 3 轮

**messages 状态（发送前）：**
```
[
  { role: "user",      content: "Create a file called hello.py ..." },
  { role: "assistant", content: [TextBlock(...), ToolUseBlock(id="toolu_01", ...)] },
  { role: "user",      content: [{ type: "tool_result", tool_use_id: "toolu_01", content: "(no output)" }] },
  { role: "assistant", content: [ToolUseBlock(id="toolu_02", ...)] },
  { role: "user",      content: [{ type: "tool_result", tool_use_id: "toolu_02", content: "Hello, World!" }] }
]
```

**模型返回**：
```
response.content = [
  TextBlock("Done! I've created hello.py and verified it works -- it prints \"Hello, World!\" as expected.")
]
response.stop_reason = "end_turn"
```

**执行操作**：
1. 追加 assistant 消息
2. stop_reason == "end_turn" → **退出循环！**

### 最终 messages 列表（6 条消息）

```
messages[0]: { role: "user",      content: "Create a file..." }         ← 用户原始输入
messages[1]: { role: "assistant", content: [Text + ToolUse(toolu_01)] } ← 模型创建文件
messages[2]: { role: "user",      content: [tool_result(toolu_01)] }    ← 创建结果
messages[3]: { role: "assistant", content: [ToolUse(toolu_02)] }        ← 模型验证文件
messages[4]: { role: "user",      content: [tool_result(toolu_02)] }    ← 验证结果
messages[5]: { role: "assistant", content: [Text("Done!")] }            ← 最终回答
```

**观察要点**：
- 消息严格交替：user → assistant → user → assistant → ...
- 每条 tool_result 的 tool_use_id 都和对应的 ToolUseBlock.id 配对
- 模型自己决定了执行两步（创建 + 验证），而不是一步

---

## 5. 变更总结表

| 组件 | 之前 | 之后 |
|---|---|---|
| Agent loop | (无) | `while True` + stop_reason |
| Tools | (无) | `bash` (单一工具) |
| Messages | (无) | 累积式消息列表 |
| Control flow | (无) | `stop_reason != "tool_use"` |

这是第一课，所以所有内容都是"从无到有"。从 s02 开始，每课都会在这个表中标出增量变化。

---

## 6. 关键设计原则

1. **模型即决策者** -- 何时调用工具、调用什么工具、何时停止，全由模型决定。harness 只是忠实执行。
2. **循环不变性** -- 后续 11 课都在此循环上叠加机制（更多工具、规划、压缩、子 agent……），但 `while True → call API → check stop_reason → execute tools` 这个骨架始终不变。
3. **累积式上下文** -- messages 只追加不删减（至少在 s01 中），模型能看到完整的行为轨迹，包括之前的错误和修正。
4. **工具结果即反馈** -- 工具的输出以 tool_result 的形式回传给模型，形成一个闭环。模型根据反馈决定下一步，就像人根据终端输出决定下一步操作一样。
5. **优雅降级** -- run_bash 函数永远返回字符串（包括错误信息），永远不抛异常。agent 不会因为一个命令执行失败就崩溃，模型会看到错误信息然后自己调整。

---

## 7. 试一试

```sh
cd learn-claude-code
python agents/s01_agent_loop.py
```

试试这些 prompt（英文 prompt 对 LLM 效果更好, 也可以用中文）:

1. `Create a file called hello.py that prints "Hello, World!"`
2. `List all Python files in this directory`
3. `What is the current git branch?`
4. `Create a directory called test_output and write 3 files in it`

**观察重点：**
- 终端中黄色的 `$` 行是模型决定执行的命令
- 注意模型可能执行多步操作（比如先创建文件，再验证）
- 注意 stop_reason 从 `tool_use` 变成 `end_turn` 的时机
