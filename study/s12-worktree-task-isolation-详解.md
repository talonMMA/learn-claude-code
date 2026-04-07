# s12: Worktree + Task Isolation (Worktree 任务隔离) -- 详细讲解

`s01 > s02 > s03 > s04 > s05 > s06 | s07 > s08 > s09 > s10 > s11 > [ s12 ]`

> *"各干各的目录, 互不干扰"* -- 任务管目标, worktree 管目录, 按 ID 绑定。
>
> **Harness 层**: 目录隔离 -- 永不碰撞的并行执行通道。

这是系列的**第十二课**，也是**第四阶段（团队协作与自治）**的最后一课。s11 实现了自治（队友自动认领任务），但所有人仍共享一个工作目录。s12 要回答的核心问题是：**多个智能体同时改代码，怎么不互相踩脚？** 答案是：**控制平面（任务板） + 执行平面（worktree）分离，用 task_id 绑定。**

---

## 1. 问题：为什么需要目录隔离？

### 回顾 s11 的不足

s11 已经有了自治队友，能自动认领任务。但有一个致命的假设：所有人在同一目录工作。

**问题：共享目录导致文件冲突**

```
s11 的场景:
  任务 #1: "重构 auth 模块"     → alice 认领
  任务 #2: "优化 login UI"      → bob 认领

  两人都在同一个工作目录下:
    /project/
      ├── config.py          ← alice 要改这个文件
      ├── auth.py            ← alice 在改
      ├── login.py           ← bob 在改
      └── config.py          ← bob 也要改这个文件！

  时间线:
    t1: alice 修改 config.py，加了 AUTH_SECRET 配置
    t2: bob 修改 config.py，加了 LOGIN_TIMEOUT 配置
    t3: alice 的改动被 bob 覆盖了！AUTH_SECRET 消失了
    t4: alice 运行测试 → 失败 → 不知道为什么

  更糟糕的是 git 状态:
    两个人的未提交改动混在一起
    git diff 看到的是两个人的混合修改
    无法干净地 git stash / git reset 某一个人的改动
    无法独立回滚某个任务
```

**问题的本质：任务板管"做什么"，但不管"在哪做"**

```
s11 的架构缺陷:

  控制平面 (.tasks/)              执行平面（???）
  ┌──────────────┐              ┌──────────────────┐
  │ task_1.json   │              │                  │
  │   owner: alice│    ───→     │  同一个目录!!!     │
  │ task_2.json   │    ───→     │  /project/        │
  │   owner: bob  │              │                  │
  └──────────────┘              └──────────────────┘

  问题: N 个任务 → 1 个目录 = 必然冲突
  解法: N 个任务 → N 个目录 = 隔离
```

### 解决思路

```
给每个任务一个独立的 git worktree 目录:

  控制平面 (.tasks/)              执行平面 (.worktrees/)
  ┌──────────────┐              ┌──────────────────┐
  │ task_1.json   │    ←──→     │ auth-refactor/    │  独立目录
  │   worktree:   │              │ branch: wt/auth   │  独立分支
  │   "auth-ref"  │              ├──────────────────┤
  │ task_2.json   │    ←──→     │ ui-login/         │  独立目录
  │   worktree:   │              │ branch: wt/ui     │  独立分支
  │   "ui-login"  │              └──────────────────┘
  └──────────────┘

  绑定方式: task_id
  task_1.json 里写 worktree: "auth-refactor"
  index.json 里 auth-refactor 写 task_id: 1
  → 双向引用，任意一侧都能找到对方
```

---

## 2. 前置知识：git worktree 是什么？

在理解代码之前，需要先理解 git worktree：

```
正常的 git 仓库:
  /project/        ← 只有一个工作目录
    ├── .git/      ← git 元数据
    ├── src/
    └── config.py

  问题: 你只能在一个分支上工作
  如果要同时在两个分支上改代码？→ 要么来回切换，要么 clone 两份

git worktree 的解法:
  /project/        ← 主工作目录 (main 分支)
    ├── .git/
    ├── src/
    └── config.py

  /project/.worktrees/auth-refactor/   ← worktree 1 (wt/auth-refactor 分支)
    ├── src/                            ← 完整的文件副本
    └── config.py                       ← 独立修改不影响主目录

  /project/.worktrees/ui-login/        ← worktree 2 (wt/ui-login 分支)
    ├── src/
    └── config.py

  关键: 所有 worktree 共享同一个 .git/（提交历史共享）
  但每个 worktree 有自己的工作目录和分支
  → 可以同时在不同分支上编辑不同文件，互不干扰
```

git worktree 的核心命令：

```bash
# 创建 worktree：在指定路径创建新分支和工作目录
git worktree add -b wt/auth-refactor .worktrees/auth-refactor HEAD

# 列出所有 worktree
git worktree list

# 删除 worktree
git worktree remove .worktrees/auth-refactor
```

---

## 3. 架构总览

```
Control plane (.tasks/)             Execution plane (.worktrees/)
+------------------+                +------------------------+
| task_1.json      |                | auth-refactor/         |
|   status: in_progress  <------>   branch: wt/auth-refactor
|   worktree: "auth-refactor"   |   task_id: 1             |
+------------------+                +------------------------+
| task_2.json      |                | ui-login/              |
|   status: pending    <------>     branch: wt/ui-login
|   worktree: "ui-login"       |   task_id: 2             |
+------------------+                +------------------------+
                                    |
                          index.json (worktree registry)
                          events.jsonl (lifecycle log)

State machines:
  Task:     pending -> in_progress -> completed
  Worktree: absent  -> active      -> removed | kept
```

### 和 s11 的关键区别

| 组件 | s11 | s12 |
|------|-----|-----|
| **执行模式** | 所有任务共享同一目录 | 每个任务可分配独立 worktree |
| **协调** | 任务板 (owner/status) | 任务板 + worktree 显式绑定 |
| **可恢复性** | 仅任务状态 | 任务状态 + worktree 索引 |
| **收尾** | 任务完成 | 任务完成 + 显式 keep/remove |
| **生命周期可见性** | 隐式日志 | 显式事件流 (events.jsonl) |
| **团队管理** | 有 (spawn/send/broadcast) | 无（聚焦隔离机制本身） |

注意：s12 没有继承 s11 的团队管理代码（spawn、MessageBus 等）。它是一个**聚焦演示**，只展示"任务 + worktree 隔离"这一个核心概念。在实际系统中，你会把 s11 的自治队友 + s12 的目录隔离组合起来。

---

## 4. 代码逐行详解

### 4.1 文件头和导入

```python
#!/usr/bin/env python3
# Harness: directory isolation -- parallel execution lanes that never collide.
"""
s12_worktree_task_isolation.py - Worktree + Task Isolation

Directory-level isolation for parallel task execution.
Tasks are the control plane and worktrees are the execution plane.
"""

import json
import os
import re
import subprocess
import time
from pathlib import Path

from anthropic import Anthropic
from dotenv import load_dotenv
```

和 s11 相比：
- **去掉了** `threading`、`uuid` -- 不再有多线程团队管理
- **新增了** `re` -- 用于 worktree 名称校验

### 4.2 环境配置

```python
load_dotenv(override=True)
if os.getenv("ANTHROPIC_BASE_URL"):
    os.environ.pop("ANTHROPIC_AUTH_TOKEN", None)

WORKDIR = Path.cwd()
client = Anthropic(base_url=os.getenv("ANTHROPIC_BASE_URL"))
MODEL = os.environ["MODEL_ID"]
```

标准配置，和以前一样。

```python
def detect_repo_root(cwd: Path) -> Path | None:
    """Return git repo root if cwd is inside a repo, else None."""
    try:
        r = subprocess.run(
            ["git", "rev-parse", "--show-toplevel"],
            cwd=cwd, capture_output=True, text=True, timeout=10,
        )
        if r.returncode != 0:
            return None
        root = Path(r.stdout.strip())
        return root if root.exists() else None
    except Exception:
        return None

REPO_ROOT = detect_repo_root(WORKDIR) or WORKDIR
```

**新增：自动检测 git 仓库根目录。** 这是因为 worktree 必须在 git 仓库内创建。如果不在 git 仓库中，退回使用 WORKDIR。

### 4.3 System Prompt

```python
SYSTEM = (
    f"You are a coding agent at {WORKDIR}. "
    "Use task + worktree tools for multi-task work. "
    "For parallel or risky changes: create tasks, allocate worktree lanes, "
    "run commands in those lanes, then choose keep/remove for closeout. "
    "Use worktree_events when you need lifecycle visibility."
)
```

注意 prompt 的设计：
1. 告诉 LLM "用 task + worktree 工具"
2. 给出操作流程模板："创建任务 → 分配 worktree → 在 lane 里执行 → keep/remove 收尾"
3. 提到 `worktree_events` 用于可见性

### 4.4 ★ EventBus -- 追加式事件流

```python
class EventBus:
    def __init__(self, event_log_path: Path):
        self.path = event_log_path
        self.path.parent.mkdir(parents=True, exist_ok=True)
        if not self.path.exists():
            self.path.write_text("")
```

EventBus 是 s12 的新组件，负责记录所有生命周期事件。

**为什么需要事件流？**

```
没有事件流时:
  alice 在 worktree auth-refactor 里工作
  崩溃了！
  重启后: 
    index.json 显示 auth-refactor 是 active 的
    但它到底经历了什么？什么时候创建的？有没有出过错？
    → 不知道，信息丢失了

有事件流后:
  events.jsonl:
    {"event": "worktree.create.before", "worktree": {"name": "auth-refactor"}, "ts": 1730000000}
    {"event": "worktree.create.after", "worktree": {"name": "auth-refactor", "status": "active"}, "ts": 1730000001}
    → 完整的创建历史，可追溯
```

```python
    def emit(self, event, task=None, worktree=None, error=None):
        payload = {
            "event": event,
            "ts": time.time(),
            "task": task or {},
            "worktree": worktree or {},
        }
        if error:
            payload["error"] = error
        with self.path.open("a", encoding="utf-8") as f:
            f.write(json.dumps(payload) + "\n")
```

**关键设计：**
- **追加式**（append-only）：只写不改，永不丢失历史
- **JSONL 格式**：每行一个 JSON，便于追加和逐行解析
- **统一结构**：每条事件都有 `event`、`ts`、`task`、`worktree` 字段

事件类型一览：

```
worktree.create.before   -- 准备创建 worktree
worktree.create.after    -- 创建成功
worktree.create.failed   -- 创建失败
worktree.remove.before   -- 准备删除 worktree
worktree.remove.after    -- 删除成功
worktree.remove.failed   -- 删除失败
worktree.keep            -- 标记为保留
task.completed           -- 任务完成
```

每个操作都有 before/after/failed 三态，这是经典的生命周期钩子模式。

```python
    def list_recent(self, limit=20):
        n = max(1, min(int(limit or 20), 200))
        lines = self.path.read_text(encoding="utf-8").splitlines()
        recent = lines[-n:]
        items = []
        for line in recent:
            try:
                items.append(json.loads(line))
            except Exception:
                items.append({"event": "parse_error", "raw": line})
        return json.dumps(items, indent=2)
```

取最近 N 条事件，解析失败的行也不会崩溃。

**重要概念：事件流是观测旁路，不是状态机替身**

```
真实状态源:
  任务状态 → .tasks/task_X.json
  worktree 状态 → .worktrees/index.json

事件流:
  → .worktrees/events.jsonl  (只读旁路，用于审计和调试)

类比: 
  数据库表 = 状态源（你查余额看数据库）
  审计日志 = 事件流（你追溯操作看日志）
  → 不会用审计日志来查余额
```

### 4.5 ★ TaskManager -- 持久化任务板

```python
class TaskManager:
    def __init__(self, tasks_dir: Path):
        self.dir = tasks_dir
        self.dir.mkdir(parents=True, exist_ok=True)
        self._next_id = self._max_id() + 1
```

和 s11 的任务管理器类似，但有关键新增。

#### 4.5.1 创建任务

```python
    def create(self, subject, description=""):
        task = {
            "id": self._next_id,
            "subject": subject,
            "description": description,
            "status": "pending",
            "owner": "",
            "worktree": "",          # ← 新增：worktree 绑定字段
            "blockedBy": [],
            "created_at": time.time(),
            "updated_at": time.time(),
        }
        self._save(task)
        self._next_id += 1
        return json.dumps(task, indent=2)
```

和 s11 对比，任务 JSON 多了 `worktree` 字段，初始值为空字符串。

#### 4.5.2 ★ 绑定 worktree -- 核心方法

```python
    def bind_worktree(self, task_id, worktree, owner=""):
        task = self._load(task_id)
        task["worktree"] = worktree        # 绑定 worktree 名称
        if owner:
            task["owner"] = owner
        if task["status"] == "pending":
            task["status"] = "in_progress"  # 自动推进状态
        task["updated_at"] = time.time()
        self._save(task)
        return json.dumps(task, indent=2)
```

**关键设计：绑定时自动推进状态。**

```
为什么绑定时自动把 pending → in_progress？

因为 "分配了执行环境" 意味着 "开始执行" 了:
  task.status == "pending" && task.worktree == ""    → 还没开始
  task.status == "in_progress" && task.worktree == "auth-refactor"  → 正在执行

如果不自动推进:
  程序员需要手动调用两步:
    1. bind_worktree(task_id=1, worktree="auth-refactor")
    2. task_update(task_id=1, status="in_progress")
  容易忘记第 2 步，导致状态不一致

自动推进 = 减少出错机会 = 状态始终正确
```

#### 4.5.3 解绑 worktree

```python
    def unbind_worktree(self, task_id):
        task = self._load(task_id)
        task["worktree"] = ""
        task["updated_at"] = time.time()
        self._save(task)
        return json.dumps(task, indent=2)
```

解绑时只清空 worktree 字段，不改变 status。因为任务可能已经 completed 了。

### 4.6 ★ WorktreeManager -- 核心类

这是 s12 最核心的类，管理 git worktree 的完整生命周期。

#### 4.6.1 初始化

```python
class WorktreeManager:
    def __init__(self, repo_root, tasks, events):
        self.repo_root = repo_root
        self.tasks = tasks           # 依赖 TaskManager
        self.events = events         # 依赖 EventBus
        self.dir = repo_root / ".worktrees"
        self.dir.mkdir(parents=True, exist_ok=True)
        self.index_path = self.dir / "index.json"
        if not self.index_path.exists():
            self.index_path.write_text(json.dumps({"worktrees": []}, indent=2))
        self.git_available = self._is_git_repo()
```

**注意依赖关系：**

```
WorktreeManager 依赖:
  ├── TaskManager   -- 用于绑定/解绑/完成任务
  ├── EventBus      -- 用于发射生命周期事件
  └── git           -- 用于实际创建/删除 worktree

三者的关系:
  TaskManager 管 "做什么"（控制平面）
  WorktreeManager 管 "在哪做"（执行平面）
  EventBus 管 "发生了什么"（观测平面）
```

`index.json` 的结构：

```json
{
  "worktrees": [
    {
      "name": "auth-refactor",
      "path": "/project/.worktrees/auth-refactor",
      "branch": "wt/auth-refactor",
      "task_id": 1,
      "status": "active",
      "created_at": 1730000000
    }
  ]
}
```

**为什么不直接用 `git worktree list`？**

```
git worktree list 的输出:
  /project                       abc1234 [main]
  /project/.worktrees/auth-refactor  def5678 [wt/auth-refactor]

问题:
  1. 没有 task_id — 不知道哪个 worktree 对应哪个任务
  2. 没有自定义状态 — 不知道是 active/kept/removed
  3. 没有时间戳 — 不知道什么时候创建的
  4. 上下文压缩后丢失 — git 命令输出不会被持久化

index.json 补充了这些元数据，是 worktree 的 "本地注册表"
```

#### 4.6.2 名称校验

```python
    def _validate_name(self, name):
        if not re.fullmatch(r"[A-Za-z0-9._-]{1,40}", name or ""):
            raise ValueError(
                "Invalid worktree name. Use 1-40 chars: letters, numbers, ., _, -"
            )
```

限制 worktree 名称：1-40 个字符，只允许字母、数字、点、下划线、横线。这是因为名称会变成目录名和 git 分支名，必须是安全的。

#### 4.6.3 ★ 创建 worktree -- 完整流程

```python
    def create(self, name, task_id=None, base_ref="HEAD"):
        self._validate_name(name)
        if self._find(name):
            raise ValueError(f"Worktree '{name}' already exists in index")
        if task_id is not None and not self.tasks.exists(task_id):
            raise ValueError(f"Task {task_id} not found")

        path = self.dir / name
        branch = f"wt/{name}"
```

创建前的防御检查：
1. 名称格式合法
2. 名称不重复
3. 如果绑定任务，任务必须存在

```python
        self.events.emit(
            "worktree.create.before",
            task={"id": task_id} if task_id is not None else {},
            worktree={"name": name, "base_ref": base_ref},
        )
```

**发射 before 事件**（操作前）。

```python
        try:
            self._run_git(["worktree", "add", "-b", branch, str(path), base_ref])
```

**执行实际的 git 命令：**

```bash
git worktree add -b wt/auth-refactor .worktrees/auth-refactor HEAD

参数解释:
  add             -- 添加 worktree
  -b wt/auth-refactor  -- 创建新分支名
  .worktrees/auth-refactor  -- worktree 路径
  HEAD            -- 基于哪个提交创建
```

```python
            entry = {
                "name": name,
                "path": str(path),
                "branch": branch,
                "task_id": task_id,
                "status": "active",
                "created_at": time.time(),
            }
            idx = self._load_index()
            idx["worktrees"].append(entry)
            self._save_index(idx)

            if task_id is not None:
                self.tasks.bind_worktree(task_id, name)

            self.events.emit("worktree.create.after", ...)
            return json.dumps(entry, indent=2)
```

创建成功后的三个步骤：
1. **更新 index.json** -- 注册新 worktree
2. **绑定任务**（如果指定了 task_id）-- 双向关联
3. **发射 after 事件** -- 记录成功

```python
        except Exception as e:
            self.events.emit("worktree.create.failed", ..., error=str(e))
            raise
```

失败时发射 failed 事件，然后重新抛出异常。

**创建的完整时序：**

```
worktree_create("auth-refactor", task_id=1)
  │
  ├─ 1. 校验名称和任务
  ├─ 2. emit("worktree.create.before")
  ├─ 3. git worktree add -b wt/auth-refactor .worktrees/auth-refactor HEAD
  ├─ 4. 写入 index.json
  ├─ 5. tasks.bind_worktree(1, "auth-refactor")
  │      └─ task_1.json: status="in_progress", worktree="auth-refactor"
  ├─ 6. emit("worktree.create.after")
  └─ 7. 返回 worktree entry JSON
```

#### 4.6.4 在 worktree 中执行命令

```python
    def run(self, name, command):
        dangerous = ["rm -rf /", "sudo", "shutdown", "reboot", "> /dev/"]
        if any(d in command for d in dangerous):
            return "Error: Dangerous command blocked"

        wt = self._find(name)
        if not wt:
            return f"Error: Unknown worktree '{name}'"
        path = Path(wt["path"])
        if not path.exists():
            return f"Error: Worktree path missing: {path}"

        try:
            r = subprocess.run(
                command, shell=True,
                cwd=path,              # ← 关键：cwd 指向 worktree 目录
                capture_output=True, text=True, timeout=300,
            )
            out = (r.stdout + r.stderr).strip()
            return out[:50000] if out else "(no output)"
        except subprocess.TimeoutExpired:
            return "Error: Timeout (300s)"
```

**核心机制：`cwd=path`**

```
这就是"隔离"的实现方式:

普通 bash 工具:
  subprocess.run(command, cwd=WORKDIR)  ← 在主目录执行

worktree_run:
  subprocess.run(command, cwd=worktree_path)  ← 在隔离目录执行

同一个命令 "git status" 在两个地方执行:
  bash("git status")
    → 显示主目录的 git 状态
  
  worktree_run("auth-refactor", "git status")
    → 显示 auth-refactor worktree 的 git 状态
    → 独立的未提交改动、独立的分支

这就是"各干各的目录"的含义。
没有什么复杂的沙箱或容器，就是 cwd 参数不同。
```

#### 4.6.5 ★ 删除 worktree -- 收尾流程

```python
    def remove(self, name, force=False, complete_task=False):
        wt = self._find(name)
        if not wt:
            return f"Error: Unknown worktree '{name}'"

        self.events.emit("worktree.remove.before", ...)
```

```python
        try:
            args = ["worktree", "remove"]
            if force:
                args.append("--force")
            args.append(wt["path"])
            self._run_git(args)
```

**执行 `git worktree remove`**，删除实际目录。`--force` 选项可以强制删除有未提交改动的 worktree。

```python
            if complete_task and wt.get("task_id") is not None:
                task_id = wt["task_id"]
                before = json.loads(self.tasks.get(task_id))
                self.tasks.update(task_id, status="completed")
                self.tasks.unbind_worktree(task_id)
                self.events.emit("task.completed", ...)
```

**如果 `complete_task=True`：一键收尾。**

```
worktree_remove("auth-refactor", complete_task=True) 做了三件事:

  1. git worktree remove .worktrees/auth-refactor  -- 删除目录
  2. task_1.json: status="completed", worktree=""  -- 完成任务 + 解绑
  3. events.jsonl: task.completed                   -- 记录事件

一个调用 = 拆除目录 + 完成任务 + 发事件
如果是手动做，需要调用 3 个工具。
```

```python
            idx = self._load_index()
            for item in idx.get("worktrees", []):
                if item.get("name") == name:
                    item["status"] = "removed"
                    item["removed_at"] = time.time()
            self._save_index(idx)

            self.events.emit("worktree.remove.after", ...)
            return f"Removed worktree '{name}'"
```

注意：删除后 index.json 中的记录不会被删掉，而是标记为 `"status": "removed"`。这是为了保留历史记录。

#### 4.6.6 保留 worktree

```python
    def keep(self, name):
        wt = self._find(name)
        if not wt:
            return f"Error: Unknown worktree '{name}'"

        idx = self._load_index()
        for item in idx.get("worktrees", []):
            if item.get("name") == name:
                item["status"] = "kept"
                item["kept_at"] = time.time()
                kept = item
        self._save_index(idx)

        self.events.emit("worktree.keep", ...)
        return json.dumps(kept, indent=2)
```

**keep vs remove 的选择：**

```
worktree_keep("ui-login"):
  → 目录保留在磁盘上
  → index.json 状态变为 "kept"
  → 后续可以继续在里面工作，或者手动删除
  → 适合：工作还没完成，或者想保留作为参考

worktree_remove("auth-refactor", complete_task=True):
  → 目录从磁盘删除
  → index.json 状态变为 "removed"
  → 绑定的任务标记完成
  → 适合：工作已完成，清理资源

这是两种不同的收尾策略，由 LLM 或用户决定使用哪种。
```

### 4.7 工具定义

s12 注册了 15 个工具，分为三类：

```
基础工具（和以前一样）:
  bash           -- 在主目录执行命令
  read_file      -- 读文件
  write_file     -- 写文件
  edit_file      -- 编辑文件

任务管理工具（控制平面）:
  task_create     -- 创建任务
  task_list       -- 列出所有任务
  task_get        -- 获取任务详情
  task_update     -- 更新任务状态/归属
  task_bind_worktree  -- 手动绑定任务到 worktree

worktree 工具（执行平面）:
  worktree_create  -- 创建 worktree（可选绑定任务）
  worktree_list    -- 列出所有 worktree
  worktree_status  -- 查看某个 worktree 的 git 状态
  worktree_run     -- 在某个 worktree 中执行命令
  worktree_keep    -- 保留 worktree
  worktree_remove  -- 删除 worktree（可选完成任务）

观测工具:
  worktree_events  -- 查看最近的生命周期事件
```

### 4.8 Agent Loop

```python
def agent_loop(messages):
    while True:
        response = client.messages.create(
            model=MODEL,
            system=SYSTEM,
            messages=messages,
            tools=TOOLS,
            max_tokens=8000,
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
                results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": str(output),
                })
        messages.append({"role": "user", "content": results})
```

这是标准的 agent loop（回忆 s01）。s12 没有多线程，只有一个 agent 在运行。LLM 通过工具调用来操作任务和 worktree。

---

## 5. 完整工作流示例

让我们走一遍完整的场景：

```
用户: "Implement auth refactor and login UI updates in parallel"

Step 1 -- LLM 创建两个任务:
  > task_create(subject="Auth refactor")
    → .tasks/task_1.json  {status: "pending", worktree: ""}
  > task_create(subject="Login UI polish")
    → .tasks/task_2.json  {status: "pending", worktree: ""}

Step 2 -- LLM 为每个任务分配 worktree:
  > worktree_create(name="auth-refactor", task_id=1)
    → git worktree add -b wt/auth-refactor .worktrees/auth-refactor HEAD
    → index.json: [{name: "auth-refactor", task_id: 1, status: "active"}]
    → task_1.json: {status: "in_progress", worktree: "auth-refactor"}
    → events.jsonl: worktree.create.before, worktree.create.after

  > worktree_create(name="ui-login", task_id=2)
    → git worktree add -b wt/ui-login .worktrees/ui-login HEAD
    → index.json: [..., {name: "ui-login", task_id: 2, status: "active"}]
    → task_2.json: {status: "in_progress", worktree: "ui-login"}

Step 3 -- LLM 在各自的 worktree 中执行命令:
  > worktree_run(name="auth-refactor", command="pytest tests/auth -q")
    → subprocess.run("pytest tests/auth -q", cwd=".worktrees/auth-refactor")
    → 在 auth-refactor 分支的独立目录中运行测试

  > worktree_run(name="ui-login", command="npm test -- login")
    → subprocess.run("npm test -- login", cwd=".worktrees/ui-login")
    → 在 ui-login 分支的独立目录中运行测试

Step 4 -- 收尾:
  > worktree_remove(name="auth-refactor", complete_task=True)
    → git worktree remove .worktrees/auth-refactor
    → task_1.json: {status: "completed", worktree: ""}
    → events.jsonl: worktree.remove.before, task.completed, worktree.remove.after

  > worktree_keep(name="ui-login")
    → 目录保留（可能 UI 还需要继续调整）
    → index.json: ui-login status="kept"
    → events.jsonl: worktree.keep

最终状态:
  .tasks/task_1.json: status="completed"    ✓ 完成
  .tasks/task_2.json: status="in_progress"  → 继续工作
  .worktrees/auth-refactor/: 已删除
  .worktrees/ui-login/: 保留中
```

---

## 6. 核心设计决策

### 6.1 共享任务板 + 隔离执行通道

```
为什么不是每个 worktree 有自己的任务板？

共享任务板的好处:
  - 全局可见性：任何人都能看到所有任务的状态
  - 协调简单：一个地方查看谁在做什么
  - 不重复：不会出现两个任务板有同一个任务的不同版本

独立执行通道的好处:
  - 不冲突：alice 的改动不影响 bob
  - 可回滚：每个 worktree 有独立的 git 状态
  - 可并行：真正的并行开发
```

### 6.2 显式生命周期管理

```
为什么收尾是显式的（keep/remove），而不是自动清理？

自动清理的风险:
  任务完成 → 自动删除 worktree → 但 worktree 里还有未 push 的代码！
  → 代码丢失！

显式管理的好处:
  任务完成 → LLM 决定 keep 还是 remove
  → 如果代码已 push → remove（安全删除）
  → 如果代码未 push → keep（保留以供后续处理）
  → 决策权在 LLM 或用户手中，不会意外丢失
```

### 6.3 事件流是旁路，不是状态机

```
状态源:
  task 状态 → .tasks/task_X.json（读写）
  worktree 状态 → .worktrees/index.json（读写）

事件流:
  → .worktrees/events.jsonl（只写，用于审计）

为什么不用事件流来存储状态？
  1. 事件流只追加，不能修改 → 不适合存储"当前状态"
  2. 查"任务 1 当前什么状态"需要重放所有事件 → 低效
  3. 事件流可能很大（几千条事件）→ 不适合频繁查询

正确做法:
  状态文件 = 快照（当前状态，快速查询）
  事件流 = 日志（历史轨迹，审计追溯）
```

---

## 7. 崩溃恢复

```
场景: 程序在创建 worktree 后、但在更新 index.json 前崩溃了

  git worktree add 成功了 → 磁盘上有 .worktrees/auth-refactor/
  但 index.json 没更新 → 程序不知道这个 worktree 存在

恢复策略:
  1. git worktree list → 列出 git 知道的所有 worktree
  2. 对比 index.json → 找到不在索引中的 worktree
  3. 补充注册或手动清理

s12 没有实现自动恢复，但提供了基础设施:
  - index.json + events.jsonl 提供了重建的数据源
  - "磁盘状态是持久的，对话记忆是易失的" 是核心原则
```

---

## 8. 和 Claude Code 真实 Worktree 的对比

Claude Code（你现在正在使用的）也有 worktree 功能。对比一下：

```
s12 教学版:
  - 手动管理 index.json
  - 手动绑定 task_id
  - 事件流写 JSONL 文件
  - 单 agent，工具调用驱动

Claude Code 真实版:
  - EnterWorktree / ExitWorktree 工具
  - 在 .claude/worktrees/ 下创建
  - 自动创建新分支基于 HEAD
  - Agent 工具支持 isolation: "worktree" 参数
  - 集成到 git 提交和 PR 工作流

核心思想完全一致:
  1. 用独立目录隔离执行
  2. 用 git 分支管理变更
  3. 完成后选择保留或删除
```

---

## 9. 全系列回顾

到 s12 为止，整个系列覆盖了构建编程智能体的完整技术栈：

```
第一阶段: 基础 Agent Loop
  s01 - Agent Loop        -- 基本循环：LLM → 工具 → 结果 → LLM
  s02 - Tool Use          -- 工具注册和执行
  s03 - Todo Write        -- 任务追踪
  s04 - Subagent          -- 子 agent 并行/隔离
  s05 - Skill Loading     -- 动态能力扩展
  s06 - Context Compact   -- 上下文压缩

第二阶段: 任务系统
  s07 - Task System       -- 持久化任务板
  s08 - Background Tasks  -- 后台执行

第三阶段: 团队协作
  s09 - Agent Teams       -- 多 agent 通信
  s10 - Team Protocols    -- 结构化协议

第四阶段: 自治与隔离
  s11 - Autonomous Agents -- 自主认领任务
  s12 - Worktree + Task Isolation -- 目录级隔离

从 s01 到 s12 的演进:
  s01: 一个 LLM + 一个工具 + 一个循环
  s12: 任务板协调 + worktree 隔离 + 事件流审计

每一步都在解决上一步遗留的问题:
  s11 的问题: 共享目录冲突
  s12 的解决: 目录隔离
```

---

## 10. 试一试

```sh
cd learn-claude-code
python agents/s12_worktree_task_isolation.py
```

建议按顺序尝试这些 prompt：

1. `Create tasks for backend auth and frontend login page, then list tasks.`
   → 观察任务创建，注意 worktree 字段为空

2. `Create worktree "auth-refactor" for task 1, then bind task 2 to a new worktree "ui-login".`
   → 观察 worktree 创建和任务状态自动推进

3. `Run "git status --short" in worktree "auth-refactor".`
   → 观察命令在隔离目录中执行

4. `Keep worktree "ui-login", then list worktrees and inspect events.`
   → 观察 keep 操作和事件流

5. `Remove worktree "auth-refactor" with complete_task=true, then list tasks/worktrees/events.`
   → 观察一键收尾：目录删除 + 任务完成 + 事件记录
