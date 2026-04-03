# s09: Agent Teams (智能体团队) -- 详细讲解

`s01 > s02 > s03 > s04 > s05 > s06 | s07 > s08 > [ s09 ] s10 > s11 > s12`

> *"任务太大一个人干不完, 要能分给队友"* -- 持久化队友 + JSONL 邮箱。
>
> **Harness 层**: 团队邮箱 -- 多个模型, 通过文件协调。

这是系列的**第九课**，也是**第三阶段（持久化与后台）**的第三课。s07 把任务持久化到磁盘（DAG），s08 把慢命令丢到后台线程。s09 要回答的核心问题是：**当一个任务大到单个 agent 干不完时，怎么让多个 agent 像团队一样协作？** 答案是：**持久化的命名队友 + 基于文件的 JSONL 收件箱通信**。

---

## 1. 问题：为什么需要智能体团队？

### 回顾 s04 子智能体的局限

s04 引入了子智能体（subagent），但它是一次性的：

```
s04 子智能体的生命周期:
  spawn("分析这段代码") → 执行 → 返回摘要 → 销毁 ← 知识全部丢失
  spawn("写测试用例") → 执行 → 返回摘要 → 销毁 ← 又要从头来
```

**痛点 1：没有记忆传递**

```
用户: "先让 agent 分析代码结构，再让它写重构方案"

s04 的做法:
  子智能体 A: 分析代码 → 返回摘要（5000 tokens 压缩成 200）→ 消亡
  子智能体 B: 写重构方案 → 只拿到 A 的 200 token 摘要 → 缺少细节

问题: B 对代码的理解只有 A 的 4%，大量上下文在摘要时丢失
```

**痛点 2：没有身份**

```
s04 的子智能体没有名字，没有角色，没有可追踪的身份。
你无法说 "让 alice 继续她上次的工作"，因为 alice 这个概念不存在。
每次都是一个匿名的、无状态的执行单元。
```

**痛点 3：无法直接通信**

```
s04: 子智能体只能和主 agent 通信（返回摘要）

  主 agent ←→ 子A    主 agent ←→ 子B    ← 星形拓扑，中心瓶颈

  子A 不能直接找子B 说 "你那边 API 接口定义好了吗？"
  所有沟通必须经过主 agent 中转
```

### s08 后台任务也不够

s08 的后台线程只跑 shell 命令，不做 LLM 引导的决策：

```
s08: bg_run("npm install")  ← 只是跑命令，没有思考能力
s09: spawn("alice", "coder", "实现登录功能")  ← 完整的 agent loop，能思考+用工具
```

### s09 的目标

```
需要的三样东西:
  1. 持久化智能体: 能跨多轮对话存活，有 idle -> working -> idle 的生命周期
  2. 身份管理: 每个智能体有名字、角色、状态，存在 config.json 里
  3. 通信通道: 智能体之间可以直接发消息，不需要经过中心节点
```

---

## 2. 架构总览

```
                          ┌─────────────────────────────┐
                          │      用户 (stdin/stdout)      │
                          └─────────┬───────────────────┘
                                    │
                          ┌─────────▼───────────────────┐
                          │      Lead Agent (主线程)      │
                          │   9 tools, SYSTEM prompt     │
                          │   可以 spawn/send/broadcast  │
                          └──┬──────────┬───────────────┘
                             │          │
              spawn("alice") │          │ spawn("bob")
                             │          │
                    ┌────────▼──┐   ┌──▼────────┐
                    │  alice    │   │   bob      │
                    │  Thread   │   │   Thread   │
                    │  6 tools  │   │   6 tools  │
                    │  (no spawn)│   │  (no spawn)│
                    └────────┬──┘   └──┬────────┘
                             │         │
                             ▼         ▼
                     .team/inbox/alice.jsonl
                     .team/inbox/bob.jsonl
                     .team/inbox/lead.jsonl
                     .team/config.json
```

**关键设计决策**: Lead（组长）拥有 9 个工具（包括 spawn/broadcast），队友只有 6 个工具（不能 spawn 新队友）。这是**职责分离**：组长负责协调，队友负责执行。

---

## 3. 核心组件逐行拆解

### 3.1 MessageBus: JSONL 收件箱

这是智能体间通信的核心。每个智能体有一个 `.jsonl` 文件作为收件箱。

```python
class MessageBus:
    def __init__(self, inbox_dir: Path):
        self.dir = inbox_dir
        self.dir.mkdir(parents=True, exist_ok=True)
```

**为什么用 JSONL（JSON Lines）而不是普通 JSON？**

```
普通 JSON 的问题:
  读: json.loads(file.read())     ← 要读整个文件
  写: 先读 → 解析 → append → 序列化 → 写回   ← 不安全！两个线程同时写会冲突

JSONL 的优势:
  读: 逐行解析，每行独立
  写: open("file.jsonl", "a").write(json_line + "\n")  ← append 模式，原子性更好
  清空: open("file.jsonl", "w").close()  ← 简单的 drain
```

`send()` 方法 -- 往目标收件箱追加一行：

```python
def send(self, sender: str, to: str, content: str,
         msg_type: str = "message", extra: dict = None) -> str:
    if msg_type not in VALID_MSG_TYPES:
        return f"Error: Invalid type '{msg_type}'. Valid: {VALID_MSG_TYPES}"
    msg = {
        "type": msg_type,        # message/broadcast/shutdown_request/...
        "from": sender,          # 谁发的
        "content": content,      # 消息内容
        "timestamp": time.time(),# 时间戳
    }
    inbox_path = self.dir / f"{to}.jsonl"
    with open(inbox_path, "a") as f:    # "a" = append 模式
        f.write(json.dumps(msg) + "\n") # 追加一行
    return f"Sent {msg_type} to {to}"
```

`read_inbox()` 方法 -- **读取并清空**（drain-on-read）：

```python
def read_inbox(self, name: str) -> list:
    inbox_path = self.dir / f"{name}.jsonl"
    if not inbox_path.exists():
        return []
    messages = []
    for line in inbox_path.read_text().strip().splitlines():
        if line:
            messages.append(json.loads(line))
    inbox_path.write_text("")  # ← drain! 读完就清空
    return messages
```

**为什么 drain-on-read？**

```
不清空的问题:
  第1轮: 读到消息 A, B
  第2轮: 读到消息 A, B, C  ← A, B 被重复处理了！

drain-on-read:
  第1轮: 读到消息 A, B → 清空
  第2轮: 读到消息 C      ← 只有新消息

这是一个简单的"消费即销毁"模式，类似于消息队列的 acknowledge 机制。
```

`broadcast()` 方法 -- 给除了自己以外的所有队友发消息：

```python
def broadcast(self, sender: str, content: str, teammates: list) -> str:
    count = 0
    for name in teammates:
        if name != sender:  # 不给自己发
            self.send(sender, name, content, "broadcast")
            count += 1
    return f"Broadcast to {count} teammates"
```

### 3.2 TeammateManager: 团队名册管理

```python
class TeammateManager:
    def __init__(self, team_dir: Path):
        self.dir = team_dir
        self.dir.mkdir(exist_ok=True)
        self.config_path = self.dir / "config.json"
        self.config = self._load_config()  # 从文件加载
        self.threads = {}                   # name -> Thread
```

**config.json 的结构：**

```json
{
  "team_name": "default",
  "members": [
    {"name": "alice", "role": "coder", "status": "working"},
    {"name": "bob", "role": "tester", "status": "idle"}
  ]
}
```

**为什么把团队信息存在文件里而不是内存里？**

这和 s07 的哲学一致：**文件系统就是协调层**。如果进程崩溃重启，读配置文件就能恢复团队状态。任何 agent（甚至人类）都可以通过读文件了解当前团队组成。

### 3.3 spawn(): 创建队友

```python
def spawn(self, name: str, role: str, prompt: str) -> str:
    member = self._find_member(name)
    if member:
        # 已存在的队友：检查状态，可以重新激活
        if member["status"] not in ("idle", "shutdown"):
            return f"Error: '{name}' is currently {member['status']}"
        member["status"] = "working"
        member["role"] = role
    else:
        # 新队友：加入名册
        member = {"name": name, "role": role, "status": "working"}
        self.config["members"].append(member)
    self._save_config()

    # 在新线程中启动 agent loop
    thread = threading.Thread(
        target=self._teammate_loop,
        args=(name, role, prompt),
        daemon=True,  # daemon=True: 主线程退出时自动清理
    )
    self.threads[name] = thread
    thread.start()
    return f"Spawned '{name}' (role: {role})"
```

**生命周期对比：**

```
s04 子智能体:  spawn → work → return → 消亡 (一次性)
s09 队友:      spawn → WORKING → IDLE → (可再次 spawn) → WORKING → ... → SHUTDOWN

关键区别: s09 的队友可以被"唤醒"——如果 alice 已经 idle，
再 spawn alice 不会创建新的，而是更新她的状态为 working 并启动新任务。
```

### 3.4 _teammate_loop(): 队友的 agent loop

这是每个队友在自己线程里跑的核心循环：

```python
def _teammate_loop(self, name: str, role: str, prompt: str):
    sys_prompt = (
        f"You are '{name}', role: {role}, at {WORKDIR}. "
        f"Use send_message to communicate. Complete your task."
    )
    messages = [{"role": "user", "content": prompt}]
    tools = self._teammate_tools()  # 6 个工具（没有 spawn）

    for _ in range(50):  # 最多 50 轮
        # 每轮开始前检查收件箱
        inbox = BUS.read_inbox(name)
        for msg in inbox:
            messages.append({"role": "user", "content": json.dumps(msg)})

        # 调用 LLM
        response = client.messages.create(
            model=MODEL, system=sys_prompt,
            messages=messages, tools=tools, max_tokens=8000,
        )
        messages.append({"role": "assistant", "content": response.content})

        if response.stop_reason != "tool_use":
            break  # 没有工具调用 → 任务完成

        # 执行工具并收集结果
        results = []
        for block in response.content:
            if block.type == "tool_use":
                output = self._exec(name, block.name, block.input)
                results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": str(output),
                })
        messages.append({"role": "user", "content": results})

    # 循环结束，状态改为 idle
    member = self._find_member(name)
    if member and member["status"] != "shutdown":
        member["status"] = "idle"
        self._save_config()
```

**收件箱检查的时机非常重要：**

```
每轮 LLM 调用前都检查收件箱:

  轮1: [检查收件箱 → 空] → LLM 调用 → 执行工具
  轮2: [检查收件箱 → 有消息!] → 注入消息 → LLM 调用 → 执行工具
  轮3: [检查收件箱 → 空] → LLM 调用 → 返回文本 → 退出

这意味着队友不是"实时"收到消息的，而是在每轮工作间隙查看。
类似于你每隔几分钟看一次邮件，而不是推送通知。
```

---

## 4. 工具系统：9 vs 6

### Lead（组长）的 9 个工具

| # | 工具名 | 说明 | 来源 |
|---|--------|------|------|
| 1 | bash | 运行 shell 命令 | s02 |
| 2 | read_file | 读文件 | s02 |
| 3 | write_file | 写文件 | s02 |
| 4 | edit_file | 编辑文件 | s02 |
| 5 | **spawn_teammate** | 创建/唤醒队友 | **s09 新增** |
| 6 | **list_teammates** | 查看团队名册 | **s09 新增** |
| 7 | **send_message** | 给队友发消息 | **s09 新增** |
| 8 | **read_inbox** | 读自己的收件箱 | **s09 新增** |
| 9 | **broadcast** | 广播给所有队友 | **s09 新增** |

### Teammate（队友）的 6 个工具

| # | 工具名 | 说明 |
|---|--------|------|
| 1 | bash | 运行 shell 命令 |
| 2 | read_file | 读文件 |
| 3 | write_file | 写文件 |
| 4 | edit_file | 编辑文件 |
| 5 | **send_message** | 给其他人发消息 |
| 6 | **read_inbox** | 读自己的收件箱 |

**队友没有的工具**: spawn_teammate, list_teammates, broadcast

**为什么这样设计？**

```
如果队友也能 spawn:
  alice spawn 了 charlie
  bob 也 spawn 了 dave
  charlie 又 spawn 了 eve...
  → 无限增殖，没人控制全局

组长垄断 spawn 权:
  只有组长能决定团队规模和人员配置
  队友专注于执行任务
  → 清晰的层级结构，可控的复杂度
```

---

## 5. 主循环（Lead 的 agent loop）

```python
def agent_loop(messages: list):
    while True:
        # 组长也检查收件箱
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
            model=MODEL, system=SYSTEM,
            messages=messages, tools=TOOLS, max_tokens=8000,
        )
        messages.append({"role": "assistant", "content": response.content})

        if response.stop_reason != "tool_use":
            return  # 回复用户

        # 执行工具
        results = []
        for block in response.content:
            if block.type == "tool_use":
                handler = TOOL_HANDLERS.get(block.name)
                output = handler(**block.input) if handler else f"Unknown tool"
                results.append({...})
        messages.append({"role": "user", "content": results})
```

注意组长的收件箱注入方式和队友不同：用 `<inbox>` XML 标签包裹，并加一个 `"Noted inbox messages."` 的假回复来保持消息交替格式。

---

## 6. 消息类型体系

s09 定义了 5 种消息类型，但目前只实现了前两种，其余留给 s10：

```python
VALID_MSG_TYPES = {
    "message",                 # 普通点对点消息
    "broadcast",               # 广播（发给所有人）
    "shutdown_request",        # 请求优雅关闭 (s10)
    "shutdown_response",       # 同意/拒绝关闭 (s10)
    "plan_approval_response",  # 同意/拒绝计划 (s10)
}
```

**为什么提前定义但不实现？**

这是面向 s10（Guardrails）的接口预留。s10 要做的是"安全关闭"和"计划审批"，需要队友之间能发特殊类型的消息。s09 把消息类型验证机制建好，s10 只需要加处理逻辑。

---

## 7. 文件系统布局

```
.team/
├── config.json              ← 团队名册（谁、什么角色、什么状态）
└── inbox/
    ├── alice.jsonl           ← alice 的收件箱（append-only, drain-on-read）
    ├── bob.jsonl             ← bob 的收件箱
    └── lead.jsonl            ← 组长的收件箱
```

**每个 .jsonl 文件的一行长这样：**

```json
{"type": "message", "from": "alice", "content": "API 接口写好了，你可以开始写测试了", "timestamp": 1711987200.0}
```

---

## 8. 对比 s04 → s08 → s09 的演进

| 维度 | s04 子智能体 | s08 后台任务 | s09 智能体团队 |
|------|-------------|-------------|---------------|
| 生命周期 | 一次性 | 一次性命令 | 持久化（idle ↔ working） |
| 身份 | 无 | task_id | 名字 + 角色 |
| 思考能力 | 有（完整 agent loop） | 无（只跑 shell） | 有（每个队友独立 agent loop） |
| 通信 | 返回摘要给主 agent | 完成通知 | JSONL 收件箱（任意对任意） |
| 持久化 | 无 | 内存中的 dict | config.json + JSONL 文件 |
| 线程 | 不一定 | daemon thread | daemon thread |
| 工具数 | 继承主 agent | 无 | 6 个（精简集） |
| 协调方式 | 主→子→主（星形） | 主→后台→通知 | 网状（任意→任意） |

---

## 9. 实际执行流程示例

用户输入: `Spawn alice (coder) and bob (tester). Have alice send bob a message.`

```
时间线:

t0: 用户输入
    └→ Lead agent 收到请求

t1: Lead 调用 spawn_teammate(name="alice", role="coder", prompt="...")
    └→ TeammateManager 创建 alice，config.json 更新
    └→ 新线程启动: alice 的 agent loop 开始

t2: Lead 调用 spawn_teammate(name="bob", role="tester", prompt="...")
    └→ TeammateManager 创建 bob，config.json 更新
    └→ 新线程启动: bob 的 agent loop 开始

t3: Lead 调用 send_message(to="alice", content="请给 bob 发消息说...")
    └→ alice.jsonl << {"type":"message","from":"lead","content":"..."}

t4: [alice 线程] 检查收件箱 → 读到 lead 的消息
    └→ alice 调用 send_message(to="bob", content="嘿 bob，准备好测试了吗？")
    └→ bob.jsonl << {"type":"message","from":"alice","content":"..."}

t5: [bob 线程] 检查收件箱 → 读到 alice 的消息
    └→ bob 处理消息...

t6: alice 完成任务 → status = "idle"
t7: bob 完成任务 → status = "idle"
```

---

## 10. 潜在问题与权衡

### 并发写入

```
如果 alice 和 bob 同时给 lead 发消息:
  alice: open("lead.jsonl", "a").write(msg_a)
  bob:   open("lead.jsonl", "a").write(msg_b)

JSONL append 模式在大多数操作系统上对小写入是原子的，
但严格来说不保证。生产环境应该加文件锁。
s09 选择简单方案，因为这是教学代码。
```

### 消息丢失风险

```
drain-on-read 的问题:
  1. read_inbox 读取文件
  2. 在"读取"和"清空"之间，有新消息写入
  3. 清空操作把新消息也删了

解决方案: 可以用临时文件 rename 来实现原子操作，
但 s09 选择了简单方案。对于教学目的足够了。
```

### 上下文膨胀

```
每个队友的 messages 列表只增不减:
  轮1: [prompt]
  轮2: [prompt, inbox_msg, response, tool_result]
  轮3: [prompt, inbox_msg, response, tool_result, inbox_msg, response, tool_result]
  ...

50 轮下来可能超过 max_tokens。
s06 的 context compaction 技术可以用在这里，但 s09 没有实现。
```

---

## 11. 与 Claude Code 的对应

Claude Code 中的 Agent 工具就是 s09 概念的生产化版本：

| s09 概念 | Claude Code 对应 |
|----------|-----------------|
| spawn_teammate | Agent tool（启动子 agent） |
| JSONL 收件箱 | 内部消息传递机制 |
| config.json | 内部 agent 注册表 |
| 队友工具子集 | subagent_type 决定工具集 |
| Lead 的 9 工具 | 主 agent 的完整工具集 |
| daemon thread | 后台 agent 进程 |

---

## 12. 关键收获

1. **持久化身份是团队协作的基础** -- 没有身份就没有问责，没有记忆传递，没有"继续上次的工作"。

2. **文件系统是最简单的协调层** -- config.json + JSONL 比消息队列、RPC、共享内存都简单，且人类可读可调试。

3. **职责分离通过工具控制实现** -- 组长有 spawn 权，队友没有。工具集就是权限系统。

4. **消息是非实时的（轮询模式）** -- 队友在每轮 LLM 调用前检查收件箱，不是推送通知。这简化了实现但引入了延迟。

5. **drain-on-read 保证消息只被处理一次** -- 读完就清空，简单有效。

---

## 13. 试一试

```sh
cd learn-claude-code
python agents/s09_agent_teams.py
```

试试这些 prompt:

1. `Spawn alice (coder) and bob (tester). Have alice send bob a message.`
2. `Broadcast "status update: phase 1 complete" to all teammates`
3. `Check the lead inbox for any messages`
4. `/team` -- 查看团队名册和状态
5. `/inbox` -- 手动检查组长的收件箱
