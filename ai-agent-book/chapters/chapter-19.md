# 第 3 卷：工具系统工程化

## 第 19 章：工具执行生命周期

### 19.1 本章目标

第 18 章讲了 Tool Schema。Schema 解决的是“模型应该怎样传参数”和“程序怎样校验参数”。但一个工具从模型发起到结果回到模型，中间不只有 schema 校验。

完整工具执行生命周期通常是：

```text
tool_use
  ↓
查找工具
  ↓
schema 校验
  ↓
业务校验 validateInput
  ↓
权限判断 checkPermission
  ↓
pre-tool hooks
  ↓
tool.call
  ↓
post-tool hooks
  ↓
结果格式化
  ↓
tool_result
  ↓
日志、用量、观测
```

mini-agent 目前只有其中几步：

```text
查找工具 -> schema 校验 -> 权限判断 -> call -> formatResult
```

本章会把这条链路补完整，让读者理解工程级 Agent 为什么不会让工具“直接执行”。

读完本章，你应该能理解：

1. 为什么 schema 校验之外还需要 `validateInput()`。
2. 为什么权限判断和工具执行要分离。
3. hooks 在 Agent 工程中解决什么问题。
4. 为什么工具结果格式化要区分“模型可见”和“用户可见”。
5. telemetry/logging 应该记录哪些信息。
6. Claude Code 的 `toolExecution.py` 做了哪些产品级增强。

### 19.2 为什么工具不能直接 call

最简单的执行方式是：

```python
output = await tool.call(input, context)
```

这看起来没问题，但缺少很多保护。

例如模型调用：

```json
{
  "name": "edit_file",
  "input": {
    "path": "../outside.txt",
    "oldString": "a",
    "newString": "b"
  }
}
```

如果直接 call，工具内部也许会拒绝，也许不会。更好的结构是让执行层统一负责：

```text
先校验参数类型
再校验业务约束
再判断权限
再执行
```

这样所有工具都有一致安全边界。

### 19.3 schema 校验和业务校验的区别

Schema 校验回答的是：

```text
字段类型对不对？
必填字段有没有？
枚举值是否合法？
```

业务校验回答的是：

```text
这个值在当前上下文里能不能用？
```

例如 `read_file`：

Schema 可以校验：

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

但 schema 不知道：

- 文件是否存在。
- path 是否在 cwd 内。
- path 是否是目录。
- 文件是否太大。
- 文件是否是二进制。

这些是业务校验。

所以工具接口可以增加：

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

定义：

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

默认：

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

### 19.4 validateInput 应该做什么

不同工具的 validateInput 不同。

`read_file` 可以检查：

- path 是否在 cwd 内。
- 文件是否存在。
- 是否是目录。
- 文件大小是否超过上限。

`shell` 可以检查：

- 命令是否为空。
- timeout 是否合理。
- 是否包含明显阻塞模式，例如 `sleep 999999`。

`edit_file` 可以检查：

- oldString 是否为空。
- oldString 和 newString 是否相同。
- 文件是否存在。

注意：validateInput 不应该问用户。问用户属于权限阶段。

### 19.5 在 runToolUse 中加入 validateInput

mini-agent 的 `runToolUse()` 可以升级：

```python
validation = await tool.validateInput.(parsed.data, context)

def if(self, validation && !validation.ok):
    return toolResult(
    toolUse.id
    "ValidationError for {tool.name}: {validation.message}"
    True
    )
```

这和 schema 错误不同：

```text
InputValidationError: 参数形状错了
ValidationError: 参数形状对，但当前不能执行
```

模型看到两者都能修正，但原因更清楚。

Claude Code 的 Tool 接口中也有 `validateInput()`，`toolExecution.py` 会在权限判断前调用它。顺序很重要：如果输入本身无效，就没必要问权限。

### 19.6 权限为什么在 validateInput 后

假设模型调用：

```json
{
  "name": "shell",
  "input": {
    "command": ""
  }
}
```

如果先问用户：

```text
Allow shell command?
```

用户会很困惑，因为命令为空。

所以顺序应该是：

```text
schema 校验
  ↓
业务校验
  ↓
权限判断
```

只有一个工具调用是“形状正确、业务上可执行”的，才值得问用户是否允许。

### 19.7 权限判断为什么在 call 前

这看起来显然，但很多简化示例会漏掉。

错误做法：

```python
output = await tool.call(input, context)
await checkPermission(...)
```

这等于先执行后询问，没有意义。

正确做法：

```python
permission = await checkToolPermission(...)
if not allowed -> return tool_result error
output = await tool.call(...)
```

权限判断必须在任何副作用之前。

### 19.8 hooks 是什么

Hooks 是工具生命周期中的扩展点。

例如：

```text
PreToolUse hook:
  在工具执行前运行

PostToolUse hook:
  在工具执行后运行
```

它们可以做：

- 记录日志。
- 阻止某些操作。
- 修改输入。
- 执行额外检查。
- 通知外部系统。
- 自动格式化。
- 安全扫描。

比如一个 pre hook：

```text
如果 shell command 包含 "rm -rf"，直接拒绝。
```

一个 post hook：

```text
如果 edit_file 修改了 .py 文件，自动运行 formatter。
```

mini-agent 暂时可以实现简单 hook 机制。

### 19.9 Hook 类型设计

定义：

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

Engine options：

```python
from typing import Any, Protocol

hooks::
    preToolUse: PreToolUseHook[]
    postToolUse: PostToolUseHook[]
```

执行：

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

post hook：

```python
import asyncio
from dataclasses import dataclass
from typing import Any

@dataclass
class HookResult:
    status: str
    message: str = ""

async def run_hook(name: str, payload: dict[str, Any], timeout: float = 5.0) -> HookResult:
    try:
        await asyncio.wait_for(asyncio.sleep(0), timeout=timeout)
        return HookResult("ok", f"hook 已执行: {name}")
    except asyncio.TimeoutError:
        return HookResult("timeout", f"hook 超时: {name}")
```

### 19.10 hooks 和权限的区别

权限系统主要回答：

```text
这个工具调用是否允许？
```

Hooks 更通用：

```text
工具执行前后还要做什么？
```

有些 hook 会参与安全决策，但 hook 不只用于安全。

例如：

- 统计工具耗时。
- 自动运行 formatter。
- 同步 IDE 状态。
- 发送通知。

Claude Code 的 hooks 系统比 mini-agent 更复杂，支持用户配置 hook、matcher、pre/post、失败处理、进度消息等。它让用户和企业可以扩展 Agent 行为，而不用改核心代码。

### 19.11 call 阶段

`tool.call()` 是真正执行动作的地方。

要求：

1. 不再做基础 schema 校验。
2. 可以假设 input 类型正确。
3. 仍然要处理运行时错误。
4. 尽量返回结构化 output。

例如 `shellTool.call()` 返回：

```python

    command
    exitCode
    stdout
    stderr
    timedOut
    truncated
```

不要只返回字符串。结构化 output 更利于：

- UI 渲染。
- 日志。
- 测试。
- 格式化给模型。

### 19.12 formatResult 阶段

`formatResult()` 把结构化 output 转成模型可见内容。

这一步非常重要。

例如 shell output：

```text
Command: pytest
Exit code: 1

STDOUT:
...

STDERR:
...
```

模型需要清楚知道：

- 运行了什么命令。
- 是否成功。
- 输出是什么。
- 是否截断。

同一个 output 也可以有不同展示：

```text
模型可见：完整但预算化的文本
用户可见：漂亮的 UI 组件
日志可见：结构化 JSON
```

Claude Code 的 Tool 接口区分：

- `mapToolResultToToolResultBlockParam`
- `renderToolResultMessage`
- `extractSearchText`

这就是因为不同消费者需要不同格式。

### 19.13 telemetry 阶段

工具执行应该记录：

- toolName。
- toolUseId。
- 开始时间。
- 结束时间。
- 是否成功。
- 错误类型。
- 是否被拒绝。
- 是否超时。
- 输出是否截断。

mini-agent 可以先简单记录：

```python
startedAt = time.time()

try:
    // run tool
    } finally:
        durationMs = time.time() - startedAt
        print("[tool] {tool.name} {durationMs}ms")
```

真实产品不应该记录敏感内容，例如完整命令参数、文件内容、密钥。Claude Code 的遥测代码里有很多类型名强调“已验证不是代码或文件路径”，这说明遥测安全是严肃问题。

### 19.14 完整 runToolUse 流水线

整合后：

```python
async def runToolUse(args):
    { toolUse, registry, context, hooks } = args
    startedAt = time.time()

    tool = registry.get(toolUse.name)
    if (not tool) return toolError(...)

    parsed = tool.inputSchema.safeParse(toolUse.input)
    if (not parsed.success) return inputValidationError(...)

    validation = await tool.validateInput.(parsed.data, context)
    if (validation  and  not validation.ok) return validationError(...)

    permission = await checkToolPermission(...)
    if denied -> return permissionError(...)
    if ask -> prompt user or return permission required

    for preHook -> maybe deny

    try:
        output = await tool.call(parsed.data, context)

        for postHook -> await hook

        return toolResult(toolUse.id, tool.formatResult(output))
        } catch (error):
            return toolResult(toolUse.id, formatToolError(error), True)
            } finally:
                log duration
```

这就是工具执行生命周期的核心。

### 19.15 错误分类

错误应该分类。

至少区分：

```text
UnknownToolError
InputValidationError
ValidationError
PermissionDenied
HookDenied
ToolExecutionError
```

为什么？

因为模型看到不同错误，应采取不同行动。

例如：

`InputValidationError`：修正参数。

`PermissionDenied`：不要重复尝试，询问用户或解释。

`ToolExecutionError`：可能换路径、换命令、读取更多上下文。

Claude Code 的 `classifyToolError()` 会把错误转成 telemetry-safe 字符串，用于日志和诊断。

### 19.16 本章练习

练习一：给 Tool 增加 `validateInput()`。

至少让 `edit_file` 检查 oldString 和 newString 是否相同。

练习二：实现 pre hook。

如果 shell 命令包含 `rm -rf`，deny。

练习三：实现 post hook。

工具执行后打印耗时。

练习四：改造 runToolUse。

按本章生命周期重新组织。

练习五：测试错误分类。

分别构造 unknown tool、schema error、permission denied、tool throw。

### 19.17 本章小结

本章我们把工具执行从“直接 call”升级成完整生命周期。

你应该理解：

1. schema 校验和业务校验不同。
2. 权限必须在副作用前。
3. hooks 是生命周期扩展点。
4. call 应返回结构化 output。
5. formatResult 面向模型，不等于 UI 渲染。
6. telemetry 要记录状态，但避免敏感内容。

下一章我们会深入并发工具调度：哪些工具可以并行，哪些必须串行，以及 Claude Code 的 StreamingToolExecutor 为什么存在。

