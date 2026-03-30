# s06: Context Compact (上下文压缩) -- 详细讲解

`s01 > s02 > s03 > s04 > s05 > [ s06 ] | s07 > s08 > s09 > s10 > s11 > s12`

> *"上下文总会满, 要有办法腾地方"* -- 三层压缩策略, 换来无限会话。
>
> **Harness 层**: 压缩 -- 干净的记忆, 无限的会话。

这是系列的**第六课**。s01 搭建了 agent 的骨架（循环），s02 扩展了它的能力（多工具 + dispatch map），s03 教会了它自我规划（TodoManager + nag reminder），s04 引入了子智能体来隔离上下文，s05 实现了按需技能加载。s06 要回答的核心问题是：**当上下文窗口即将被填满时，如何在不丢失关键信息的前提下继续工作？** 答案是三层压缩策略：微压缩（静默瘦身）、自动压缩（阈值触发）、手动压缩（模型主动请求）。

---

## 1. 问题：为什么需要上下文压缩？

原文一句话：
> 上下文窗口是有限的。读一个 1000 行的文件就吃掉 ~4000 token; 读 30 个文件、跑 20 条命令, 轻松突破 100k token。不压缩, 智能体根本没法在大项目里干活。

### 展开理解

想象一个真实的开发场景：你让 agent "分析项目中所有 Python 文件并总结每一个"。

**上下文消耗速度有多快？**

```
操作                           大约消耗 tokens
──────────────────────────────────────────────
读 1 个 200 行的 Python 文件     ~800 tokens
读 1 个 500 行的 Python 文件     ~2000 tokens
执行一条 bash 命令 + 输出        ~200-500 tokens
agent 的每次回复                 ~200-800 tokens
```

如果项目有 30 个 Python 文件，平均每个 300 行：

```
30 文件 × ~1200 tokens/文件 = ~36,000 tokens（仅文件内容）
+ 30 次工具调用的请求/响应开销  = ~10,000 tokens
+ agent 的 30 次分析回复        = ~15,000 tokens
────────────────────────────────────────────
总计 ≈ 61,000 tokens
```

而上下文窗口通常是 128k 或 200k tokens。看起来还行？但别忘了：

1. **System prompt 占位**：基础指令 + 工具定义已经占了 ~3000 tokens
2. **后续工作还要空间**：分析完还要写总结、改代码，需要留余量
3. **累积效应**：这只是一个任务，用户可能连续做多个任务，上下文只增不减
4. **质量衰退**：即使没超限，上下文太长时模型的注意力也会分散（"lost in the middle" 问题）

**不压缩的后果**

```
场景 1：硬撞上限
  用户: "读完剩下的文件"
  API: Error 400 - context_length_exceeded
  → 会话直接死掉，所有上下文丢失

场景 2：质量衰退
  messages[] 有 100k tokens
  → 模型开始"忘记"早期的重要信息
  → 重复已经做过的工作
  → 给出与之前矛盾的回答

场景 3：成本爆炸
  每次 API 调用都要发送全部历史
  → 100k tokens 输入 × $3/M tokens = $0.30/次调用
  → 一个任务 50 次调用 = $15
  → 毫无必要，因为大部分旧内容已经不重要了
```

### 核心矛盾

```
需求：agent 要能处理大型任务（读大量文件、做长时间开发）
约束：上下文窗口有硬上限，且越长质量越差、成本越高

矛盾：任务越大 → 上下文越长 → 撞上限 / 质量下降 / 成本暴增
```

### 解决思路

人类程序员是怎么处理长时间工作的？

- **短期记忆**：只记住当前正在做的事情的细节
- **长期记忆**：重要的决策和结论记在笔记里
- **选择性遗忘**：3 小时前 `git diff` 的具体输出？早忘了。但"我把登录逻辑从 session 改成了 JWT"这个决策还记得

Agent 也应该这样工作：**战略性遗忘**。不是丢信息，而是把详细信息存到磁盘（transcript），只在上下文中保留摘要。

---

## 2. 解决方案的架构

### 三层压缩策略图

```
Every turn:
+------------------+
| Tool call result |
+------------------+
        |
        v
[Layer 1: micro_compact]        (静默, 每一轮都执行)
  把 3 轮以前的 tool_result 内容
  替换为 "[Previous: used {tool_name}]"
        |
        v
[Check: tokens > 50000?]
   |               |
   no              yes
   |               |
   v               v
continue    [Layer 2: auto_compact]
              把完整对话保存到 .transcripts/
              让 LLM 做一次摘要。
              用 [摘要] 替换所有 messages。
                    |
                    v
            [Layer 3: compact tool]
              模型主动调用 compact 工具。
              执行与 auto_compact 相同的摘要机制。
```

### 架构图逐元素解读

**Layer 1: micro_compact（微压缩）**
- **触发条件**：每一轮 LLM 调用前都执行
- **作用对象**：旧的 tool_result 内容（保留最近 3 个）
- **压缩方式**：把大于 100 字符的内容替换为 `[Previous: used read_file]` 这样的占位符
- **用户感知**：无，完全静默
- **类比**：就像你打开了很多文件，但只保留最近 3 个文件的内容在屏幕上，其他的只保留文件名标签

**Layer 2: auto_compact（自动压缩）**
- **触发条件**：估算 token 数超过 THRESHOLD（50000）
- **作用对象**：整个 messages 列表
- **压缩方式**：① 保存完整对话到磁盘（.jsonl 文件） ② 让 LLM 做一次摘要 ③ 用 2 条消息替换整个历史
- **用户感知**：打印 `[auto_compact triggered]`
- **类比**：就像你记了 50 页笔记后，写一页总结，然后翻到新的一页继续工作。旧笔记不扔，放抽屉里

**Layer 3: compact tool（手动压缩）**
- **触发条件**：模型主动调用 `compact` 工具
- **作用对象**：同 auto_compact
- **压缩方式**：同 auto_compact
- **用户感知**：打印 `[manual compact]`
- **类比**：你觉得笔记本太乱了，主动决定"我来整理一下"

### 三层递进关系

```
激进程度:   低 ──────────────────────────────────── 高
            |                |                      |
        Layer 1          Layer 2                Layer 3
      micro_compact     auto_compact         manual compact
            |                |                      |
触发:    每轮自动         阈值自动              模型主动
范围:    只替换旧结果     替换整个历史          替换整个历史
信息损失: 极小            有（但有 transcript） 有（但有 transcript）
频率:    非常高           偶尔                  罕见
```

### 设计哲学

1. **分层防御**：不是一次性暴力压缩，而是层层递进。大多数时候 Layer 1 就够了，只有真正需要时才触发 Layer 2/3
2. **信息不灭**：transcript 保存在磁盘上，压缩后的摘要里也包含了 transcript 的路径，理论上可以恢复
3. **模型参与**：让 LLM 自己做摘要（Layer 2/3），因为只有 LLM 知道哪些信息"重要"——这是比规则引擎更智能的压缩方式
4. **渐进式遗忘**：先忘细节（Layer 1），再忘全部但保留总结（Layer 2），就像人的记忆衰退曲线

---

## 3. 代码逐行详解

### 3.1 文件头和导入

```python
#!/usr/bin/env python3
# Harness: compression -- clean memory for infinite sessions.
"""
s06_context_compact.py - Compact
...
Key insight: "The agent can forget strategically and keep working forever."
"""
```

核心洞见就在文档字符串里：**"agent 可以战略性遗忘，从而永远工作下去"**。这句话概括了整课的哲学。

```python
import json
import os
import subprocess
import time
from pathlib import Path

from anthropic import Anthropic
from dotenv import load_dotenv
```

- `time`：**新增**——用于给 transcript 文件名加时间戳
- 其余导入与 s05 相同

### 3.2 环境配置

```python
load_dotenv(override=True)

if os.getenv("ANTHROPIC_BASE_URL"):
    os.environ.pop("ANTHROPIC_AUTH_TOKEN", None)

WORKDIR = Path.cwd()
client = Anthropic(base_url=os.getenv("ANTHROPIC_BASE_URL"))
MODEL = os.environ["MODEL_ID"]
```

与 s05 完全一致，没有变化。

### 3.3 压缩相关常量

```python
SYSTEM = f"You are a coding agent at {WORKDIR}. Use tools to solve tasks."

THRESHOLD = 50000
TRANSCRIPT_DIR = WORKDIR / ".transcripts"
KEEP_RECENT = 3
```

这是 s06 **新增**的三个关键常量：

| 常量 | 值 | 含义 |
|---|---|---|
| `THRESHOLD` | 50000 | 估算 token 数超过此值触发 auto_compact |
| `TRANSCRIPT_DIR` | `.transcripts/` | 压缩前保存完整对话的目录 |
| `KEEP_RECENT` | 3 | micro_compact 保留最近 N 个 tool_result 不压缩 |

**为什么 THRESHOLD 是 50000？**

- 模型上下文窗口通常 128k-200k tokens
- 50000 大约是窗口的 25%-40%
- 留足余量让模型在压缩后还有足够空间继续工作
- 如果设太高（比如 100000），压缩时可能来不及，LLM 的摘要质量也会因输入太长而下降

**为什么 KEEP_RECENT 是 3？**

- 最近 3 个工具结果通常是当前正在进行的工作链的一部分
- 比如：读文件 → 编辑文件 → 验证修改，这三步的结果都应该保留
- 如果设太小（1），模型可能丢失正在进行的多步操作的中间状态
- 如果设太大（10），微压缩的效果就太弱了

### 3.4 Token 估算函数（新增）

```python
def estimate_tokens(messages: list) -> int:
    """Rough token count: ~4 chars per token."""
    return len(str(messages)) // 4
```

**为什么是"估算"而不是精确计算？**

- 精确计算需要 tokenizer（如 `tiktoken`），引入额外依赖
- 4 字符 ≈ 1 token 是英文的经验值，对中文不太准确（中文大约 1-2 字符 = 1 token）
- 但对于"是否触发压缩"这个判断来说，精确度不重要——我们要的是"大概快满了"的信号
- `str(messages)` 会把整个 messages 列表序列化为字符串，包括所有嵌套结构

**注意**：这个函数会把 messages 列表转成字符串，包含了 JSON 格式的开销（引号、花括号等），所以估算值会偏高。但偏高比偏低安全——宁可早压缩，不要撞上限。

### 3.5 Layer 1: micro_compact（新增核心）

```python
def micro_compact(messages: list) -> list:
    # Collect (msg_index, part_index, tool_result_dict) for all tool_result entries
    tool_results = []
    for msg_idx, msg in enumerate(messages):
        if msg["role"] == "user" and isinstance(msg.get("content"), list):
            for part_idx, part in enumerate(msg["content"]):
                if isinstance(part, dict) and part.get("type") == "tool_result":
                    tool_results.append((msg_idx, part_idx, part))
```

**第一段：收集所有 tool_result**

- 遍历 messages 列表，找出所有 `tool_result` 类型的内容块
- `tool_result` 只出现在 `role: "user"` 的消息中（因为 tool result 是 harness 以用户身份回传给 API 的）
- `content` 是列表形式时才可能包含 tool_result（单条用户输入的 content 是字符串）
- 记录三元组 `(msg_idx, part_idx, part)`，其中 `part` 是引用（后面直接修改它就能修改原始 messages）

**为什么要先全部收集再处理？** 因为需要知道总数才能决定"最近 3 个"是哪些。

```python
    if len(tool_results) <= KEEP_RECENT:
        return messages
```

**短路返回**：如果 tool_result 总数不超过 3 个，没什么可压缩的，直接返回。

```python
    # Find tool_name for each result by matching tool_use_id in prior assistant messages
    tool_name_map = {}
    for msg in messages:
        if msg["role"] == "assistant":
            content = msg.get("content", [])
            if isinstance(content, list):
                for block in content:
                    if hasattr(block, "type") and block.type == "tool_use":
                        tool_name_map[block.id] = block.name
```

**第二段：构建 tool_use_id → tool_name 的映射**

- 每个 `tool_result` 有一个 `tool_use_id`，对应 assistant 消息中某个 `tool_use` block 的 `id`
- 这个映射让我们能把 `"toolu_01XYZ..."` 翻译成人类可读的 `"read_file"`
- 注意：assistant 消息的 content 是 API 返回的对象（有 `.type`, `.id`, `.name` 属性），不是普通字典，所以用 `hasattr` 和点号访问

```python
    # Clear old results (keep last KEEP_RECENT)
    to_clear = tool_results[:-KEEP_RECENT]
    for _, _, result in to_clear:
        if isinstance(result.get("content"), str) and len(result["content"]) > 100:
            tool_id = result.get("tool_use_id", "")
            tool_name = tool_name_map.get(tool_id, "unknown")
            result["content"] = f"[Previous: used {tool_name}]"
    return messages
```

**第三段：执行压缩**

- `tool_results[:-KEEP_RECENT]`：除了最后 3 个之外的所有 tool_result
- 只压缩内容超过 100 字符的结果——短结果（如 "ok"、"Wrote 42 bytes"）保留原样，压缩它们没有意义
- 替换为 `"[Previous: used read_file]"` 这样的占位符
- **直接修改 result 字典**（原地修改），因为 `result` 是对 messages 中元素的引用

**设计巧妙之处**：

```
压缩前 messages 中的一条 tool_result:
{
  "type": "tool_result",
  "tool_use_id": "toolu_01ABC",
  "content": "#!/usr/bin/env python3\nimport os\nimport sys\n... (500行代码)"
}

压缩后:
{
  "type": "tool_result",
  "tool_use_id": "toolu_01ABC",
  "content": "[Previous: used read_file]"
}
```

结构不变，只是 content 被替换了。API 不会报错，模型也能理解"之前用过这个工具"。

### 3.6 Layer 2: auto_compact（新增核心）

```python
def auto_compact(messages: list) -> list:
    # Save full transcript to disk
    TRANSCRIPT_DIR.mkdir(exist_ok=True)
    transcript_path = TRANSCRIPT_DIR / f"transcript_{int(time.time())}.jsonl"
    with open(transcript_path, "w") as f:
        for msg in messages:
            f.write(json.dumps(msg, default=str) + "\n")
    print(f"[transcript saved: {transcript_path}]")
```

**第一段：保存完整对话到磁盘**

- 创建 `.transcripts/` 目录（如果不存在）
- 文件名用 Unix 时间戳确保唯一：`transcript_1711800000.jsonl`
- JSONL 格式（每行一条 JSON），而不是一整个 JSON 数组——这样便于流式写入和逐行读取
- `default=str`：处理不可序列化的对象（比如 API 返回的 ContentBlock 对象），转成字符串而不是报错

**为什么要保存？** 这是信息不灭原则的体现。压缩后上下文中的详细信息没了，但磁盘上有完整记录。理论上可以恢复、可以审计、可以 debug。

```python
    # Ask LLM to summarize
    conversation_text = json.dumps(messages, default=str)[:80000]
    response = client.messages.create(
        model=MODEL,
        messages=[{"role": "user", "content":
            "Summarize this conversation for continuity. Include: "
            "1) What was accomplished, 2) Current state, 3) Key decisions made. "
            "Be concise but preserve critical details.\n\n" + conversation_text}],
        max_tokens=2000,
    )
    summary = response.content[0].text
```

**第二段：让 LLM 做摘要**

- 把对话序列化为 JSON 字符串，截断到 80000 字符（约 20000 tokens），防止摘要请求本身也超上下文
- 摘要 prompt 要求包含三个维度：
  1. **What was accomplished**（完成了什么）——结果
  2. **Current state**（当前状态）——进行到哪了
  3. **Key decisions made**（关键决策）——为什么这么做
- `max_tokens=2000`：摘要控制在 2000 tokens 以内，约 1500 字

**为什么让 LLM 做摘要，而不是用规则？** 因为只有 LLM 能理解对话的**语义**。规则只能基于长度或时间裁剪，LLM 能判断"虽然这段对话很长，但关键结论就是把登录从 session 改成了 JWT"。

```python
    # Replace all messages with compressed summary
    return [
        {"role": "user", "content": f"[Conversation compressed. Transcript: {transcript_path}]\n\n{summary}"},
        {"role": "assistant", "content": "Understood. I have the context from the summary. Continuing."},
    ]
```

**第三段：用 2 条消息替换整个历史**

- 第 1 条（user）：标记这是压缩后的对话 + transcript 文件路径 + LLM 生成的摘要
- 第 2 条（assistant）：让模型"接受"摘要，表示理解上下文

**为什么需要 2 条消息？** API 要求 messages 以 user 开头，user/assistant 交替。如果只有一条 user 消息，下一次循环 agent 的回复就变成第二条消息，格式没问题。但加上 assistant 的确认消息更好——它建立了一个"我已经读过摘要"的锚点，让后续回复更连贯。

**压缩效果**：假设原来有 50000 tokens 的 messages，压缩后只剩 ~2500 tokens（摘要 2000 + 确认 500）。释放了 47500 tokens 的空间。

### 3.7 工具实现（继承自 s05）

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

四个工具函数（bash, read_file, write_file, edit_file）与 s05 完全一致。这说明压缩机制是独立于工具系统的——它是 harness 层的基础设施，不需要改变工具的实现。

### 3.8 工具 Dispatch Map 和定义（有变化）

```python
TOOL_HANDLERS = {
    "bash":       lambda **kw: run_bash(kw["command"]),
    "read_file":  lambda **kw: run_read(kw["path"], kw.get("limit")),
    "write_file": lambda **kw: run_write(kw["path"], kw["content"]),
    "edit_file":  lambda **kw: run_edit(kw["path"], kw["old_text"], kw["new_text"]),
    "compact":    lambda **kw: "Manual compression requested.",
}
```

**与 s05 的对比**：

- 去掉了 s05 的 `load_skill` 工具（s06 简化了，不再展示 skill 机制）
- **新增 `compact` 工具**：handler 只返回一个标记字符串，实际的压缩在 agent_loop 中处理

**为什么 compact 的 handler 不直接执行压缩？** 因为压缩需要修改整个 messages 列表，而工具 handler 只负责返回结果字符串。压缩逻辑属于 harness 的循环层，不属于工具层。

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
    {"name": "compact", "description": "Trigger manual conversation compression.",
     "input_schema": {"type": "object", "properties": {"focus": {"type": "string", "description": "What to preserve in the summary"}}}},
]
```

**compact 工具的 schema 设计**：
- 只有一个可选参数 `focus`：告诉压缩器"重点保留什么"
- 注意 `required` 字段缺失（或者说没有必填参数），模型可以不带参数调用
- 当前实现中 `focus` 参数没有被使用（handler 是硬编码的字符串）——这是一个可以扩展的点

### 3.9 Agent Loop（核心修改）

```python
def agent_loop(messages: list):
    while True:
        # Layer 1: micro_compact before each LLM call
        micro_compact(messages)
```

**每轮循环开始：先执行 Layer 1**

- `micro_compact` 直接修改 messages 列表（原地操作）
- 在 LLM 调用之前执行，这样发送给 API 的就已经是压缩后的版本了
- 即使没有什么可压缩的也安全调用（内部有 `len <= KEEP_RECENT` 的短路检查）

```python
        # Layer 2: auto_compact if token estimate exceeds threshold
        if estimate_tokens(messages) > THRESHOLD:
            print("[auto_compact triggered]")
            messages[:] = auto_compact(messages)
```

**Layer 1 之后检查 Layer 2**

- 如果 micro_compact 后 token 仍然超过阈值，触发 auto_compact
- `messages[:] = ...`：这是 Python 的列表切片赋值，**原地替换列表内容**而不是创建新列表
  - 为什么不用 `messages = auto_compact(messages)`？因为 `messages` 是函数参数，重新赋值只会改变局部变量，不会影响调用者持有的列表引用
  - `messages[:] = ...` 修改的是列表对象本身，调用者的引用指向同一个对象，所以能看到变化
- 注意顺序：Layer 1 先尝试微压缩（代价小），不够的话再触发 Layer 2（代价大）

```python
        response = client.messages.create(
            model=MODEL, system=SYSTEM, messages=messages,
            tools=TOOLS, max_tokens=8000,
        )
        messages.append({"role": "assistant", "content": response.content})
        if response.stop_reason != "tool_use":
            return
```

这部分与之前的课程一致：调用 API → 追加回复 → 检查是否结束。

```python
        results = []
        manual_compact = False
        for block in response.content:
            if block.type == "tool_use":
                if block.name == "compact":
                    manual_compact = True
                    output = "Compressing..."
                else:
                    handler = TOOL_HANDLERS.get(block.name)
                    try:
                        output = handler(**block.input) if handler else f"Unknown tool: {block.name}"
                    except Exception as e:
                        output = f"Error: {e}"
                print(f"> {block.name}: {str(output)[:200]}")
                results.append({"type": "tool_result", "tool_use_id": block.id, "content": str(output)})
        messages.append({"role": "user", "content": results})
```

**工具执行部分的变化**：

- 新增 `manual_compact` 标志
- 遇到 `compact` 工具时，设置标志为 True，返回 "Compressing..." 作为 tool_result
- **为什么不立即压缩？** 因为：
  1. 这轮可能有多个工具调用（模型可能同时调用 compact 和其他工具）
  2. 所有 tool_result 都需要先回传给 API（否则消息格式不完整）
  3. 压缩应该在 tool_result 追加之后、下次 LLM 调用之前执行

```python
        # Layer 3: manual compact triggered by the compact tool
        if manual_compact:
            print("[manual compact]")
            messages[:] = auto_compact(messages)
```

**Layer 3：在所有 tool_result 追加完毕后执行**

- 复用 `auto_compact` 函数——Layer 2 和 Layer 3 的压缩逻辑完全相同
- 区别只在于触发方式：Layer 2 是 harness 根据 token 阈值自动触发，Layer 3 是模型主动请求

### 3.10 交互式 REPL

```python
if __name__ == "__main__":
    history = []
    while True:
        try:
            query = input("\033[36ms06 >> \033[0m")
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

与之前课程结构一致：
- 提示符变为 `s06 >>`
- `history` 列表在多轮对话间保持——这很关键，因为压缩机制需要看到完整的历史才能起作用
- 注意：`agent_loop(history)` 可能会通过 `messages[:] = ...` 修改 `history` 的内容（当触发 auto_compact 时）

---

## 4. 完整执行流程示例

### 场景：用户要求读取所有 Python 文件

```
用户输入: "Read every Python file in the agents/ directory one by one"
```

#### 第 1 轮

```
messages = [
  {role: "user", content: "Read every Python file in the agents/ directory one by one"}
]
```

1. `micro_compact(messages)` → 没有 tool_result，直接返回
2. `estimate_tokens(messages) = ~20` → 远低于 50000，不触发 auto_compact
3. API 调用 → 模型回复要先列出文件，调用 bash

```
messages = [
  {role: "user", content: "Read every Python file..."},
  {role: "assistant", content: [TextBlock("I'll read..."), ToolUse(bash, {command: "ls agents/*.py"})]},
]
```

4. 执行 bash → 返回文件列表
5. stop_reason = "tool_use" → 继续循环

```
messages = [
  ...,
  {role: "user", content: [{type: "tool_result", tool_use_id: "t1", content: "s01_agent_loop.py\ns02_tool_use.py\n..."}]},
]
```

#### 第 2-7 轮：逐个读取文件

每一轮模型调用 `read_file` 读取一个文件。

到第 5 轮时，messages 中已经有 4 个 tool_result：

```
tool_results 列表:
  [0] bash: "s01_agent_loop.py\ns02_tool_use.py\n..."     ← 第 1 轮
  [1] read_file: "(s01 的完整代码, ~100行)"                 ← 第 2 轮
  [2] read_file: "(s02 的完整代码, ~150行)"                 ← 第 3 轮
  [3] read_file: "(s03 的完整代码, ~200行)"                 ← 第 4 轮
  [4] read_file: "(s04 的完整代码, ~250行)"                 ← 第 5 轮（当前）
```

第 6 轮开始时，`micro_compact` 执行：

```
KEEP_RECENT = 3, len(tool_results) = 5
to_clear = tool_results[:2] = [bash结果, s01代码]

压缩后:
  [0] bash: "[Previous: used bash]"                        ← 被压缩！
  [1] read_file: "[Previous: used read_file]"              ← 被压缩！
  [2] read_file: "(s03 的完整代码)"                         ← 保留
  [3] read_file: "(s04 的完整代码)"                         ← 保留
  [4] read_file: "(s05 的完整代码)"                         ← 保留（最新的）
```

#### 第 10+ 轮：token 超过阈值

假设读到第 8 个文件时，`estimate_tokens(messages)` 返回 52000：

```
[auto_compact triggered]
[transcript saved: .transcripts/transcript_1711800123.jsonl]
```

auto_compact 执行后：

```
messages = [
  {role: "user", content: "[Conversation compressed. Transcript: .transcripts/transcript_1711800123.jsonl]\n\nSummary: The user asked to read all Python files in agents/. Files read so far: s01 through s08. s01 implements the basic agent loop with a while loop checking stop_reason. s02 adds tool dispatch. s03 adds TodoManager. s04 adds subagent spawning. s05 adds skill loading. s06 adds context compression. s07 implements a task system. s08 adds background tasks. Remaining: s09-s12 and s_full."},
  {role: "assistant", content: "Understood. I have the context from the summary. Continuing."},
]
```

token 数从 ~52000 降到 ~800。agent 继续读取剩下的文件。

---

## 5. 消息列表的状态变化全景

### 正常工作时（micro_compact 活跃）

```
messages[] 结构：

[0] user:      "Read every Python file..."
[1] assistant: [TextBlock, ToolUse(bash)]
[2] user:      [{tool_result: "[Previous: used bash]"}]          ← 已压缩
[3] assistant: [TextBlock, ToolUse(read_file)]
[4] user:      [{tool_result: "[Previous: used read_file]"}]     ← 已压缩
[5] assistant: [TextBlock, ToolUse(read_file)]
[6] user:      [{tool_result: "[Previous: used read_file]"}]     ← 已压缩
[7] assistant: [TextBlock, ToolUse(read_file)]
[8] user:      [{tool_result: "(s06完整代码)"}]                   ← 保留（最近3个之一）
[9] assistant: [TextBlock, ToolUse(read_file)]
[10] user:     [{tool_result: "(s07完整代码)"}]                   ← 保留（最近3个之一）
[11] assistant: [TextBlock, ToolUse(read_file)]
[12] user:     [{tool_result: "(s08完整代码)"}]                   ← 保留（最近3个之一）
```

### auto_compact 触发后

```
messages[] 结构（极简）：

[0] user:      "[Conversation compressed...]\n\n{LLM 生成的摘要}"
[1] assistant: "Understood. I have the context from the summary. Continuing."
```

### 继续工作后

```
messages[] 结构：

[0] user:      "[Conversation compressed...]\n\n{摘要}"
[1] assistant: "Understood. Continuing."
[2] user:      (隐式继续，或新的用户指令)
[3] assistant: [TextBlock, ToolUse(read_file)]
[4] user:      [{tool_result: "(s09完整代码)"}]
...
```

---

## 6. 变更总结表

| 组件 | 之前 (s05) | 之后 (s06) | 说明 |
|---|---|---|---|
| 工具数量 | 5 (bash, read, write, edit, load_skill) | 5 (bash, read, write, edit, compact) | compact 替换 load_skill |
| 上下文管理 | 无 | 三层压缩 | 全新机制 |
| micro_compact | 无 | 旧 tool_result → 占位符 | 每轮静默执行 |
| auto_compact | 无 | token 阈值触发 LLM 摘要 | 自动防御 |
| manual compact | 无 | 模型调用 compact 工具触发 | 模型自主决策 |
| Transcripts | 无 | 保存到 `.transcripts/` | 信息不灭 |
| 新增常量 | — | THRESHOLD, TRANSCRIPT_DIR, KEEP_RECENT | 压缩策略参数 |
| 新增函数 | — | estimate_tokens, micro_compact, auto_compact | 压缩实现 |
| Skill 机制 | SkillManager + 两层注入 | 去掉（简化） | s06 聚焦压缩，不同时展示 skill |
| agent_loop | 基础循环 | 插入 3 层压缩逻辑 | 核心变化 |

---

## 7. 关键设计原则

### 原则 1：分层防御（Defense in Depth）

不依赖单一压缩策略，而是多层递进。Layer 1 每轮执行代价极小，Layer 2 偶尔触发代价中等，Layer 3 极少使用。大多数时候 Layer 1 就够了。

### 原则 2：信息不灭（No Data Loss）

压缩不是删除。完整对话保存在 `.transcripts/` 目录的 JSONL 文件中，摘要中也包含了 transcript 的路径。信息只是从活跃上下文移到了磁盘。

### 原则 3：语义压缩优于规则压缩

让 LLM 做摘要，而不是简单地截断最早的 N 条消息。LLM 理解对话的语义，能保留真正重要的信息（关键决策、当前状态），丢弃不重要的细节（具体的文件内容、命令输出）。

### 原则 4：原地修改（In-Place Mutation）

`messages[:] = ...` 而不是 `messages = ...`。这确保所有持有 messages 引用的代码都能看到更新后的列表。这是 Python 中处理共享可变状态的常见模式。

### 原则 5：模型也有自主权（Agent Autonomy）

Layer 3 让模型可以主动决定"我需要压缩了"。这比纯 harness 驱动更灵活——模型可能在 token 还没到阈值时就觉得上下文太杂乱了，主动清理。

### 原则 6：压缩与工具解耦

压缩逻辑在 agent_loop 中实现，不在工具 handler 中。工具只负责返回结果，压缩只负责管理上下文。各自独立，互不干扰。

---

## 8. 与 Claude Code 真实实现的对照

Claude Code 的压缩机制与 s06 的三层模型高度对应：

| s06 概念 | Claude Code 真实行为 |
|---|---|
| micro_compact | 系统自动压缩旧的 tool result，用户看到 "prior messages compressed" |
| auto_compact | 当上下文接近限制时自动触发，提示 "conversation was automatically compressed" |
| compact tool | 用户可以通过 `/compact` 命令手动触发压缩 |
| transcript 保存 | Claude Code 保留完整对话历史在内存/磁盘中 |
| KEEP_RECENT | 最近的工具调用结果保留完整内容 |

---

## 9. 试一试

```sh
cd learn-claude-code
python agents/s06_context_compact.py
```

### 推荐实验

**实验 1：观察 micro_compact**
```
s06 >> Read every Python file in the agents/ directory one by one
```
当模型读取第 4+ 个文件时，观察终端输出——早期的 tool_result 应该已经被替换为占位符（虽然这个替换是静默的，你可以在代码中加 print 来观察）。

**实验 2：触发 auto_compact**
```
s06 >> Keep reading files until compression triggers automatically
```
持续让 agent 读文件，直到看到 `[auto_compact triggered]` 输出。查看 `.transcripts/` 目录中生成的文件。

**实验 3：手动触发 compact**
```
s06 >> Use the compact tool to manually compress the conversation
```
观察模型是否调用了 `compact` 工具，以及压缩后的行为。

**实验 4（进阶）：修改阈值观察行为**

试试把 `THRESHOLD` 改成 5000（很低），然后让 agent 读几个文件，观察 auto_compact 被频繁触发的情况。或者把 `KEEP_RECENT` 改成 1，观察更激进的微压缩效果。

---

## 10. 思考题

1. **micro_compact 的副作用**：如果模型需要引用 4 轮前的工具输出（比如"你之前读的那个文件里第 30 行是什么？"），micro_compact 已经把内容替换掉了，模型会怎么做？这算不算信息丢失？

2. **摘要的质量**：auto_compact 让 LLM 做摘要，但摘要可能遗漏重要细节。有没有办法提高摘要质量？（提示：考虑 compact 工具的 `focus` 参数）

3. **多次压缩**：如果一个很长的会话触发了多次 auto_compact，每次都是"摘要的摘要"，信息会不会逐渐失真？如何缓解？

4. **与子智能体的关系**：s04 的子智能体有独立的 messages 列表。如果子智能体的上下文也满了，应该怎么处理？子智能体的压缩策略应该和主 agent 一样吗？
