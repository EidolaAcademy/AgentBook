# 第 30 章：子 Agent、任务分解与多 Agent 协作总览

前面几卷我们一直在构建“一个 Agent”：

- 它能和模型对话。
- 它能调用工具。
- 它能管理上下文。
- 它能执行命令和编辑文件。
- 它有权限、沙箱、hooks、审计。

现在我们进入一个新的阶段：

```text
一个 Agent 不够用了。
```

复杂工程任务经常需要多条探索线并行推进。

例如用户说：

```text
帮我分析这个项目的认证系统、权限系统和数据库迁移逻辑，然后给出重构方案。
```

主 Agent 如果自己做所有事，会遇到几个问题：

- 上下文很快膨胀。
- 搜索路径太多。
- 任务之间互相干扰。
- 主线推理被细节淹没。
- 很难并行探索。

子 Agent 的价值就是：

```text
把一个大任务拆成多个小任务，让专门的 Agent 去完成，再把结果汇总给主 Agent。
```

这一章是第 5 卷的总览。我们先讲为什么需要子 Agent、子 Agent 应该怎么定义、主 Agent 和子 Agent 的职责如何划分。下一章再深入 Claude Code 的 AgentTool 实现。

## 30.1 什么是子 Agent

子 Agent 是由主 Agent 启动的另一个 Agent。

它通常有：

- 自己的系统提示。
- 自己的任务 prompt。
- 自己的工具集合。
- 自己的上下文窗口。
- 自己的权限模式。
- 自己的执行生命周期。
- 自己的结果输出。

主 Agent 不直接执行所有细节，而是委派：

```text
主 Agent：
  请 Explore agent 搜索认证相关代码。

Explore agent：
  搜索文件、阅读代码、返回发现。

主 Agent：
  根据 Explore agent 的结果继续制定方案。
```

这和普通函数调用不一样。

函数调用是确定性代码。  
子 Agent 是一个新的推理循环。

它可以自己：

- 搜索。
- 读取。
- 调用工具。
- 多轮推理。
- 形成结论。

## 30.2 为什么不是让主 Agent 一直做

主 Agent 可以做很多事，但它有天然限制。

第一，上下文窗口有限。

如果主 Agent 搜索整个项目，把所有文件都读进来，很快会撑爆上下文。

第二，注意力有限。

模型在一个巨大上下文里，很容易漏掉细节。

第三，任务分支多。

认证、权限、数据库、前端状态、部署脚本，每条线都可以单独探索。

第四，并行性不足。

主 Agent 一次只沿着一条思路走。子 Agent 可以并发探索多个方向。

第五，角色不同。

有些 Agent 应该只读，有些 Agent 可以编辑，有些 Agent 专门验证，有些 Agent 专门写测试。

所以成熟系统会引入：

```text
主 Agent 负责规划、协调、最终决策。
子 Agent 负责专门探索、执行或验证。
```

## 30.3 子 Agent 和工具调用的区别

你可能会问：

```text
子 Agent 不也是一种工具吗？
```

在 Claude Code 里，AgentTool 确实是一个工具。但它和普通工具不同。

普通工具：

```text
输入 -> 执行一次 -> 输出
```

子 Agent 工具：

```text
输入任务 -> 启动一个 Agent 循环 -> 多次模型调用和工具调用 -> 输出总结
```

普通工具像函数。  
子 Agent 像派出去的一位同事。

这带来更多工程问题：

- 子 Agent 能用哪些工具？
- 子 Agent 能不能再启动子 Agent？
- 子 Agent 的权限是什么？
- 子 Agent 运行多久？
- 子 Agent 失败怎么办？
- 子 Agent 输出太长怎么办？
- 子 Agent 是否共享主 Agent 上下文？
- 子 Agent 能不能修改文件？
- 子 Agent 是否要在隔离 worktree 里运行？

这些都是第 5 卷要解决的问题。

## 30.4 Claude Code 的 AgentTool

Claude Code 中负责启动子 Agent 的工具是 `AgentTool`。

它的输入大致包含：

```text
description
  任务短描述。

prompt
  给子 Agent 的完整任务。

subagent_type
  使用哪种子 Agent。

model
  可选模型覆盖。

run_in_background
  是否后台运行。

isolation
  是否使用 worktree 或远程隔离。

cwd
  子 Agent 工作目录覆盖。
```

这已经说明 AgentTool 不是普通工具。

它要处理：

- agent 类型选择。
- agent 定义加载。
- 权限和工具池。
- MCP server。
- 后台任务。
- 进度。
- worktree 隔离。
- 远程执行。
- fork subagent。
- 多 Agent team。

这一章先不追每个分支，先建立概念。

## 30.5 AgentDefinition：子 Agent 的说明书

Claude Code 中每个 Agent 都有 AgentDefinition。

它包含：

- agentType
- whenToUse
- tools
- disallowedTools
- skills
- mcpServers
- hooks
- model
- effort
- permissionMode
- maxTurns
- background
- isolation
- memory
- getSystemPrompt

你可以把 AgentDefinition 理解成：

```text
这种子 Agent 的说明书。
```

例如 Explore agent：

```text
agentType: Explore
whenToUse: 快速探索代码库
tools: Read, Grep, Glob, Bash read-only
disallowedTools: Edit, Write, Agent
model: haiku 或 inherit
omitClaudeMd: true
```

它的角色非常明确：

```text
只读探索，不改文件。
```

这就是好 AgentDefinition 的特点：职责清楚，工具受限，输出目标明确。

## 30.6 子 Agent 类型

常见子 Agent 类型包括：

探索 Agent：

```text
负责搜索和阅读代码。
通常只读。
适合快速收集事实。
```

计划 Agent：

```text
负责分析方案。
通常不改文件。
适合给出重构或迁移计划。
```

验证 Agent：

```text
负责检查实现是否满足要求。
可以读代码、运行测试。
通常不直接修改。
```

实现 Agent：

```text
负责执行一部分改动。
需要编辑权限。
最好配合 worktree 隔离。
```

文档 Agent：

```text
负责整理文档、生成总结、更新说明。
```

运维 Agent：

```text
负责查看 CI、部署配置、日志。
需要更严格权限。
```

不要让所有子 Agent 都是“通用 Agent”。专业化是子 Agent 的价值。

## 30.7 主 Agent 和子 Agent 的职责边界

主 Agent 应该负责：

- 理解用户目标。
- 拆分任务。
- 选择合适子 Agent。
- 给子 Agent 清晰 prompt。
- 汇总结果。
- 做最终决策。
- 和用户沟通。

子 Agent 应该负责：

- 完成一个边界清楚的子任务。
- 不擅自扩大任务范围。
- 报告事实、证据和结论。
- 明确不确定性。
- 不直接代表用户做最终产品决策。

例如主 Agent 不应该这样派任务：

```text
帮我看看项目。
```

太模糊。

应该这样：

```text
搜索项目中认证相关代码。重点找登录、session、token、middleware、权限校验。不要修改文件。返回文件路径、关键函数和你的结论。
```

子 Agent prompt 越清楚，结果越可用。

## 30.8 子 Agent 工具必须受限

子 Agent 不应该默认拥有主 Agent 的全部工具。

原因很简单：

```text
委派不是放权。
```

探索 Agent 不需要 Edit。  
验证 Agent 不一定需要 Write。  
文档 Agent 不需要 Bash 的危险命令。  
Slack Agent 不需要文件写权限。

Claude Code 的 AgentDefinition 支持：

- tools
- disallowedTools
- permissionMode

Explore agent 明确 disallow：

- Agent
- ExitPlanMode
- Edit
- Write
- NotebookEdit

这防止 Explore agent 越权修改项目，或者递归启动更多子 Agent。

教学版也应该：

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

创建子 Agent 工具池：

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

## 30.9 子 Agent 的上下文应该隔离

子 Agent 不应该总是继承主 Agent 的完整上下文。

为什么？

第一，浪费 token。

主 Agent 对话里很多内容和子任务无关。

第二，容易干扰。

子 Agent 只需要完成一个明确任务，太多历史会让它偏题。

第三，隐私和权限。

主 Agent 可能知道一些子 Agent 不该知道的信息。

第四，缓存和成本。

每个子 Agent 都带完整上下文，会非常贵。

所以默认方式应该是：

```text
子 Agent 只收到任务 prompt + 必要系统提示 + 必要环境上下文。
```

Claude Code 也有特殊 fork subagent 模式，可以继承父上下文，但那是专门设计过的路径，不是默认全部复制。

## 30.10 子 Agent 输出应该是总结，不是全量日志

子 Agent 执行过程中可能读了很多文件，跑了很多命令。

主 Agent 不需要它的全部过程。

主 Agent 需要：

- 结论。
- 证据路径。
- 关键片段摘要。
- 不确定性。
- 下一步建议。

所以子 Agent 的最终输出应该像报告：

```text
Findings:
1. Auth entry point is src/auth/login.py.
2. Session validation happens in src/middleware/auth.py.
3. Permission checks are scattered across ...

Evidence:
- src/auth/login.py: ...
- src/middleware/auth.py: ...

Risks:
- Token refresh has no expiry check...
```

不要把子 Agent 的所有 tool results 原样塞回主 Agent。

## 30.11 同步子 Agent 和后台子 Agent

子 Agent 可以同步运行，也可以后台运行。

同步：

```text
主 Agent 等子 Agent 完成。
适合短任务、必须马上用结果。
```

后台：

```text
子 Agent 独立运行。
主 Agent 或用户稍后查看结果。
适合长任务。
```

Claude Code 的 AgentTool 支持 `run_in_background`，也有后台任务注册、进度更新、输出文件路径等机制。

教学版可以先做同步：

```python
result = await runAgent(task)
return result.summary
```

再做后台：

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

后台子 Agent 的难点：

- 取消。
- 进度。
- 输出存储。
- 用户通知。
- 权限提示。
- 结果恢复。

## 30.12 子 Agent 和隔离 worktree

如果子 Agent 要修改代码，最好给它隔离环境。

否则多个 Agent 同时编辑同一个工作区，很容易冲突。

隔离 worktree 的好处：

- 子 Agent 可以独立修改。
- 主工作区不被直接污染。
- 可以检查 diff 后再合并。
- 多个实现 Agent 可以并行尝试不同方案。

Claude Code 的 AgentTool 支持：

```text
isolation: "worktree"
```

教学版可以先理解为：

```text
给子 Agent 创建一个临时 git worktree，让它在那里工作。
```

流程：

```text
创建 worktree。
子 Agent 在 worktree cwd 中运行。
完成后检查 diff。
主 Agent 或用户决定是否合并。
清理 worktree。
```

这是多 Agent 写代码时非常重要的工程能力。

## 30.13 子 Agent 的权限模式

子 Agent 可以有自己的 permissionMode。

例如：

```text
Explore: default / read-only
Plan: plan
Implementation: acceptEdits
Verification: default
```

Claude Code 的 AgentDefinition 支持 `permissionMode`。

AgentTool 创建 workerPermissionContext 时，会基于父 context 复制，但设置子 Agent 自己的 mode。

教学版：

```python
childPermissionContext =:
    ...parentPermissionContext
    "mode": agentDefinition.permissionMode  or  "default"
```

注意：不要让子 Agent 默认继承父 Agent 的所有宽权限。

如果父 Agent 因某次任务临时获得了广泛权限，子 Agent 不一定应该自动继承。

## 30.14 子 Agent 和 MCP

某些子 Agent 需要特定外部工具。

例如：

- Slack agent 需要 Slack MCP。
- GitHub agent 需要 GitHub MCP。
- Database agent 需要数据库 MCP。

Claude Code 的 AgentDefinition 支持：

```text
mcpServers
requiredMcpServers
```

AgentTool 在启动前会检查 required MCP servers 是否有 tools 可用。如果还在 pending，会等待一段时间。如果缺失，会报错并提示用 `/mcp` 配置。

这说明：

```text
子 Agent 的能力依赖需要在启动前验证。
```

否则子 Agent 运行到一半才发现没有 Slack 工具，会浪费时间和上下文。

## 30.15 子 Agent 不能无限递归

如果子 Agent 可以无限启动子 Agent，很容易失控：

```text
主 Agent -> 子 Agent A -> 子 Agent B -> 子 Agent C ...
```

会带来：

- 成本失控。
- 上下文混乱。
- 权限边界模糊。
- 任务难以取消。

所以要限制：

- 某些 Agent 禁止使用 AgentTool。
- 最大深度。
- 最大并发数。
- 最大总 token。
- 最大运行时间。

Explore agent 就 disallow AgentTool，这是很好的例子。

教学版可以加：

```python
from typing import Any

def example(context: dict[str, Any]) -> dict[str, Any]:
    return {"ok": True, "context": context}
```

## 30.16 子 Agent 的成本控制

子 Agent 很强，但也很容易贵。

成本来自：

- 多个模型调用。
- 多个工具调用。
- 大量文件读取。
- 并发探索。
- 后台任务。
- 重试。

所以要控制：

- maxTurns。
- maxTokens。
- maxRuntimeMs。
- maxToolCalls。
- maxCostUsd。
- 子 Agent 数量。

Claude Code 的 AgentDefinition 有 `maxTurns`。其他地方也有预算、token 统计、进度 tracker。

教学版可以先实现：

```python
from typing import Any

def example(context: dict[str, Any]) -> dict[str, Any]:
    return {"ok": True, "context": context}
```

每轮检查：

```python
if (turns >= budget.maxTurns) stop("max turns reached")
if (time.time() - start > budget.maxRuntimeMs) stop("timeout")
```

没有预算的子 Agent 是危险的。

## 30.17 子 Agent 的结果质量

子 Agent 最终要给主 Agent 提供可用结果。

好的子 Agent 输出：

- 结构化。
- 有证据。
- 有文件路径。
- 区分事实和推测。
- 有不确定性。
- 不过度展开。

坏输出：

```text
我看了一下，好像认证系统挺复杂。
```

好输出：

```text
认证入口：
- src/auth/login.py: handleLogin()
- src/auth/session.py: createSession()

权限校验：
- src/middleware/requireAuth.py
- src/lib/permissions.py

结论：
权限校验分散在 middleware 和 route handler，建议先抽象 requirePermission().

不确定：
没有找到 refresh token 轮换逻辑，可能在外部服务。
```

主 Agent 才能基于它继续工作。

## 30.18 教学版 AgentDefinition

先实现一个简化 AgentDefinition：

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

定义 Explore：

```python
"exploreAgent": AgentDefinition =:
    "agentType": "Explore"
    "whenToUse": "Search and read code without modifying files"
    "model": "haiku"
    "permissionMode": "default"
    "disallowedTools": ["Edit", "Write", "Agent"]
    "maxTurns": 6
    "systemPrompt": "
    You are a read-only code exploration agent.
    Search and read files. Do not modify files.
    Return concise findings with file paths.
    ".strip()
```

这个定义已经足够训练我们后续实现 AgentTool。

## 30.19 教学版 AgentTool 输入

AgentTool 输入：

```python
from typing import Any

def example(context: dict[str, Any]) -> dict[str, Any]:
    return {"ok": True, "context": context}
```

含义：

```text
description
  给 UI 和任务列表看的短描述。

prompt
  给子 Agent 的真实任务。

subagent_type
  选择 AgentDefinition。

model
  临时覆盖模型。

run_in_background
  是否后台运行。

isolation
  是否隔离执行。
```

注意：`description` 不是 prompt。  
description 是给人看的短标签。  
prompt 是给子 Agent 的任务内容。

## 30.20 教学版 runAgent

子 Agent 运行本质上还是调用我们前面实现的 Agent 循环。

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

这就是子 Agent 的核心。

## 30.21 主 Agent 如何决定是否使用子 Agent

主 Agent 使用子 Agent 的标准应该清楚。

适合使用子 Agent：

- 需要搜索大量文件。
- 有多个独立方向。
- 需要验证实现。
- 需要只读探索，避免污染主上下文。
- 任务可以并行。

不适合使用子 Agent：

- 简单单文件修改。
- 用户要求快速直接回答。
- 子任务边界不清。
- 需要强一致实时交互。
- 权限风险太高。

系统提示可以教主 Agent：

```text
Use subagents for independent research or verification tasks.
Do not use subagents for trivial edits.
Give each subagent a clear, bounded task and ask for concise findings.
```

## 30.22 子 Agent 的失败处理

子 Agent 可能失败：

- agent type 不存在。
- required MCP server 不可用。
- 权限被拒绝。
- 运行超时。
- 超出 maxTurns。
- 工具失败。
- 输出为空。

主 Agent 不应该崩溃。它应该得到结构化错误：

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

然后主 Agent 可以：

- 换一个 Agent。
- 缩小任务。
- 直接自己处理。
- 告诉用户缺少配置。

## 30.23 子 Agent 和用户信任

多 Agent 很酷，但也更难让用户理解。

用户需要知道：

- 启动了哪个子 Agent。
- 它在做什么。
- 是否后台运行。
- 是否会修改文件。
- 是否有隔离。
- 最终结果是什么。

UI 上可以显示：

```text
Explore agent: Searching auth code...
Verification agent: Running tests...
Implementation agent: Editing in worktree agent-a1b2c3...
```

不要让子 Agent 在用户看不见的地方悄悄做大量动作。

## 30.24 常见错误

错误一：所有子 Agent 都有全部工具。

正确做法：按角色限制 tools 和 disallowedTools。

错误二：子 Agent 继承完整主上下文。

正确做法：默认只传任务和必要上下文。

错误三：子 Agent 输出全量日志。

正确做法：返回结构化总结和证据。

错误四：允许无限递归启动子 Agent。

正确做法：限制深度、数量、预算。

错误五：后台子 Agent 没有输出文件或任务 ID。

正确做法：后台任务必须可查询、可取消、可恢复。

错误六：写代码子 Agent 不做隔离。

正确做法：使用 worktree 或类似隔离。

错误七：required MCP 不检查。

正确做法：启动前验证依赖能力。

错误八：子 Agent 权限默认过宽。

正确做法：子 Agent 有自己的 permissionMode。

## 30.25 本章练习

练习一：定义 `AgentDefinition`。

至少包含：

- agentType
- whenToUse
- systemPrompt
- tools
- disallowedTools
- model
- permissionMode
- maxTurns

练习二：定义 Explore agent。

要求：

- 只读。
- 禁止 Edit、Write、Agent。
- 输出文件路径和结论。

练习三：实现 `buildAgentTools()`。

根据 tools 和 disallowedTools 过滤工具。

练习四：实现 `runAgent()`。

复用前面实现的 agentLoop。

练习五：实现 AgentTool。

输入包含 description、prompt、subagent_type。

练习六：给子 Agent 添加 maxTurns。

练习七：实现后台子 Agent。

返回 taskId 和 outputFile。

练习八：设计 worktree isolation 流程。

练习九：写一个验证 Agent。

它只能读文件和运行测试，不允许编辑。

练习十：思考题：

```text
什么时候子 Agent 应该继承主 Agent 上下文？什么时候不应该？
```

## 30.26 本章小结

本章我们进入了多 Agent 世界。

你应该理解：

1. 子 Agent 是由主 Agent 启动的独立 Agent 循环。
2. 子 Agent 适合处理边界清楚的探索、验证、实现子任务。
3. AgentDefinition 是子 Agent 的说明书。
4. 子 Agent 应该有专门角色和受限工具。
5. 子 Agent 默认不应继承完整主上下文。
6. 子 Agent 输出应该是总结和证据，不是全量日志。
7. 后台子 Agent 需要 taskId、进度、输出和取消机制。
8. 会修改代码的子 Agent 最好在 worktree 中隔离。
9. 子 Agent 需要预算，防止递归和成本失控。
10. 用户应该能看到子 Agent 正在做什么。

下一章我们会深入 AgentTool 的实现细节：如何选择 agent type、如何构造子 Agent 工具池、如何运行 runAgent、如何收集进度和最终结果。
