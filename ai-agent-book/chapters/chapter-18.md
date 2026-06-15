# 第 3 卷：工具系统工程化

## 第 18 章：Tool Schema，模型和程序之间的合同

### 18.1 本章目标

前面我们已经实现了多个工具。每个工具都有 Zod schema，也有给 API 的 JSON Schema。现在需要停下来深入理解：Tool Schema 到底是什么？为什么它对 Agent 如此关键？

本章目标：

1. 理解 schema 是模型和程序之间的合同。
2. 区分 Zod schema 和 JSON Schema。
3. 理解模型为什么经常生成错误参数。
4. 学会设计模型友好的工具参数。
5. 学会把参数错误变成模型可修正反馈。
6. 理解 MCP 工具 schema 如何接入。
7. 对照 Claude Code 的 `tool.py`、`toolToAPISchema` 和 deferred tool schema。

### 18.2 Schema 不是类型注解那么简单

在普通 Python 项目里，类型主要帮助开发者：

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

但运行时并没有这个类型。如果外部传进来：

```json
{ "path": 123 }
```

Python 不会自动阻止。

Agent 工具参数来自模型，是运行时数据。必须用运行时 schema 校验。

所以我们需要：

```python
inputSchema =:
    "path": str
```

它不仅是给 Python 看，也是给运行时看。

### 18.3 Schema 是三方合同

Tool Schema 同时服务三方：

第一，模型。

模型通过 schema 知道应该生成哪些字段、字段是什么类型、哪些必填。

第二，程序。

程序通过 schema 校验模型输出，得到安全的 typed input。

第三，用户和文档。

好的字段描述能让工具行为更清楚，错误时也更好解释。

所以 schema 不是随便写的。它会直接影响模型调用工具的成功率。

### 18.4 坏 schema 示例

不好的工具 schema：

```python

    "data": str
```

问题：

1. `data` 太泛。
2. 模型不知道应该放路径、内容还是命令。
3. 错误时用户也看不懂。

更好的 schema：

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

字段名和描述都明确，模型更容易正确调用。

### 18.5 字段设计原则

原则一：字段名具体。

用：

```text
filePath
oldString
newString
timeoutMs
maxResults
```

不要用：

```text
data
value
input
options
```

原则二：避免多个字段表达同一件事。

不要同时有：

```python
path
file
filePath
filename
```

模型会混淆。

原则三：布尔值要有清晰语义。

`overwrite` 比 `flag` 好。

原则四：数字要有单位。

`timeoutMs` 比 `timeout` 好。

原则五：复杂结构先别过度嵌套。

模型更容易生成扁平对象。

### 18.6 可选字段的风险

可选字段很方便，但也会让模型不确定。

例如：

```python

    "path": str
    "offset": int | None
    "limit": int | None
```

模型可能不知道何时使用 offset/limit。工具描述要补充：

```text
Only provide offset and limit when reading a portion of a large file.
```

Claude Code 的 FileReadTool prompt 中也会告诉模型，文件太大时使用 offset/limit。

### 18.7 枚举比自由字符串更安全

如果字段只有几个值，用 enum。

例如权限模式：

```python
from typing import Any

def example(context: dict[str, Any]) -> dict[str, Any]:
    return {"ok": True, "context": context}
```

比：

```python
str
```

更安全。

模型如果输出 `"planning"`，schema 会拒绝，并告诉模型合法值是什么。

### 18.8 严格对象

Zod 默认对象可能允许额外字段，取决于写法。工具参数建议严格：

```python
z.strictObject(:
    "path": str
```

这样模型输出多余字段时会被发现。

为什么要在意多余字段？

因为多余字段可能说明模型误解了工具。例如：

```json
{
  "path": "README.md",
  "mode": "delete"
}
```

`read_file` 没有 mode。严格 schema 会让模型知道这个字段无效。

Claude Code 的很多工具使用 strict object 或等价策略，避免参数漂移。

### 18.9 参数错误如何反馈给模型

当 schema 校验失败，不要只返回：

```text
Invalid input
```

这对模型帮助不大。

更好：

```text
InputValidationError for read_file:
- path: expected string, received number

Please call read_file with:
{
  "path": "relative/path/to/file"
}
```

模型看到具体错误后，下一轮更可能修正。

mini-agent 可以改进 `runToolUse()`：

```python
from typing import Any

def example(context: dict[str, Any]) -> dict[str, Any]:
    return {"ok": True, "context": context}
```

Claude Code 的 `toolExecution.py` 会格式化 Zod 错误，并在 deferred tool schema 未发送时给特殊提示。

### 18.10 JSON Schema 和 Zod Schema 的区别

Zod schema 用于程序运行时校验：

```python
inputSchema.safeParse(input)
```

JSON Schema 用于告诉模型工具参数结构：

```json
{
  "type": "object",
  "properties": {
    "path": { "type": "string" }
  },
  "required": ["path"]
}
```

两者最好保持一致。

如果 Zod 要求 `path`，但 JSON Schema 写成 `filePath`，模型会生成 `filePath`，客户端校验失败。

这类不一致非常常见，也很难调试。

真实项目可以：

1. 从 Zod 自动生成 JSON Schema。
2. 或写测试确保两者字段一致。
3. 或让工具只定义一种 schema，再统一转换。

Claude Code 允许工具有 `inputSchema`，MCP 工具有时直接提供 `inputJSONSchema`。

### 18.11 MCP Schema

MCP server 返回工具时，会提供类似 JSON Schema 的 input schema。

客户端要把 MCP tool 包装成内部 Tool：

```text
MCP tool schema
  ↓
内部 Tool inputJSONSchema
  ↓
模型可见 tool schema
```

但 MCP 工具来自外部 server，不能完全信任。

需要处理：

1. 工具名称规范化。
2. 描述长度限制。
3. schema 兼容性。
4. metadata 清洗。
5. 只读/破坏性注解。

Claude Code 的 `services/mcp/client.py` 中 `fetchToolsForClient` 就做了这类转换。它会给 MCP 工具加 `mcp__server__tool` 前缀，避免命名冲突。

### 18.12 Tool Schema 和 ToolSearch

工具很多时，把所有 schema 都发给模型会浪费上下文。

解决办法之一是 ToolSearch：

```text
先只给模型工具名称和简短 hint
模型调用 ToolSearch 查找相关工具
再加载具体工具 schema
```

Claude Code 中有 deferred tools 和 `ToolSearchTool`。如果某个工具 schema 没有发送，模型可能会生成错误参数。源码里甚至有专门提示：

```text
This tool's schema was not sent...
Load the tool first...
```

这说明 schema 不只是校验问题，也是上下文预算问题。

mini-agent 工具少，暂时全部发送。等工具很多时，再引入 ToolSearch。

### 18.13 Schema 测试

你应该测试每个工具 schema。

例如：

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

还可以测试 JSON Schema 字段：

```python
expect(readFileTool.inputJsonSchema.required).toContain("path")
```

### 18.14 本章练习

练习一：检查所有工具字段名。

把模糊字段改成明确字段。

练习二：为所有工具写 `inputJsonSchema`。

确保和 Zod schema 一致。

练习三：改进 Zod 错误反馈。

让模型看到具体字段错误。

练习四：写 schema 测试。

每个工具至少一个成功案例、一个失败案例。

练习五：思考 ToolSearch。

如果工具数量增加到 100 个，你会如何避免全部 schema 进入上下文？

### 18.15 本章小结

本章我们深入理解了 Tool Schema。

你应该记住：

1. Schema 是模型和程序之间的合同。
2. Zod 用于运行时校验，JSON Schema 用于告诉模型。
3. 两者必须保持一致。
4. 字段名和描述会影响模型调用成功率。
5. 参数错误应该反馈给模型，让它能修正。
6. MCP 工具 schema 需要包装和清洗。
7. 工具多时，schema 本身也会成为上下文负担。

下一章我们会深入工具执行生命周期，把 validateInput、checkPermission、hooks、call、format、telemetry 串成完整流水线。
