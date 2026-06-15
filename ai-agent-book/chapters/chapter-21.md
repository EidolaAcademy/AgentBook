# 第 3 卷：工具系统工程化

## 第 21 章：工具结果预算，大输出不能直接塞进上下文

### 21.1 本章目标

工具让 Agent 能观察世界，但工具结果也会迅速撑爆上下文。

例如：

```bash
cat package-lock.json
find .
pytest -- --verbose
rg "a"
```

这些命令可能返回几万行。即使模型能处理很长上下文，把巨大输出直接塞进去也会带来：

1. 成本增加。
2. 延迟增加。
3. 上下文被噪音淹没。
4. 模型注意力分散。
5. prompt too long。

本章目标：

1. 理解工具结果预算的必要性。
2. 区分“工具本次输出预算”和“历史累计预算”。
3. 实现大输出预览。
4. 实现工具结果落盘。
5. 在历史中用占位摘要替换大结果。
6. 对照 Claude Code 的 `toolResultStorage` 和 `applyToolResultBudget`。

### 21.2 两种预算

工具结果预算至少有两层。

第一层：单次工具结果预算。

例如 shell 单次最多返回 20,000 字符。

第二层：历史累计预算。

即使每次只返回 20,000 字符，执行 20 次工具也会有 400,000 字符。长会话仍然会爆。

所以需要：

```text
工具执行时限制输出
  +
发送模型前压缩历史工具结果
```

mini-agent 之前已经有单次输出截断。现在要加入更高级策略。

### 21.3 只截断有什么问题

截断简单，但有缺点。

如果工具返回：

```text
前 20000 字符
[truncated]
```

后面的内容丢了。

有时后面的内容很重要。例如测试日志的关键错误可能在末尾。

更好的方式是：

1. 完整输出保存到磁盘。
2. 给模型一个预览。
3. 告诉模型完整文件路径。
4. 如果需要，模型可以用 `read_file` 读取更具体部分。

Claude Code 的大输出处理就采用类似思想：过大的工具结果会持久化，模型看到预览和路径，而不是全部内容。

### 21.4 ToolResultRecord

定义一个记录：

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

工具结果落盘后，模型看到：

```text
Tool result was too large and has been saved to:
.mini-agent/tool-results/tool-result-abc.txt

Original size: 120000 chars
Preview:
...
```

### 21.5 生成预览

预览可以保留开头和结尾。

为什么要保留结尾？

很多命令关键错误在最后。例如：

```text
...
FAIL test/login.test.py
Expected ...
```

实现：

```python
from typing import Any

def example(context: dict[str, Any]) -> dict[str, Any]:
    return {"ok": True, "context": context}
```

这比只保留开头更有用。

### 21.6 保存工具结果

创建 `src/mini_agent/tools/resultStorage.py`：

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

### 21.7 在 runToolUse 中应用

`formatResult()` 后得到模型可见字符串：

```python
content = tool.formatResult(output)
```

然后：

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

这让模型既不会被巨大输出淹没，又能知道完整结果在哪里。

### 21.8 历史累计预算

即使单次结果已经限制，历史还是会增长。

我们可以在发送模型前替换旧工具结果：

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

这个版本只保留最近 N 个工具结果的完整内容。

真实实现更细：

- 根据字符数/token 数预算。
- 不替换当前轮必需结果。
- 对已经落盘的结果保留路径。
- 保证 prompt cache 稳定。

Claude Code 的 `applyToolResultBudget` 就是更成熟的版本。

### 21.9 预览内容也要安全

工具结果可能包含密钥。

例如：

```bash
env
cat .env
```

输出里可能有 API key。

预算系统不是安全扫描。它只控制大小，不负责识别秘密。但真实产品通常还需要 secret scanner 或输出过滤。

Claude Code 中有 team memory secret guard、secret scanner 等相关模块。Agent 工具输出和记忆写入都要防止泄露敏感信息。

mini-agent 后续安全章节再处理 secret scanning。

### 21.10 哪些工具不应该落盘

并不是所有工具结果都适合落盘。

例如 `read_file` 读取一个巨大文件，如果落盘结果里又写“完整结果保存到某文件”，模型可能再读这个文件，形成奇怪循环。

Claude Code 的 Tool 接口有 `maxResultSizeChars`，某些工具可以设置 Infinity 或特殊策略，避免不合适的持久化。

mini-agent 可以先统一处理，但要知道这是简化。

### 21.11 模型提示很重要

当结果被存储和预览时，一定要告诉模型如何继续：

```text
Use read_file with offset/limit or a more specific command if more detail is needed.
```

这不是废话。模型会根据提示决定下一步。

如果只说：

```text
truncated
```

模型可能不知道该怎么办。

工具结果要像一个好同事：不仅告诉你信息不完整，还告诉你怎样获取更多。

### 21.12 和 Claude Code 对照

Claude Code 相关模块：

```text
src/mini_agent/utils/toolResultStorage.py
src/query.py
src/services/tools/toolExecution.py
```

关键思想：

1. 工具定义自己的最大结果大小。
2. 大结果可以被持久化。
3. 模型收到预览和路径。
4. query 前应用总预算。
5. 替换记录可持久化，保证 resume 后行为一致。

这说明工具结果预算不是一个小优化，而是长会话 Agent 的基础设施。

### 21.13 本章练习

练习一：实现 `createPreview()`。

要求保留开头和结尾。

练习二：实现 `storeLargeToolResult()`。

保存到 `.mini-agent/tool-results/`。

练习三：在 runToolUse 中接入大结果保存。

超过 20,000 字符时保存完整结果，返回预览。

练习四：实现 `replaceOldToolResults()`。

只保留最近 5 个工具结果完整内容。

练习五：测试 shell 大输出。

运行一个会产生大量输出的命令，确认模型只收到预览。

### 21.14 本章小结

本章我们让工具结果有了预算系统。

你应该理解：

1. 工具输出是上下文膨胀的主要来源。
2. 单次工具结果和历史累计都要预算。
3. 大输出最好保存到磁盘，并给模型预览和路径。
4. 预览应保留开头和结尾。
5. 模型提示要告诉它如何获取更多细节。
6. Claude Code 的 tool result storage 是长会话能力的重要基础。

下一章我们会深入 Bash 安全。Shell 工具虽然强大，但如果只靠简单字符串判断，很容易被 shell 语法绕过。

