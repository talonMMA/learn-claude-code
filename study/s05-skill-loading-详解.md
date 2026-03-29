# s05: Skill Loading (技能加载) -- 详细讲解

`s01 > s02 > s03 > s04 > [ s05 ] s06 | s07 > s08 > s09 > s10 > s11 > s12`

> *"用到什么知识, 临时加载什么知识"* -- 通过 tool_result 注入, 不塞 system prompt。
>
> **Harness 层**: 按需知识 -- 模型开口要时才给的领域专长。

这是系列的**第五课**。s01 搭建了 agent 的骨架（循环），s02 扩展了它的能力（多工具 + dispatch map），s03 教会了它自我规划（TodoManager + nag reminder），s04 引入了子智能体来隔离上下文。s05 要回答的核心问题是：**当智能体需要特定领域的专业知识时，如何既能提供完整的知识内容，又不浪费宝贵的上下文窗口？** 答案是两层注入：系统提示放摘要目录（便宜），tool_result 放完整内容（按需）。

---

## 1. 问题：为什么需要 Skill 机制？

原文一句话：
> 你希望智能体遵循特定领域的工作流: git 约定、测试模式、代码审查清单。全塞进系统提示太浪费 -- 10 个技能, 每个 2000 token, 就是 20,000 token, 大部分跟当前任务毫无关系。

### 展开理解

想象你有一个万能 coding agent，它可能需要以下领域知识：

1. **Git 工作流**：提交规范、分支策略、PR 模板（~2000 tokens）
2. **代码审查**：安全检查清单、性能检查、可维护性（~2000 tokens）
3. **PDF 处理**：如何读取、创建、合并 PDF（~2000 tokens）
4. **MCP 服务器构建**：协议规范、模板代码（~2000 tokens）
5. **Agent 设计**：架构模式、最佳实践（~2000 tokens）

**如果全部塞进 system prompt 会怎样？**

```
System prompt 总量 = 基础指令 (500 tokens) + 5个技能 × 2000 tokens = 10,500 tokens
```

问题有三个：

1. **浪费钱**：每次 API 调用都要为 10,500 tokens 的输入付费，即使用户只是问个简单问题
2. **噪音干扰**：模型要从 10,500 tokens 中找到相关内容，注意力被稀释
3. **不可扩展**：技能从 5 个增长到 20 个时，system prompt 膨胀到 40,000+ tokens

**真实世界的例子：Claude Code**

Claude Code 有几十个 skill（commit、review-pr、pdf、xlsx 等），如果全放在 system prompt 里，每次对话的基础成本就高得离谱。实际上，绝大多数对话只会用到 0-2 个 skill。

### 核心矛盾

```
需求：agent 要有广泛的领域知识
约束：上下文窗口是稀缺资源，每个 token 都有成本

矛盾：知识越多 → 上下文越胖 → 成本越高 + 质量越差
```

### 类比：图书馆 vs 书包

- **全塞 system prompt** = 每天背着整个图书馆上学 —— 太重了
- **两层注入** = 带一张图书馆目录卡，需要时去借 —— 轻便高效

---

## 2. 解决方案的架构

```
System prompt (Layer 1 -- always present):
+--------------------------------------+
| You are a coding agent.              |
| Skills available:                    |
|   - git: Git workflow helpers        |  ~100 tokens/skill
|   - test: Testing best practices     |
|   - pdf: Process PDF files           |
|   - code-review: Review code         |
+--------------------------------------+

When model calls load_skill("git"):
+--------------------------------------+
| tool_result (Layer 2 -- on demand):  |
| <skill name="git">                   |
|   Full git workflow instructions...  |  ~2000 tokens
|   Step 1: ...                        |
| </skill>                             |
+--------------------------------------+
```

### 架构解读

**Layer 1（系统提示层）—— 便宜的目录**

- 始终存在于每次 API 调用中
- 每个技能只占 ~100 tokens（一行名称 + 描述）
- 作用：让模型**知道有什么技能可用**
- 类比：书架上的标签

**Layer 2（工具结果层）—— 按需的全文**

- 只在模型主动调用 `load_skill` 时才注入
- 每个技能完整内容 ~2000 tokens
- 作用：给模型**实际的领域知识**
- 类比：从书架上取下的那本书

### 设计哲学

这个设计体现了一个关键原则：**让模型自己决定什么时候需要什么知识**。

Harness（你的代码）的角色不是"决定加载哪个 skill"，而是：
1. 告诉模型"你有哪些 skill 可以用"（Layer 1）
2. 当模型说"我需要这个 skill"时，把内容给它（Layer 2）

这和 s02 的工具使用思想一脉相承：**相信模型的判断力**。

### 为什么用 tool_result 注入而不是修改 system prompt？

你可能会想：为什么不在模型请求技能时，动态修改 system prompt 重新调用？

原因：
1. **system prompt 修改意味着重启对话**，之前的 messages 历史中的 system prompt 引用会不一致
2. **tool_result 是对话历史的自然延续**，模型"问"了一个问题（load_skill），得到了"回答"（技能内容）
3. **符合既有的工具使用模式**，不需要新机制，只是 dispatch map 多了一个条目

---

## 3. 技能文件的结构

### 目录组织

```
skills/
├── agent-builder/
│   └── SKILL.md          # agent 设计知识
├── code-review/
│   └── SKILL.md          # 代码审查清单
├── mcp-builder/
│   └── SKILL.md          # MCP 服务器模板
└── pdf/
    └── SKILL.md          # PDF 处理知识
```

### SKILL.md 文件格式

每个 SKILL.md 都有 YAML frontmatter + Markdown 正文：

```markdown
---
name: code-review
description: Perform thorough code reviews with security, performance,
  and maintainability analysis. Use when user asks to review code,
  check for bugs, or audit a codebase.
---

# Code Review Skill

You now have expertise in conducting comprehensive code reviews...

## Review Checklist
### 1. Security (Critical)
...
```

**frontmatter 字段说明：**

| 字段 | 作用 | 去向 |
|------|------|------|
| `name` | 技能标识符，模型用这个名字来加载 | Layer 1 目录 + Layer 2 加载 key |
| `description` | 一句话描述，帮模型判断什么时候该加载 | Layer 1 目录中展示 |
| `tags`（可选） | 额外分类标签 | Layer 1 目录中展示（如果有） |

**正文部分**是模型加载后会看到的完整内容。写法上有讲究：
- 以"You now have expertise in..."开头，暗示模型现在拥有了这个能力
- 提供具体的检查清单、代码模板、命令示例
- 结构化组织（标题 + 清单 + 代码块），方便模型提取信息

---

## 4. 代码逐行详解

### 4.1 文件头和导入

```python
#!/usr/bin/env python3
# Harness: on-demand knowledge -- domain expertise, loaded when the model asks.
"""
s05_skill_loading.py - Skills

Two-layer skill injection that avoids bloating the system prompt:

    Layer 1 (cheap): skill names in system prompt (~100 tokens/skill)
    Layer 2 (on demand): full skill body in tool_result
"""

import os
import re                  # ← 新增：用于解析 YAML frontmatter
import subprocess
from pathlib import Path

from anthropic import Anthropic
from dotenv import load_dotenv
```

**与 s04 的对比：**
- 新增 `re` 模块 — 用正则表达式解析 SKILL.md 的 frontmatter
- 移除了 s04 的 `json` 和 `threading` — 不再需要子智能体和 TodoManager

> 注意：s05 回到了"单一 agent"的结构，不再有 TodoManager 或子智能体。每一课聚焦一个新概念。

### 4.2 环境配置

```python
load_dotenv(override=True)

if os.getenv("ANTHROPIC_BASE_URL"):
    os.environ.pop("ANTHROPIC_AUTH_TOKEN", None)

WORKDIR = Path.cwd()
client = Anthropic(base_url=os.getenv("ANTHROPIC_BASE_URL"))
MODEL = os.environ["MODEL_ID"]
SKILLS_DIR = WORKDIR / "skills"    # ← 新增：技能文件的根目录
```

**新增：`SKILLS_DIR`**

指向项目根目录下的 `skills/` 文件夹。SkillLoader 会递归扫描这个目录下所有的 `SKILL.md` 文件。

### 4.3 SkillLoader 类 —— 本课核心

这是 s05 的核心新增内容，一共 ~50 行，但设计精巧。

#### 4.3.1 构造函数和扫描

```python
class SkillLoader:
    def __init__(self, skills_dir: Path):
        self.skills_dir = skills_dir
        self.skills = {}           # {name: {"meta": {...}, "body": "...", "path": "..."}}
        self._load_all()

    def _load_all(self):
        if not self.skills_dir.exists():
            return                 # 目录不存在就跳过，不报错
        for f in sorted(self.skills_dir.rglob("SKILL.md")):
            text = f.read_text()
            meta, body = self._parse_frontmatter(text)
            name = meta.get("name", f.parent.name)  # fallback 到目录名
            self.skills[name] = {"meta": meta, "body": body, "path": str(f)}
```

**逐行解析：**

1. `self.skills = {}` — 技能注册表，key 是技能名，value 包含元数据、正文和路径
2. `self.skills_dir.rglob("SKILL.md")` — 递归搜索所有 SKILL.md 文件
   - `rglob` 意味着支持嵌套目录：`skills/category/sub/SKILL.md` 也能找到
3. `sorted(...)` — 按字母顺序排列，确保输出稳定
4. `meta.get("name", f.parent.name)` — 优先用 frontmatter 中的 name，没有就用目录名
   - 这是一个健壮的 fallback 设计：即使 frontmatter 写错了，目录名还能兜底

**设计决策：为什么在启动时一次性加载所有技能？**

因为 Layer 1（描述列表）需要在 system prompt 中，而 system prompt 在对话开始前就确定了。所以必须在构造函数中扫描完所有技能。但 Layer 2（完整内容）是延迟的——只在 `get_content()` 被调用时才返回。

虽然 body 已经在内存中了（`self.skills[name]["body"]`），但关键是**它不会出现在 API 调用的 system prompt 中**，只在模型主动调用 `load_skill` 后才通过 tool_result 进入对话。

#### 4.3.2 frontmatter 解析器

```python
    def _parse_frontmatter(self, text: str) -> tuple:
        """Parse YAML frontmatter between --- delimiters."""
        match = re.match(r"^---\n(.*?)\n---\n(.*)", text, re.DOTALL)
        if not match:
            return {}, text        # 没有 frontmatter 就整个文件当 body
        meta = {}
        for line in match.group(1).strip().splitlines():
            if ":" in line:
                key, val = line.split(":", 1)
                meta[key.strip()] = val.strip()
        return meta, match.group(2).strip()
```

**逐行解析：**

1. 正则 `r"^---\n(.*?)\n---\n(.*)"` 匹配标准 YAML frontmatter 格式：
   ```
   ---
   name: pdf
   description: Process PDF files
   ---
   正文内容...
   ```
   - `(.*?)` 非贪婪匹配 frontmatter 区域
   - `(.*)` 匹配剩余所有内容（正文）
   - `re.DOTALL` 让 `.` 匹配换行符

2. 简易 YAML 解析：只处理 `key: value` 格式，不支持嵌套
   - 这是**有意的简化**：SKILL.md 的 frontmatter 只需要扁平的键值对
   - 不引入 PyYAML 依赖，保持轻量

3. `line.split(":", 1)` — 只在第一个冒号处分割
   - 确保 `description: This is a: complex description` 也能正确解析

**注意一个小细节：** 如果 `description` 字段使用 YAML 的多行语法（如 `|`），这个简易解析器只会拿到 `|` 本身。但对于我们的示例 SKILL.md 来说够用了。

#### 4.3.3 Layer 1：描述列表（给 system prompt 用）

```python
    def get_descriptions(self) -> str:
        """Layer 1: short descriptions for the system prompt."""
        if not self.skills:
            return "(no skills available)"
        lines = []
        for name, skill in self.skills.items():
            desc = skill["meta"].get("description", "No description")
            tags = skill["meta"].get("tags", "")
            line = f"  - {name}: {desc}"
            if tags:
                line += f" [{tags}]"
            lines.append(line)
        return "\n".join(lines)
```

**输出示例：**
```
  - agent-builder: Design and build AI agents for any domain...
  - code-review: Perform thorough code reviews with security...
  - mcp-builder: Build MCP servers that give Claude new capabilities...
  - pdf: Process PDF files - extract text, create PDFs...
```

**每个技能只占一行**——这就是 Layer 1 的精髓：用最少的 token 让模型知道"有什么可用的"。

#### 4.3.4 Layer 2：完整内容（给 tool_result 用）

```python
    def get_content(self, name: str) -> str:
        """Layer 2: full skill body returned in tool_result."""
        skill = self.skills.get(name)
        if not skill:
            return f"Error: Unknown skill '{name}'. Available: {', '.join(self.skills.keys())}"
        return f"<skill name=\"{name}\">\n{skill['body']}\n</skill>"
```

**输出示例**（当模型调用 `load_skill("code-review")` 时）：
```xml
<skill name="code-review">
# Code Review Skill

You now have expertise in conducting comprehensive code reviews...
## Review Checklist
### 1. Security (Critical)
...
</skill>
```

**注意 `<skill>` XML 标签包裹**——这是给模型的结构化信号，让它明确知道"这是一个技能的内容"，方便模型理解边界。

**错误处理**也很友好：如果模型请求一个不存在的技能名，返回错误信息**并列出所有可用的技能名**，帮模型自我纠正。

### 4.4 SkillLoader 实例化和 System Prompt

```python
SKILL_LOADER = SkillLoader(SKILLS_DIR)

# Layer 1: skill metadata injected into system prompt
SYSTEM = f"""You are a coding agent at {WORKDIR}.
Use load_skill to access specialized knowledge before tackling unfamiliar topics.

Skills available:
{SKILL_LOADER.get_descriptions()}"""
```

**这就是两层注入的第一层。** 实际的 system prompt 大约长这样：

```
You are a coding agent at /path/to/project.
Use load_skill to access specialized knowledge before tackling unfamiliar topics.

Skills available:
  - agent-builder: Design and build AI agents for any domain...
  - code-review: Perform thorough code reviews with security...
  - mcp-builder: Build MCP servers that give Claude new capabilities...
  - pdf: Process PDF files - extract text, create PDFs...
```

**关键指令**：`Use load_skill to access specialized knowledge before tackling unfamiliar topics.`

这句话非常重要——它告诉模型：
1. 有一个工具叫 `load_skill`
2. 在处理不熟悉的主题**之前**先加载
3. 暗示了"先加载知识，再行动"的工作流程

### 4.5 工具函数（从 s04 继承）

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

这些工具函数和 s04 基本一致，是 agent 的基础能力。不再赘述。

### 4.6 Dispatch Map —— 加入 load_skill

```python
TOOL_HANDLERS = {
    "bash":       lambda **kw: run_bash(kw["command"]),
    "read_file":  lambda **kw: run_read(kw["path"], kw.get("limit")),
    "write_file": lambda **kw: run_write(kw["path"], kw["content"]),
    "edit_file":  lambda **kw: run_edit(kw["path"], kw["old_text"], kw["new_text"]),
    "load_skill": lambda **kw: SKILL_LOADER.get_content(kw["name"]),  # ← 新增
}
```

**与 s04 的对比：**

| s04 的 dispatch map | s05 的 dispatch map |
|---|---|
| bash, read_file, write_file, edit_file | bash, read_file, write_file, edit_file |
| **run_agent**（子智能体） | **load_skill**（技能加载） |

`load_skill` 的处理极其简洁——就是调用 `SKILL_LOADER.get_content()`。**它不执行任何副作用，只返回文本。** 这让它成为最安全的工具之一。

### 4.7 工具定义（给 API 的 schema）

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
    {"name": "load_skill",
     "description": "Load specialized knowledge by name.",
     "input_schema": {"type": "object", "properties": {
         "name": {"type": "string", "description": "Skill name to load"}
     }, "required": ["name"]}},
]
```

**`load_skill` 的 schema 设计：**
- 只有一个参数 `name`（字符串类型）
- `description` 中加了 `"Skill name to load"`，帮模型理解参数含义
- 工具的顶层 `description` 是 `"Load specialized knowledge by name."`——简洁但明确

### 4.8 Agent 循环（从 s02 继承，无改动）

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

**这就是 s02 以来不变的 agent 循环骨架。** load_skill 只是 dispatch map 里多了一个条目，循环代码完全不需要改动。

这再次验证了 s02 dispatch map 设计的威力：**新增工具只需要修改 TOOL_HANDLERS 和 TOOLS，核心循环零改动。**

### 4.9 交互式 REPL（从 s02 继承）

```python
if __name__ == "__main__":
    history = []
    while True:
        try:
            query = input("\033[36ms05 >> \033[0m")
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

提示符变成了 `s05 >>` 以区分不同课程的运行环境。

---

## 5. 完整执行流程示例

### 场景：用户请求代码审查

```
用户输入: "Review the file agents/s05_skill_loading.py"
```

#### 第一轮：模型决定先加载知识

```
API 请求:
  system: "You are a coding agent at /path/to/project.\n
           Use load_skill to access specialized knowledge...\n
           Skills available:\n
             - agent-builder: Design and build AI agents...\n
             - code-review: Perform thorough code reviews...\n
             - mcp-builder: Build MCP servers...\n
             - pdf: Process PDF files..."
  messages: [
    {role: "user", content: "Review the file agents/s05_skill_loading.py"}
  ]
  tools: [bash, read_file, write_file, edit_file, load_skill]

API 响应:
  stop_reason: "tool_use"
  content: [
    {type: "text", text: "I'll load the code review skill first..."},
    {type: "tool_use", id: "tu_1", name: "load_skill",
     input: {"name": "code-review"}},                          ← 模型主动加载技能
    {type: "tool_use", id: "tu_2", name: "read_file",
     input: {"path": "agents/s05_skill_loading.py"}}           ← 同时读取文件
  ]
```

**关键观察**：模型看到 system prompt 中列出了 `code-review` 技能，判断当前任务需要它，于是主动调用 `load_skill`。同时并行读取要审查的文件。

#### Harness 执行工具

```python
# load_skill("code-review") 执行:
SKILL_LOADER.get_content("code-review")
# → 返回 <skill name="code-review">\n# Code Review Skill\nYou now have expertise...\n</skill>

# read_file("agents/s05_skill_loading.py") 执行:
run_read("agents/s05_skill_loading.py")
# → 返回文件的 227 行内容
```

messages 变为：

```
messages: [
  {role: "user", content: "Review the file agents/s05_skill_loading.py"},
  {role: "assistant", content: [
    {type: "text", text: "I'll load the code review skill first..."},
    {type: "tool_use", id: "tu_1", name: "load_skill", input: {"name": "code-review"}},
    {type: "tool_use", id: "tu_2", name: "read_file", input: {"path": "agents/s05_skill_loading.py"}}
  ]},
  {role: "user", content: [
    {type: "tool_result", tool_use_id: "tu_1",
     content: "<skill name=\"code-review\">\n# Code Review Skill\nYou now have expertise..."},
    {type: "tool_result", tool_use_id: "tu_2",
     content: "#!/usr/bin/env python3\n# Harness: on-demand knowledge..."}
  ]}
]
```

#### 第二轮：模型用加载的知识进行审查

```
API 响应:
  stop_reason: "end_turn"
  content: [
    {type: "text", text: "## Code Review: s05_skill_loading.py\n\n### Summary\n
     Well-structured skill loading implementation...\n\n### Security\n
     ✅ Path traversal protection via safe_path()...\n\n### Issues\n
     1. Dangerous command blacklist is easily bypassed..."}
  ]
```

模型现在拥有了 code-review 技能的完整清单，按照清单逐项审查代码文件。

#### 最终 messages 状态

```
messages: [
  {role: "user",      content: "Review the file agents/s05_skill_loading.py"},
  {role: "assistant", content: [text + tool_use(load_skill) + tool_use(read_file)]},
  {role: "user",      content: [tool_result(skill内容) + tool_result(文件内容)]},
  {role: "assistant", content: [text(审查报告)]}    ← stop_reason: "end_turn"
]
```

总共 2 轮 API 调用，技能内容只在需要时出现在对话中。

---

## 6. 两层注入的完整数据流

```
启动时:
┌──────────────────────────────────────────────────────┐
│  SKILL_LOADER = SkillLoader(skills/)                 │
│                                                      │
│  扫描: skills/agent-builder/SKILL.md                 │
│        skills/code-review/SKILL.md                   │
│        skills/mcp-builder/SKILL.md                   │
│        skills/pdf/SKILL.md                           │
│                                                      │
│  self.skills = {                                     │
│    "agent-builder": {meta: {...}, body: "...", ...}, │
│    "code-review":   {meta: {...}, body: "...", ...}, │
│    "mcp-builder":   {meta: {...}, body: "...", ...}, │
│    "pdf":           {meta: {...}, body: "...", ...}, │
│  }                                                   │
└──────────────────────────────────────────────────────┘
              │
              ▼
┌──────────────────────────────────────────────────────┐
│  SYSTEM prompt (Layer 1):                            │
│  "Skills available:                                  │
│    - agent-builder: Design and build AI agents...    │  ← ~400 tokens 总计
│    - code-review: Perform thorough code reviews...   │
│    - mcp-builder: Build MCP servers...               │
│    - pdf: Process PDF files..."                      │
└──────────────────────────────────────────────────────┘
              │
              ▼  (每次 API 调用都带上)
┌──────────────────────────────────────────────────────┐
│  模型看到: "我有 4 个技能可以用"                       │
│  模型判断: "用户要审查代码 → 需要 code-review"        │
│  模型输出: tool_use(load_skill, name="code-review")  │
└──────────────────────────────────────────────────────┘
              │
              ▼  (Harness 执行)
┌──────────────────────────────────────────────────────┐
│  SKILL_LOADER.get_content("code-review")             │
│  → "<skill name=\"code-review\">                     │
│      # Code Review Skill                             │  ← ~2000 tokens
│      You now have expertise...                       │
│      ## Review Checklist                             │
│      ..."                                            │
└──────────────────────────────────────────────────────┘
              │
              ▼  (注入到 tool_result)
┌──────────────────────────────────────────────────────┐
│  messages 新增:                                       │
│  {role: "user", content: [{                          │
│    type: "tool_result",                              │
│    tool_use_id: "tu_xxx",                            │  ← Layer 2: 按需
│    content: "<skill name=\"code-review\">..."        │
│  }]}                                                 │
└──────────────────────────────────────────────────────┘
```

---

## 7. 变更总结表

| 组件 | 之前 (s04) | 之后 (s05) |
|------|-----------|-----------|
| **核心新增** | SubAgent + TodoManager | SkillLoader |
| **Tools** | 5 (基础 + run_agent) | 5 (基础 + load_skill) |
| **系统提示** | 静态 + todo nag 提醒 | 静态 + 技能描述列表 |
| **知识管理** | 无 | skills/\*/SKILL.md 文件 |
| **注入方式** | 无 | 两层（system prompt + tool_result） |
| **新增模块** | `re` | `re`（frontmatter 解析） |
| **新增文件** | 无 | 4 个 SKILL.md |
| **Agent 循环** | 不变 | 不变 |
| **代码行数** | ~230 行 | ~227 行（更简洁） |

---

## 8. 关键设计原则

### 原则 1：按需加载优于预加载
**一句话：** 不要把所有知识塞进 system prompt，让模型自己决定什么时候需要什么。
**解释：** 上下文窗口是稀缺资源。预加载所有知识是 O(n) 成本，按需加载是 O(1) ~ O(k)（k 是实际用到的技能数）。

### 原则 2：目录和内容分离
**一句话：** 便宜的摘要放 system prompt，昂贵的全文放 tool_result。
**解释：** 这是经典的"索引+数据"分层。模型先看索引（~100 tokens/技能），再按需取数据（~2000 tokens/技能）。这个模式在数据库、文件系统、搜索引擎中无处不在。

### 原则 3：Dispatch Map 的可扩展性
**一句话：** 新增 load_skill 工具不需要修改 agent 循环一行代码。
**解释：** s02 建立的 dispatch map 架构在 s05 再次证明了它的价值。load_skill 就是又一个 `name → handler` 映射。

### 原则 4：信任模型的判断
**一句话：** 让模型自己判断什么时候需要加载技能，不需要 harness 预判。
**解释：** 模型看到 system prompt 中的技能列表和用户请求，有能力判断是否需要加载。这比 harness 硬编码规则（如"用户提到 review 就自动加载 code-review"）更灵活。

### 原则 5：文件即配置
**一句话：** 技能用 Markdown 文件定义，不需要写代码。
**解释：** 添加新技能只需要创建一个目录和 SKILL.md 文件，不需要修改 Python 代码。这降低了知识贡献的门槛——非开发者也能写 SKILL.md。

---

## 9. 与 Claude Code 真实实现的对比

Claude Code 实际使用的 Skill 机制和 s05 的教学版本非常接近：

| 特性 | s05 教学版 | Claude Code 实际 |
|------|-----------|-----------------|
| 技能文件 | `skills/*/SKILL.md` | 同样是 SKILL.md 文件 |
| 两层注入 | system prompt + tool_result | system prompt 摘要 + Skill 工具注入 |
| 触发方式 | 模型调用 `load_skill` | 模型调用 `Skill` 工具 |
| 技能来源 | 本地 `skills/` 目录 | 多种来源（内置、用户自定义、MCP 等） |
| 描述在 prompt 中 | 是 | 是（在 system-reminder 中列出） |

你在本次对话的 system prompt 中就能看到真实的 Skill 列表（如 `commit`、`review-pr`、`pdf` 等），这就是 Layer 1。当我调用某个 Skill 时，它的完整内容才会被加载——这就是 Layer 2。

---

## 10. 试一试

```sh
cd learn-claude-code
python agents/s05_skill_loading.py
```

推荐试验的 prompt：

1. `What skills are available?`
   — 模型应该列出 system prompt 中的技能清单，不需要调用任何工具

2. `Load the agent-builder skill and follow its instructions`
   — 观察模型调用 `load_skill("agent-builder")`，然后按照技能内容行动

3. `I need to do a code review -- load the relevant skill first`
   — 模型应该先加载 code-review 技能，然后询问要审查什么文件

4. `Build an MCP server using the mcp-builder skill`
   — 模型加载 mcp-builder 技能后，应该能使用其中的模板代码

### 进阶试验

5. **不提示就加载**：`Review agents/s05_skill_loading.py for security issues`
   — 看模型是否会**自动**判断需要加载 code-review 技能

6. **加载不存在的技能**：`Load the kubernetes skill`
   — 观察错误处理：返回可用技能列表

7. **写一个新技能**：用 `write_file` 创建 `skills/testing/SKILL.md`，然后重启
   — 验证新技能是否自动出现在可用列表中
