# 第 26 章：权限规则，allow、deny、ask 如何匹配工具和输入

上一章我们建立了 Agent 安全总图：模型提出工具调用，程序做权限决策，沙箱限制执行范围，日志记录发生了什么。

这一章开始深入权限规则。

权限规则是 Agent 安全系统里最容易被低估的一层。很多人会以为规则就是：

```text
允许 Read。
禁止 Bash。
```

但真实系统远比这细。

因为用户真正想表达的通常不是：

```text
永远允许 Bash。
```

而是：

```text
允许 Bash 运行 pytest。
禁止 Bash 运行 rm。
允许 Read 读取当前项目。
禁止 Read 读取 .env。
允许 Edit 修改 src/**。
修改 .github/workflows/** 时要问我。
```

也就是说，规则不只匹配工具名，还要匹配工具输入。

这一章我们会从 Claude Code 的规则格式出发，设计一个新手也能实现的权限规则系统。

## 26.1 权限规则解决什么问题

权限规则的作用是把用户或系统的安全偏好变成可执行的判断。

自然语言：

```text
你可以运行测试，但不要安装依赖。
```

需要变成规则：

```text
allow Bash(pytest)
deny Bash(pip install)
deny Bash(ppip install)
deny Bash(yarn install)
```

自然语言：

```text
你可以改 src 目录，但不要动配置文件。
```

需要变成规则：

```text
allow Edit(src/**)
deny Edit(.env)
deny Edit(.github/**)
deny Edit(pyproject.toml)
```

权限规则把模糊意图转成机器可判断的条件。

如果没有规则系统，Agent 每次都只能问用户，或者粗暴地全允许、全拒绝。

这两种都不好。

## 26.2 三种行为：allow、deny、ask

权限规则最核心的行为是三种：

```text
allow
  命中后允许。

deny
  命中后拒绝。

ask
  命中后强制询问。
```

为什么需要 `ask`？

因为有些操作不是绝对危险，也不是绝对安全。

例如：

```bash
pip install
```

它可能是合理的，也可能引入风险。

再比如编辑：

```text
.github/workflows/deploy.yml
```

这可能只是修 CI，也可能影响部署权限。

这类操作最适合 `ask`：

```text
允许模型提出，但必须让用户确认。
```

所以权限系统不要只做二元判断：

```text
允许 / 禁止
```

要做三元判断：

```text
允许 / 询问 / 拒绝
```

## 26.3 规则至少包含三部分

一条权限规则至少应该包含：

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

其中：

```text
toolName
  规则匹配哪个工具。

ruleContent
  规则匹配该工具的哪类输入。可选。

behavior
  命中后如何处理。

source
  规则来自哪里。
```

`ruleContent` 是关键。

如果没有 `ruleContent`：

```text
Bash
```

表示匹配整个 Bash 工具。

如果有 `ruleContent`：

```text
Bash(pytest)
```

表示只匹配 Bash 工具中某类命令。

Claude Code 的规则值也是这个思想：`PermissionRuleValue` 包含 `toolName` 和可选 `ruleContent`。

## 26.4 Claude Code 的规则字符串格式

Claude Code 使用一种非常直观的规则字符串：

```text
ToolName
ToolName(content)
```

例子：

```text
Bash
Bash(pytest)
Read(src/**)
Edit(src/components/**)
Agent(Explore)
mcp__slack
mcp__slack__send_message
```

这有几个优点。

第一，可读。

用户配置文件里可以直接写：

```json
{
  "permissions": {
    "allow": ["Read(src/**)", "Bash(pytest)"],
    "deny": ["Read(.env)", "Bash(rm *)"]
  }
}
```

第二，可序列化。

规则能放进 JSON、CLI 参数、项目设置、会话记录。

第三，可扩展。

不同工具可以解释自己的 `ruleContent`。

例如：

```text
Bash(pytest)
```

由 Bash 工具解释。

```text
Read(src/**)
```

由文件权限逻辑解释。

```text
Agent(Explore)
```

由 Agent 工具解释。

这就是“统一外壳，工具自定义内容”的设计。

## 26.5 解析规则字符串

我们先实现一个教学版解析器。

输入：

```text
Bash(pytest)
```

输出：

```python

    "toolName": "Bash"
    "ruleContent": "pytest"
```

最简单版本：

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

这个版本能跑，但还不够安全。

因为内容里可能有括号：

```text
Bash(python -c "print(1)")
```

如果你只找第一个 `(` 和最后一个 `)`，这个例子还能工作。但如果内容里有转义括号，或者规则格式不规范，就会出问题。

Claude Code 的解析器处理了：

- 找第一个未转义的左括号。
- 找最后一个未转义的右括号。
- 内容里的括号需要转义。
- 旧工具名要规范化成新工具名。

教学版可以先支持转义：

```python
def findFirstUnescaped(str: str, ch: str):
    for i in range(0, str.length):
        if (str[i] != ch) continue

        backslashes = 0
        j = i - 1
        while j >= 0 && str[j] === "\\":
            backslashes++
            j--

        if (backslashes % 2 == 0) return i

    return -1
```

然后解析：

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

这已经接近源码思路。

## 26.6 为什么需要工具名别名

Claude Code 的规则解析里还有 legacy tool name alias。

原因很现实：工具会改名。

比如早期某个工具叫：

```text
Task
```

后来改成：

```text
Agent
```

用户配置文件里可能已经保存了：

```text
Task
```

如果程序不兼容旧名字，用户升级后权限规则突然失效。

所以需要：

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

这是一种很朴素但重要的工程意识：

```text
权限规则一旦持久化，就成了兼容性协议。
```

不要随便改规则格式。  
不要随便改工具名。  
改了就要提供兼容映射。

## 26.7 工具级规则和输入级规则

规则可以分成两类。

第一类是工具级规则：

```text
Bash
Read
mcp__slack
```

它们没有 `ruleContent`，表示匹配整个工具或一组工具。

第二类是输入级规则：

```text
Bash(pytest)
Read(src/**)
Edit(src/index.py)
Agent(Explore)
```

它们有 `ruleContent`，表示匹配该工具的某类输入。

两者的处理方式不一样。

工具级规则可以在工具列表阶段就使用。

例如用户 deny：

```text
mcp__slack
```

那么所有 Slack MCP 工具都可以直接从可见工具列表里过滤掉。

输入级规则必须等模型真的调用工具后才能判断。

例如：

```text
Bash(pytest)
```

只有看到具体输入：

```json
{ "command": "pytest" }
```

才知道是否命中。

所以权限系统有两个时机：

```text
发送工具给模型前：过滤工具级 deny。
执行工具前：检查输入级规则。
```

这也解释了为什么第 24 章说 ToolSearch 之前要先做权限过滤。

## 26.8 工具名匹配和别名

工具可能有别名。

Claude Code 的 `toolMatchesName()` 会判断：

```text
tool.name === name || tool.aliases includes name
```

教学版也可以这样：

```python
from typing import Any

def example(context: dict[str, Any]) -> dict[str, Any]:
    return {"ok": True, "context": context}
```

别名有两个用途。

第一，兼容旧模型输出。

模型可能还会调用旧工具名。

第二，兼容旧配置。

用户规则可能还写着旧名称。

不过要小心：MCP 工具名可能和内置工具名冲突。

例如某个 MCP server 也有一个叫 `Read` 的工具。生产系统需要用完整 MCP 名称：

```text
mcp__server__Read
```

否则规则：

```text
Read
```

可能错误匹配 MCP 工具。

Claude Code 对 MCP 工具有额外处理，尤其是 server-level rule。

## 26.9 MCP server 级规则

MCP 工具命名通常是：

```text
mcp__server__tool
```

例如：

```text
mcp__slack__send_message
mcp__slack__list_channels
mcp__github__create_issue
```

如果用户想禁止整个 Slack server，不应该一条条写：

```text
deny mcp__slack__send_message
deny mcp__slack__list_channels
deny mcp__slack__...
```

应该支持：

```text
deny mcp__slack
```

或者：

```text
deny mcp__slack__*
```

Claude Code 的工具级匹配里就支持 server-level permission：规则 `mcp__server1` 可以匹配 `mcp__server1__tool1`。

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

这样：

```text
mcp__slack
```

可以匹配：

```text
mcp__slack__send_message
```

但不会匹配：

```text
mcp__github__send_message
```

## 26.10 规则匹配顺序

规则匹配顺序非常重要。

一个保守顺序是：

```text
1. deny
2. ask
3. allow
4. mode default
```

为什么 deny 在最前？

因为用户或项目显式拒绝的东西，不应该被更宽泛的 allow 覆盖。

例如：

```text
allow Read(src/**)
deny Read(src/secrets.py)
```

读 `src/secrets.py` 时应该拒绝。

如果 allow 先匹配，就会错误允许。

为什么 ask 在 allow 前？

因为 ask 通常表示“这个范围需要强确认”。

例如：

```text
allow Edit(src/**)
ask Edit(src/payments/**)
```

编辑 `src/payments/checkout.py` 时应该问用户，而不是被 `allow Edit(src/**)` 直接放行。

所以教学版执行前可以这样：

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

这个顺序要写测试。

## 26.11 规则来源和持久化位置

规则来源很重要，因为它决定：

- 能不能被用户编辑。
- 是否应该持久化。
- 冲突时谁优先。
- UI 应该怎么解释。

常见来源：

```text
policySettings
  企业或组织管理策略。

userSettings
  用户全局设置。

projectSettings
  项目共享设置。

localSettings
  当前机器本地设置。

cliArg
  命令行参数。

session
  当前会话临时规则。
```

例如用户点击“Always allow for this session”，应该写到 `session`。

用户点击“Always allow in this project”，应该写到 `projectSettings` 或 local project settings，具体看产品设计。

企业策略一般不应该被普通用户覆盖。

教学版可以先支持两个来源：

```text
project
session
```

但类型上预留：

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

这样以后扩展不会改太多。

## 26.12 once、session、permanent

当用户看到权限弹窗时，通常会有多个选择：

```text
Allow once
Always allow this session
Deny
Always deny
```

这些选择对应不同的规则生命周期。

`Allow once`：

```text
只返回本次 allow，不添加规则。
```

`Always allow this session`：

```text
添加 session allow 规则。
```

`Always deny`：

```text
添加 session 或 project deny 规则。
```

不要把 `Allow once` 错误持久化。

这是安全系统常见 bug。

教学版 UI 可以返回：

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

然后：

```python
if choice.type === "add_rule":
    permissionStore.add(choice.rule)
```

## 26.13 规则内容由工具解释

`ruleContent` 没有全局统一语义。

对 Bash 来说：

```text
Bash(pytest)
```

匹配命令。

对 Read 来说：

```text
Read(src/**)
```

匹配文件路径。

对 Agent 来说：

```text
Agent(Explore)
```

匹配子 Agent 类型。

对 MCP 来说：

```text
mcp__slack__send_message(channel:C123)
```

可能匹配参数里的 channel。

所以规则系统应该把 `ruleContent` 交给工具或工具分类器解释。

可以在 Tool 上加：

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

通用匹配：

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

这比在一个大函数里写所有工具规则更清楚。

## 26.14 文件路径规则

文件路径规则通常支持 glob。

例如：

```text
Read(src/**)
Edit(src/components/*.python)
deny Read(.env)
```

新手第一版可以使用成熟的 glob 库，不要自己写复杂 glob。

匹配流程：

```text
1. 把输入路径解析为绝对路径。
2. 把规则路径相对到某个 root。
3. 用 glob/ignore 库匹配。
4. 处理 symlink 和路径穿越。
```

教学版简化：

```python
from typing import Any

def example(context: dict[str, Any]) -> dict[str, Any]:
    return {"ok": True, "context": context}
```

生产版还要处理：

- Windows 路径转换。
- `/**` 结尾语义。
- root 来源。
- ignore 库规则。
- 不存在路径的父目录 realpath。
- deny within allow。

但第一版先明确“规则是相对 root 匹配”。

## 26.15 Bash 规则

Bash 规则更难。

最简单规则：

```text
Bash(pytest)
```

可以精确匹配命令字符串：

```python
def bashRuleMatches(command: str, ruleContent: str):
    return command.strip() == ruleContent.strip()
```

但这很快不够。

用户可能想允许：

```text
Bash(pytest *)
```

或者：

```text
Bash(git status)
Bash(git diff *)
```

你可以先支持前缀规则：

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

但请记住第 22 章的警告：这只是基础规则，不是完整 Bash 安全。

真实系统还要解析：

- 管道。
- 重定向。
- 子命令。
- 命令替换。
- 变量。
- 引号。
- 多行命令。

Bash 规则可以做“允许常见命令”，但危险命令识别要走更强的 Bash 安全分析。

## 26.16 ask 规则覆盖 allow 规则

我们看一个具体例子。

规则：

```text
allow Edit(src/**)
ask Edit(src/payments/**)
deny Edit(src/payments/secrets.py)
```

输入：

```text
src/payments/checkout.py
```

应该结果：

```text
ask
```

输入：

```text
src/payments/secrets.py
```

应该结果：

```text
deny
```

输入：

```text
src/ui/button.python
```

应该结果：

```text
allow
```

这就是规则优先级：

```text
deny > ask > allow
```

测试代码可以这样写：

```python
expect(decide("Edit", "src/payments/secrets.py")).toBe("deny")
expect(decide("Edit", "src/payments/checkout.py")).toBe("ask")
expect(decide("Edit", "src/ui/button.python")).toBe("allow")
```

如果这三个测试里任何一个失败，权限系统就不可信。

## 26.17 shadowed rules：被遮蔽的规则

规则系统成熟后，会出现 shadowed rule，也就是某条规则永远不会生效。

例如：

```text
deny Bash
allow Bash(pytest)
```

如果工具级 deny `Bash` 优先，那么 `allow Bash(pytest)` 永远不会生效。

再比如：

```text
allow Read(src/**)
allow Read(src/components/**)
```

第二条可能被第一条覆盖，虽然不一定危险，但可能冗余。

Claude Code 源码里有 shadowed rule detection 相关模块，用于识别规则冲突和遮蔽。

教学版可以先做一个简单提示：

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

这不是完整算法，但能帮助用户发现明显冲突。

## 26.18 规则建议要尽量窄

当用户确认一次操作后，系统通常会建议：

```text
Always allow similar operations?
```

这里的“similar”很危险。

如果用户允许：

```bash
pytest
```

你不应该建议：

```text
allow Bash
```

这太宽了。

应该建议：

```text
allow Bash(pytest)
```

或者稍微宽一点：

```text
allow Bash(pytest *)
```

如果用户允许编辑：

```text
src/components/Button.python
```

可以建议：

```text
allow Edit(src/components/**)
```

而不是：

```text
allow Edit
```

规则建议越宽，后续风险越高。

教学版可以让每个工具提供规则建议：

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

Edit 工具可以建议目录级规则。  
Bash 工具可以建议命令级规则。  
MCP 工具可以建议工具级或 server 级规则。

## 26.19 deny 规则也要谨慎持久化

允许规则太宽有风险，拒绝规则太宽也有问题。

如果用户拒绝了一次：

```bash
pip install
```

你不应该自动写：

```text
deny Bash
```

否则之后连测试都不能跑。

更合理：

```text
deny Bash(pip install)
```

或者只 deny once，不持久化。

用户可能只是拒绝当前上下文，不代表永远拒绝该类操作。

所以 UI 要区分：

```text
Deny once
Always deny this command
Always deny this tool
```

不要替用户做过宽推断。

## 26.20 规则和工具可见性

规则不仅影响执行，还影响工具可见性。

例如：

```text
deny mcp__slack
```

如果工具列表里仍然给模型展示：

```text
mcp__slack__send_message
```

模型可能反复尝试调用，浪费上下文和轮次。

所以工具级 deny 应该在发送工具列表前过滤：

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

注意：输入级 deny 不能在这里过滤。

例如：

```text
deny Bash(rm *)
```

不能把 Bash 工具整个隐藏，因为 Bash 仍然可以运行：

```text
pytest
```

只有工具级 deny 才能隐藏整个工具。

## 26.21 规则解释

权限系统应该能解释命中的规则。

例如：

```text
Denied by projectSettings rule: Bash(rm *)
```

或者：

```text
Allowed by session rule: Edit(src/components/**)
```

这需要 PermissionRule 保留：

- source
- behavior
- toolName
- ruleContent

格式化函数：

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

错误信息：

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

解释不是为了好看。它会影响模型下一步能否修正，也影响用户是否信任 Agent。

## 26.22 测试规则系统

规则系统必须测试。

最低测试清单：

1. `Bash` 能解析成工具级规则。
2. `Bash(pytest)` 能解析成输入级规则。
3. 内容里的转义括号能正确解析。
4. 旧工具名能规范化。
5. deny 优先于 ask。
6. ask 优先于 allow。
7. 工具级 deny 会过滤工具列表。
8. 输入级 deny 不会过滤整个工具。
9. MCP server 级规则能匹配 server 下所有工具。
10. MCP server 级规则不匹配其他 server。
11. path glob 能匹配 `src/**`。
12. path glob 不允许 `../` 逃出 root。
13. session allow 不会写入 permanent settings。
14. shadowed rule 能被检测。

权限系统的 bug 往往不是“程序崩了”，而是“悄悄允许了不该允许的事”。测试要覆盖冲突和边界。

## 26.23 教学版：完整决策流程

最后把本章内容组合成一个简化流程：

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

这不是生产代码，但它表达了一个清晰骨架：

```text
显式规则优先。
模式兜底。
读写区别处理。
需要用户参与时返回 ask。
```

## 26.24 常见错误

错误一：只支持工具级规则。

结果：无法表达“允许 pytest，但禁止 rm”。

错误二：allow 优先于 deny。

结果：宽泛 allow 覆盖敏感 deny。

错误三：输入级 deny 隐藏整个工具。

结果：禁止 `Bash(rm *)` 后，连 `Bash(pytest)` 都不能用。

错误四：规则格式不处理转义括号。

结果：Python、shell、正则类命令解析失败。

错误五：工具改名后旧规则失效。

结果：用户升级后安全配置失效。

错误六：MCP server 级规则不支持。

结果：用户无法一键禁止整个外部集成。

错误七：用户允许一次被持久化为永久允许。

结果：权限逐渐失控。

错误八：规则命中后不解释来源。

结果：用户和模型都不知道为什么允许或拒绝。

## 26.25 本章练习

练习一：实现 `parsePermissionRuleValue()`。

支持：

- `ToolName`
- `ToolName(content)`
- 转义括号

练习二：实现 `permissionRuleValueToString()`。

要求能把括号转义后保存。

练习三：实现工具名别名。

例如：

```text
Task -> Agent
KillShell -> TaskStop
```

练习四：实现 MCP server 级匹配。

要求：

```text
mcp__slack
```

匹配：

```text
mcp__slack__send_message
```

但不匹配：

```text
mcp__github__send_message
```

练习五：实现 deny、ask、allow 优先级。

练习六：实现文件路径 glob 匹配。

练习七：实现 Bash 的简单命令匹配。

练习八：实现工具级 deny 过滤工具列表。

练习九：实现 session allow，不写入永久配置。

练习十：写 shadowed rule 检测。

## 26.26 本章小结

本章我们深入了权限规则。

你应该理解：

1. 权限规则不能只匹配工具名，也要匹配工具输入。
2. `allow`、`deny`、`ask` 是三种基本行为。
3. `ToolName(content)` 是一种简单而强大的规则格式。
4. 规则解析要处理转义、旧工具名和格式兼容。
5. 工具级规则可以影响工具可见性，输入级规则只能在执行前判断。
6. deny 应该优先于 ask，ask 应该优先于 allow。
7. MCP 需要 server 级规则。
8. 规则来源决定持久化、解释和优先级。
9. 用户确认的一次性决定和持久规则必须区分。
10. 规则建议要尽量窄。

下一章我们会继续第 4 卷，深入沙箱：当权限系统允许执行命令后，如何用执行环境限制文件系统、网络和进程副作用。
