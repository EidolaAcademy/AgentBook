# 第 2 卷：最小 Agent 实现

## 第 10 章：写文件与编辑文件，让 Agent 真正改变项目

### 10.1 本章目标

到目前为止，mini-agent 已经能：

1. 读取文件。
2. 搜索项目。
3. 执行部分 shell 命令。
4. 在危险操作前询问用户。

但它还不能修改项目。一个 coding Agent 如果不能写文件，就永远停留在“分析助手”阶段。它可以告诉你应该怎么改，却不能替你改。

本章我们要加入两个工具：

```text
write_file
edit_file
```

`write_file` 用于创建新文件或完整覆盖文件。

`edit_file` 用于把文件中的一段旧文本替换成新文本。

读完本章，你会理解：

1. 为什么写文件比读文件危险得多。
2. 为什么要区分 write 和 edit。
3. 为什么 edit 应该要求 old_string 精确匹配。
4. 如何在写入前进行权限确认。
5. 如何在工具结果中告诉模型写入是否成功。
6. Claude Code 的 FileWriteTool 和 FileEditTool 解决了哪些更复杂的问题。

### 10.2 为什么不只做一个 write_file

最简单的修改方式是让模型直接生成完整文件内容，然后调用：

```json
{
  "name": "write_file",
  "input": {
    "path": "src/index.py",
    "content": "完整文件内容..."
  }
}
```

这当然可行，但有几个问题。

第一，大文件成本高。一个 1000 行文件，只改一行，却要让模型重写全部内容，很浪费上下文。

第二，容易误删。模型如果漏掉某段代码，完整覆盖会直接把它删掉。

第三，不利于审查。用户很难看出具体改了哪里。

所以需要 `edit_file`：

```json
{
  "name": "edit_file",
  "input": {
    "path": "src/index.py",
    "oldString": "port = 3000",
    "newString": "port = 4000"
  }
}
```

这种方式只替换局部文本，更安全，也更接近“补丁”。

Claude Code 也区分 `FileWriteTool` 和 `FileEditTool`。写新文件、编辑已有文件、Notebook 编辑都有不同工具，因为它们的风险和 UI 展示方式不同。

### 10.3 写入工具的安全原则

写入工具必须比读取工具更严格。

原则一：路径必须在 cwd 内。

不能让模型写到项目外。

原则二：默认需要权限。

写文件会改变用户项目，必须询问。

原则三：编辑必须精确。

如果 `oldString` 匹配不到，不能猜着改。

原则四：避免多处误改。

如果 `oldString` 出现多次，要么拒绝，要么要求模型提供更具体上下文。

原则五：结果要清楚。

模型需要知道文件是否创建、覆盖、替换成功，以及替换了几处。

原则六：未来要能展示 diff。

本章先不实现 diff UI，但工具设计要为 diff 留空间。

### 10.4 write_file 输入设计

`write_file` schema：

```python
inputSchema =:
    "path": str
    "content": str
    "overwrite": bool | None
```

字段含义：

`path`：目标路径。

`content`：完整文件内容。

`overwrite`：是否允许覆盖已有文件。

为什么需要 `overwrite`？

因为创建新文件和覆盖已有文件风险不同。默认情况下，如果文件已存在但模型没明确说 overwrite，就应该拒绝。

### 10.5 edit_file 输入设计

`edit_file` schema：

```python
inputSchema =:
    "path": str
    "oldString": str
    "newString": str
```

字段含义：

`oldString`：文件中必须存在的旧文本。

`newString`：替换后的新文本。

为什么不使用行号？

因为模型看到的行号可能过期。用户或工具可能已经修改文件，行号会变化。文本匹配通常更稳。

但文本匹配也有要求：

1. `oldString` 不能太短。
2. `oldString` 必须唯一。
3. 换行和空格要精确。

Claude Code 的 FileEditTool 对这些问题处理更严格。它会要求 old_string 精确匹配，并在失败时给模型反馈。

### 10.6 复用路径安全函数

上一章 `read_file` 中有：

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

现在应该把它抽到公共文件：

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

读、写、编辑、搜索都应该复用同一个路径边界函数。不要每个工具手写一份，否则很容易出现不一致。

### 10.7 实现 write_file

创建 `src/mini_agent/tools/writeFiletool.py`：

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

注意：

1. `isReadOnly` 是 false。
2. `isConcurrencySafe` 是 false。
3. `checkPermission` 总是 ask。
4. 已存在文件默认不覆盖。
5. 自动创建父目录。

### 10.8 实现 edit_file

创建 `src/mini_agent/tools/editFiletool.py`：

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

这个工具做了三个重要保护：

1. `oldString` 必须存在。
2. `oldString` 必须唯一。
3. 不唯一时拒绝，要求更具体输入。

这能避免模型误改多处。

### 10.9 为什么 edit_file 不直接支持 replace all

有时批量替换很有用，但对新手 Agent 来说很危险。

例如把：

```text
user
```

替换成：

```text
account
```

如果全局替换，可能改到注释、字符串、测试快照、文档、变量名的一部分。模型可能以为只改一处，结果改了几十处。

所以教学版 `edit_file` 只允许唯一匹配。

未来可以增加：

```python
from typing import Any

def example(context: dict[str, Any]) -> dict[str, Any]:
    return {"ok": True, "context": context}
```

但必须配合 diff 和用户确认。

### 10.10 注册写入工具

更新注册表：

```python
registry.register(writeFileTool)
registry.register(editFileTool)
```

现在工具池是：

```text
read_file
glob_files
grep_files
shell
write_file
edit_file
```

这已经能完成一个小型修改任务：

```text
用户：创建 hello.txt
  ↓
模型：write_file("hello.txt", "hello")
  ↓
权限：询问用户
  ↓
工具：写文件
  ↓
模型：报告完成
```

### 10.11 FakeModel 模拟写文件

让 FakeModel 支持：

```text
创建 hello 文件
```

返回：

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

运行时应该出现权限确认：

```text
Allow writing file: hello.txt [y/N]
```

输入 `y` 后，工具执行并返回：

```text
File created.
Path: ...
Bytes: 6
```

### 10.12 工具结果如何帮助模型继续

写入成功后，模型可能会说：

```text
文件已创建。
```

写入失败时，模型要知道原因。

例如文件已存在：

```text
Error: File already exists: hello.txt. Set overwrite: true to replace it.
```

模型看到后可以：

1. 问用户是否覆盖。
2. 换文件名。
3. 如果用户明确要求覆盖，再调用 `write_file` with `overwrite: true`。

编辑失败时：

```text
Error: oldString appears 3 times in file. Use a more specific oldString.
```

模型看到后应该读取更多上下文，构造更具体的 oldString。

这就是工具错误作为反馈的价值。

### 10.13 和 Claude Code FileEditTool 对照

Claude Code 的 FileEditTool 比我们的版本复杂很多。它要处理：

1. 精确 old_string 匹配。
2. 文件不存在。
3. 多处匹配。
4. diff 展示。
5. 用户批准。
6. 文件历史。
7. IDE 通知。
8. sed 命令模拟编辑。
9. 编码和换行。
10. Notebook 编辑。
11. 写入后的 UI 消息。

其中最重要的是 diff 和权限。

真实 coding Agent 修改文件前，用户通常需要看到“将要改什么”。我们的教学版只询问：

```text
Allow editing file: src/index.py
```

这还不够。后面产品化章节会加入 diff 展示。

Claude Code 的 UI 会渲染文件编辑 diff，让用户可以批准或拒绝。这是 coding Agent 用户体验中非常关键的一环。

### 10.14 写入工具的常见坑

坑一：默认覆盖已有文件。

危险。必须要求 `overwrite: true`。

坑二：oldString 不唯一仍然替换。

危险。可能误改多处。

坑三：不做路径限制。

模型可能写到项目外。

坑四：不询问用户。

写入是状态修改，默认必须 ask。

坑五：不保留错误反馈。

模型需要知道为什么写入失败。

### 10.15 本章练习

练习一：实现 `write_file`。

要求：

- cwd 内写入。
- 默认不覆盖。
- 自动创建父目录。
- 写入前 ask。

练习二：实现 `edit_file`。

要求：

- oldString 必须存在。
- oldString 必须唯一。
- 替换后写回文件。
- 编辑前 ask。

练习三：测试 plan 模式。

在 `/mode plan` 下尝试创建文件，应该被拒绝。

练习四：测试 default 模式。

在 `/mode default` 下创建文件，应该询问用户。

练习五：测试 bypass 模式。

在 `/mode bypass` 下创建文件，应该不询问直接执行。

### 10.16 本章小结

本章 mini-agent 第一次拥有了改变项目的能力。

你应该理解：

1. 写文件和编辑文件应该分成两个工具。
2. 写入默认需要权限。
3. 覆盖已有文件必须显式声明。
4. 编辑应该使用唯一 oldString。
5. 编辑失败要给模型可行动的错误反馈。
6. 真实产品还需要 diff 展示和更严格的文件保护。

下一章我们会把这些工具串起来，实现一个完整的小任务：搜索文件、读取文件、编辑文件、运行命令验证。

---
