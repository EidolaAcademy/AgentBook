# 第 31 章：深入 AgentTool，如何把一个任务交给子 Agent

上一章我们从宏观上讨论了子 Agent 和多 Agent 协作：为什么需要拆任务，什么时候适合拆任务，子 Agent 和普通工具调用有什么区别，以及一个成熟系统为什么不能简单地“多开几个模型”。本章开始进入源码级细节。我们要拆解 Claude Code 中非常关键的一个工具：`AgentTool`。

如果说 `Read`、`Grep`、`Bash`、`Edit` 这些工具让 Agent 可以“看见”和“改变”项目，那么 `AgentTool` 让 Agent 可以“委派”。委派不是把一段文本发给另一个模型这么简单。真正的委派至少包含八个问题：

1. 父 Agent 如何描述任务？
2. 系统如何选择子 Agent 类型？
3. 子 Agent 能使用哪些工具？
4. 子 Agent 是否继承父 Agent 的权限？
5. 子 Agent 是否继承父 Agent 的上下文？
6. 子 Agent 的执行结果如何回到父 Agent？
7. 子 Agent 失败、超时、被中断时如何处理？
8. 多个子 Agent 同时存在时，如何管理进度、日志和成本？

Claude Code 的 `AgentTool` 就是在回答这些问题。它表面上是一个工具，实际上更像一个“子任务运行时”。本章会先解释它在源码中的结构，然后把这些设计转化成一个新手可实现的教学版本。读完这一章，你应该能理解：为什么子 Agent 系统是 Agent 工程从“能跑”到“可扩展”的关键分水岭。

## 31.1 AgentTool 为什么是一个特殊工具

普通工具通常有明确的输入和输出。例如：

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

模型调用 `Read`，程序读取文件，然后把文件内容返回给模型。整个过程虽然有权限、缓存、截断等细节，但它的形状是单步的。

`AgentTool` 不一样。它的输入是一个任务，输出可能是另一个 Agent 的完整工作结果。调用 `AgentTool` 后，系统内部会再启动一个新的 Agent 循环。这个循环可能会：

- 读取文件。
- 搜索代码。
- 调用 Bash。
- 使用 MCP 工具。
- 修改文件。
- 触发权限检查。
- 产生自己的会话记录。
- 被放到后台运行。
- 被放进独立 worktree。
- 最终把总结返回给父 Agent。

也就是说，`AgentTool` 的输出不是一个简单函数的返回值，而是另一个 Agent 执行若干轮推理和工具调用后的结果。

从工程角度看，这意味着 `AgentTool` 不是普通工具，而是一个“调度器”。它要做的事情包括：

- 根据输入选择一个 Agent 定义。
- 根据 Agent 定义构造系统提示词。
- 根据权限上下文构造工具池。
- 根据同步或异步模式注册任务。
- 根据隔离配置决定工作目录。
- 调用 `runAgent` 启动子 Agent。
- 收集子 Agent 消息流。
- 提取最终结果。
- 记录审计信息和进度信息。

这也是为什么我们要单独花一章讲它。很多新手在做多 Agent 时，会把“调用另一个模型”当成子 Agent。那只是最小原型，不是工程系统。真正的子 Agent 必须被权限、工具、上下文、日志、预算、错误恢复这些机制包住。

## 31.2 从输入 schema 看 AgentTool 的产品形态

我们先看 `AgentTool` 的输入字段。源码中的基础输入大致可以概括成下面这样：

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

不要急着看字段数量。我们要逐个理解它们背后的意图。

`description` 是短描述。它通常用于 UI、日志、进度展示和任务列表。它不应该承载完整任务细节。比如：

```txt
review auth flow
```

`prompt` 是真正交给子 Agent 的任务说明。它应该包含目标、约束、输出格式、必要背景。例如：

```txt
请检查 src/auth 下的登录流程，重点找出 token 刷新、错误处理和并发请求中的潜在问题。
不要修改文件，只返回发现的问题、证据文件路径和建议修复方向。
```

`subagent_type` 用来指定子 Agent 类型。比如你可以有：

- `general-purpose`
- `Explore`
- `Plan`
- `security-reviewer`
- `frontend-designer`
- `test-writer`

不同类型的 Agent 会有不同系统提示词、工具权限、模型配置和工作方式。

`model` 是模型覆盖。某些任务更适合便宜快速的模型，某些任务需要更强推理。源码中允许调用方传入 `sonnet`、`opus`、`haiku` 这样的模型别名，但最终模型选择还会结合 Agent 定义和当前权限模式。

`run_in_background` 表示是否后台运行。如果子 Agent 任务很长，例如全仓库审查、长时间测试、生成大报告，就不应该阻塞父 Agent 当前回合。

`name` 和 `team_name` 与更高级的多 Agent 团队模式有关。普通子 Agent 是一次性委派；命名 teammate 则更像一个可寻址的协作者，之后可以通过消息继续交流。

`mode` 是权限模式。比如让某个 teammate 以 plan 模式运行，必须先提出计划再执行。

`isolation` 决定隔离方式。`worktree` 表示在独立 git worktree 中运行，避免多个 Agent 同时修改同一份工作区互相踩踏。

`cwd` 指定子 Agent 的工作目录。它和 worktree 隔离互斥，因为 worktree 本身就会建立新的目录上下文。

这些字段说明一件事：`AgentTool` 不是“把 prompt 发给另一个模型”的工具，而是“启动一个受控工作单元”的工具。

## 31.3 description 和 prompt 的区别

很多人第一次设计子 Agent 工具时，会只留一个 `task` 字段：

```python
from typing import Any

def example(context: dict[str, Any]) -> dict[str, Any]:
    return {"ok": True, "context": context}
```

这能跑，但很快会出问题。因为同一段任务文本同时承担了三个角色：

- 给模型看的完整任务。
- 给用户看的短标题。
- 给日志和后台任务列表看的索引。

这些角色的需求不同。

给模型看的内容越详细越好，里面可能有多段约束、上下文、输出格式。给用户看的短标题则必须简短，否则任务列表会难以阅读。给日志看的字段最好稳定，方便搜索和聚合。

所以 Claude Code 把它拆成 `description` 和 `prompt`。

教学实现中也建议这样拆：

```python
from typing import Any

def example(context: dict[str, Any]) -> dict[str, Any]:
    return {"ok": True, "context": context}
```

然后在调用子 Agent 时：

```python
taskTitle = input.description
childUserMessage =:
    "role": 'user'
    "content": input.prompt
```

这样你可以在 UI 上显示：

```txt
Running agent: review auth flow
```

同时给子 Agent 的内容仍然是完整说明：

```txt
请检查 src/auth 下的登录流程，重点找出 token 刷新、错误处理和并发请求中的潜在问题……
```

这是一个小设计，但它体现了成熟 Agent 系统的一个原则：不要让一个字段承担太多语义。

## 31.4 子 Agent 类型是怎么被选择的

`AgentTool` 的核心步骤之一是选择 `selectedAgent`。逻辑可以简化成：

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

真实源码比这个复杂，因为它还要处理 fork subagent、权限过滤、allowedAgentTypes、多 Agent team 和实验开关。但对新手来说，先理解普通路径就够了。

Agent 类型本质上是一份配置。它通常包含：

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

例如一个只读探索 Agent：

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

再比如一个测试修复 Agent：

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

父 Agent 调用 `AgentTool` 时，可以显式说：

```json
{
  "description": "fix failing tests",
  "subagent_type": "test-fixer",
  "prompt": "请运行测试并修复当前失败的用例。只做必要修改。"
}
```

如果不传 `subagent_type`，系统会使用默认 Agent。Claude Code 中默认通常会落到 general-purpose 或 fork path，取决于功能开关。你的教学版可以先固定为 `general-purpose`。

## 31.5 为什么要过滤 Agent

真实系统中，不是所有 Agent 都永远可用。源码里在展示提示词和执行调用时都会过滤 Agent。常见过滤原因包括：

- 某个 Agent 需要的 MCP server 当前不可用。
- 权限规则禁止使用某类 Agent。
- 当前工具定义只允许调用特定 Agent 类型。
- 当前运行模式不允许 teammate 再创建 teammate。
- 某些后台任务能力被禁用。

这听起来像边角逻辑，但它非常重要。因为一旦 Agent 可以委派，就可能绕过限制。

举个例子，假设父 Agent 当前不能调用 `Bash`，但它可以调用一个拥有 `Bash` 的子 Agent。如果系统不做限制，父 Agent 就可以通过子 Agent 间接执行命令。这叫权限绕过。

再举一个例子，某个 `slack-researcher` Agent 依赖 Slack MCP。如果 Slack MCP 未认证，模型仍然看到这个 Agent，就可能不断调用它，然后得到失败。更好的方式是在 prompt 展示阶段就把不可用 Agent 过滤掉。

教学版可以实现两个过滤器：

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

完整选择流程可以写成：

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

这里的关键不是代码本身，而是观念：Agent 类型也是一种能力，必须进入权限系统。

## 31.6 required MCP servers：能力不可用时不要假装能用

Claude Code 中的 Agent 定义可以声明 `requiredMcpServers`。这表示某个 Agent 只有在对应 MCP server 可用并且有工具时才应该被使用。

比如：

```yaml
---
name: github-reviewer
description: Review GitHub issues and pull requests
requiredMcpServers:
  - github
---
```

如果 GitHub MCP 没配置，或者配置了但未登录，系统不应该让模型以为 `github-reviewer` 可以工作。

源码中有两个关键点：

第一，prompt 阶段会过滤掉 MCP 要求不满足的 Agent。这样模型在工具说明里不会看到不可用选项。

第二，call 阶段仍然重新检查。因为 prompt 展示和实际调用之间可能有状态变化。比如 MCP server 刚开始是 pending，后来失败了；或者模型通过旧上下文知道某个 Agent 名称，仍然尝试调用。

教学版可以这样实现：

```python
from typing import Any

def example(context: dict[str, Any]) -> dict[str, Any]:
    return {"ok": True, "context": context}
```

调用时：

```python
from typing import Any

def example(context: dict[str, Any]) -> dict[str, Any]:
    return {"ok": True, "context": context}
```

注意这里有一个工程细节：等待 pending 状态是合理的，但不能无限等。Agent 系统里任何外部依赖都要有超时，否则一次委派就可能把整个任务卡死。

## 31.7 构造子 Agent 的系统提示词

选中 Agent 后，下一步是构造系统提示词。普通模型调用通常只有一个全局 system prompt，而子 Agent 系统至少有三层提示词：

1. 主系统提示词：整个产品的通用行为规则。
2. Agent 类型提示词：这个子 Agent 的专业角色和工作边界。
3. 环境提示词：当前目录、工具能力、上下文限制等运行时信息。

Claude Code 中每个 Agent 都有 `getSystemPrompt`。对于 built-in agent，它可能依赖当前 `toolUseContext` 动态生成；对于自定义 agent，它通常来自 Markdown frontmatter 后面的正文。

一个教学版构造函数可以这样写：

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

举个例子，`Explore` Agent 的系统提示词可能强调：

- 只读。
- 先搜索，再阅读。
- 返回证据路径。
- 不要修改文件。
- 不要运行破坏性命令。

`test-fixer` Agent 的系统提示词可能强调：

- 优先复现失败。
- 只做最小修改。
- 每次修改后运行相关测试。
- 不要扩大范围。

这些差异就是子 Agent 的价值。如果所有子 Agent 都用同一个 system prompt，只是换个名字，那多 Agent 系统不会变聪明，只会变贵。

## 31.8 构造子 Agent 的 prompt messages

除了系统提示词，还要构造传给子 Agent 的消息列表。最简单的方式是只传一个用户消息：

```python
promptMessages = [

    "role": 'user'
    "content": input.prompt
]
```

这对应 Claude Code 中普通路径的做法：把 `prompt` 包装成 user message，作为子 Agent 的初始任务。

但是复杂系统会有更多路径。

一种路径是普通子 Agent。它不继承父 Agent 的完整对话，只收到任务说明。这样成本低、边界清晰。

另一种路径是 fork subagent。它会继承父 Agent 的更多上下文，用来保持 prompt cache 或让子 Agent 接着父 Agent 的状态往下推理。这种方式更强，但也更贵、更复杂，而且要特别处理未完成工具调用、历史消息压缩和上下文一致性。

新手阶段建议先做普通路径。也就是：

```python
def buildChildMessages(prompt: str):
    return [
    
        "role": 'user'
        "content": prompt
    ]
```

等普通委派跑通之后，再考虑是否需要上下文继承。

这里有一个非常重要的经验：默认不要把父 Agent 的完整历史塞给子 Agent。原因有三个：

第一，成本会迅速膨胀。父 Agent 的历史可能已经很长，如果每个子 Agent 都复制一份，token 消耗会呈倍数增长。

第二，子 Agent 容易被无关信息干扰。一个代码搜索任务不需要知道父 Agent 前面十轮关于 UI 颜色的讨论。

第三，权限边界会变模糊。父 Agent 历史中可能有不适合给某类子 Agent 的信息。

更好的方式是让父 Agent 写清楚任务，把必要上下文压缩进 `prompt`。

## 31.9 子 Agent 的工具池不能随便继承

源码里一个非常关键的设计是：子 Agent 的工具池由系统重新组装，而不是直接使用父 Agent 当前工具池。

简化理解：

```python
workerPermissionContext =:
    ...parentPermissionContext
    "mode": selectedAgent.permissionMode  or  'acceptEdits'

workerTools = assembleToolPool(workerPermissionContext, mcpTools)
```

为什么不直接继承父工具？

因为父 Agent 当前看到的工具集合，可能是为父 Agent 当前模式定制的。子 Agent 有自己的角色和权限。例如：

- 父 Agent 可能在 plan 模式，不能编辑。
- 某个子 Agent 可能被定义为专门修复测试，可以编辑。
- 某个只读子 Agent 必须禁用 Edit 和 Write。
- 某个后台子 Agent 不能弹出交互式权限确认。

所以更合理的做法是：

1. 先根据子 Agent 的权限模式组装完整工具池。
2. 再根据 Agent 定义中的 `tools` 和 `disallowedTools` 过滤。
3. 再根据运行模式过滤，例如异步任务不允许需要交互确认的工具。

教学版可以实现：

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

这段代码表达了一个原则：工具权限要从 Agent 定义中推导出来，而不是从模型意愿中推导出来。

模型可以请求工具，但不能决定自己拥有什么工具。

## 31.10 permissionMode：子 Agent 是否能自己申请权限

在 Agent 系统中，权限不仅是“允许还是不允许”。还有一个问题：如果遇到需要用户确认的操作，谁来确认？

同步子 Agent 还比较简单。它运行在父 Agent 当前回合里，如果需要用户确认，可以冒泡到当前界面。

异步子 Agent 更麻烦。它可能在后台跑，用户不一定正在看它。如果它需要弹窗确认，就会造成非常差的体验。因此源码里对异步 Agent 会设置类似 `shouldAvoidPermissionPrompts` 的上下文，让它尽量不要触发交互式权限确认。

教学版可以把权限模式简化成：

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

含义如下：

- `readOnly`：只能读，不能写，不能执行危险命令。
- `acceptEdits`：允许文件编辑，但危险命令仍需检查。
- `ask`：需要用户确认。
- `bypassPermissions`：跳过权限检查，通常只在受信环境使用。
- `bubble`：子 Agent 的权限请求冒泡给父界面。

创建子 Agent 上下文时：

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

这个函数背后的原则是：子 Agent 可以拥有自己的权限模式，但它不能随意突破系统边界。父上下文里的组织级 deny 规则、工作目录限制、安全策略仍然应该保留。

## 31.11 同步子 Agent 与异步子 Agent

`AgentTool` 可以同步运行，也可以后台运行。

同步运行的流程是：

```txt
父 Agent 调用 AgentTool
        |
        v
启动子 Agent
        |
        v
等待子 Agent 完成
        |
        v
把结果返回父 Agent
        |
        v
父 Agent 继续推理
```

异步运行的流程是：

```txt
父 Agent 调用 AgentTool
        |
        v
注册后台任务
        |
        v
立即返回 agentId / outputFile
        |
        v
父 Agent 可继续工作
        |
        v
子 Agent 后台完成后发送通知或写结果文件
```

同步适合短任务，例如：

- 搜索某个函数定义。
- 检查一小段代码。
- 总结一个目录结构。

异步适合长任务，例如：

- 全仓库安全审查。
- 跑完整测试套件并修复。
- 大规模迁移。
- 生成长报告。

教学版可以先实现同步，再实现异步。同步版本大致如下：

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

异步版本则需要任务注册表：

```python
from typing import Any

def example(context: dict[str, Any]) -> dict[str, Any]:
    return {"ok": True, "context": context}
```

启动后台任务：

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

这只是最小版本。生产系统还要考虑取消、进度、输出文件、日志、恢复和资源清理。

## 31.12 runAgentParams：启动子 Agent 的参数包

源码中 `AgentTool` 最终会组装一个大参数包传给 `runAgent`。这个参数包非常值得学习，因为它展示了启动一个子 Agent 到底需要哪些东西。

可以把它抽象成：

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

逐个解释：

`agentDefinition` 是 Agent 的身份和规则。

`promptMessages` 是子 Agent 的初始用户消息。

`toolUseContext` 是父级运行上下文，里面有 app state、权限上下文、abort controller、MCP client、会话信息等。

`canUseTool` 是权限判断函数。子 Agent 不应该绕过工具权限系统。

`isAsync` 决定子 Agent 的生命周期、权限弹窗策略和任务注册方式。

`querySource` 用来标记请求来源，方便日志、统计和调试。

`model` 是模型覆盖。

`availableTools` 是子 Agent 可见的工具池。

`allowedTools` 可以进一步限制工具允许列表，防止父会话的临时 allow 规则泄漏到子 Agent。

`maxTurns` 控制最多推理轮数，避免子 Agent 无限循环。

`worktreePath` 表示如果启用了 worktree 隔离，子 Agent 应该在隔离目录里工作。

`description` 用于后台任务显示、元数据和恢复。

你可以看到，一个成熟的 `runAgent` 不是只需要 `prompt`。它需要的是完整运行时上下文。

## 31.13 子 Agent 的上下文隔离

一个新手常犯错误是让所有 Agent 共享同一个 messages 数组。这样做看起来方便，但后果很糟：

- 子 Agent 的工具调用会污染父 Agent 历史。
- 父 Agent 的未完成工具调用可能导致子 Agent API 请求非法。
- 子 Agent 的大量搜索过程会把父 Agent 上下文挤爆。
- 多个子 Agent 并发写同一个数组会造成顺序混乱。

正确做法是给子 Agent 创建自己的上下文：

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

创建时：

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

注意 abort controller 的差异。同步子 Agent 可以共享父 Agent 的 abort signal，因为用户取消当前回合时，子 Agent 也应该停。异步子 Agent 通常需要自己的 abort controller，因为它已经脱离父回合继续运行。

## 31.14 进度事件：不要让用户面对黑盒

当子 Agent 跑很久时，如果界面没有反馈，用户会感觉系统卡住了。Claude Code 的 `AgentTool` 会处理 progress，并在后台任务、SDK event、UI 展示之间同步状态。

教学版可以先实现一个简单的进度回调：

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

运行子 Agent 时：

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

进度事件不只是 UI 装饰。它还有工程价值：

- 可以判断任务是否还活着。
- 可以在日志中定位慢工具。
- 可以向 SDK 用户暴露状态。
- 可以让父 Agent 决定是否等待或转入后台。

一个看不见进度的子 Agent 系统，很难被用户信任。

## 31.15 子 Agent 的结果应该怎么返回

同步子 Agent 的返回值通常包括：

```python
from typing import Any

def example(context: dict[str, Any]) -> dict[str, Any]:
    return {"ok": True, "context": context}
```

异步子 Agent 的返回值通常包括：

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

为什么异步不直接返回结果？因为结果还没有产生。它只能返回“你可以在哪里查这个任务”。

Claude Code 的输出 schema 中也有类似区分：同步完成时返回 completed；后台启动时返回 async_launched，并包含 agentId、description、prompt、outputFile 等信息。

教学版可以这样写：

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

调用函数：

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

这里还有一个细节：返回给父 Agent 的结果不要太大。即使子 Agent 读了大量文件，最终也应该总结为父 Agent 可以使用的短结果。否则父 Agent 的上下文会被子 Agent 输出淹没。

## 31.16 错误处理：子 Agent 失败不等于整个系统崩溃

子 Agent 可能失败。失败原因包括：

- Agent 类型不存在。
- 需要的 MCP server 不可用。
- 模型 API 失败。
- 工具权限被拒绝。
- Bash 命令失败。
- 超过 maxTurns。
- 用户取消。
- worktree 创建失败。

`AgentTool` 需要把这些错误变成可理解的结果，而不是让整个进程崩掉。

教学版可以定义：

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

选择 Agent 时：

```python
def if(self, !selectedAgent):
    throw AgentToolError(
    "Agent type '{effectiveType}' not found"
    'AGENT_NOT_FOUND'
    )
```

能力缺失时：

```python
throw AgentToolError(
"Agent '{agent.agentType}' requires missing capability '{capability}'"
'CAPABILITY_MISSING'
)
```

父 Agent 收到错误后，可以把它作为工具调用失败处理。模型可以据此改用其他方案。例如：

```txt
Agent type 'github-reviewer' requires GitHub MCP, but GitHub MCP is not available.
```

父 Agent 可能接着说：

```txt
GitHub MCP 不可用。我将改为只基于本地代码进行审查。
```

这就是错误可解释性的价值。错误不是为了责备模型，而是为了帮助模型恢复。

## 31.17 maxTurns：给子 Agent 设置回合上限

子 Agent 如果没有回合上限，可能陷入循环。比如：

1. 搜索文件。
2. 读文件。
3. 觉得还需要搜索。
4. 又读文件。
5. 又觉得还需要搜索。

这在代码库很大时尤其常见。

所以 Agent 定义中应该支持 `maxTurns`：

```python
from typing import Any

def example(context: dict[str, Any]) -> dict[str, Any]:
    return {"ok": True, "context": context}
```

运行时：

```python
from typing import Any

def example(context: dict[str, Any]) -> dict[str, Any]:
    return {"ok": True, "context": context}
```

不同 Agent 的上限可以不同：

- `Explore`：8 到 12 轮通常够用。
- `test-fixer`：15 到 25 轮比较合理。
- `migration-agent`：可能需要更高，但最好后台运行。

不要把 maxTurns 当成随便填的数字。它直接影响成本、延迟和安全。

## 31.18 worktree isolation：让子 Agent 有独立工作区

当多个子 Agent 都可能修改文件时，共享工作区会非常危险。

想象父 Agent 同时启动两个子 Agent：

- A 负责修复认证测试。
- B 负责重构认证组件。

如果它们在同一个目录编辑同一个文件，可能出现：

- A 读到旧文件，B 已经改了。
- A 写入时覆盖 B 的修改。
- 测试结果无法归因。
- 最终 diff 混在一起，难以审查。

worktree isolation 的思路是：为子 Agent 创建一个临时 git worktree，让它在独立目录里修改。完成后，再由父 Agent 或用户决定如何合并。

教学版可以先不完整实现 git worktree，但要保留接口：

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

生产系统里还要处理：

- worktree 创建失败。
- 子 Agent 运行完成后是否保留 worktree。
- worktree 中是否有未提交修改。
- 如何把修改带回主工作区。
- 路径提示如何告诉子 Agent 当前目录已变化。

本书后面的章节会单独讲 worktree isolation，这里先记住：当子 Agent 能写文件时，隔离不是锦上添花，而是并发修改的安全基础。

## 31.19 自定义 Agent 文件如何变成 AgentDefinition

Claude Code 支持从 Markdown 或 JSON 中加载自定义 Agent。一个典型 Markdown Agent 可能长这样：

```md
---
name: api-reviewer
description: Review API design and backend changes
tools:
  - Read
  - Grep
  - Bash
model: sonnet
permissionMode: readOnly
maxTurns: 12
---

你是一个 API 审查 Agent。
你应该重点检查接口设计、错误处理、认证授权、数据兼容性和测试覆盖。
不要修改文件，只返回结构化审查结果。
```

加载器要做几件事：

1. 读取 agents 目录下的 Markdown 文件。
2. 解析 frontmatter。
3. 校验必填字段。
4. 把正文变成 `getSystemPrompt`。
5. 合并 built-in、plugin、user、project、policy 等来源。
6. 处理同名覆盖优先级。

教学版可以简化成：

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

同名覆盖可以用 Map：

```python
from typing import Any

def example(context: dict[str, Any]) -> dict[str, Any]:
    return {"ok": True, "context": context}
```

注意覆盖顺序很重要。Claude Code 中会区分 built-in、plugin、userSettings、projectSettings、flagSettings、policySettings 等来源。教学版可以先定义：

```python
activeAgents = mergeAgents([
builtInAgents
pluginAgents
userAgents
projectAgents
policyAgents
```

越靠后的来源优先级越高。

## 31.20 一个完整的教学版 AgentTool

现在我们把前面的内容拼成一个可读的教学版。这个版本不是 Claude Code 源码的逐行复刻，而是保留核心结构，去掉实验开关、远程任务和复杂 UI。

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

这段代码包含了 AgentTool 的骨架：

- 输入解析。
- 默认 Agent 类型。
- Agent 过滤。
- 能力检查。
- 同步/异步判断。
- 子权限上下文。
- 子工具池。
- 子 Agent 执行。
- 结果返回。

你可以先用这个版本跑通，再逐渐加入：

- 进度事件。
- transcript。
- output file。
- worktree。
- MCP server 初始化和清理。
- hooks。
- agent memory。
- fork path。

这就是学习复杂源码的正确方法：先抓骨架，再补肌肉，最后理解神经系统。

## 31.21 一个端到端例子：让父 Agent 委派代码审查

假设我们有三个 Agent：

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

父 Agent 想审查认证模块：

```python
await callAgentTool(

    "description": 'review auth module'
    "subagent_type": 'code-reviewer'
    "prompt": "
    请审查 src/auth 目录。
    重点关注 token 刷新、权限判断、错误处理和测试覆盖。
    不要修改文件。
    输出格式：
    1. 严重问题
    2. 中等问题
    3. 建议
    每条都要带文件路径。
    "
runtime
)
```

系统会：

1. 找到 `code-reviewer`。
2. 确认它没有被权限规则禁止。
3. 根据它的 `permissionMode` 创建只读权限上下文。
4. 根据 `tools` 只给它 `Read`、`Grep`、`Glob`。
5. 构造代码审查系统提示词。
6. 把 prompt 作为用户消息交给它。
7. 启动子 Agent 循环。
8. 收集最终审查结果。
9. 返回父 Agent。

父 Agent 收到结果后，可以继续：

```txt
我已经让 code-reviewer 检查了认证模块。它发现两个中等风险：
1. refresh token 失败时错误被吞掉。
2. 并发刷新可能导致旧 token 覆盖新 token。

接下来我会读取对应文件并准备最小修复。
```

这就是“委派”的基本体验。

## 31.22 常见错误一：子 Agent 任务写得太短

坏例子：

```txt
检查代码
```

这个任务对人类都不清楚，对子 Agent 更不清楚。它不知道检查哪里、检查什么、是否能修改、输出什么。

好例子：

```txt
请检查 src/payment 下最近修改的支付流程。
重点关注金额精度、失败重试、幂等性和日志脱敏。
不要修改文件。
输出每个问题的文件路径、风险等级、证据和建议修复方式。
```

委派任务时，父 Agent 应该像一个合格的技术负责人，而不是只丢一句模糊指令。

## 31.23 常见错误二：所有子 Agent 都给全工具

很多原型系统会让所有子 Agent 都拥有全部工具。这在演示里方便，在真实项目里危险。

一个只负责搜索的 Agent 不应该拥有 `Edit`。一个只负责总结文档的 Agent 不应该拥有 `Bash`。一个后台 Agent 不应该轻易触发交互式权限弹窗。

建议默认策略是：

- 子 Agent 默认只读。
- 只有明确需要修改时才给 Edit/Write。
- 只有明确需要运行命令时才给 Bash。
- 后台任务要更保守。
- 高风险工具必须经过权限系统。

工具越强，系统越要克制。

## 31.24 常见错误三：父 Agent 不验证子 Agent 结果

子 Agent 的结果不是绝对真理。它可能漏看文件、误读代码、测试没跑全，或者因为权限限制没有完成任务。

父 Agent 收到子 Agent 结果后，应该进行最低限度验证。例如：

- 子 Agent 说某文件有问题，父 Agent 可以再读该文件。
- 子 Agent 说测试通过，父 Agent 可以查看测试命令输出。
- 子 Agent 给出修改建议，父 Agent 可以检查是否符合项目约束。

这不是不信任子 Agent，而是协作系统的基本质量控制。

一个好的父 Agent 提示词应该包含：

```txt
当你收到子 Agent 的结果时，把它视为辅助证据。
在执行关键修改或向用户汇报之前，必要时读取相关文件或运行相关检查。
```

## 31.25 常见错误四：后台任务没有可查询结果

异步子 Agent 不能只返回“已启动”。它必须提供后续查询方式。

最少也要返回：

- taskId 或 agentId。
- description。
- 当前状态。
- 输出文件路径或查询接口。

否则父 Agent 和用户都不知道任务最后怎么样了。

教学版可以提供一个查询工具：

```python
def getTaskStatus(taskId: str):
    return tasks.get(taskId)
```

更完整的系统可以把后台任务结果写入文件：

```txt
.agent/tasks/<taskId>/result.md
.agent/tasks/<taskId>/events.jsonl
```

这样即使进程重启，也可以恢复任务记录。

## 31.26 从 Claude Code 学到的设计原则

本章对应的源码很多，但最值得带走的是这些原则。

第一，子 Agent 是运行时对象，不是 prompt 字符串。

它有身份、系统提示词、工具、权限、模型、上下文、日志和生命周期。

第二，Agent 选择必须经过过滤。

不能因为某个 Agent 定义存在，就让模型随便调用。MCP 依赖、权限规则、allowed types、运行模式都可能影响可用性。

第三，工具池应该为子 Agent 重新构造。

不要把父 Agent 当前看到的工具原样传下去。子 Agent 的工具应该由系统根据它的角色和权限生成。

第四，同步和异步是两种不同生命周期。

同步适合短任务，异步适合长任务。异步任务需要任务注册、进度、查询和取消。

第五，结果要总结，不要倾倒。

子 Agent 可以看很多东西，但返回给父 Agent 的应该是高密度、可行动的结果。

第六，隔离是多 Agent 修改代码的前提。

如果多个 Agent 可能同时写文件，worktree 或类似隔离机制非常重要。

第七，错误要可解释。

`Agent not found`、`MCP missing`、`permission denied`、`max turns exceeded` 应该清楚地返回给父 Agent，让它能恢复。

## 31.27 本章小结

这一章我们深入拆解了 `AgentTool`。它表面上是工具，实际上是子 Agent 调度器。它把一个任务从父 Agent 手里接过来，完成 Agent 类型选择、权限过滤、能力检查、系统提示词构造、工具池组装、同步或异步执行、进度跟踪和结果返回。

对新手来说，最重要的是不要一开始就追求复杂多 Agent。你应该按下面顺序实现：

1. 定义 `AgentDefinition`。
2. 实现默认子 Agent。
3. 实现 `description` 和 `prompt` 分离。
4. 实现 Agent 类型选择。
5. 实现工具白名单和黑名单。
6. 实现同步子 Agent。
7. 实现 maxTurns。
8. 实现异步任务注册。
9. 实现进度事件。
10. 实现 worktree 隔离。

这条路径走完，你就不再只是“调用模型”，而是在搭建一个真正的 Agent 运行时。

下一章我们会继续深入子 Agent 的上下文隔离、结果汇总和成本控制。那一章会回答一个更难的问题：当子 Agent 越来越多时，如何避免上下文爆炸、成本爆炸和结果混乱。
