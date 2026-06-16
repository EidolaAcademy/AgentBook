# 第 2 卷：最小 Agent 实现

## 第 9 章：权限系统，给 Agent 装上刹车

### 9.1 本章目标

到目前为止，mini-agent 有了四类工具：

```text
read_file
glob_files
grep_files
shell
```

前三个是只读工具。`shell` 则非常特殊：它既可以只读，也可以写入；既可以安全，也可以危险。上一章我们用一个临时策略处理它：明显只读就运行，不明显只读就拒绝。

这不够。

一个真正可用的 coding Agent 必须能在用户允许时运行测试、安装依赖、生成文件、执行脚本。它也必须能在危险操作前停下来，让用户决定。

这就是权限系统的作用。

本章目标：

1. 理解为什么权限系统是 Agent 的核心，不是附加功能。
2. 设计 `PermissionDecision`。
3. 给 Tool 接口加入 `checkPermission()`。
4. 实现三种权限模式：default、plan、bypass。
5. 实现 CLI 中的用户确认。
6. 把 Shell 工具接入权限系统。
7. 对照 Claude Code 的 `useCanUseTool` 和权限上下文。

### 9.2 权限系统解决什么问题

权限系统解决的是“模型想做”和“系统允许做”之间的边界。

模型可能想做：

```bash
pytest
```

这通常可以允许，但最好让用户知道。

模型可能想做：

```bash
rm -rf dist
```

这可能是合理清理，也可能误删。

模型可能想做：

```bash
git reset --hard
```

这非常危险，可能丢失用户未提交修改。

模型可能想做：

```bash
curl https://example.com/install.sh | sh
```

这涉及网络和执行远程代码，必须高度谨慎。

如果没有权限系统，你只有两个极端：

1. 什么都不让做，Agent 很弱。
2. 什么都自动做，Agent 很危险。

权限系统提供中间道路：

```text
明显安全 -> 自动允许
明显危险 -> 自动拒绝
不确定 -> 问用户
```

这就是 Agent 的刹车。

### 9.3 权限决策类型

先定义权限决策。

创建 `src/permissions/types.py`：

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

三种行为：

`allow`：允许执行。

`deny`：拒绝执行，不询问用户。

`ask`：需要用户确认。

为什么要有 `reason`？

因为权限系统必须可解释。用户看到“拒绝”或“询问”时，需要知道为什么。模型收到拒绝结果时，也需要知道下一步怎么做。

例如：

```python
from dataclasses import dataclass
from typing import Any, Literal

Decision = Literal["allow", "deny", "ask"]

@dataclass
class PermissionDecision:
    decision: Decision
    reason: str = ""
    rule: str | None = None

def check_tool_permission(tool_name: str, tool_input: dict[str, Any]) -> PermissionDecision:
    command = str(tool_input.get("command", ""))
    if tool_name == "Bash" and command.startswith("rm -rf"):
        return PermissionDecision("deny", "危险删除命令", "Bash(rm -rf*)")
    if tool_name in {"Read", "Grep"}:
        return PermissionDecision("allow")
    return PermissionDecision("ask", f"需要用户确认: {tool_name}")
```

这比简单返回 `false` 好得多。

Claude Code 的权限结果更复杂，会记录 decision reason、rule source、classifier、hook、permission prompt 等信息。我们先从最小可解释决策开始。

### 9.4 权限模式

再定义权限模式：

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

三种模式：

`default`：只读工具自动允许，写入或不确定操作询问用户。

`plan`：计划模式。只允许观察，不允许修改和执行不确定命令。

`bypass`：绕过权限。所有操作允许。只应该在用户明确选择、受信环境中使用。

为什么需要 plan？

很多时候用户只是让 Agent 分析、规划、阅读代码，而不是立刻修改。计划模式可以防止模型在没有批准方案前直接动手。

为什么需要 bypass？

在某些自动化环境里，用户可能明确希望 Agent 不停询问，直接执行。但这是高风险模式，必须显式开启。

Claude Code 中权限模式更多，例如 default、acceptEdits、plan、bypassPermissions、auto 等。我们的三种模式足够覆盖教学项目。

### 9.5 PermissionContext

权限判断需要上下文。

创建：

```python
from dataclasses import dataclass
from typing import Any, Literal

Decision = Literal["allow", "deny", "ask"]

@dataclass
class PermissionDecision:
    decision: Decision
    reason: str = ""
    rule: str | None = None

def check_tool_permission(tool_name: str, tool_input: dict[str, Any]) -> PermissionDecision:
    command = str(tool_input.get("command", ""))
    if tool_name == "Bash" and command.startswith("rm -rf"):
        return PermissionDecision("deny", "危险删除命令", "Bash(rm -rf*)")
    if tool_name in {"Read", "Grep"}:
        return PermissionDecision("allow")
    return PermissionDecision("ask", f"需要用户确认: {tool_name}")
```

后面可以扩展：

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

Claude Code 的 `ToolPermissionContext` 非常丰富，包括：

- 当前模式。
- 额外工作目录。
- always allow rules。
- always deny rules。
- always ask rules。
- 是否可用 bypass。
- 是否 auto mode 可用。
- 后台 Agent 是否应避免权限弹窗。
- plan mode 前的模式。

这些都是从真实产品需求长出来的。我们先从 mode 开始。

### 9.6 给 Tool 接口加入 checkPermission

上一章的 Tool 接口没有权限方法。现在升级：

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

然后更新 `buildTool()` 默认值：

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

注意：默认策略不能无脑 allow。只读允许，非只读询问。

这和前面讲的“安全默认值保守”一致。

### 9.7 通用权限判断

可以写一个通用函数：

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

这里有两层：

第一层是全局模式。

第二层是工具自己的权限逻辑。

例如 `read_file` 在 default 模式下自动允许，因为它只读；`shell` 要根据命令判断。

Claude Code 也是类似分层，只是层数更多：

- permission mode。
- allow/deny/ask rules。
- tool-specific checkPermissions。
- hooks。
- classifier。
- interactive prompt。
- coordinator/swarm worker 特殊处理。

### 9.8 更新 runToolUse

现在 `runToolUse()` 在 call 前要检查权限：

```python
from dataclasses import dataclass
from typing import Any, Literal

Decision = Literal["allow", "deny", "ask"]

@dataclass
class PermissionDecision:
    decision: Decision
    reason: str = ""
    rule: str | None = None

def check_tool_permission(tool_name: str, tool_input: dict[str, Any]) -> PermissionDecision:
    command = str(tool_input.get("command", ""))
    if tool_name == "Bash" and command.startswith("rm -rf"):
        return PermissionDecision("deny", "危险删除命令", "Bash(rm -rf*)")
    if tool_name in {"Read", "Grep"}:
        return PermissionDecision("allow")
    return PermissionDecision("ask", f"需要用户确认: {tool_name}")
```

这只是第一版。它还不会真的问用户，只会告诉模型需要权限。

下一节我们会让 CLI 提供确认能力。

### 9.9 为什么 ask 不能在工具内部直接 question

你可能想在 `shellTool.call()` 里直接写：

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

不建议。

工具本身应该只描述“我需要权限”，不要知道 UI 怎么问用户。原因：

1. 交互式 CLI 可以问用户。
2. 非交互模式不能问，只能拒绝。
3. 未来 TUI 可能用弹窗。
4. SDK 模式可能把权限请求发给宿主应用。
5. 子 Agent 或后台任务可能不能显示 UI。

所以权限询问应该由运行环境处理，而不是工具内部处理。

Claude Code 的 `useCanUseTool.tsx` 正是这个思想。工具和通用权限逻辑先得出 allow/deny/ask，交互式场景再把 ask 放入确认队列，UI 渲染权限弹窗。

### 9.10 PermissionPrompter

在 mini-agent 中，我们可以定义一个简单接口：

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

CLI 实现：

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

然后 AgentEngine 持有：

```python
from typing import Any, Protocol

permissionPrompter: PermissionPrompter
```

非交互模式可以不传，遇到 ask 就拒绝。

### 9.11 runToolUse 支持 ask

更新 `runToolUse()` 参数：

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

处理 ask：

```python
from dataclasses import dataclass
from typing import Any, Literal

Decision = Literal["allow", "deny", "ask"]

@dataclass
class PermissionDecision:
    decision: Decision
    reason: str = ""
    rule: str | None = None

def check_tool_permission(tool_name: str, tool_input: dict[str, Any]) -> PermissionDecision:
    command = str(tool_input.get("command", ""))
    if tool_name == "Bash" and command.startswith("rm -rf"):
        return PermissionDecision("deny", "危险删除命令", "Bash(rm -rf*)")
    if tool_name in {"Read", "Grep"}:
        return PermissionDecision("allow")
    return PermissionDecision("ask", f"需要用户确认: {tool_name}")
```

如果用户允许，就继续执行工具。

这里还有一个产品问题：用户是否只允许这一次，还是以后都允许？Claude Code 支持更丰富的规则来源和持久化。mini-agent 先只做“一次性允许”。

### 9.12 shellTool 接入权限

上一章 shellTool 在 `call()` 里直接拒绝非只读命令。现在改成 `checkPermission()`：

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

现在 `pytest` 不会自动拒绝，而是 ask。CLI 用户输入 `y` 后就会执行。

### 9.13 plan 模式

如何让用户进入 plan 模式？先用 CLI 命令：

```text
/mode plan
/mode default
/mode bypass
```

AgentEngine 保存：

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

构造 ToolContext 时：

```python
from dataclasses import dataclass
from typing import Any, Literal

Decision = Literal["allow", "deny", "ask"]

@dataclass
class PermissionDecision:
    decision: Decision
    reason: str = ""
    rule: str | None = None

def check_tool_permission(tool_name: str, tool_input: dict[str, Any]) -> PermissionDecision:
    command = str(tool_input.get("command", ""))
    if tool_name == "Bash" and command.startswith("rm -rf"):
        return PermissionDecision("deny", "危险删除命令", "Bash(rm -rf*)")
    if tool_name in {"Read", "Grep"}:
        return PermissionDecision("allow")
    return PermissionDecision("ask", f"需要用户确认: {tool_name}")
```

CLI 里处理：

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

现在 plan 模式下，非只读工具会被拒绝。

### 9.14 bypass 模式的警告

`bypass` 很危险。不要悄悄开启。

CLI 可以要求用户明确输入：

```text
/mode bypass
```

并打印：

```text
Warning: bypass mode allows all tools without asking.
```

更严格一点，可以要求二次确认。教学项目先打印警告即可。

Claude Code 的 bypass permissions 也非常敏感。它会检查是否可用、是否被策略限制，也会有用户确认和配置迁移。

### 9.15 权限结果如何反馈给模型

当用户拒绝命令：

```bash
pytest
```

工具结果应该类似：

```text
Permission denied by user: Allow shell command: pytest
```

模型看到后应该停止尝试执行同一命令，转而解释：

```text
我无法运行测试，因为你拒绝了权限。你可以手动运行 pytest，或允许我执行该命令。
```

这就是为什么权限拒绝也要作为 tool_result 回到模型，而不是只在 UI 打印。

Claude Code 中权限拒绝会变成模型可见的工具结果，例如 rejected tool use message。这样模型可以调整计划。

### 9.16 常见坑

坑一：权限判断散落在工具 call 内部。

这样 UI、非交互模式、子 Agent 都不好统一处理。

坑二：ask 时工具自己读 stdin。

这会把工具和 CLI 绑定死，未来无法迁移到 TUI 或 SDK。

坑三：用户拒绝后不告诉模型。

模型会以为工具丢失或系统错误，可能重复尝试。

坑四：bypass 默认开启。

高风险。必须显式开启。

坑五：plan 模式仍允许写操作。

计划模式的价值就是先观察和规划，不应修改。

### 9.17 和 Claude Code 权限系统对照

Claude Code 的权限系统关键文件包括：

```text
src/Tool.ts
src/hooks/useCanUseTool.tsx
src/mini_agent/utils/permissions/
src/hooks/toolPermission/
src/components/permissions/
```

它比 mini-agent 多很多能力：

1. 多种权限模式。
2. allow/deny/ask 规则。
3. 规则来源区分：session、user settings、project settings、policy settings。
4. Bash classifier 自动判断。
5. hooks 可以参与权限。
6. 交互式 UI 队列。
7. coordinator 和 swarm worker 特殊处理。
8. 后台 Agent 避免权限弹窗。
9. 权限决定日志。
10. 企业策略限制。

但核心思想和我们一样：

```text
工具请求 -> 校验参数 -> 权限判断 -> allow/deny/ask -> 执行或返回拒绝结果
```

学会 mini-agent 的权限系统后，再看 Claude Code 的实现，就不会觉得它是魔法，只是把更多真实场景加进来了。

### 9.18 本章练习

练习一：定义 `PermissionDecision`。

支持：

- allow
- deny
- ask

练习二：给 Tool 加 `checkPermission()`。

默认策略：

- 只读工具 allow。
- 非只读工具 ask。

练习三：实现三种模式。

要求：

- default：按工具判断。
- plan：非只读 deny。
- bypass：全部 allow。

练习四：实现 CLI 确认。

当 shell 请求 `pytest` 时，询问用户：

```text
Allow shell command: pytest [y/N]
```

练习五：测试权限结果。

测试：

```text
/mode plan
运行测试

/mode default
运行测试

/mode bypass
运行测试
```

观察三种模式的不同结果。

### 9.19 本章小结

本章我们给 Agent 装上了第一版刹车。

你现在应该理解：

1. 权限系统不是附加功能，而是 Agent 安全运行的核心。
2. 权限结果应该是 allow、deny、ask，而不是简单 boolean。
3. 工具不应该自己处理 UI 询问。
4. plan/default/bypass 是三种基础权限模式。
5. 权限拒绝也要反馈给模型。

下一章我们会实现写文件和编辑文件工具。到那时，权限系统会真正发挥作用，因为写入工具默认必须询问用户。

---
