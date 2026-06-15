# 第 39 章：从 mini-agent 到生产级 Agent 平台的完整路线图

前面 38 章，我们从概念、模型调用、工具协议、权限系统、文件编辑、上下文预算、会话持久化、测试、源码映射、MCP、ToolSearch、沙箱、hooks、子 Agent、多 Agent、memory、skills、plugins、评估、回放和可观测性一路讲下来。

如果你是新手，读到这里很可能会有一个真实的问题：

> 这些内容我都大概知道了，但真正动手时，到底应该按什么顺序做？

这章就是回答这个问题。

我们会把“从零搭建 AI Agent”拆成一条可执行路线。你不需要一天内做完所有东西，也不应该一开始就模仿 Claude Code 的全部复杂度。正确路线是：先做一个能跑的最小闭环，再逐步加工具、权限、上下文、持久化、评估、多 Agent 和部署。

这章会非常具体。每个阶段都会讲：

1. 学习目标。
2. 你要实现什么。
3. 推荐目录结构。
4. 最小代码形态。
5. 验收标准。
6. 常见错误。
7. 什么时候进入下一阶段。

你可以把这一章当作整本书的“施工图”。

## 39.1 总路线：不要从生产平台开始

很多人学习 Agent，一上来就想做：

1. 多模型支持。
2. 插件市场。
3. MCP。
4. 多 Agent 编排。
5. 向量数据库记忆。
6. 复杂权限系统。
7. Web UI。
8. 云端部署。

结果通常是项目很快失控。目录很多，抽象很多，真正能完成任务的能力却很弱。

正确顺序应该是：

1. 阶段 0：理解 Agent 的最小循环。
2. 阶段 1：做一个 CLI，可以接收用户输入。
3. 阶段 2：接入真实模型，完成一次问答。
4. 阶段 3：定义消息协议和工具调用格式。
5. 阶段 4：实现 Read、Grep、Bash 三个核心工具。
6. 阶段 5：实现权限系统。
7. 阶段 6：实现 Edit/Write，让 Agent 能修改项目。
8. 阶段 7：实现上下文预算和工具结果截断。
9. 阶段 8：实现会话持久化和 resume。
10. 阶段 9：实现 eval 和测试。
11. 阶段 10：实现子 Agent。
12. 阶段 11：实现 MCP、ToolSearch、skills、custom agents。
13. 阶段 12：实现观测、回放、审计和部署。

这不是唯一顺序，但它符合一个原则：

> 先让 Agent 在小范围里真实完成任务，再扩大能力边界。

如果 Agent 连一个文件都读不准，就不要急着做插件市场。如果 Agent 连权限拒绝都处理不好，就不要急着让它执行部署命令。如果 Agent 没有 transcript，就不要急着做复杂多 Agent 回放。

## 39.2 阶段 0：先理解 Agent 的最小循环

学习目标：

1. 知道 Agent 和普通聊天机器人的区别。
2. 理解 tool_use 和 tool_result。
3. 理解 Agent loop。
4. 明白“模型负责决定，程序负责执行”。

最小循环：

```text
用户输入
  -> 组装 messages
  -> 调模型
  -> 模型返回普通文本或 tool_use
  -> 如果是普通文本，输出给用户
  -> 如果是 tool_use，程序执行工具
  -> 把 tool_result 加回 messages
  -> 再调模型
```

这一阶段不需要写复杂代码。你可以先画图，或者用伪代码理解：

```python
from dataclasses import dataclass, field
from datetime import datetime, timezone
from typing import Any

@dataclass
class Message:
    role: str
    content: list[dict[str, Any]] | str

@dataclass
class TranscriptEntry:
    kind: str
    session_id: str
    message: Message
    uuid: str
    parent_uuid: str | None = None
    created_at: str = field(default_factory=lambda: datetime.now(timezone.utc).isoformat())
```

验收标准：

1. 你能解释 tool_use 为什么不是程序自己猜出来的，而是模型请求的。
2. 你能解释工具执行结果为什么要回填给模型。
3. 你能解释为什么需要 maxTurns。
4. 你能解释为什么工具必须有 schema。

常见错误：

1. 把模型当成能直接读本地文件的程序。
2. 让模型输出自然语言命令，然后程序用正则硬解析。
3. 忘记把工具结果回填给模型。
4. 没有循环终止条件。

进入下一阶段的条件：你能不用看书，独立写出 Agent loop 的伪代码。

## 39.3 阶段 1：做一个最小 CLI

学习目标：

1. 熟悉 Python 项目结构。
2. 学会从命令行读取用户输入。
3. 学会组织最小可运行程序。

推荐目录：

```text
mini-agent/
  pyproject.toml
  src/
    cli.py
    agent.py
```

`cli.py`：

```python
import asyncio

from agent import run_agent

async def main() -> None:
    while True:
        prompt = input("> ").strip()
        if prompt in {"exit", "quit"}:
            break
        answer = await run_agent(prompt)
        print(answer)

if __name__ == "__main__":
    asyncio.run(main())
```

`agent.py` 先不要接模型：

```python
async def run_agent(prompt: str) -> str:
    return f"你说的是：{prompt}"
```

验收标准：

1. 能启动 CLI。
2. 能输入多轮问题。
3. 输入 exit 能退出。
4. 错误能打印出来。

常见错误：

1. 一开始就做 Web UI。
2. CLI 还不能稳定运行就接模型。
3. 没有统一错误入口。

为什么先做 CLI？

因为 Agent 的核心是循环、工具、权限和上下文。CLI 能让你最快验证这些核心逻辑。Web UI 会带来状态管理、组件、样式、网络接口等额外复杂度，容易分散注意力。

进入下一阶段的条件：CLI 稳定运行，能多轮输入。

## 39.4 阶段 2：接入真实模型

学习目标：

1. 理解模型 API。
2. 理解 messages 结构。
3. 理解 system prompt 和 user prompt。
4. 知道如何处理 API key。

推荐目录：

```text
src/
  cli.py
  agent.py
  model/
    client.py
    types.py
```

模型接口不要和具体供应商强绑定：

```python
from typing import Any

def example(context: dict[str, Any]) -> dict[str, Any]:
    return {"ok": True, "context": context}
```

`agent.py`：

```python
from model.client import Message, ModelClient

async def run_agent(prompt: str, model: ModelClient) -> str:
    messages = [
        Message(role="system", content="你是一个严谨的编程助手。"),
        Message(role="user", content=prompt),
    ]

    response = await model.complete(messages)
    return response.message.content
```

验收标准：

1. API key 不写死在代码里。
2. 模型能返回文本。
3. 网络错误能被捕获。
4. messages 结构清晰。
5. system prompt 和 user prompt 分开。

常见错误：

1. API key 提交到 git。
2. 把所有 prompt 拼成一个字符串。
3. 没有处理模型错误。
4. 每个文件都直接调用模型 SDK，后期难替换。

进入下一阶段的条件：你能通过统一 ModelClient 完成一次问答，并且不依赖具体 UI。

## 39.5 阶段 3：定义工具协议

学习目标：

1. 定义 Tool 接口。
2. 定义 tool schema。
3. 定义 tool_use 和 tool_result。
4. 让模型能请求工具。

推荐目录：

```text
src/
  tools/
    types.py
    registry.py
```

核心类型：

```python
from pathlib import Path

def resolve_inside_workspace(workspace: Path, user_path: str) -> Path:
    root = workspace.resolve()
    target = (root / user_path).resolve()
    if root != target and root not in target.parents:
        raise ValueError(f"Path is outside current workspace: {user_path}")
    return target

def read_text_file(workspace: Path, user_path: str) -> str:
    return resolve_inside_workspace(workspace, user_path).read_text(encoding="utf-8")
```

工具注册表：

```python
from pathlib import Path

def resolve_inside_workspace(workspace: Path, user_path: str) -> Path:
    root = workspace.resolve()
    target = (root / user_path).resolve()
    if root != target and root not in target.parents:
        raise ValueError(f"Path is outside current workspace: {user_path}")
    return target

def read_text_file(workspace: Path, user_path: str) -> str:
    return resolve_inside_workspace(workspace, user_path).read_text(encoding="utf-8")
```

验收标准：

1. 工具有 name、description、inputSchema。
2. 工具执行通过 registry。
3. 未知工具会报错。
4. tool_use 有唯一 id。
5. tool_result 能关联 tool_use id。

常见错误：

1. 工具没有 schema，模型只能猜输入。
2. 工具返回结构不稳定。
3. 直接用 if else 分发所有工具。
4. tool_result 不带 toolUseId，模型无法对应。

进入下一阶段的条件：你能注册一个假工具，比如 `EchoTool`，模型请求后程序能执行并把结果回填。

## 39.6 阶段 4：实现 Read 工具

学习目标：

1. 让 Agent 读取本地文件。
2. 学会路径限制。
3. 学会控制输出大小。
4. 理解文件工具的安全边界。

Read 工具定义：

```python
from pathlib import Path

def resolve_inside_workspace(workspace: Path, user_path: str) -> Path:
    root = workspace.resolve()
    target = (root / user_path).resolve()
    if root != target and root not in target.parents:
        raise ValueError(f"Path is outside current workspace: {user_path}")
    return target

def read_text_file(workspace: Path, user_path: str) -> str:
    return resolve_inside_workspace(workspace, user_path).read_text(encoding="utf-8")
```

实现重点：

1. file_path 必须在 workspace 内。
2. 默认不要读太多行。
3. 大文件要提示被截断。
4. 二进制文件不要当文本读。

验收标准：

1. 能读取普通文本文件。
2. 路径越界会失败。
3. 大文件不会全部塞进上下文。
4. 返回内容带行号更好。

常见错误：

1. 允许读取任意绝对路径。
2. 不限制文件大小。
3. 文件不存在时返回空字符串。
4. 工具错误不清晰。

进入下一阶段的条件：Agent 能根据用户问题主动读取文件，并基于文件内容回答。

## 39.7 阶段 5：实现 Grep 工具

学习目标：

1. 让 Agent 能探索项目。
2. 理解搜索比盲读更重要。
3. 学会限制搜索范围。

Grep 工具的价值在于：Agent 不知道文件在哪里时，需要先搜索。

最小 schema：

```python
from pathlib import Path

def resolve_inside_workspace(workspace: Path, user_path: str) -> Path:
    root = workspace.resolve()
    target = (root / user_path).resolve()
    if root != target and root not in target.parents:
        raise ValueError(f"Path is outside current workspace: {user_path}")
    return target

def read_text_file(workspace: Path, user_path: str) -> str:
    return resolve_inside_workspace(workspace, user_path).read_text(encoding="utf-8")
```

实现方式可以先用 JavaScript 遍历文件，后续再换成 ripgrep。新手先关注接口和输出格式。

返回格式建议：

```text
src/auth/token.py:42: def verify_token(...)
src/middleware/auth.py:18: user = verify_token(...)
```

验收标准：

1. 能搜索指定 pattern。
2. 能限制 path。
3. 能限制结果数量。
4. 忽略 node_modules、.git、dist。
5. 搜索结果足够短。

常见错误：

1. 默认搜索整个机器。
2. 返回几千行结果。
3. 不忽略依赖目录。
4. 正则错误时崩溃。

进入下一阶段的条件：Agent 能先 Grep 找文件，再 Read 关键文件。

## 39.8 阶段 6：实现 Bash 工具，但先加安全限制

学习目标：

1. 让 Agent 能运行命令。
2. 理解 shell 是最高风险工具。
3. 实现 timeout。
4. 捕获 stdout、stderr、exitCode。

Bash 工具很强，也很危险。不要一开始就给完全自由。

最小 schema：

```python
from pathlib import Path

def resolve_inside_workspace(workspace: Path, user_path: str) -> Path:
    root = workspace.resolve()
    target = (root / user_path).resolve()
    if root != target and root not in target.parents:
        raise ValueError(f"Path is outside current workspace: {user_path}")
    return target

def read_text_file(workspace: Path, user_path: str) -> str:
    return resolve_inside_workspace(workspace, user_path).read_text(encoding="utf-8")
```

执行结果：

```python
import asyncio
from dataclasses import dataclass

@dataclass
class CommandResult:
    command: str
    exit_code: int | None
    stdout: str
    stderr: str
    timed_out: bool = False

async def run_command(command: str, timeout: float = 30.0) -> CommandResult:
    process = await asyncio.create_subprocess_shell(command, stdout=asyncio.subprocess.PIPE, stderr=asyncio.subprocess.PIPE)
    try:
        stdout, stderr = await asyncio.wait_for(process.communicate(), timeout=timeout)
        return CommandResult(command, process.returncode, stdout.decode(), stderr.decode())
    except asyncio.TimeoutError:
        process.kill()
        return CommandResult(command, None, "", "命令超时", True)
```

必须做的限制：

1. timeout。
2. 输出截断。
3. 工作目录限制在 workspace。
4. 危险命令进入权限系统。
5. 交互式命令要避免。

验收标准：

1. 能运行 `pytest` 这类命令。
2. 命令超时会停止。
3. stdout/stderr 都能返回。
4. 非 0 exit code 不会让整个 Agent 崩溃。
5. 危险命令不能静默执行。

常见错误：

1. 没有 timeout。
2. 使用 shell 拼接用户输入导致注入。
3. 把 stderr 当作程序异常直接抛出。
4. 输出过大塞爆上下文。
5. 允许 watch mode 一直运行。

进入下一阶段的条件：Agent 能运行测试命令，并能根据失败输出继续修复。

## 39.9 阶段 7：实现权限系统

学习目标：

1. 理解 allow、deny、ask。
2. 知道权限判断发生在工具执行前。
3. 学会给不同工具不同规则。
4. 能记录权限审计。

权限系统最小结构：

```python
from dataclasses import dataclass
from typing import Literal

Decision = Literal["allow", "deny", "ask"]

@dataclass
class PermissionRule:
    source: str
    behavior: Decision
    tool_name: str
    rule_content: str = ""

def format_rule(rule: PermissionRule) -> str:
    content = f"({rule.rule_content})" if rule.rule_content else ""
    return f"{rule.source}:{rule.behavior}:{rule.tool_name}{content}"

def deny_message(rule: PermissionRule) -> dict[str, str]:
    return {
        "behavior": "deny",
        "message": f"Denied by {format_rule(rule)}",
        "reason": "Matched deny rule",
    }
```

规则示例：

```python
rules = [
{ tool: "Read", decision: "allow" }
{ tool: "Grep", decision: "allow" }
{ tool: "Bash", commandPrefix: "pytest", decision: "allow" }
{ tool: "Bash", commandPrefix: "git push", decision: "ask" }
{ tool: "Bash", commandPrefix: "rm -rf", decision: "deny" }
{ tool: "Edit", decision: "ask" }
]
```

验收标准：

1. 每次工具执行前都经过权限检查。
2. deny 后工具不执行。
3. ask 能暂停并等待用户批准。
4. 权限决策有审计记录。
5. Agent 能理解 deny 并尝试替代方案。

常见错误：

1. 只在 Bash 上做权限，忘记 Edit/Write。
2. 权限规则只看工具名，不看输入。
3. deny 后抛错导致整个任务结束，模型没有机会恢复。
4. 用户批准一次后永久放开所有类似操作。

进入下一阶段的条件：你能演示 Read 自动允许、危险 Bash 被拒绝、Edit 需要询问。

## 39.10 阶段 8：实现 Edit 和 Write

学习目标：

1. 让 Agent 能修改文件。
2. 学会最小改动。
3. 防止误伤。
4. 生成可审查 diff。

Edit 工具应该优先于 Write，因为 Edit 更适合局部修改。

Edit schema：

```python
from pathlib import Path

def resolve_inside_workspace(workspace: Path, user_path: str) -> Path:
    root = workspace.resolve()
    target = (root / user_path).resolve()
    if root != target and root not in target.parents:
        raise ValueError(f"Path is outside current workspace: {user_path}")
    return target

def read_text_file(workspace: Path, user_path: str) -> str:
    return resolve_inside_workspace(workspace, user_path).read_text(encoding="utf-8")
```

关键规则：

1. old_string 必须唯一匹配。
2. 修改前最好要求已读文件。
3. 修改后返回 diff 摘要。
4. 文件不存在时不要静默创建，除非用 Write。
5. 大范围替换要 ask。

Write schema：

```python
from pathlib import Path

def resolve_inside_workspace(workspace: Path, user_path: str) -> Path:
    root = workspace.resolve()
    target = (root / user_path).resolve()
    if root != target and root not in target.parents:
        raise ValueError(f"Path is outside current workspace: {user_path}")
    return target

def read_text_file(workspace: Path, user_path: str) -> str:
    return resolve_inside_workspace(workspace, user_path).read_text(encoding="utf-8")
```

Write 更危险。建议：

1. 创建新文件可以 ask 或 allow，视项目策略。
2. 覆盖已有文件必须 ask。
3. 不能写出 workspace。
4. 不能写敏感路径，如 `.env`，除非用户明确要求。

验收标准：

1. Agent 能修一个小 bug。
2. 修改范围最小。
3. 修改后能运行测试。
4. 任务结束能列出 changed files。
5. 错误匹配不会破坏文件。

常见错误：

1. 用 Write 覆盖整个文件来做小修改。
2. old_string 多处匹配仍然替换。
3. 修改前不读取文件。
4. 修改失败后模型不知道原因。

进入下一阶段的条件：Agent 能完成“读代码、改代码、跑测试、总结”的完整闭环。

## 39.11 阶段 9：实现上下文预算

学习目标：

1. 理解 token budget。
2. 控制工具结果大小。
3. 避免历史消息无限增长。
4. 学会 compact 的基本思想。

上下文预算有三个入口：

1. 输入消息。
2. 工具 schema。
3. 工具结果。

新手最容易忽略工具结果。一个 Bash 命令可能输出几万行，一个 Grep 可能返回几千条匹配，一个 Read 可能读取大文件。如果直接塞进 messages，Agent 很快失控。

最小策略：

```python
limits =:
    "readMaxLines": 300
    "grepMaxResults": 100
    "bashMaxOutputChars": 12000
    "maxTurns": 20
```

截断工具结果时要告诉模型：

```text
Output truncated to 12000 characters. Original output was 84620 characters.
Use a narrower command or search pattern if more detail is needed.
```

验收标准：

1. 大文件不会完整进入上下文。
2. 大命令输出会截断。
3. Agent 能根据截断提示缩小范围。
4. 超过 maxTurns 会停止。
5. 对话历史可以摘要或裁剪。

常见错误：

1. 截断但不告诉模型。
2. 只限制 Read，不限制 Bash。
3. maxTurns 太高导致成本失控。
4. compact 摘要丢掉用户原始约束。

进入下一阶段的条件：你能让 Agent 在大项目里搜索和读取，而不会马上塞爆上下文。

## 39.12 阶段 10：实现会话持久化

学习目标：

1. 保存 transcript。
2. 支持 resume。
3. 支持 session id。
4. 区分 transcript 和运行时事件。

最小 transcript JSONL：

```json
{"type":"user","uuid":"1","sessionId":"s1","message":{"role":"user","content":"修复测试"}}
{"type":"assistant","uuid":"2","sessionId":"s1","message":{"role":"assistant","content":"我先查看测试。"}}
```

目录：

```text
.mini-agent/
  sessions/
    s1.jsonl
```

写入：

```python
import json
from pathlib import Path
from typing import Any

def read_json(path: Path) -> dict[str, Any]:
    return json.loads(path.read_text(encoding="utf-8"))

def read_jsonl(path: Path) -> list[dict[str, Any]]:
    if not path.exists():
        return []
    return [json.loads(line) for line in path.read_text(encoding="utf-8").splitlines() if line.strip()]

def load_agent_task(task_id: str, root: Path) -> dict[str, Any]:
    metadata = read_json(root / ".agent" / "tasks" / task_id / "metadata.json")
    transcript = read_jsonl(root / ".agent" / "subagents" / f"{task_id}.jsonl")
    return {"id": task_id, "metadata": metadata, "transcript": transcript}
```

读取：

```python
from dataclasses import dataclass, field
from datetime import datetime, timezone
from typing import Any

@dataclass
class Message:
    role: str
    content: list[dict[str, Any]] | str

@dataclass
class TranscriptEntry:
    kind: str
    session_id: str
    message: Message
    uuid: str
    parent_uuid: str | None = None
    created_at: str = field(default_factory=lambda: datetime.now(timezone.utc).isoformat())
```

验收标准：

1. 每轮消息保存到 JSONL。
2. 重启 CLI 后可以 resume。
3. session id 稳定。
4. 工具结果能恢复。
5. progress 不混进 transcript。

常见错误：

1. 用纯文本保存，后续难解析。
2. 写入不追加，导致历史丢失。
3. transcript 存太多 UI 状态。
4. resume 时没有恢复 tool_result。

进入下一阶段的条件：你能关闭程序后重新打开，继续同一个任务。

## 39.13 阶段 11：实现测试与 eval

学习目标：

1. 不靠手感判断 Agent 好坏。
2. 写工具单元测试。
3. 写 Agent eval。
4. 记录回归。

测试分层：

1. 工具测试：Read、Grep、Bash、Edit 是否正确。
2. 权限测试：allow、deny、ask 是否正确。
3. Agent loop 测试：tool_use -> tool_result -> next model call 是否正确。
4. Eval case：在 fixture repo 上执行任务。

工具测试示例：

```python
from pathlib import Path

def resolve_inside_workspace(workspace: Path, user_path: str) -> Path:
    root = workspace.resolve()
    target = (root / user_path).resolve()
    if root != target and root not in target.parents:
        raise ValueError(f"Path is outside current workspace: {user_path}")
    return target

def read_text_file(workspace: Path, user_path: str) -> str:
    return resolve_inside_workspace(workspace, user_path).read_text(encoding="utf-8")
```

权限测试：

```python
from dataclasses import dataclass
from typing import Literal

Decision = Literal["allow", "deny", "ask"]

@dataclass
class PermissionRule:
    source: str
    behavior: Decision
    tool_name: str
    rule_content: str = ""

def format_rule(rule: PermissionRule) -> str:
    content = f"({rule.rule_content})" if rule.rule_content else ""
    return f"{rule.source}:{rule.behavior}:{rule.tool_name}{content}"

def deny_message(rule: PermissionRule) -> dict[str, str]:
    return {
        "behavior": "deny",
        "message": f"Denied by {format_rule(rule)}",
        "reason": "Matched deny rule",
    }
```

Eval case：

```json
{
  "id": "fix-simple-test",
  "fixtureRepo": "fixtures/simple-node",
  "prompt": "修复失败的测试。",
  "expected": {
    "filesChanged": ["src/add.py"],
    "commandsRun": ["pytest"],
    "commandsNotRun": ["git push"]
  }
}
```

验收标准：

1. 每个工具都有测试。
2. 权限危险场景有测试。
3. 至少 5 个 eval case。
4. eval 输出失败原因。
5. 每次重大改动跑 eval。

常见错误：

1. 只测 happy path。
2. 不测权限拒绝。
3. eval 没有固定 fixture。
4. 只看最终回答，不看文件改动。

进入下一阶段的条件：你能比较两个版本哪个更好，而不是凭感觉。

## 39.14 阶段 12：实现流式响应

学习目标：

1. 让用户更早看到输出。
2. 理解 streaming chunk。
3. 处理 tool_use 的增量构造。
4. 更新 UI/CLI 状态。

流式响应不是必须第一天做，但产品体验会明显改善。

流式处理要注意：

1. 文本 chunk 可以立刻显示。
2. tool_use chunk 需要组装完整后执行。
3. 首 token 时间要记录。
4. 用户中断要能取消流。
5. 流结束后再把完整 assistant message 写入 transcript。

验收标准：

1. 模型生成文本时 CLI 能实时显示。
2. 工具调用不会被半截 JSON 触发。
3. Ctrl+C 能中断。
4. transcript 保存完整消息，而不是零散 chunk。

常见错误：

1. 每个 chunk 都写 transcript，导致历史混乱。
2. tool_use 没组装完就执行。
3. 中断后没有清理 running tool。
4. 流式输出和工具进度混在一起。

进入下一阶段的条件：用户能看到实时输出，并且 transcript 仍然干净。

## 39.15 阶段 13：实现子 Agent

学习目标：

1. 理解任务分解。
2. 实现 AgentTool。
3. 隔离子 Agent 上下文。
4. 保存 sidechain transcript。

不要太早做子 Agent。等主 Agent 闭环稳定后再做。

最小 AgentTool：

```python
from typing import Any

def example(context: dict[str, Any]) -> dict[str, Any]:
    return {"ok": True, "context": context}
```

子 Agent 运行时：

1. 使用独立 messages。
2. 使用独立 maxTurns。
3. 使用受限工具列表。
4. 记录 sidechain transcript。
5. 最终只返回结构化摘要给主 Agent。

验收标准：

1. 主 Agent 可以派发子任务。
2. 子 Agent 不污染主上下文。
3. 子 Agent 有工具限制。
4. 子 Agent 结果可审计。
5. 子 Agent 失败不会拖垮整个任务。

常见错误：

1. 主子 Agent 共用同一个 messages 数组。
2. 子 Agent 拥有全部工具。
3. 子 Agent 输出过长。
4. 不保存 sidechain，无法复盘。

进入下一阶段的条件：你能让 code-reviewer 只读代码并返回审查摘要。

## 39.16 阶段 14：实现 worktree isolation

学习目标：

1. 让并行任务不互相污染。
2. 理解 git worktree。
3. 隔离子 Agent 修改。
4. 汇总 diff。

当子 Agent 需要写文件时，最好不要直接写主工作区。可以使用 worktree isolation。

基本思路：

1. 为子任务创建临时 worktree。
2. 子 Agent 在 worktree 中修改。
3. 任务结束后生成 diff。
4. 主 Agent 决定是否合并。
5. 清理 worktree。

验收标准：

1. 子 Agent 修改不会直接影响主目录。
2. 能查看子 Agent diff。
3. 合并冲突能被报告。
4. 任务取消后 worktree 清理。

常见错误：

1. 并行子 Agent 写同一目录。
2. worktree 没清理。
3. 忽略未提交文件。
4. 自动合并冲突文件。

进入下一阶段的条件：你能同时派发两个修改任务，并安全查看各自 diff。

## 39.17 阶段 15：实现 MCP 接入

学习目标：

1. 理解外部工具服务器。
2. 把 MCP 工具包装成本地 Tool。
3. 加权限边界。
4. 处理连接失败。

MCP 的核心价值是扩展工具来源。Agent 不需要把所有能力都内置，只要能发现和调用外部工具。

接入步骤：

1. 读取 MCP server 配置。
2. 启动或连接 server。
3. list tools。
4. 把 MCP tool schema 转成本地 ToolDefinition。
5. 调用时走统一权限系统。
6. 工具结果回填模型。

验收标准：

1. 能列出 MCP 工具。
2. 能调用一个简单 MCP 工具。
3. MCP 工具也经过权限判断。
4. MCP server 失败有清晰错误。
5. 不可信 MCP 工具不能直接获得高权限。

常见错误：

1. MCP 工具绕过权限系统。
2. 连接失败导致整个 Agent 崩溃。
3. 工具 schema 太多全部注入上下文。
4. 不区分项目 MCP 和用户 MCP 的信任边界。

进入下一阶段的条件：MCP 工具和本地工具在 Agent loop 里行为一致。

## 39.18 阶段 16：实现 ToolSearch

学习目标：

1. 解决工具太多的问题。
2. 延迟加载工具 schema。
3. 控制上下文成本。

当工具数量少时，可以把所有 schema 注入模型。当工具很多时，这会浪费上下文。ToolSearch 的思路是：先给模型一个搜索工具，让模型按需查找可用工具。

流程：

```text
用户任务
  -> 模型看到少量核心工具 + ToolSearch
  -> 模型调用 ToolSearch("github issue")
  -> 系统返回相关工具 schema
  -> 模型再调用具体工具
```

验收标准：

1. 默认注入工具数量减少。
2. 模型能通过 ToolSearch 找到工具。
3. 搜索结果包含工具名、描述、schema 摘要。
4. 相关性排序稳定。
5. 找不到工具时有清晰反馈。

常见错误：

1. ToolSearch 返回过多结果。
2. 工具描述太模糊，无法搜索。
3. 搜索结果不包含足够 schema。
4. 模型不知道何时该搜索工具。

进入下一阶段的条件：系统有几十个工具时，主上下文仍然可控。

## 39.19 阶段 17：实现 skills 和 memory

学习目标：

1. 区分长期知识和当前上下文。
2. 学会按需加载技能说明。
3. 给 Agent 提供可复用流程。

Memory 和 skills 不要神化。

Memory 适合保存：

1. 用户偏好。
2. 项目约定。
3. 长期任务背景。
4. 常用命令。

Skills 适合保存：

1. 特定领域操作流程。
2. 文档生成规范。
3. 测试策略。
4. 特定工具使用说明。

不要把所有历史聊天都塞进 memory。那不是记忆，是噪声。

验收标准：

1. memory 可读可写。
2. skills 可按任务加载。
3. skills 不会全部默认塞进上下文。
4. 用户能查看和删除 memory。
5. memory 中不保存 secret。

常见错误：

1. 把 memory 当数据库乱写。
2. skills 太长，加载后占满上下文。
3. 没有来源标记。
4. 用户无法管理记忆。

进入下一阶段的条件：Agent 能根据项目约定稳定执行任务，而不是每次都重新说明。

## 39.20 阶段 18：实现 custom agents 和 plugins

学习目标：

1. 让用户定义专用 Agent。
2. 支持 Markdown frontmatter。
3. 管理加载优先级。
4. 处理插件分发和安全。

自定义 Agent 文件：

```md
---
name: test-runner
description: Run tests and diagnose failures.
tools:
  - Read
  - Grep
  - Bash
maxTurns: 12
---

你是测试诊断 Agent。你的任务是运行相关测试，分析失败原因，并给出最小修复建议。
```

加载来源：

1. built-in agents。
2. project agents。
3. user agents。
4. plugin agents。

要定义覆盖规则。比如 user agents 可以覆盖 built-in，但 plugin agents 应该命名空间化，避免冲突。

验收标准：

1. 能加载 Markdown agent。
2. frontmatter 解析失败不拖垮系统。
3. 同名覆盖规则明确。
4. plugin agents 有命名空间。
5. 插件不能静默声明高风险 hooks 和 permissions。

常见错误：

1. 任意插件 Agent 覆盖内置 Agent。
2. 解析失败导致所有 Agent 不可用。
3. tools: [] 和 tools: undefined 混淆。
4. 自定义 Agent 默认获得全部工具。

进入下一阶段的条件：用户能定义一个只读 reviewer，并被主 Agent 正确调用。

## 39.21 阶段 19：实现可观测性和回放

学习目标：

1. 保存 transcript。
2. 保存 structured events。
3. 保存 profiler。
4. 支持 replay。

这是上一章的工程落地。

最小要求：

1. 每个 session 有 transcript。
2. 每个工具调用有事件。
3. 每个权限决策有审计。
4. 每次 query 有 profiler。
5. 可以 deterministic replay。

验收标准：

1. 出错后能解释 Agent 做了什么。
2. 能看到慢在哪里。
3. 能看到权限为什么 allow/deny/ask。
4. 能打开子 Agent sidechain。
5. eval 失败时能输出 artifacts。

常见错误：

1. 只打印 console.log。
2. 所有数据混进 transcript。
3. 远程上传敏感内容。
4. replay 实际重新执行工具，导致历史不可信。

进入下一阶段的条件：你能拿一次失败任务，复盘它的完整轨迹。

## 39.22 阶段 20：做发布与部署

学习目标：

1. 把书稿、文档或工具发布出去。
2. 理解 GitHub Pages。
3. 理解 CI workflow。
4. 理解版本发布。

对于本书这样的文档项目，部署路线很清楚：

1. 章节放在 `chapters/`。
2. `README.md` 作为仓库首页。
3. `index.md` 作为 Pages 首页。
4. `_config.yml` 配置 Jekyll。
5. `.github/workflows/pages.yml` 自动部署。
6. 推送到 GitHub。
7. 在仓库设置里启用 Pages。

对于 mini-agent 工具项目，发布路线不同：

1. 编译 Python。
2. 配置 bin entry。
3. 加 Python package metadata。
4. 写 README。
5. 加 license。
6. 加 release workflow。
7. 发布到 PyPI 或 GitHub release。

验收标准：

1. 新机器能按 README 跑起来。
2. CI 能跑测试。
3. 文档能在线访问。
4. 版本号清晰。
5. 发布流程可重复。

常见错误：

1. 本地能跑，README 没写依赖。
2. 没有 lockfile 或依赖版本漂移。
3. GitHub Pages workflow 没权限。
4. 发布前没跑 eval。

进入下一阶段的条件：别人不用问你，也能 clone 后按文档使用。

## 39.23 学习顺序：新手 30 天计划

如果你想按一个月学习，可以这样安排。

第 1 到 3 天：理解 Agent loop。

任务：

1. 读第 1 到 5 章。
2. 画出 tool_use/tool_result 流程。
3. 写一个不接模型的假 Agent loop。

验收：

1. 能解释模型和工具的分工。
2. 能写出 maxTurns 循环。

第 4 到 6 天：CLI 和模型调用。

任务：

1. 搭 Python 项目。
2. 写 CLI。
3. 接真实模型。

验收：

1. 能多轮聊天。
2. API key 不进代码。

第 7 到 10 天：工具系统。

任务：

1. 写 Tool 接口。
2. 写 ToolRegistry。
3. 实现 Read 和 Grep。

验收：

1. Agent 能读项目文件。
2. Agent 能搜索再读取。

第 11 到 14 天：Bash、权限和 Edit。

任务：

1. 实现 Bash。
2. 实现权限系统。
3. 实现 Edit。

验收：

1. Agent 能修一个小 bug。
2. 危险命令被拒绝或询问。
3. 修改后能跑测试。

第 15 到 18 天：上下文和持久化。

任务：

1. 工具输出截断。
2. maxTurns。
3. transcript JSONL。
4. resume。

验收：

1. 大输出不爆上下文。
2. 会话可恢复。

第 19 到 22 天：测试和 eval。

任务：

1. 写工具测试。
2. 写权限测试。
3. 写 5 个 eval case。

验收：

1. 能自动发现回归。
2. eval 报告包含失败原因。

第 23 到 26 天：子 Agent。

任务：

1. 实现 AgentTool。
2. 实现 sidechain transcript。
3. 写一个 code-reviewer。

验收：

1. 主 Agent 能委托审查。
2. 子 Agent 上下文不污染主链。

第 27 到 30 天：生产化。

任务：

1. 加 profiler。
2. 加 permission audit。
3. 加 replay。
4. 写部署文档。

验收：

1. 失败任务可复盘。
2. 慢任务可定位。
3. 项目可被别人运行。

## 39.24 技术栈建议

对于新手，我建议用 Python 起步。

原因：

1. Claude Code 源码风格更接近 Python 生态。
2. 工具 schema、消息类型、事件类型都适合静态类型。
3. Python 调用文件系统和 shell 方便。
4. Python 生态方便做 CLI、自动化和测试工具。

推荐技术：

1. Python：核心语言。
2. Python fs/path/child_process：文件和命令。
3. pydantic：输入校验。
4. pytest：测试。
5. argparse 或 typer：CLI 参数。
6. subprocess 或 asyncio.create_subprocess_shell：更友好的命令执行。
7. simple-git：git 操作。
8. JSONL：transcript 存储。
9. GitHub Actions：CI。
10. GitHub Pages：文档部署。

什么时候用 Python？

如果你的 Agent 主要面向数据分析、机器学习、Notebook、pandas、科学计算，Python 很合适。但如果你要学习 Claude Code 这类开发者工具 Agent，Python 会更贴近。

什么时候用数据库？

一开始不要用。先用文件存 transcript 和 eval artifact。等你有多用户、多机器、云端同步需求，再引入 SQLite、Postgres 或对象存储。

什么时候用向量数据库？

不要在第一阶段用。很多新手一上来就做 RAG，结果连工具循环都没稳定。向量检索适合文档和长期知识，不是 Agent 的必需起点。

## 39.25 目录结构最终形态

当 mini-agent 逐渐成熟，可以演进成这样：

```text
mini-agent/
  pyproject.toml
  src/
    cli/
      index.py
      commands/
        chat.py
        resume.py
        eval.py
        replay.py
    agent/
      runAgent.py
      loop.py
      messages.py
      compact.py
    model/
      client.py
      providers/
        anthropic.py
        openai.py
      types.py
    tools/
      types.py
      registry.py
      read.py
      grep.py
      bash.py
      edit.py
      write.py
      agenttool.py
    permissions/
      engine.py
      rules.py
      audit.py
    context/
      budget.py
      attachments.py
      memory.py
      skills.py
    sessions/
      transcript.py
      storage.py
      resume.py
    agents/
      definitions.py
      loader.py
      builtin/
        code-reviewer.md
        test-runner.md
    mcp/
      client.py
      toolAdapter.py
    observability/
      events.py
      profiler.py
      tracer.py
    eval/
      types.py
      runner.py
      checks.py
      fixtures.py
    replay/
      deterministic.py
      render.py
    utils/
      ids.py
      paths.py
      jsonl.py
```

这个结构不是一开始就要有。它是演进结果。

判断一个目录是否应该拆出来，可以问：

1. 这个模块是否有独立职责？
2. 是否被多个地方使用？
3. 是否有独立测试价值？
4. 是否已经让当前文件太大？

不要为了“看起来架构好”提前拆 30 个目录。小项目初期保持简单是优点。

## 39.26 每阶段的质量门槛

这里给你一张质量门槛表。

| 阶段 | 不能妥协的门槛 |
| --- | --- |
| CLI | 可重复启动、可退出、错误可见 |
| 模型 | API key 安全、错误处理、统一 client |
| 工具 | schema 清晰、结果稳定、未知工具报错 |
| Read | 路径限制、大小限制、错误清楚 |
| Grep | 范围限制、结果限制、忽略依赖目录 |
| Bash | timeout、输出截断、exitCode 保留 |
| 权限 | 工具执行前判断、deny 不执行、可审计 |
| Edit | old_string 唯一、修改前读文件、diff 可见 |
| 上下文 | maxTurns、工具结果预算、截断提示 |
| 会话 | JSONL、sessionId、resume、progress 分离 |
| Eval | fixture 固定、检查轨迹、检查安全 |
| 子 Agent | 上下文隔离、工具限制、sidechain |
| MCP | 统一权限、失败隔离、schema 控制 |
| 观测 | transcript、events、profiler、replay |
| 部署 | README、CI、版本、发布步骤 |

这张表可以贴在你的项目 README 里。每次想加新功能，先看当前阶段门槛是否达标。

## 39.27 如何判断你正在过度设计

过度设计的信号：

1. 你还没有一个工具，却已经有插件系统。
2. 你还没有 eval，却在调 prompt 细节。
3. 你还没有权限系统，却接入了 Bash。
4. 你还没有 transcript，却做多 Agent。
5. 你还不能修一个小 bug，却在做自动软件工程平台。
6. 你还没有用户，却设计了复杂账号系统。
7. 你还没遇到性能问题，却引入分布式队列。

工程上要允许自己“幼稚地先跑起来”。但这个“幼稚”不是不安全，而是范围小。

好的早期版本：

1. 工具少。
2. 权限保守。
3. UI 简单。
4. eval 少但真实。
5. 日志足够。
6. 能完成一个窄任务。

坏的早期版本：

1. 抽象很多。
2. 功能入口很多。
3. 权限宽松。
4. 没有测试。
5. 不能稳定完成任务。

## 39.28 如何判断你正在欠设计

欠设计的信号：

1. 所有逻辑都在一个 `agent.py`。
2. 工具输入没有 schema。
3. 权限判断散落在工具内部。
4. transcript 是纯文本。
5. tool_result 格式每个工具都不同。
6. 错误处理靠 catch 后打印。
7. eval 只能人工看。
8. 子 Agent 和主 Agent 共用上下文。

欠设计会导致后期改不动。你要在“过度设计”和“欠设计”之间找平衡。

一个简单判断：

> 如果某个概念已经被三个地方使用，就应该给它一个名字和类型。

比如 toolUse、toolResult、permissionDecision、transcriptEntry、agentDefinition。这些概念会贯穿系统，必须结构化。

## 39.29 新手最应该反复练的五个能力

第一，读代码能力。

Agent 工程不是只写 prompt。你要读 SDK、读工具输出、读错误堆栈、读源码结构。

第二，设计接口能力。

Tool 接口、ModelClient 接口、PermissionEngine 接口、TranscriptStore 接口，这些接口会决定系统能否演进。

第三，控制边界能力。

哪些工具能执行？哪些路径能读写？哪些数据能上传？哪些子 Agent 能修改文件？边界比能力更重要。

第四，调试过程能力。

Agent 出错不一定是最终答案错。你要会看 transcript、events、profiler、diff、permission audit。

第五，写 eval 能力。

没有 eval，你不知道改动有没有让系统变好。Agent 系统越复杂，越需要自动评估。

## 39.30 从本书映射到实践任务

你可以这样对应前面章节：

1. 第 1 到 5 章：理解概念和消息协议。
2. 第 6 到 8 章：实现 Read、Grep、Bash。
3. 第 9 到 11 章：权限、编辑和完整闭环。
4. 第 12 到 15 章：真实模型、流式、上下文和持久化。
5. 第 16 到 17 章：测试和源码映射。
6. 第 18 到 24 章：工具 schema、生命周期、并发、预算、MCP、ToolSearch。
7. 第 25 到 29 章：权限、安全、沙箱、hooks、审计。
8. 第 30 到 37 章：子 Agent、多 Agent、memory、skills、plugins。
9. 第 38 章：评估、回放、调试和可观测性。
10. 第 39 章：学习和实现路线图。

不要平均用力。对新手来说，最关键的是第 5 到 11 章的闭环：工具调用、读文件、搜索、Bash、权限、编辑、完整任务。只要这条链路扎实，后面的复杂能力都有地方挂。

## 39.31 你的第一个完整作品应该是什么

我建议你的第一个完整作品不要叫“通用 AI Agent 平台”。范围太大。

更好的目标：

> 做一个能在本地 Python 项目里修复简单测试失败的 CLI Agent。

它的能力边界很清楚：

1. 只在当前项目工作。
2. 只支持 Read、Grep、Bash、Edit。
3. Bash 默认只允许测试相关命令。
4. 写文件前询问。
5. 最多 15 轮。
6. 保存 transcript。
7. 有 5 个 eval case。

这个项目做完，你就真正理解了 Agent 工程的核心。

验收任务：

1. 准备一个有失败测试的 fixture。
2. 用户输入：“修复失败测试。”
3. Agent 运行测试。
4. Agent 读取失败文件。
5. Agent 搜索相关实现。
6. Agent 修改一个文件。
7. Agent 再次运行测试。
8. Agent 汇报改动和验证结果。
9. transcript 可回放。
10. eval 能自动判断通过。

如果你能做到这十步，比看十篇 Agent 概念文章有用得多。

## 39.32 第二个作品：只读代码审查 Agent

第二个作品可以做 code-reviewer。

能力边界：

1. 只读。
2. 不能 Edit/Write/Bash，或者 Bash 只允许静态检查。
3. 输出 findings。
4. 每条 finding 包含文件路径、风险、建议。

为什么做只读 reviewer？

因为它适合练习：

1. 搜索策略。
2. 上下文预算。
3. 结构化输出。
4. 子 Agent。
5. 证据引用。

输出格式：

```json
{
  "findings": [
    {
      "severity": "high",
      "file": "src/auth.py",
      "summary": "JWT expiration is not checked.",
      "evidence": "verifyToken decodes token but does not validate exp.",
      "recommendation": "Reject expired tokens before returning user identity."
    }
  ]
}
```

验收标准：

1. 不修改任何文件。
2. findings 有证据。
3. 不编造不存在的文件。
4. 输出不超过约定长度。
5. 能作为子 Agent 被主 Agent 调用。

## 39.33 第三个作品：文档生成 Agent

第三个作品可以做 docs-writer。

能力边界：

1. 读取项目结构。
2. 读取 package、README、src 入口。
3. 生成或更新文档。
4. 不运行危险命令。

这个作品适合练习 Write、Edit、结构化规划、用户确认。

流程：

1. 读取 README。
2. 读取 pyproject.toml。
3. 搜索 CLI 入口或 API 入口。
4. 生成文档大纲。
5. 请求用户确认。
6. 写入 docs。
7. 总结变更。

验收标准：

1. 文档和项目真实结构一致。
2. 不夸大不存在的功能。
3. 修改前有计划。
4. 生成内容格式稳定。

## 39.34 第四个作品：安全权限演练 Agent

第四个作品可以专门练权限。

场景：

1. 用户要求读取 `.env`。
2. 用户要求执行 `rm -rf`。
3. 用户要求 `git push`。
4. 用户要求修改 pyproject.toml。
5. 用户要求运行测试。

你要让 Agent 正确区分：

1. 什么可以自动做。
2. 什么必须询问。
3. 什么必须拒绝。
4. 拒绝后如何解释。
5. 有无替代方案。

这个作品不一定功能复杂，但会让你真正理解安全边界。

## 39.35 最后路线：从个人工具到团队平台

当你的 mini-agent 稳定后，才考虑团队平台。

个人工具阶段：

1. 本地 CLI。
2. 本地 transcript。
3. 本地权限。
4. 少量工具。

团队工具阶段：

1. 项目级配置。
2. 共享 custom agents。
3. 统一权限模板。
4. eval suite。
5. CI 集成。

平台阶段：

1. 插件系统。
2. MCP 管理。
3. 多用户审计。
4. 集中 telemetry。
5. 组织策略。
6. Web 控制台。
7. 成本分析。

不要跳级。每一级都有不同问题。

个人工具的核心是“我能不能高效完成任务”。团队工具的核心是“别人能不能安全复用”。平台的核心是“组织能不能治理和演进”。

## 39.36 本章小结

这一章把全书内容变成了一条可执行路线。

最重要的原则是：从小闭环开始，不要从大平台开始。

你应该先实现 CLI、模型调用、工具协议、Read、Grep、Bash、权限、Edit 和完整任务闭环。然后再做上下文预算、会话持久化、测试、eval、流式响应、子 Agent、MCP、ToolSearch、skills、custom agents、可观测性和部署。

对新手来说，真正的转折点不是“我理解了 Agent 概念”，而是“我做出了一个能读项目、改文件、跑测试、被权限约束、能回放、能 eval 的小 Agent”。

做到这一步，你就不再只是看懂别人做的 Agent，而是具备了搭建 Agent 系统的工程肌肉。

下一章会进入发布与部署：我们会把这本书本身作为一个文档站项目，讲清楚如何整理 Markdown、配置 GitHub Pages、准备仓库、提交代码、启用 Pages，以及在没有授权信息时如何安全地交接部署步骤。
