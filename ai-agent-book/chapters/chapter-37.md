# 第 37 章：自定义 Agent、插件 Agent 与分发机制

前面我们已经知道 AgentDefinition 是子 Agent 系统的核心。一个 AgentDefinition 决定了 Agent 的名字、用途、工具、模型、权限、记忆、技能和系统提示词。

但是一个真实 Agent 平台不能只靠内置 Agent。用户需要自己写 Agent，团队需要共享 Agent，插件需要分发 Agent。于是系统会出现多种 Agent 来源：

- built-in agents：产品内置。
- user agents：用户个人配置。
- project agents：项目仓库配置。
- managed/policy agents：组织策略配置。
- plugin agents：插件提供。
- flag/json agents：实验或配置注入。

这一章会讲如何加载、解析、合并、覆盖和分发 Agent。重点不是“读 Markdown 文件”这么简单，而是多来源配置背后的治理问题。

## 37.1 为什么要支持自定义 Agent

内置 Agent 无法覆盖所有团队需求。

一个前端团队可能需要：

```txt
design-system-reviewer
accessibility-checker
storybook-maintainer
```

一个后端团队可能需要：

```txt
migration-reviewer
api-contract-checker
observability-debugger
```

一个安全团队可能需要：

```txt
threat-modeler
secrets-scanner
dependency-risk-reviewer
```

这些角色都有自己的知识、工具范围和输出格式。把它们都塞进一个 general-purpose prompt 会让 Agent 变得臃肿。更好的方式是让用户定义专门 Agent。

## 37.2 Markdown Agent 文件

最自然的自定义格式是 Markdown 加 frontmatter。

```md
---
name: api-reviewer
description: Review API changes for compatibility and correctness
tools:
  - Read
  - Grep
  - Bash
model: sonnet
permissionMode: readOnly
maxTurns: 12
---

你是一个 API 审查 Agent。

重点检查：
- 向后兼容性
- 错误处理
- 鉴权边界
- 请求和响应 schema
- 测试覆盖

不要修改文件。输出具体问题、证据文件和建议。
```

frontmatter 是结构化元数据，正文是系统提示词。

解析后：

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

## 37.3 必填字段

最少需要：

```yaml
name: api-reviewer
description: Review API changes
```

`name` 用于调用：

```json
{
  "subagent_type": "api-reviewer"
}
```

`description` 或 `whenToUse` 用于告诉模型什么时候使用它。

如果缺少 name 或 description，应该报告解析错误，而不是静默生成一个坏 Agent。

教学版：

```python
from typing import Any

def example(context: dict[str, Any]) -> dict[str, Any]:
    return {"ok": True, "context": context}
```

## 37.4 frontmatter 字段

常见字段：

```yaml
tools:
disallowedTools:
skills:
initialPrompt:
mcpServers:
hooks:
model:
effort:
permissionMode:
maxTurns:
background:
memory:
isolation:
color:
```

逐个解释：

`tools`：允许工具列表。如果缺省，通常表示使用默认工具集合；如果为空，表示不给工具。

`disallowedTools`：从可用工具中排除某些工具。

`skills`：启动时预加载的 skills。

`initialPrompt`：启动 Agent 时追加的初始提示。

`mcpServers`：Agent 额外需要的 MCP server。

`hooks`：Agent 生命周期 hooks。

`model`：默认模型。

`effort`：推理努力程度。

`permissionMode`：权限模式。

`maxTurns`：最大回合数。

`background`：是否默认后台运行。

`memory`：是否启用 memory，以及 scope。

`isolation`：是否默认 worktree 隔离。

`color`：UI 展示颜色。

## 37.5 tools 的语义

tools 字段最容易出错。

可以有三种状态：

```txt
缺失：使用默认工具集合
空：不允许任何工具
['Read', 'Grep']：只允许这些工具
['*']：允许全部默认工具
```

教学版解析：

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

这里 `undefined` 和 `[]` 含义不同。新手实现时一定要分清。

## 37.6 disallowedTools

如果一个 Agent 允许大部分工具，但明确禁止某些工具，可以用 disallowedTools。

例如 Explore Agent：

```yaml
disallowedTools:
  - Edit
  - Write
  - Bash
  - Agent
```

解析后：

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

白名单和黑名单同时存在时，建议顺序是：

1. 先按 Agent tools 白名单选。
2. 再按 disallowedTools 排除。
3. 再按异步/权限模式做系统过滤。

## 37.7 model 与 inherit

Agent 可以声明模型：

```yaml
model: sonnet
```

也可以声明：

```yaml
model: inherit
```

`inherit` 表示跟随父 Agent 或 team lead 的模型。源码中会把 `inherit` 作为特殊值处理，而不是传给模型 API。

教学版：

```python
from typing import Any

def example(context: dict[str, Any]) -> dict[str, Any]:
    return {"ok": True, "context": context}
```

## 37.8 maxTurns 校验

maxTurns 必须是正整数。

```yaml
maxTurns: 12
```

错误示例：

```yaml
maxTurns: many
maxTurns: -1
```

不要让坏值默默变成无限循环。解析时应该警告或拒绝。

```python
def parsePositiveInt(value: Any):
    if (value == None) return None
    n = Number(value)
    if (not Number.isInteger(n)  or  n <= 0) return None
    return n
```

## 37.9 memory、skills、isolation 的组合

一个较完整的 Agent：

```md
---
name: migration-reviewer
description: Review database migrations
tools:
  - Read
  - Grep
  - Bash
skills:
  - migration-review
memory: project
isolation: worktree
maxTurns: 15
---

你是数据库迁移审查 Agent。
不要修改迁移文件，只运行只读检查命令。
```

这里：

- skill 提供迁移审查流程。
- memory 提供项目长期数据库约定。
- isolation 保证如果它运行命令产生文件，也不污染主工作区。

AgentDefinition 是这些能力的组合点。

## 37.10 JSON Agent 定义

除了 Markdown，也可以通过 JSON 配置 Agent。

```json
{
  "api-reviewer": {
    "description": "Review API changes",
    "tools": ["Read", "Grep"],
    "prompt": "你是一个 API 审查 Agent..."
  }
}
```

JSON 适合：

- settings 注入。
- 管理策略。
- 实验开关。
- 自动生成。

Markdown 适合人类手写。JSON 适合程序配置。

两者最后都应该变成同一个 `AgentDefinition`。

## 37.11 多来源加载

一个成熟系统会从多个来源加载 Agent：

```txt
built-in
plugin
user settings
project settings
flag settings
policy settings
```

教学版：

```python
async def loadAllAgents(cwd: str):
    builtIn = getBuiltInAgents()
    plugin = await loadPluginAgents()
    user = await loadUserAgents()
    project = await loadProjectAgents(cwd)
    policy = await loadPolicyAgents()

    return mergeAgents([
    builtIn
    plugin
    user
    project
    policy
```

## 37.12 覆盖优先级

如果多个来源定义同名 Agent，谁生效？

源码中会按 group 顺序写入 Map，后写覆盖前写。这样可以实现优先级。

```python
from typing import Any

def example(context: dict[str, Any]) -> dict[str, Any]:
    return {"ok": True, "context": context}
```

越靠后的来源优先级越高。

为什么 managed/policy 最高？因为组织策略通常应该覆盖用户配置。

## 37.13 解析失败不要拖垮系统

如果一个用户写错了自定义 Agent，不应该导致整个 Agent 系统不可用。

更好的做法：

- 记录 failedFiles。
- 日志中记录错误。
- UI 或命令中展示错误。
- 继续加载其他 Agent。
- 最差也返回 built-in agents。

教学版：

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

## 37.14 插件 Agent 的命名空间

插件 Agent 不能直接叫：

```txt
reviewer
```

否则不同插件可能冲突。更好的方式是带插件命名空间：

```txt
security-tools:reviewer
product-tools:reviewer
```

如果插件里还有目录 namespace：

```txt
security-tools:web:reviewer
```

源码中插件 Agent 会把 pluginName、namespace、baseAgentName 拼起来：

```python
from typing import Any

def example(context: dict[str, Any]) -> dict[str, Any]:
    return {"ok": True, "context": context}
```

这让插件可以安全分发 Agent。

## 37.15 插件变量替换

插件 Agent 的 prompt 可能需要引用插件目录下的文件：

```txt
请参考 {CLAUDE_PLUGIN_ROOT}/references/security.md
```

加载时替换：

```python
systemPrompt = substitutePluginVariables(markdownContent,:
    "path": pluginPath
    "source": sourceName
```

插件也可能有用户配置：

```txt
API endpoint: {user_config.endpoint}
```

但敏感配置不能直接展开到 prompt。敏感值应该替换成占位符或通过受控工具访问。

## 37.16 插件 Agent 的安全限制

插件来自第三方，信任边界不同于用户自己写的 `.claude/agents/`。

源码中插件 Agent 明确忽略这些 per-agent 字段：

- `permissionMode`
- `hooks`
- `mcpServers`

为什么？

因为这些字段会提升能力：

- permissionMode 可能改变权限行为。
- hooks 可能执行命令。
- mcpServers 可能接入外部能力。

如果一个插件里藏了某个 Agent 文件，安装后悄悄声明 hooks 或 MCP，就会越过用户安装时的预期。

正确做法是：插件可以在 manifest 级别声明需要的 hooks 或 MCP，让用户安装时明确授权。单个插件 Agent 文件不应该偷偷加。

教学版也建议这样：

```python
if source === 'plugin':
    ignore(frontmatter.permissionMode)
    ignore(frontmatter.hooks)
    ignore(frontmatter.mcpServers)
```

## 37.17 插件 Agent 能声明什么

插件 Agent 适合声明：

- name / description
- tools
- disallowedTools
- skills
- model
- effort
- maxTurns
- memory
- isolation
- background
- color
- systemPrompt

这些主要影响 Agent 行为和成本，不应该绕过安装授权边界。

如果插件确实需要 MCP 或 hooks，应该在插件 manifest 中声明，而不是 Agent frontmatter 中声明。

## 37.18 去重与 symlink

加载 Markdown 文件时，可能通过不同路径读到同一个文件：

- symlink。
- 插件路径重复配置。
- 用户配置和项目配置指向同一目录。

源码中会通过文件 identity 检测 duplicate path。教学版可以简单用 realpath：

```python
from typing import Any

def example(context: dict[str, Any]) -> dict[str, Any]:
    return {"ok": True, "context": context}
```

这避免重复注册同一个 Agent。

## 37.19 缓存和清理

Agent definitions 通常会 memoize。否则每次 prompt 都扫描文件系统会很慢。

```python
from typing import Any

def example(context: dict[str, Any]) -> dict[str, Any]:
    return {"ok": True, "context": context}
```

当用户修改 Agent 文件或插件更新时，需要清缓存：

```python
def clearAgentDefinitionsCache():
    getAgentDefinitionsWithOverrides.cache.clear.()
    clearPluginAgentCache()
```

缓存是性能优化，但必须有清理入口。

## 37.20 自定义 Agent 分发方式

团队可以用几种方式分发 Agent。

第一，项目仓库：

```txt
.claude/agents/api-reviewer.md
```

优点：和代码一起版本控制，团队共享。

第二，用户配置：

```txt
~/.claude/agents/personal-reviewer.md
```

优点：个人定制，不影响团队。

第三，插件：

```txt
my-plugin/
  agents/
    reviewer.md
  skills/
    review-checklist.md
  plugin.json
```

优点：可打包、可安装、可复用。

第四，组织策略：

```json
{
  "agents": {
    "compliance-reviewer": { ... }
  }
}
```

优点：集中治理。

## 37.21 Agent 文件质量清单

一个好的自定义 Agent 文件应该满足：

1. name 简短稳定。
2. description 明确什么时候用。
3. system prompt 说明角色边界。
4. tools 最小化。
5. disallowedTools 明确禁止高风险工具。
6. maxTurns 有上限。
7. 输出格式清楚。
8. 不把一次性任务写进 Agent prompt。
9. 如果需要 memory，选择正确 scope。
10. 如果需要修改代码，考虑 isolation: worktree。

坏例子：

```md
---
name: helper
description: helps with stuff
tools: ["*"]
---

Do anything necessary.
```

好例子：

```md
---
name: test-failure-analyst
description: Analyze test failures and identify likely root causes without editing files
tools:
  - Read
  - Grep
  - Bash
maxTurns: 12
---

你是测试失败分析 Agent。
先读取失败输出，再定位相关测试和源码。
不要修改文件。
输出：失败命令、失败用例、可能根因、证据文件、建议下一步。
```

## 37.22 新手实现路线

按这个顺序做：

1. 定义 AgentDefinition。
2. 支持 built-in agents。
3. 支持 Markdown agent 文件。
4. 解析 name、description、tools、model、maxTurns。
5. 加载 project agents。
6. 加载 user agents。
7. 实现同名覆盖。
8. 返回 failedFiles。
9. 支持 skills、memory、isolation。
10. 支持 plugin agents 命名空间。
11. 加插件安全限制。
12. 加缓存和清缓存。

不要一开始就支持所有 frontmatter 字段。先让核心闭环稳定。

## 37.23 本章小结

这一章我们讲了 Agent 的自定义和分发。

Agent 平台的可扩展性来自 AgentDefinition，但多来源加载会带来治理问题。用户 Agent、项目 Agent、插件 Agent、组织策略 Agent 的信任边界不同，不能用同一套权限规则粗暴处理。

最重要的原则：

1. Markdown Agent 用 frontmatter 表达结构化元数据。
2. 多来源 Agent 需要明确覆盖优先级。
3. 解析失败不能拖垮整个系统。
4. 插件 Agent 必须命名空间化。
5. 插件 Agent 不能悄悄声明 hooks、MCP、permissionMode。
6. tools 的 undefined 和 [] 含义不同。
7. Agent 文件应最小工具、明确边界、限制 maxTurns。
8. 分发方式决定信任边界。

下一章会讲评估、回放、调试与生产可观测性。一个 Agent 系统不只要会运行，还要能解释为什么这样运行，能复现失败，能评估质量，能在生产环境里被治理。
