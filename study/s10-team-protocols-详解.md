# s10: Team Protocols (团队协议) -- 详细讲解

`s01 > s02 > s03 > s04 > s05 > s06 | s07 > s08 > s09 > [ s10 ] s11 > s12`

> *"队友之间要有统一的沟通规矩"* -- 一个 request-response 模式驱动所有协商。
>
> **Harness 层**: 协议 -- 模型之间的结构化握手。

这是系列的**第十课**，也是**第三阶段（持久化与后台）**的第四课。s09 让多个 agent 能协作，但缺少结构化的协调机制。s10 要回答的核心问题是：**当团队需要做关键决策（关机、审批）时，怎么保证过程可追踪、可审计、不会出乱子？** 答案是：**基于 request_id 关联的请求-响应协议 + 有限状态机（FSM）**。

---

## 1. 问题：为什么需要协议？

### 回顾 s09 的不足

s09 实现了团队通信（JSONL 收件箱），但通信是"自由"的 -- 没有结构、没有流程、没有追踪：

**问题 1：直接杀线程会出事**

```
s09 的关机方式:
  用户: "alice 干完了，关掉她"
  做法: 直接让 alice 的线程自然退出（loop 结束）

  但如果 alice 正在写文件呢？
    t1: alice 开始写 config.json（写了一半）
    t2: 线程被终止
    t3: config.json 是损坏的 ← 灾难！

  需要的是"优雅关机":
    t1: 领导请求 alice 关机
    t2: alice 收到请求，检查自己的状态
    t3: alice 完成手头工作，回复"OK 我可以关了"
    t4: alice 线程正常退出
```

**问题 2：高风险操作没有审批**

```
s09 的做法:
  用户: "让 bob 重构认证模块"
  bob: 收到！开干！（直接开始改核心代码）

  问题: 如果 bob 的重构方案有问题呢？
  - 删了不该删的代码
  - 引入了安全漏洞
  - 和 alice 的工作冲突

  需要的是"计划审批":
    t1: bob 提交重构计划
    t2: 领导审查计划内容
    t3: 领导批准 → bob 开始执行
    或: 领导拒绝 → bob 修改方案
```

### 两个问题的共同本质

仔细看这两个场景，它们的结构完全一样：

```
关机协议:                     计划审批协议:
  发起方: 领导                  发起方: 队友
  接收方: 队友                  接收方: 领导
  请求:   "请关机"              请求:   "请审批这个计划"
  响应:   approve/reject        响应:   approve/reject
  追踪:   request_id            追踪:   request_id

共同模式 = 请求-响应 + ID 关联 + 状态机
```

这就是 s10 的核心洞察：**一套 FSM 模式，套用两个领域**。

---

## 2. 架构总览

```
                         ┌─────────────────────────────┐
                         │      用户 (stdin/stdout)      │
                         └─────────┬───────────────────┘
                                   │
                         ┌─────────▼───────────────────┐
                         │      Lead Agent (主线程)      │
                         │   12 tools (+3 from s09)     │
                         │   shutdown_request            │
                         │   shutdown_response (check)   │
                         │   plan_approval (review)      │
                         └──┬──────────┬───────────────┘
                            │          │
             spawn("alice") │          │ spawn("bob")
                            │          │
                   ┌────────▼──┐   ┌──▼────────┐
                   │  alice    │   │   bob      │
                   │  Thread   │   │   Thread   │
                   │  8 tools  │   │  8 tools   │
                   │ +shutdown │   │ +shutdown  │
                   │  response │   │  response  │
                   │ +plan     │   │ +plan      │
                   │  approval │   │  approval  │
                   └────────┬──┘   └──┬────────┘
                            │         │
                            ▼         ▼
                    .team/inbox/alice.jsonl
                    .team/inbox/bob.jsonl
                    .team/inbox/lead.jsonl
                    .team/config.json

    ┌──────────────────────────────────┐
    │        Request Trackers          │
    │  shutdown_requests = {req_id: {  │
    │    target, status                │
    │  }}                              │
    │  plan_requests = {req_id: {      │
    │    from, plan, status            │
    │  }}                              │
    └──────────────────────────────────┘
```

**相对 s09 的变化**: Lead 从 9 个工具增加到 12 个，队友从 6 个增加到 8 个。新增了请求追踪器（tracker）和线程锁。

---

## 3. 核心概念：请求-响应协议

### 3.1 什么是 request_id 关联？

在分布式系统中，请求和响应可能不是连续发生的。request_id 的作用是把它们关联起来：

```
没有 request_id 的世界:
  领导: "alice 请关机"
  领导: "bob 请关机"
  ...过了一会儿...
  收到响应: "approve"  ← 这是 alice 的还是 bob 的？？？

有 request_id 的世界:
  领导: "alice 请关机" (req_id: abc)
  领导: "bob 请关机"   (req_id: xyz)
  ...过了一会儿...
  收到响应: {req_id: "abc", approve: true}  ← 明确是 alice 的响应
  收到响应: {req_id: "xyz", approve: false} ← 明确是 bob 拒绝了
```

这在现实世界随处可见：

```
类比:
  HTTP:        request_id ≈ Connection ID + 序列号
  数据库事务:   request_id ≈ transaction ID
  快递:        request_id ≈ 快递单号
  银行转账:    request_id ≈ 交易流水号
```

### 3.2 有限状态机（FSM）

每个请求都有一个简单的三状态 FSM：

```
   ┌─────────┐
   │ pending  │  ← 刚创建，等待响应
   └────┬─────┘
        │
   ┌────┴────┐
   │         │
   ▼         ▼
┌──────┐  ┌──────┐
│approved│  │rejected│
└──────┘  └──────┘

状态转换规则:
  - pending → approved  (响应方同意)
  - pending → rejected  (响应方拒绝)
  - approved/rejected → ??? (终态，不可变)

只有三个状态，只有两种转换。简单到不可能出错。
```

### 3.3 请求追踪器（Tracker）

追踪器是一个字典，key 是 request_id，value 是请求的元数据和状态：

```python
# 关机请求追踪器
shutdown_requests = {}
# 例: {"abc123": {"target": "alice", "status": "pending"}}

# 计划审批追踪器
plan_requests = {}
# 例: {"xyz789": {"from": "bob", "plan": "重构认证模块...", "status": "pending"}}

# 线程锁：因为多个线程可能同时读写追踪器
_tracker_lock = threading.Lock()
```

**为什么需要线程锁？**

```
没有锁的问题:
  线程 A（alice）: 读取 shutdown_requests["abc"] → status = "pending"
  线程 B（lead） : 同时读取同一条 → status = "pending"
  线程 A: 写入 status = "approved"
  线程 B: 写入 status = "rejected"  ← 覆盖了 A 的写入！

  最终状态取决于谁最后写入，这是经典的竞态条件 (race condition)

有锁:
  线程 A: 获取锁 → 读+写 → 释放锁
  线程 B: 等待锁... → 获取锁 → 读+写 → 释放锁
  → 操作串行化，状态一致
```

---

## 4. 关机协议：逐行拆解

### 4.1 领导发起关机请求

```python
def handle_shutdown_request(teammate: str) -> str:
    req_id = str(uuid.uuid4())[:8]               # 1. 生成唯一 ID（取 UUID 前 8 位）
    with _tracker_lock:                            # 2. 加锁
        shutdown_requests[req_id] = {              # 3. 在追踪器中记录
            "target": teammate,
            "status": "pending"
        }
    BUS.send(                                      # 4. 通过收件箱发送请求
        "lead", teammate, "Please shut down gracefully.",
        "shutdown_request", {"request_id": req_id},
    )
    return f"Shutdown request {req_id} sent to '{teammate}' (status: pending)"
```

**执行流程：**

```
handle_shutdown_request("alice")

  ① uuid.uuid4() → "a1b2c3d4-e5f6-..."
     取前 8 位 → "a1b2c3d4"

  ② shutdown_requests["a1b2c3d4"] = {
       "target": "alice",
       "status": "pending"
     }

  ③ alice.jsonl 追加一行:
     {
       "type": "shutdown_request",
       "from": "lead",
       "content": "Please shut down gracefully.",
       "timestamp": 1711987200.0,
       "request_id": "a1b2c3d4"
     }

  ④ 返回: "Shutdown request a1b2c3d4 sent to 'alice' (status: pending)"
```

### 4.2 队友响应关机请求

队友的 `_exec` 方法中处理 `shutdown_response` 工具：

```python
if tool_name == "shutdown_response":
    req_id = args["request_id"]        # 1. 从参数取回 request_id
    approve = args["approve"]           # 2. 取决定：同意还是拒绝
    with _tracker_lock:                 # 3. 加锁更新追踪器
        if req_id in shutdown_requests:
            shutdown_requests[req_id]["status"] = (
                "approved" if approve else "rejected"
            )
    BUS.send(                           # 4. 发送响应到领导收件箱
        sender, "lead", args.get("reason", ""),
        "shutdown_response",
        {"request_id": req_id, "approve": approve},
    )
    return f"Shutdown {'approved' if approve else 'rejected'}"
```

### 4.3 关键：should_exit 标志

队友的 agent loop 中有一个特殊处理：

```python
def _teammate_loop(self, name, role, prompt):
    # ...
    should_exit = False                    # ← 退出标志
    for _ in range(50):
        inbox = BUS.read_inbox(name)
        for msg in inbox:
            messages.append(...)
        if should_exit:                    # ← 如果已批准关机，在处理完收件箱后退出
            break
        # ... LLM 调用 ...
        for block in response.content:
            if block.type == "tool_use":
                output = self._exec(name, block.name, block.input)
                # ...
                if block.name == "shutdown_response" and block.input.get("approve"):
                    should_exit = True     # ← 批准了关机，设置标志
        # ...

    # 循环结束后更新状态
    member = self._find_member(name)
    if member:
        member["status"] = "shutdown" if should_exit else "idle"
        self._save_config()
```

**为什么不立即退出，而是设标志？**

```
立即退出的问题:
  轮 N:
    1. LLM 调用 → 返回 [shutdown_response(approve=True), write_file("report.txt", ...)]
    2. 执行 shutdown_response → 立即 break！
    3. write_file 没有执行 → 报告丢失！

should_exit 标志:
  轮 N:
    1. LLM 调用 → 返回 [shutdown_response(approve=True), write_file("report.txt", ...)]
    2. 执行 shutdown_response → should_exit = True
    3. 继续执行 write_file → 报告已保存 ✓
    4. 本轮工具全部执行完毕
  轮 N+1:
    1. 检查收件箱（处理最后的消息）
    2. if should_exit: break ← 现在安全退出
    3. 状态更新为 "shutdown"

核心原则: 完成当前轮的所有工具调用，再在下一轮开始时干净退出。
```

### 4.4 完整关机流程时序图

```
时间  |  领导 (Lead)          |  队友 (alice)              |  追踪器状态
------+----------------------+---------------------------+------------------
t0    | shutdown_request      |                           |
      | → req_id = "abc"     |                           | abc: pending
      | → alice.jsonl << msg |                           |
------+----------------------+---------------------------+------------------
t1    |                      | (轮开始) 检查收件箱        |
      |                      | 读到 shutdown_request     |
      |                      | req_id = "abc"            |
------+----------------------+---------------------------+------------------
t2    |                      | LLM 决定: 我可以关了       |
      |                      | → shutdown_response       |
      |                      |   {req_id:"abc",          |
      |                      |    approve:true}          | abc: approved
      |                      | → lead.jsonl << response  |
      |                      | should_exit = True        |
------+----------------------+---------------------------+------------------
t3    |                      | (下一轮) 检查收件箱        |
      |                      | if should_exit: break     |
      |                      | status → "shutdown"       |
------+----------------------+---------------------------+------------------
t4    | 读收件箱              |                           |
      | 看到 alice 同意了      |  (线程已结束)             |
```

---

## 5. 计划审批协议：逐行拆解

### 5.1 队友提交计划

```python
if tool_name == "plan_approval":
    plan_text = args.get("plan", "")
    req_id = str(uuid.uuid4())[:8]            # 1. 队友这边生成 request_id
    with _tracker_lock:
        plan_requests[req_id] = {              # 2. 在追踪器中记录
            "from": sender,
            "plan": plan_text,
            "status": "pending"
        }
    BUS.send(                                  # 3. 发送计划到领导收件箱
        sender, "lead", plan_text,
        "plan_approval_response",
        {"request_id": req_id, "plan": plan_text},
    )
    return f"Plan submitted (request_id={req_id}). Waiting for lead approval."
```

**注意方向的反转：**

```
关机协议:     领导 → 队友 → 领导     (领导发起)
计划审批:     队友 → 领导 → 队友     (队友发起)

但 FSM 完全一样: pending → approved | rejected
追踪器结构一样: {req_id: {metadata, status}}
消息模式一样:   request + response + ID 关联
```

### 5.2 领导审查计划

```python
def handle_plan_review(request_id: str, approve: bool, feedback: str = "") -> str:
    with _tracker_lock:
        req = plan_requests.get(request_id)    # 1. 通过 request_id 找到计划
    if not req:
        return f"Error: Unknown plan request_id '{request_id}'"
    with _tracker_lock:
        req["status"] = "approved" if approve else "rejected"  # 2. 更新状态
    BUS.send(                                  # 3. 发送审批结果到队友收件箱
        "lead", req["from"], feedback,
        "plan_approval_response",
        {"request_id": request_id, "approve": approve, "feedback": feedback},
    )
    return f"Plan {req['status']} for '{req['from']}'"
```

### 5.3 完整计划审批流程

```
时间  |  队友 (bob)             |  领导 (Lead)             |  追踪器状态
------+------------------------+-------------------------+------------------
t0    | plan_approval          |                         |
      | plan="重构认证模块:     |                         | xyz: pending
      |  1. 提取接口            |                         |
      |  2. 替换实现"           |                         |
      | → req_id = "xyz"       |                         |
      | → lead.jsonl << msg    |                         |
------+------------------------+-------------------------+------------------
t1    |                        | 读收件箱                 |
      |                        | 看到 bob 的计划          |
      |                        | 审查内容...              |
------+------------------------+-------------------------+------------------
t2    | (等待中)               | plan_approval            |
      |                        |  {req_id:"xyz",          | xyz: approved
      |                        |   approve:true,          |
      |                        |   feedback:"LGTM"}       |
      |                        | → bob.jsonl << response  |
------+------------------------+-------------------------+------------------
t3    | 读收件箱                |                         |
      | 看到 approved + LGTM   |                         |
      | 开始执行重构!           |                         |
```

---

## 6. 工具系统变化：9 → 12 和 6 → 8

### Lead（组长）的 12 个工具

| # | 工具名 | 说明 | 来源 |
|---|--------|------|------|
| 1 | bash | 运行 shell 命令 | s02 |
| 2 | read_file | 读文件 | s02 |
| 3 | write_file | 写文件 | s02 |
| 4 | edit_file | 编辑文件 | s02 |
| 5 | spawn_teammate | 创建/唤醒队友 | s09 |
| 6 | list_teammates | 查看团队名册 | s09 |
| 7 | send_message | 给队友发消息 | s09 |
| 8 | read_inbox | 读自己的收件箱 | s09 |
| 9 | broadcast | 广播给所有队友 | s09 |
| 10 | **shutdown_request** | 请求队友关机 | **s10 新增** |
| 11 | **shutdown_response** | 检查关机请求状态 | **s10 新增** |
| 12 | **plan_approval** | 审批队友的计划 | **s10 新增** |

### Teammate（队友）的 8 个工具

| # | 工具名 | 说明 | 来源 |
|---|--------|------|------|
| 1 | bash | 运行 shell 命令 | s02 |
| 2 | read_file | 读文件 | s02 |
| 3 | write_file | 写文件 | s02 |
| 4 | edit_file | 编辑文件 | s02 |
| 5 | send_message | 给其他人发消息 | s09 |
| 6 | read_inbox | 读自己的收件箱 | s09 |
| 7 | **shutdown_response** | 响应关机请求（approve/reject）| **s10 新增** |
| 8 | **plan_approval** | 提交计划等待审批 | **s10 新增** |

**注意同名工具的不同行为：**

```
"shutdown_response" 在领导端 vs 队友端:
  领导: 检查状态 → _check_shutdown_status(req_id) → 返回 JSON 状态
  队友: 发送响应 → 更新追踪器 + 发送消息到 lead 收件箱

"plan_approval" 在领导端 vs 队友端:
  领导: 审批计划 → handle_plan_review(req_id, approve, feedback)
  队友: 提交计划 → 生成 req_id + 记录追踪器 + 发送到 lead 收件箱

同一个工具名，两种角色，两种行为。
这是通过在 _exec (队友) 和 TOOL_HANDLERS (领导) 中分别定义实现的。
```

---

## 7. 领导端工具分派：TOOL_HANDLERS 字典

s10 用一个字典替代了 if-elif 链来分派工具调用：

```python
TOOL_HANDLERS = {
    "bash":              lambda **kw: _run_bash(kw["command"]),
    "read_file":         lambda **kw: _run_read(kw["path"], kw.get("limit")),
    "write_file":        lambda **kw: _run_write(kw["path"], kw["content"]),
    "edit_file":         lambda **kw: _run_edit(kw["path"], kw["old_text"], kw["new_text"]),
    "spawn_teammate":    lambda **kw: TEAM.spawn(kw["name"], kw["role"], kw["prompt"]),
    "list_teammates":    lambda **kw: TEAM.list_all(),
    "send_message":      lambda **kw: BUS.send("lead", kw["to"], kw["content"], ...),
    "read_inbox":        lambda **kw: json.dumps(BUS.read_inbox("lead"), indent=2),
    "broadcast":         lambda **kw: BUS.broadcast("lead", kw["content"], ...),
    "shutdown_request":  lambda **kw: handle_shutdown_request(kw["teammate"]),
    "shutdown_response": lambda **kw: _check_shutdown_status(kw.get("request_id", "")),
    "plan_approval":     lambda **kw: handle_plan_review(kw["request_id"], kw["approve"], ...),
}
```

**为什么用字典而不是 if-elif？**

```
if-elif 方式 (s09):
  if tool_name == "bash": ...
  elif tool_name == "read_file": ...
  elif tool_name == "write_file": ...
  ...12 个分支，越来越长

字典方式 (s10):
  handler = TOOL_HANDLERS.get(block.name)
  output = handler(**block.input) if handler else "Unknown tool"

优势:
  1. 查找是 O(1)，不是 O(n)
  2. 新增工具只需加一行，不需要改控制流
  3. 工具列表一目了然
  4. 可以动态增删工具
```

---

## 8. 队友系统提示的变化

```python
# s09 的系统提示:
sys_prompt = (
    f"You are '{name}', role: {role}, at {WORKDIR}. "
    f"Use send_message to communicate. Complete your task."
)

# s10 的系统提示:
sys_prompt = (
    f"You are '{name}', role: {role}, at {WORKDIR}. "
    f"Submit plans via plan_approval before major work. "    # ← 新增: 要求先提交计划
    f"Respond to shutdown_request with shutdown_response."   # ← 新增: 要求响应关机请求
)
```

这两行提示非常关键 -- 它们让 LLM **知道协议的存在**：

```
没有提示的话:
  LLM 收到 shutdown_request 消息 → 不知道该用什么工具响应
  LLM 想开始工作 → 直接用 bash/write_file，不知道要先提交计划

有提示:
  LLM 收到 shutdown_request → "哦，我应该用 shutdown_response 来回复"
  LLM 想开始工作 → "等等，我应该先用 plan_approval 提交计划"

系统提示 = 协议的执行保障
工具定义 = 协议的能力边界
```

---

## 9. 一套模式，两种用途

这是 s10 最重要的设计哲学。让我们抽象出通用模式：

```
通用请求-响应协议:
  ┌─────────────┐
  │  发起方       │
  │  生成 req_id  │──── request(req_id, payload) ────▶ ┌──────────────┐
  │  tracker[id]  │                                     │   接收方      │
  │  = pending    │◀── response(req_id, approve) ──── │   决策+响应    │
  │  → approved   │                                     └──────────────┘
  │    /rejected  │
  └─────────────┘

应用到关机:
  发起方 = 领导
  接收方 = 队友
  payload = "请关机"
  追踪器 = shutdown_requests

应用到计划审批:
  发起方 = 队友
  接收方 = 领导
  payload = 计划文本
  追踪器 = plan_requests

未来可以轻松扩展:
  代码审查: 发起方=队友, 接收方=领导, payload=代码差异
  资源申请: 发起方=队友, 接收方=领导, payload=资源需求
  任务委派: 发起方=领导, 接收方=队友, payload=任务描述
```

**协议的四个组成部分：**

| 组成部分 | 关机协议 | 计划审批 | 通用模式 |
|---------|---------|---------|---------|
| 唯一标识 | uuid[:8] | uuid[:8] | request_id |
| 状态机 | pending→approved/rejected | 同左 | FSM |
| 追踪器 | shutdown_requests{} | plan_requests{} | dict{req_id: metadata} |
| 通信 | BUS.send + 收件箱 | 同左 | 消息传递 |

---

## 10. 相对 s09 的完整变更清单

| 组件 | s09 | s10 | 变化原因 |
|------|-----|-----|---------|
| Lead 工具数 | 9 | 12 | +shutdown_req/resp +plan_approval |
| 队友工具数 | 6 | 8 | +shutdown_response +plan_approval |
| 关机方式 | 自然退出 | 请求-响应握手 | 防止写到一半被杀 |
| 计划门控 | 无 | 提交→审查→审批 | 防止高风险操作无人审查 |
| 请求关联 | 无 | request_id (uuid[:8]) | 多个请求同时进行时不混乱 |
| 状态追踪 | 无 | FSM: pending→approved/rejected | 可审计 |
| 线程安全 | 无锁 | _tracker_lock | 多线程写追踪器需要同步 |
| 工具分派 | if-elif | TOOL_HANDLERS 字典 | 12 个工具用字典更清晰 |
| 队友退出 | loop 自然结束 | should_exit 标志 | 确保当前轮工具全部执行完 |
| 队友状态 | idle/working | idle/working/shutdown | 新增 shutdown 终态 |
| 系统提示 | 简单任务描述 | 包含协议指引 | LLM 需要知道协议的存在 |

---

## 11. 消息类型体系（完整版）

s09 提前定义了 5 种消息类型，s10 全部用上：

```python
VALID_MSG_TYPES = {
    "message",                 # 普通点对点消息 (s09 实现)
    "broadcast",               # 广播 (s09 实现)
    "shutdown_request",        # 关机请求 (s10 实现)
    "shutdown_response",       # 关机响应 (s10 实现)
    "plan_approval_response",  # 计划审批响应 (s10 实现)
}
```

**消息流向矩阵：**

```
消息类型              发送方 → 接收方    触发时机
─────────────────────────────────────────────────
message              任意 → 任意        日常沟通
broadcast            领导 → 全体        团队公告
shutdown_request     领导 → 队友        领导发起关机
shutdown_response    队友 → 领导        队友响应关机
plan_approval_response 双向            队友提交/领导审批
```

---

## 12. 实际执行流程示例

### 示例 1: 完整的关机流程

用户输入: `Spawn alice as a coder. Then request her shutdown.`

```
t0: 用户输入
    └→ Lead agent 收到请求

t1: Lead 调用 spawn_teammate(name="alice", role="coder", prompt="...")
    └→ config.json 更新: alice, status=working
    └→ 新线程启动: alice 的 agent loop 开始

t2: alice 开始执行任务...

t3: Lead 调用 shutdown_request(teammate="alice")
    └→ req_id = "a1b2c3d4"
    └→ shutdown_requests["a1b2c3d4"] = {target:"alice", status:"pending"}
    └→ alice.jsonl << shutdown_request 消息

t4: [alice 线程] 下一轮开始，检查收件箱
    └→ 读到 shutdown_request
    └→ LLM 决定: "我可以关了"
    └→ 调用 shutdown_response(req_id="a1b2c3d4", approve=true)
    └→ shutdown_requests["a1b2c3d4"]["status"] = "approved"
    └→ lead.jsonl << shutdown_response 消息
    └→ should_exit = True

t5: [alice 线程] 下一轮开始
    └→ if should_exit: break
    └→ config.json 更新: alice, status=shutdown
    └→ 线程结束

t6: Lead 读收件箱，看到 alice 已关机
```

### 示例 2: 计划被拒绝

用户输入: `Spawn bob with a risky refactoring task. Review and reject his plan.`

```
t0: Lead spawn bob

t1: [bob 线程] bob 的系统提示说要先提交计划
    └→ bob 调用 plan_approval(plan="删除旧的认证模块，替换为新实现...")
    └→ req_id = "x7y8z9"
    └→ plan_requests["x7y8z9"] = {from:"bob", plan:"...", status:"pending"}
    └→ lead.jsonl << plan 消息

t2: Lead 读收件箱，看到 bob 的计划
    └→ Lead（或用户通过 Lead）审查内容
    └→ 调用 plan_approval(req_id="x7y8z9", approve=false, feedback="太激进了，分步来")
    └→ plan_requests["x7y8z9"]["status"] = "rejected"
    └→ bob.jsonl << rejection 消息

t3: [bob 线程] 检查收件箱，读到拒绝和反馈
    └→ LLM 调整方案...
    └→ 可以再次调用 plan_approval 提交修改后的计划
```

---

## 13. 潜在问题与权衡

### 请求超时

```
如果队友挂了，永远不会响应关机请求:
  shutdown_requests["abc"] 永远停在 "pending"

s10 没有实现超时机制。生产环境应该加:
  - 创建请求时记录时间戳
  - 定期检查超时的请求
  - 超时后自动标记为 "timeout" 并强制清理
```

### 单点追踪器

```
shutdown_requests 和 plan_requests 都在内存中:
  - 进程崩溃 → 所有追踪状态丢失
  - 无法分布式部署

生产改进: 把追踪器也持久化到文件（类似 config.json）
```

### 协议执行依赖 LLM 遵守

```
系统提示说"先提交计划再开始工作"，但 LLM 可能直接跳过:
  bob 收到 "重构认证模块" → 直接开始 bash + write_file → 没提交计划

s10 的协议是"建议性的"（advisory），不是"强制性的"（enforced）。
真正强制需要在工具执行层加逻辑:
  - 如果队友没有 pending 的 approved plan，拒绝执行 bash/write_file
  - 但这会大大增加复杂度
```

---

## 14. 对比现实世界的协议设计

| 概念 | s10 实现 | 现实世界类比 |
|------|---------|-------------|
| request_id | uuid[:8] | HTTP Request-Id, 数据库事务 ID |
| FSM | pending→approved/rejected | HTTP 状态码 (100→200/400) |
| 追踪器 | Python dict | 数据库表 + 状态字段 |
| 收件箱通信 | JSONL 文件 | 消息队列 (Kafka, RabbitMQ) |
| 关机协议 | shutdown_request/response | SIGTERM + graceful shutdown |
| 计划审批 | plan_approval | Pull Request review |
| 线程锁 | threading.Lock | 分布式锁 (Redis, Zookeeper) |

---

## 15. 与 Claude Code 的对应

| s10 概念 | Claude Code 对应 |
|----------|-----------------|
| shutdown 协议 | Agent 生命周期管理（优雅终止） |
| plan_approval | EnterPlanMode / ExitPlanMode（计划审批流程） |
| request_id 关联 | tool_use_id（工具调用和结果的关联） |
| FSM 状态追踪 | TodoWrite 的 pending/in_progress/completed |
| 系统提示中的协议指引 | 系统提示中的 safety rules 和 action guidelines |
| 线程锁 | 并发工具调用的内部同步机制 |

---

## 16. 关键收获

1. **一套 FSM，多种用途** -- pending→approved/rejected 这个简单状态机可以套用到任何需要"请求-响应"的场景。不要为每种协议发明新的状态管理。

2. **request_id 是分布式协调的基石** -- 在异步世界里，请求和响应之间可能隔着很长时间。唯一 ID 是把它们关联起来的唯一可靠方式。

3. **优雅退出比强制杀掉重要得多** -- should_exit 标志保证当前轮工具全部执行完才退出。写到一半的文件比多跑几秒钟的线程代价大得多。

4. **协议需要显式告知 LLM** -- 光定义工具不够，必须在系统提示中明确说明协议规则，LLM 才会遵守。

5. **同名工具可以有不同实现** -- shutdown_response 在领导端查状态，在队友端发响应。角色不同，行为不同。

6. **线程安全不是可选的** -- 多线程写共享状态（追踪器）必须加锁，否则竞态条件会导致数据不一致。

---

## 17. 试一试

```sh
cd learn-claude-code
python agents/s10_team_protocols.py
```

试试这些 prompt (英文 prompt 对 LLM 效果更好, 也可以用中文):

1. `Spawn alice as a coder. Then request her shutdown.`
2. `List teammates to see alice's status after shutdown approval`
3. `Spawn bob with a risky refactoring task. Review and reject his plan.`
4. `Spawn charlie, have him submit a plan, then approve it.`
5. 输入 `/team` 监控状态
6. 输入 `/inbox` 手动检查组长的收件箱
