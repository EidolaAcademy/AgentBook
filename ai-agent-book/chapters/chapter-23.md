# 第 3 卷：工具系统工程化

## 第 23 章：MCP 工具包装，把外部能力接进 Agent

### 23.1 本章目标

到目前为止，mini-agent 的工具都是本地内置工具。真实 Agent 不可能把所有能力都写在自己项目里。它需要连接外部服务，例如：

- 数据库。
- 浏览器。
- GitHub。
- Slack。
- 文档系统。
- 内部 API。
- 自定义企业工具。

MCP 的目标就是让外部工具以统一协议暴露给 Agent。

本章目标：

1. 理解 MCP 工具为什么重要。
2. 学会把 MCP tool 包装成内部 Tool。
3. 处理 MCP tool 命名空间。
4. 处理 MCP schema。
5. 处理 MCP 权限注解。
6. 理解外部工具的信任边界。
7. 对照 Claude Code 的 `services/mcp/client.py`。

### 23.2 MCP 在 Agent 架构中的位置

没有 MCP 时：

```text
Agent -> 内置工具
```

有 MCP 后：

```text
Agent -> 内部 Tool 接口 -> MCP wrapper -> MCP server -> 外部系统
```

关键思想是：无论工具来自哪里，进入 Agent 后都要变成同一种 Tool。

这样后续能力都能复用：

- schema 校验。
- 权限判断。
- hooks。
- 并发调度。
- tool_result 格式。
- 日志。
- 上下文预算。

### 23.3 MCP tool 的基本形状

一个 MCP server 通常会返回工具列表，概念上类似：

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

然后客户端可以调用：

```python
from dataclasses import dataclass

@dataclass
class SearchResult:
    tool_name: str
    score: int
    description: str

def search_tools(query: str, tools: list[dict[str, str]], limit: int = 5) -> list[SearchResult]:
    terms = [term.lower() for term in query.split() if term.strip()]
    results: list[SearchResult] = []
    for tool in tools:
        haystack = f"{tool.get('name', '')} {tool.get('description', '')}".lower()
        score = sum(1 for term in terms if term in haystack)
        if score:
            results.append(SearchResult(tool.get("name", ""), score, tool.get("description", "")))
    return sorted(results, key=lambda item: item.score, reverse=True)[:limit]
```

我们要把它包装成：

```python
from dataclasses import dataclass
from typing import Any

@dataclass
class McpTool:
    server_name: str
    name: str
    description: str
    input_schema: dict[str, Any]

def wrap_mcp_tool(tool: McpTool) -> dict[str, Any]:
    return {
        "name": f"mcp__{tool.server_name}__{tool.name}",
        "description": tool.description,
        "input_schema": tool.input_schema,
    }
```

### 23.4 命名空间问题

如果 MCP server 暴露工具：

```text
search
read
write
```

可能和内置工具冲突。

例如 mini-agent 已经有：

```text
read_file
write_file
```

另一个 MCP server 也可能有：

```text
read_file
```

所以需要命名空间。

常见格式：

```text
mcp__serverName__toolName
```

例如：

```text
mcp__github__search_issues
mcp__slack__send_message
mcp__docs__read_page
```

Claude Code 也使用类似前缀。`services/mcp/client.py` 会构造 fully qualified tool name，避免冲突。

### 23.5 包装函数

概念代码：

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

这里有一个简化点：`jsonSchemaToZod` 并不好写。生产项目可能直接使用 JSON Schema 校验库，或者让 Tool 支持 `inputJSONSchema` 而不是强制 Zod。

Claude Code 的 Tool 接口就支持 `inputJSONSchema`，因为 MCP 工具天然给 JSON Schema。

### 23.6 JSON Schema 校验

如果你不想把 JSON Schema 转 Zod，可以让工具执行层支持两种 schema：

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

执行时：

```python
from dataclasses import dataclass
from typing import Any

@dataclass
class ToolUse:
    id: str
    name: str
    input: dict[str, Any]

async def run_agent_step(model: Any, messages: list[dict[str, Any]], tools: dict[str, Any]) -> dict[str, Any]:
    response = await model.complete(messages, tools)
    messages.append(response)
    return response
```

JSON Schema 校验可以用 Ajv。

教学项目可以先不实现 Ajv，但要理解 MCP 工具为什么需要这条路径。

### 23.7 MCP 注解不能完全信任

MCP tool 可能声明：

```python
from pathlib import Path

def resolve_inside_workspace(workspace: Path, user_path: str) -> Path:
    root = workspace.resolve()
    target = (root / user_path).resolve()
    if target != root and root not in target.parents:
        raise ValueError(f"路径越界: {user_path}")
    return target

def read_text_file(workspace: Path, user_path: str, limit: int = 200) -> str:
    file_path = resolve_inside_workspace(workspace, user_path)
    lines = file_path.read_text(encoding="utf-8").splitlines()
    return "\n".join(lines[:limit])
```

但外部 server 是否可信？

如果是你自己写的 MCP server，可以更信任。

如果是第三方 MCP server，就不能完全信任注解。

策略：

1. 默认新 MCP server 需要用户信任。
2. readOnlyHint 可以降低权限级别，但不应完全绕过策略。
3. destructiveHint 应该触发强提示。
4. openWorldHint 表示工具可能访问外部世界，需要额外注意。

Claude Code 的 MCP 系统有 server approval、channel allowlist、auth、policy filtering 等机制，都是为了解决信任边界。

### 23.8 MCP 描述需要清洗

MCP server 提供的 description 会进入模型上下文。

如果 description 非常长，浪费上下文。

如果 description 包含奇怪格式或注入式内容，也可能影响模型。

所以要：

```python
from dataclasses import dataclass
from typing import Any

@dataclass
class McpTool:
    server_name: str
    name: str
    description: str
    input_schema: dict[str, Any]

def wrap_mcp_tool(tool: McpTool) -> dict[str, Any]:
    return {
        "name": f"mcp__{tool.server_name}__{tool.name}",
        "description": tool.description,
        "input_schema": tool.input_schema,
    }
```

Claude Code 会限制 MCP description 长度，并清洗 unicode/whitespace。

### 23.9 MCP 工具结果格式

MCP tool result 可能有：

- text content。
- structured content。
- metadata。
- error。

不要一律 `JSON.stringify` 后塞给模型。更好策略：

```text
如果有 text content -> 直接拼接文本
如果有 structuredContent -> 格式化 JSON
如果有 _meta -> 只传需要给模型看的部分
```

某些 metadata 只给客户端，不应给模型。

Claude Code 的 MCPTool 会处理 structuredContent、_meta，并传给 SDK 消费者。

### 23.10 MCP 权限规则

MCP 工具名带 server 前缀后，可以做权限规则：

```text
allow mcp__github__search_issues
deny mcp__slack__send_message
deny mcp__dangerousServer
```

也可以按 server 级别控制：

```text
deny all tools from server "production-db"
```

Claude Code 的工具过滤会处理 MCP server-prefix rules：如果用户 deny 某个 MCP server，模型甚至看不到这些工具。

这很重要。不要只在执行时拒绝，也要在发送工具列表给模型前过滤。否则模型会不断尝试调用不可用工具。

### 23.11 MCP 连接生命周期

MCP server 可能：

- 未连接。
- 连接中。
- 已连接。
- 认证失败。
- 工具列表加载失败。
- 中途断开。

Agent 要能处理。

教学版可以简单：

```text
启动时加载 MCP tools，失败就跳过。
```

生产版需要：

- 重连。
- auth flow。
- pending server 状态。
- UI 显示。
- 工具动态刷新。

Claude Code 的 MCPConnectionManager、client、auth、approval 等模块就是为这些情况服务。

### 23.12 本章练习

练习一：设计 `McpTool` 类型。

包含：

- name
- description
- inputSchema
- annotations

练习二：实现 MCP 命名空间。

把 server/tool 转成：

```text
mcp__server__tool
```

练习三：实现 `wrapMcpTool()`。

包装成内部 Tool。

练习四：设计 MCP 权限。

readOnlyHint allow，destructiveHint ask。

练习五：思考信任边界。

第三方 MCP server 声明 readOnlyHint，你是否应该自动信任？为什么？

### 23.13 本章小结

本章我们把外部 MCP 工具接入内部 Tool 系统。

你应该理解：

1. MCP 工具进入 Agent 后也应该变成统一 Tool。
2. 命名空间避免工具冲突。
3. JSON Schema 和 Zod 需要兼容策略。
4. MCP 注解有用，但不能盲目信任。
5. 工具描述和结果都需要清洗。
6. 权限可以按工具或 server 控制。

下一章我们会讲 ToolSearch：当工具数量变多时，不应该把所有 schema 都塞进上下文。

