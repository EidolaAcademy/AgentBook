# 第 32 章：子 Agent 上下文隔离、结果汇总与成本控制

上一章我们拆解了 `AgentTool`：父 Agent 如何选择子 Agent、如何构造工具池、如何启动同步或异步任务。这一章继续往下走，讨论子 Agent 运行起来之后最难的三个问题：

1. 上下文隔离：子 Agent 的消息、工具状态、文件读取缓存、权限判断、取消信号，哪些应该独立，哪些可以共享？
2. 结果汇总：子 Agent 可能读了很多文件、执行了很多工具，最后应该怎样把结果交还给父 Agent？
3. 成本控制：多个子 Agent 会快速放大 token、模型调用次数、工具结果大小和后台资源占用，系统怎样避免失控？

很多多 Agent 原型之所以不稳定，不是因为模型不够强，而是因为这三个问题没处理好。它们通常会出现这些症状：

- 父 Agent 的上下文被子 Agent 的搜索过程塞满。
- 子 Agent 的工具调用污染父 Agent 历史。
- 后台子 Agent 卡住后没人知道。
- 子 Agent 读了大量文件，最终返回一大坨不可用文本。
- 多个子 Agent 同时写文件，互相覆盖。
- 子 Agent 运行成本比父 Agent 还高，却没有带来更高质量结果。

Claude Code 的源码在这些地方做了大量工程处理。本章不会逐行照抄源码，而是提炼它背后的设计方法，并给出新手可以照着实现的简化版本。

## 32.1 为什么子 Agent 必须有独立上下文

先看一个错误设计：

```python
async def runSubAgentBad(parent: AgentRuntime, prompt: str):
    parent.messages.append(:
        "role": 'user'
        "content": prompt

    return runAgentLoop(parent)
```

这段代码的意思是：把子任务 prompt 直接塞进父 Agent 的 messages，然后用同一个运行时继续跑。

它看起来简单，但会带来严重问题。

第一，父子消息混在一起。父 Agent 原本在解决主任务，子 Agent 的搜索、尝试、失败、工具结果都会进入父 Agent 历史。之后父 Agent 再推理时，会看到大量低层过程信息。

第二，工具调用关系可能被破坏。模型 API 通常要求 assistant 的 tool_use 后面必须有对应 tool_result。如果你把父 Agent 和子 Agent 的消息混在一起，很容易出现孤立 tool_use 或顺序错乱。

第三，并发时不可控。如果两个子 Agent 同时往同一个 messages 数组写入，消息顺序就会变成竞争条件。一次运行正常，下一次运行可能完全不同。

第四，成本爆炸。子 Agent 的中间过程会永久留在父 Agent 上下文里，导致后续每次模型调用都重复携带这些内容。

正确设计是：父 Agent 和子 Agent 共享必要的环境能力，但拥有独立的会话上下文。

可以把父 Agent 和子 Agent 的关系想成：

```txt
父 Agent 会话
  - 主 messages
  - 主权限上下文
  - 主工具状态
  - 主 UI 状态
  - 主取消信号

子 Agent 会话
  - 子 messages
  - 子权限上下文
  - 子工具状态
  - 子 transcript
  - 子取消信号

共享部分
  - 项目文件系统
  - MCP 连接
  - 顶层任务注册表
  - 审计日志
  - 部分 UI/进度通道
```

这不是“完全隔离”，而是“受控共享”。成熟系统的关键就在这里：不要把所有东西都隔离，也不要把所有东西都共享。

## 32.2 ToolUseContext 是子 Agent 隔离的核心对象

Claude Code 中有一个非常重要的概念：`ToolUseContext`。它不是单纯的工具参数，而是工具执行时需要的运行时上下文。子 Agent 创建时，会基于父 `ToolUseContext` 生成一个新的子上下文。

教学版可以定义成：

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

子 Agent 隔离的本质，就是创建一个新的 `ToolUseContext`：

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

这段代码里有几个重点。

`messages` 必须新建。子 Agent 的对话历史不能直接引用父 Agent 的数组。

`readFileState` 建议克隆。文件读取缓存可以帮助避免重复注入同一文件内容，但子 Agent 的读取行为不应该污染父 Agent。

`abortController` 要按生命周期决定。同步子 Agent 可以共享父取消信号；异步子 Agent 通常需要自己的取消信号，但父任务结束时也可能需要级联取消，这取决于产品设计。

`setAppState` 要谨慎共享。同步子 Agent 可以更新 UI 或任务状态；异步子 Agent 如果直接修改主状态，容易造成后台写入和界面状态冲突。

`recordTranscript` 应该写入子 Agent 自己的 transcript，而不是主会话 transcript。

这就是子 Agent 上下文隔离的最小模型。

## 32.3 哪些状态应该克隆

不是所有状态都要重新创建。我们先看应该克隆的部分。

第一，消息列表。

```python
childMessages = [...initialMessages]
```

消息列表是 Agent 的思想轨迹。父子共享同一个列表会导致上下文污染。

第二，文件读取状态。

很多 Agent 系统会记录“某个文件是否已经读过”“某个文件内容是否已经注入过”。这类状态应该克隆，而不是共享。

```python
childReadFileState = parent.readFileState.clone()
```

为什么不是新建空状态？如果子 Agent 是 fork 类型，需要处理父历史中的工具结果，那么克隆可以保持一致的替换决策。为什么不是共享？因为普通子 Agent 的读文件行为不应该改变父 Agent 的读取缓存。

第三，工具结果压缩状态。

成熟系统会把大工具结果替换成引用或摘要。这个替换过程需要记录哪些 tool_use_id 已经被替换、替换成什么。fork 类子 Agent 如果继承父消息，就需要克隆这份状态，以免相同消息在父子请求中被不同方式压缩，导致缓存失效。

教学版可以写：

```python
childReplacementState = parent.replacementState
? parent.replacementState.clone()
: None
```

第四，权限拒绝计数。

如果一个异步子 Agent 的 `setAppState` 是 no-op，那么它不能依赖父状态记录权限拒绝重试次数。否则它可能反复尝试同一个被拒绝操作。可以给子 Agent 一个本地 denial tracking。

```python
denialTracking = options.shareState
? parent.denialTracking
: createDenialTrackingState()
```

## 32.4 哪些状态应该新建

下面这些状态通常应该给子 Agent 新建。

第一，`agentId`。

每个子 Agent 都应该有独立 ID。这个 ID 用于：

- transcript 归属。
- 后台任务查询。
- 日志过滤。
- 取消任务。
- 进度事件。
- worktree 命名。

```python
agentId = createAgentId()
```

第二，嵌套触发集合。

比如已经加载过哪些 memory、哪些 skill、哪些动态资源。这些集合如果父子共享，会导致子 Agent 认为某些内容“已经展示过”，但实际上它自己的上下文里没有。

```python
from typing import Any

def example(context: dict[str, Any]) -> dict[str, Any]:
    return {"ok": True, "context": context}
```

第三，子 Agent 自己的 abort controller。

异步子 Agent 尤其需要独立取消控制。

```python
abortController = AbortController()
```

第四，子 transcript 链。

每个子 Agent 的消息应该写入 sidechain transcript。sidechain 的意思是：它属于同一个项目和会话体系，但不是主对话链的一部分。

```python
await recordSidechainTranscript(agentId, childMessages)
```

这样做的好处是：

- 主上下文不被污染。
- 子 Agent 过程可审计。
- 后台任务可以恢复。
- UI 可以查看子 Agent 细节。

## 32.5 哪些状态可以共享

完全隔离不是目标。子 Agent 仍然需要共享一些能力。

第一，MCP client 连接可以共享。

如果父 Agent 已经连接了 GitHub MCP、文件系统 MCP、浏览器 MCP，子 Agent 没必要每次重新连接。当然，如果 Agent 定义中声明了自己的 MCP server，则需要在启动时额外初始化，并在结束时清理。

第二，根任务注册表应该共享。

后台 Bash、后台 Agent、监控任务都需要被顶层系统看见。否则子 Agent 启动的后台任务可能变成没人管理的孤儿任务。

第三，进度和指标通道可以共享。

子 Agent 的 token 使用、首 token 延迟、工具执行进度，可以汇总到父界面或 SDK 事件流中。

第四，组织级安全策略应该共享。

即使子 Agent 有自己的 permissionMode，组织级 deny 规则、工作目录限制、沙箱策略也不应该被绕过。

第五，同步子 Agent 可以共享部分 UI 状态更新。

同步子 Agent 正在父 Agent 当前回合里运行，用户正在等待它完成。此时共享进度和响应长度统计是合理的。

## 32.6 shareSetAppState：隔离与共享之间的开关

Claude Code 的 `createSubagentContext` 有一个值得学习的设计：默认把 mutation callbacks 变成 no-op，但允许调用者显式选择共享。

教学版可以这样写：

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

这里最重要的是默认值：默认不共享可变状态。只有调用方明确知道自己在做什么时，才打开共享。

同步 Agent 可以这样：

```python
ctx = createSubagentContext(parent,:
    "shareSetAppState": True
    "shareAbortController": True
    "shareProgress": True
```

异步 Agent 可以这样：

```python
ctx = createSubagentContext(parent,:
    "shareSetAppState": False
    "shareAbortController": False
    "shareProgress": True
```

这个设计看起来普通，但它避免了大量隐性 bug。因为大多数并发问题都来自“无意共享”。

## 32.7 sidechain transcript：记录过程，但不污染主上下文

子 Agent 的中间过程很有价值，但不应该全部塞回父 Agent。

它有审计价值：

- 子 Agent 看了哪些文件？
- 执行了哪些命令？
- 哪个工具失败了？
- 为什么得出这个结论？

它有恢复价值：

- 后台 Agent 运行到一半进程重启，能否恢复？
- 用户能否查看子 Agent 已经做到哪里？

它有调试价值：

- 为什么子 Agent 输出了错误结论？
- 它是不是没看到关键文件？
- 它是不是被某个权限拒绝挡住了？

但它不应该污染父上下文。解决方法是 sidechain transcript。

教学版可以设计目录：

```txt
.agent/
  sessions/
    main.jsonl
  subagents/
    <agentId>.jsonl
    <agentId>.meta.json
```

主会话只记录父 Agent 的关键消息。子 Agent 的完整消息写入自己的 jsonl。

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

还可以写元数据：

```python
from dataclasses import dataclass
from pathlib import Path
import subprocess

@dataclass
class AgentWorktree:
    path: Path
    branch: str

def create_agent_worktree(repo: Path, branch: str, target: Path) -> AgentWorktree:
    subprocess.run(["git", "worktree", "add", str(target), branch], cwd=repo, check=True)
    return AgentWorktree(path=target, branch=branch)

def remove_agent_worktree(repo: Path, worktree: AgentWorktree) -> None:
    subprocess.run(["git", "worktree", "remove", str(worktree.path)], cwd=repo, check=True)
```

这样父 Agent 只需要拿到最终结果，但用户和系统仍然能追溯完整过程。

## 32.8 子 Agent 结果不是 transcript，而是提炼后的交付物

一个常见错误是把子 Agent 的完整最后几轮消息原样返回给父 Agent。这样父 Agent 会收到大量过程内容，例如：

```txt
我先搜索 auth。
找到了 src/auth/token.py。
现在读取这个文件。
这个文件里有 refreshToken。
我还要搜索 tests。
……
```

父 Agent 真正需要的是：

```txt
发现 2 个问题：
1. src/auth/token.py: refreshToken 在失败时吞掉异常，调用方无法区分网络失败和未登录。
2. src/auth/session.py: 并发刷新时旧 token 可能覆盖新 token。

建议：
- refreshToken 返回结构化错误。
- 引入 refresh lock 或请求去重。
```

所以子 Agent 的结果应该是“交付物”，不是“过程录像”。

可以定义：

```python
from dataclasses import dataclass, field
from typing import Literal

@dataclass
class Finding:
    file_path: str
    message: str
    severity: Literal["low", "medium", "high"]

@dataclass
class AgentResult:
    summary: str
    findings: list[Finding] = field(default_factory=list)
    files_read: list[str] = field(default_factory=list)
    commands_run: list[str] = field(default_factory=list)
    changed_files: list[str] = field(default_factory=list)
    confidence: Literal["low", "medium", "high"] = "medium"

def format_agent_result(result: AgentResult) -> str:
    parts = [f"Summary:\n{result.summary}"]
    if result.findings:
        findings = "\n".join(f"- {item.severity}: {item.file_path}: {item.message}" for item in result.findings)
        parts.append(f"Findings:\n{findings}")
    if result.changed_files:
        parts.append("Changed files:\n" + "\n".join(result.changed_files))
    parts.append(f"Confidence: {result.confidence}")
    return "\n\n".join(parts)
```

如果输出给模型，可以压成文本：

```python
from dataclasses import dataclass, field
from typing import Literal

@dataclass
class Finding:
    file_path: str
    message: str
    severity: Literal["low", "medium", "high"]

@dataclass
class AgentResult:
    summary: str
    findings: list[Finding] = field(default_factory=list)
    files_read: list[str] = field(default_factory=list)
    commands_run: list[str] = field(default_factory=list)
    changed_files: list[str] = field(default_factory=list)
    confidence: Literal["low", "medium", "high"] = "medium"

def format_agent_result(result: AgentResult) -> str:
    parts = [f"Summary:\n{result.summary}"]
    if result.findings:
        findings = "\n".join(f"- {item.severity}: {item.file_path}: {item.message}" for item in result.findings)
        parts.append(f"Findings:\n{findings}")
    if result.changed_files:
        parts.append("Changed files:\n" + "\n".join(result.changed_files))
    parts.append(f"Confidence: {result.confidence}")
    return "\n\n".join(parts)
```

高质量子 Agent 输出应该满足五个要求：

1. 短。
2. 具体。
3. 可验证。
4. 可行动。
5. 带边界说明。

“我检查了一些代码，整体还可以”不是好结果。“我没有权限运行测试，所以只做了静态检查”才是有用边界。

## 32.9 父 Agent 应该怎样使用子 Agent 结果

父 Agent 收到子 Agent 结果后，不应该立刻把它当最终答案。更好的流程是：

```txt
收到子 Agent 结果
        |
        v
判断结果是否足够具体
        |
        v
必要时读取证据文件
        |
        v
必要时运行验证命令
        |
        v
整合到主任务计划
        |
        v
向用户汇报或继续执行
```

父 Agent 的系统提示词可以加入：

```txt
子 Agent 的结果是辅助证据，不是最终真理。
当结果涉及文件修改、测试状态、安全风险或用户决策时，你应该根据需要验证关键证据。
不要把子 Agent 的长过程原样转述给用户，只提炼和当前任务有关的结论。
```

这里有个重要观念：多 Agent 系统中，父 Agent 不是“老板”，而是“集成者”。它不只是分配任务，还要整合、验证、取舍。

## 32.10 结果汇总的三种模式

子 Agent 结果汇总通常有三种模式。

第一种，一次性总结。

子 Agent 完成任务后，最后输出一个 summary。父 Agent 只接收这个 summary。这是最简单、成本最低的模式。

适合：

- 代码搜索。
- 小范围审查。
- 文档总结。

第二种，结构化结果。

子 Agent 输出 JSON 或类似结构，父 Agent 可以稳定解析。

```json
{
  "summary": "认证模块存在两个中等风险",
  "findings": [
    {
      "severity": "medium",
      "file": "src/auth/token.py",
      "issue": "refreshToken swallows errors",
      "recommendation": "return structured errors"
    }
  ],
  "confidence": "medium"
}
```

适合：

- 审查。
- 测试分析。
- 安全扫描。
- 多个子 Agent 结果合并。

第三种，工件输出。

子 Agent 生成文件，例如：

```txt
reports/security-review.md
reports/test-failure-analysis.md
```

父 Agent 只收到文件路径和摘要。

适合：

- 长报告。
- 大型迁移计划。
- 后台任务。
- 需要用户查看的产物。

教学版可以从一次性总结开始，然后支持结构化结果，最后支持工件输出。

## 32.11 成本控制第一层：限制子 Agent 的上下文输入

成本控制从输入开始。子 Agent 的初始上下文越大，后面每轮调用越贵。

常见错误是把父 Agent 完整历史传给每个子 Agent：

```python
childMessages = [...parent.messages, createUserMessage(prompt)]
```

这会让成本按子 Agent 数量倍增。

更好的默认策略是只传任务：

```python
childMessages = [createUserMessage(prompt)]
```

如果子 Agent 需要背景，由父 Agent 明确写进 prompt：

```txt
背景：
- 用户正在修复登录失败问题。
- 相关目录可能是 src/auth 和 src/api。
- 不要修改文件，只找原因。

任务：
请搜索相关代码并总结可能原因。
```

这叫“有选择地转述上下文”，比“复制完整上下文”更可控。

只有在 fork subagent、缓存共享、会话总结这类高级场景下，才考虑继承父消息。

## 32.12 成本控制第二层：限制工具结果大小

工具结果是上下文膨胀的最大来源之一。一个 `grep` 可能返回几百行，一个 `cat` 可能返回上万行，一个测试命令可能输出几万字符。

子 Agent 如果把这些结果全部带进后续每轮模型调用，成本会迅速失控。

工具结果控制有几种方式。

第一，工具本身截断。

```python
from typing import Any

def example(context: dict[str, Any]) -> dict[str, Any]:
    return {"ok": True, "context": context}
```

第二，搜索工具默认返回匹配片段，而不是完整文件。

第三，大结果写入磁盘，只把引用放进上下文。

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

第四，microcompact。也就是在不做完整会话压缩的情况下，把旧工具结果替换成摘要或占位引用。

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

Claude Code 源码中有 content replacement、tool use summary、microcompact 等机制，目的都是减少工具结果对上下文的长期占用。

## 32.13 成本控制第三层：限制子 Agent 回合数

上一章讲过 `maxTurns`，这里从成本角度再强调一次。

每多一轮，子 Agent 都会带着已有上下文再次调用模型。假设每轮平均输入 30k token，一个子 Agent 跑 20 轮，成本非常可观。

所以每个 Agent 类型都应该有合理默认值：

```python
from typing import Any

def example(context: dict[str, Any]) -> dict[str, Any]:
    return {"ok": True, "context": context}
```

运行时：

```python
maxTurns =
agent.maxTurns  or 
DEFAULT_MAX_TURNS_BY_AGENT[agent.agentType]  or 
10
```

超过上限时，不要静默失败。应该返回清楚结果：

```txt
Agent reached maxTurns=12 before completing the task.
Partial result:
- 已检查 src/auth/token.py
- 尚未检查测试文件
- 建议继续搜索 refreshToken tests
```

部分结果比空失败更有价值。

## 32.14 成本控制第四层：为普通子 Agent 关闭高成本推理

在 Claude Code 的 `runAgent` 中，普通子 Agent 会关闭 thinking 配置，而 fork child 在需要缓存一致时才继承父 thinking config。这个设计很有启发。

原因是：不是每个子任务都值得使用高成本推理。

代码搜索、文件定位、简单总结通常不需要最强推理模式。真正需要高推理的是：

- 架构设计。
- 复杂 bug 定位。
- 安全推理。
- 大规模迁移策略。

教学版可以给 Agent 定义增加：

```python
from typing import Any

def example(context: dict[str, Any]) -> dict[str, Any]:
    return {"ok": True, "context": context}
```

默认：

```python
from typing import Any

def example(context: dict[str, Any]) -> dict[str, Any]:
    return {"ok": True, "context": context}
```

原则是：子 Agent 默认便宜，只有明确需要时才变贵。

## 32.15 成本控制第五层：选择合适模型

多 Agent 系统中，模型选择很重要。所有子 Agent 都用最强模型，成本会很高；所有子 Agent 都用最便宜模型，质量可能不稳定。

可以按任务分层：

```txt
轻量模型：
- 文件定位
- 简单搜索
- 日志摘要
- 格式转换

中等模型：
- 代码审查
- 测试失败分析
- 小范围修复

强模型：
- 跨模块架构判断
- 高风险安全分析
- 复杂重构计划
```

Agent 定义中可以写：

```yaml
---
name: Explore
model: haiku
maxTurns: 8
---
```

```yaml
---
name: security-reviewer
model: sonnet
maxTurns: 12
---
```

```yaml
---
name: architecture-planner
model: opus
maxTurns: 10
---
```

模型选择不应该完全交给父 Agent。父 Agent 可以请求覆盖，但系统应该有默认策略和上限。

## 32.16 子 Agent 的消息记录应该保留多少

子 Agent 的 transcript 有两个相反目标：

- 要足够完整，方便审计和恢复。
- 要足够克制，避免磁盘、索引和 UI 都被大量中间消息拖慢。

建议把消息分成三类。

第一类，必须保留：

- 用户给子 Agent 的初始任务。
- assistant 的最终结果。
- 工具调用名称和输入摘要。
- 工具失败信息。
- 权限拒绝信息。
- maxTurns、取消、异常等生命周期事件。

第二类，可以保留但需要压缩：

- 大工具输出。
- 长测试日志。
- 大文件读取结果。
- 搜索结果全集。

第三类，可以不进入父上下文，但保留在 sidechain：

- 中间推理文本。
- 重复搜索过程。
- 低价值进度消息。

可以实现清洗函数：

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

这跟“不要记录”不同。我们不是删除证据，而是把大内容移到合适的位置。

## 32.17 子 Agent 结果的可信度

子 Agent 输出应该说明可信度。可信度不是玄学，可以根据完成条件判断。

高可信度：

- 读取了关键文件。
- 执行了相关测试。
- 工具没有被权限拒绝。
- 没有达到 maxTurns。
- 输出包含具体证据。

中可信度：

- 读取了部分文件。
- 未运行完整测试。
- 某些搜索可能遗漏。
- 输出有证据但范围有限。

低可信度：

- 关键工具不可用。
- 任务中途取消。
- 达到 maxTurns。
- 只能基于用户描述推断。

可以定义：

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

父 Agent 使用结果时，可以参考 confidence。比如低可信度结果只能作为线索，不能直接作为最终结论。

## 32.18 多个子 Agent 结果如何合并

当父 Agent 同时委派多个子 Agent，结果合并会变复杂。

例如：

- `Explore` 找相关文件。
- `security-reviewer` 找安全问题。
- `test-fixer` 尝试修复测试。

三个结果可能互相矛盾：

```txt
Explore: 认证逻辑主要在 src/auth/token.py。
security-reviewer: 风险主要在 src/session/refresh.py。
test-fixer: 测试失败来自 src/api/client.py。
```

父 Agent 不能简单拼接。它需要做归并：

```python
from dataclasses import dataclass, field
from typing import Literal

@dataclass
class Finding:
    file_path: str
    message: str
    severity: Literal["low", "medium", "high"]

@dataclass
class AgentResult:
    summary: str
    findings: list[Finding] = field(default_factory=list)
    files_read: list[str] = field(default_factory=list)
    commands_run: list[str] = field(default_factory=list)
    changed_files: list[str] = field(default_factory=list)
    confidence: Literal["low", "medium", "high"] = "medium"

def format_agent_result(result: AgentResult) -> str:
    parts = [f"Summary:\n{result.summary}"]
    if result.findings:
        findings = "\n".join(f"- {item.severity}: {item.file_path}: {item.message}" for item in result.findings)
        parts.append(f"Findings:\n{findings}")
    if result.changed_files:
        parts.append("Changed files:\n" + "\n".join(result.changed_files))
    parts.append(f"Confidence: {result.confidence}")
    return "\n\n".join(parts)
```

合并流程可以是：

```txt
收集所有结果
        |
        v
按文件路径分组
        |
        v
按问题类型分组
        |
        v
找重复结论
        |
        v
标记冲突结论
        |
        v
决定下一步验证动作
```

代码示例：

```python
from typing import Any

def example(context: dict[str, Any]) -> dict[str, Any]:
    return {"ok": True, "context": context}
```

合并结果时要保留来源。父 Agent 最后汇报时可以说：

```txt
这个问题被 security-reviewer 和 test-fixer 都提到，可信度较高。
```

或者：

```txt
两个子 Agent 对根因判断不一致，我会先读取相关文件验证。
```

这才是多 Agent 协作的价值。

## 32.19 异步子 Agent 的结果回流

异步子 Agent 完成时，父 Agent 可能已经不在原来的回合。结果回流有几种方式。

第一，任务通知。

```txt
Background agent completed: review auth module
```

然后系统把结果作为一条 attachment 或 notification 注入主循环。

第二，输出文件。

异步 Agent 把结果写到：

```txt
.agent/tasks/<taskId>/result.md
```

父 Agent 可以通过工具读取。

第三，任务查询工具。

父 Agent 或用户可以调用：

```txt
TaskStatus({ taskId })
```

得到：

```json
{
  "status": "completed",
  "summary": "...",
  "resultPath": ".agent/tasks/abc/result.md"
}
```

第四，恢复时重建。

如果进程重启，系统从 sidechain transcript 和 metadata 恢复任务状态。

教学版可以先实现输出文件：

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

这比只把结果存在内存里更可靠。

## 32.20 清理：子 Agent 结束后必须释放资源

源码里 `runAgent` 的 finally 块做了很多清理工作。清理不是可选项。子 Agent 结束后至少要处理：

- agent-specific MCP server cleanup。
- session hooks cleanup。
- prompt cache tracking cleanup。
- file state cache cleanup。
- transcript subdir mapping cleanup。
- todos entry cleanup。
- 子 Agent 启动的后台 shell 任务清理。
- perf tracing registry cleanup。

教学版可以简化：

```python
try:
    return await runChildLoop(ctx)
    } finally:
        await cleanupMcpServers(ctx)
        clearAgentHooks(ctx.agentId)
        ctx.readFileState.clear()
        clearAgentTodos(ctx.agentId)
        killShellTasksForAgent(ctx.agentId)
```

很多系统的诡异 bug 都来自没有清理：

- 后台命令还在跑。
- 旧 hooks 影响新任务。
- 内存缓存越来越大。
- 任务列表里残留已完成 Agent。
- 取消按钮取消到错误任务。

只要你实现子 Agent，就必须实现生命周期清理。

## 32.21 子 Agent 取消策略

取消有几种情况。

第一，用户取消父 Agent 当前回合。

同步子 Agent 应该一起取消，因为它正在当前回合里工作。

第二，用户取消后台子 Agent。

应该只取消该子 Agent，不影响父 Agent 和其他后台任务。

第三，父 Agent 结束，但后台子 Agent 继续。

这取决于产品设计。有些系统允许后台任务继续，有些系统在会话关闭时全部终止。

第四，子 Agent 内部工具长时间无响应。

应该有工具级超时或任务级超时。

可以实现：

```python
from typing import Any

def example(context: dict[str, Any]) -> dict[str, Any]:
    return {"ok": True, "context": context}
```

异步 Agent 如果不想被父回合取消，可以不链接父 signal。但如果父会话关闭时要清理后台任务，就需要另一个 session-level controller。

## 32.22 上下文压缩：不要等爆了才处理

上下文压缩不是主 Agent 独有的问题。子 Agent 也可能在长任务中接近上下文限制。

压缩策略有三层。

第一，局部工具结果压缩。

优先压缩大工具结果，而不是总结整个对话。这保留了更多结构信息。

第二，自动 compact。

当上下文接近阈值时，把旧消息总结成 compact boundary 后的新上下文。

第三，结果级总结。

子 Agent 完成后，只把最终总结返回给父 Agent，而不是返回完整压缩后的历史。

教学版可以先实现一个简单阈值：

```python
def if(self, estimateTokens(messages) > contextBudget * 0.8):
    messages = await compactMessages(messages)
```

但要注意，compact 不能破坏工具调用配对。assistant 的 tool_use 和 user 的 tool_result 必须一起保留或一起被摘要替换。

## 32.23 任务预算：把成本变成显式约束

成熟 Agent 系统不应该只在事后统计成本。它应该在任务开始时就有预算。

可以定义：

```python
from typing import Any

def example(context: dict[str, Any]) -> dict[str, Any]:
    return {"ok": True, "context": context}
```

不同 Agent 类型有不同预算：

```python
budgets =:
    "Explore"::
        "maxTurns": 8
        "maxInputTokens": 120_000
        "maxOutputTokens": 8_000
        "maxToolCalls": 20
        "maxWallClockMs": 60_000
    'test-fixer'::
        "maxTurns": 20
        "maxInputTokens": 300_000
        "maxOutputTokens": 20_000
        "maxToolCalls": 60
        "maxWallClockMs": 10 * 60_000
```

每次循环前检查：

```python
def assertBudgetRemaining(budget: TaskBudget, usage: TaskUsage):
    if usage.turns >= budget.maxTurns:
        throw BudgetExceededError('maxTurns exceeded')
    if usage.toolCalls >= budget.maxToolCalls:
        throw BudgetExceededError('maxToolCalls exceeded')
    if Date.now() - usage.startedAt >= budget.maxWallClockMs:
        throw BudgetExceededError('maxWallClockMs exceeded')
```

预算不是为了限制能力，而是为了让系统可预测。

## 32.24 子 Agent prompt 应该包含预算意识

除了程序层预算，提示词也应该告诉子 Agent 如何节约。

例如 Explore Agent：

```txt
你应该优先使用搜索工具定位文件。
不要读取无关大文件。
每次读取文件前，先判断它是否可能包含任务相关信息。
如果信息足够，请停止搜索并总结。
```

代码审查 Agent：

```txt
只报告具体、可证据支持的问题。
不要列出泛泛最佳实践。
如果没有发现问题，请说明检查范围。
```

测试修复 Agent：

```txt
优先运行最小相关测试。
不要在每次小修改后运行全量测试，除非任务要求或最终验证需要。
```

模型会根据提示词调整行为。程序预算是硬刹车，prompt 预算意识是软引导。两者都需要。

## 32.25 子 Agent 不应该无限递归委派

如果子 Agent 也能调用 `AgentTool`，系统可能出现递归委派：

```txt
父 Agent -> 子 Agent A -> 子 Agent B -> 子 Agent C -> ...
```

这可能导致：

- 成本不可控。
- 权限边界混乱。
- 任务树难以展示。
- 错误恢复困难。

可以设置最大深度：

```python
from typing import Any

def example(context: dict[str, Any]) -> dict[str, Any]:
    return {"ok": True, "context": context}
```

启动子 Agent 时：

```python
from typing import Any

def example(context: dict[str, Any]) -> dict[str, Any]:
    return {"ok": True, "context": context}
```

还可以按 Agent 类型禁止递归。例如只读 Explore Agent 不允许再启动 Agent。

## 32.26 上下文隔离的测试方法

子 Agent 隔离必须写测试。否则很多 bug 只有在长任务和并发任务中才出现。

测试一：子 Agent messages 不污染父 messages。

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

测试二：异步子 Agent 不共享 setAppState。

```python
from typing import Any

def example(context: dict[str, Any]) -> dict[str, Any]:
    return {"ok": True, "context": context}
```

测试三：同步子 Agent 可以响应父取消。

```python
from typing import Any

def example(context: dict[str, Any]) -> dict[str, Any]:
    return {"ok": True, "context": context}
```

测试四：子 Agent transcript 单独记录。

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

这些测试会逼着你的设计保持边界清楚。

## 32.27 本章小结

这一章我们深入讨论了子 Agent 启动之后的工程问题。

子 Agent 必须有独立上下文。它应该拥有自己的 messages、agentId、transcript、读取状态、取消策略和权限上下文。它可以共享 MCP 连接、进度通道、根任务注册表和组织级安全策略，但共享必须是显式的。

子 Agent 的完整过程应该写入 sidechain transcript，而不是塞回父 Agent。父 Agent 需要的是提炼后的交付物：具体、短、可验证、带边界和可信度。

成本控制要从输入、工具结果、回合数、模型选择、推理配置、任务预算多个层面同时做。多 Agent 系统不是 Agent 越多越好，而是每个 Agent 都有明确角色、明确预算和明确产出。

下一章我们会进一步讨论 worktree isolation 与并行实现策略。那一章会聚焦一个非常实际的问题：当多个子 Agent 都可能修改代码时，怎样让它们并行工作而不互相覆盖。
