# 第 1 卷：入门与总览

## 第 4 章：开发环境与第一个 CLI

### 4.1 本章目标

从这一章开始，我们不再只看概念和源码地图，而是动手做一个自己的小 Agent。这个项目暂时不追求功能完整，它的目标是建立最小骨架。

读完本章，你会完成：

1. 选择技术栈。
2. 设计项目目录。
3. 创建命令行入口。
4. 定义消息类型。
5. 写一个最小聊天循环。
6. 为后续工具调用预留结构。

请注意，本章写的是“教学项目”，不是复刻 Claude Code。Claude Code 的源码包含大量产品级逻辑，如果一开始照着它的结构建项目，新手很容易迷路。正确做法是先写一个小而清楚的版本，然后逐步升级。

### 4.2 为什么选择 Python

本书示例主要使用 Python，有几个原因。

第一，虽然 Claude Code 源码来自不同技术栈，但本书教学代码统一改用 Python；阅读源码时重点看架构、协议和工程边界，动手实现时用 Python 复现同样思想。

第二，Agent 工程非常依赖结构化数据。消息、工具、权限、工具参数、工具结果、配置、状态都需要清晰类型。Python 结合 dataclass、TypedDict、Protocol、pydantic 和类型检查工具，也能帮助我们尽早发现字段写错、类型不匹配、返回值遗漏等问题。

第三，CLI、自动化、文件处理、测试和后端生态里 Python 工具链成熟，适合做命令行 Agent。

当然，Agent 思想不绑定 Python。你也可以用 Go、Rust 或其他语言实现同样架构。但本书为了减少切换成本，会一直使用 Python 做主线。

### 4.3 项目目标：mini-agent

我们要做的项目叫 `mini-agent`。

它最终会支持：

1. 命令行聊天。
2. 读取文件。
3. 搜索文件。
4. 执行 shell 命令。
5. 写入和编辑文件。
6. 权限确认。
7. 工具调用循环。
8. 上下文压缩。
9. 子 Agent。
10. MCP 工具。

但本章只做前三个基础：

```text
命令行入口
消息历史
普通模型调用
```

如果你暂时没有模型 API key，也可以先用 fake model。先把程序结构跑通，比一开始接真实 API 更重要。

### 4.4 推荐目录结构

先设计一个非常简单的目录：

```text
mini-agent/
  pyproject.toml
  src/
    cli.py
    agent/
      engine.py
      messages.py
    model/
      client.py
      fake_client.py
    tools/
      tool.py
      registry.py
    utils/
      errors.py
```

每个目录的职责：

`src/mini_agent/cli.py` 是命令行入口，负责读取用户输入、打印输出、启动 engine。

`src/mini_agent/agent/engine.py` 是 Agent 引擎。现在它只负责普通聊天，后面会加入工具循环。

`src/mini_agent/agent/messages.py` 定义内部消息类型。

`src/mini_agent/model/client.py` 定义模型客户端接口。真实模型和 fake model 都实现这个接口。

`src/mini_agent/model/fake_client.py` 是测试用假模型，方便我们不用 API key 也能跑通。

`src/mini_agent/tools/tool.py` 先预留工具接口。后面 Read、Bash、Edit 都会实现它。

`src/mini_agent/tools/registry.py` 管理工具列表。

这个结构比 Claude Code 简单得多，但主线是一致的：

```text
cli -> engine -> model client -> tools
```

### 4.5 初始化 pyproject.toml

教学项目的 `pyproject.toml` 可以这样写：

```toml
[project]
name = "mini-agent"
version = "0.1.0"
description = "A teaching mini AI agent built with Python"
requires-python = ">=3.11"
dependencies = [
  "anthropic>=0.40.0",
  "pydantic>=2.0.0",
  "typer>=0.12.0",
  "rich>=13.0.0",
]

[project.scripts]
mini-agent = "mini_agent.cli:main"

[project.optional-dependencies]
dev = [
  "pytest>=8.0.0",
  "pytest-asyncio>=0.23.0",
  "mypy>=1.8.0",
  "ruff>=0.6.0",
]
```

版本号在你实际安装时可能不同，使用当前最新稳定版本即可。这里重点不是具体版本，而是依赖类别：

- `anthropic`：模型 API。
- `pydantic`：工具参数 schema 校验。
- `typer`：命令行入口。
- `rich`：更友好的终端输出。
- `pytest`：测试。
- `mypy`：类型检查。
- `ruff`：代码风格和静态检查。

如果你还不想接真实模型，可以暂时不安装 SDK，只用 fake client。等工具循环跑通后再接真实 API。

### 4.6 Python 项目配置

一个简单的 `pyproject.toml` 可以这样写：

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
asyncio_mode = "auto"

[tool.mypy]
python_version = "3.11"
strict = true
files = ["src"]

[tool.ruff]
line-length = 100
target-version = "py311"
```

请打开 `strict`。Agent 项目里有大量结构化对象，关闭严格类型会让很多错误拖到运行时才暴露。

### 4.7 定义消息类型

先写 `src/mini_agent/agent/messages.py`。

教学版可以从最小类型开始：

```python
from dataclasses import dataclass
from typing import Literal

@dataclass
class Message:
    role: Literal["system", "user", "assistant"]
    content: str
```

这还不支持工具调用。没关系，先让聊天跑通。

后面我们会扩展成：

```python
from dataclasses import dataclass
from typing import Any, Literal

@dataclass
class TextBlock:
    type: Literal["text"]
    text: str

@dataclass
class ToolUseBlock:
    type: Literal["tool_use"]
    id: str
    name: str
    input: dict[str, Any]

@dataclass
class ToolResultBlock:
    type: Literal["tool_result"]
    tool_use_id: str
    content: str
    is_error: bool = False

ContentBlock = TextBlock | ToolUseBlock | ToolResultBlock
```

为什么不一开始就定义完整？因为新手最容易被类型吓住。先跑通纯文本，再引入 block，会更自然。

Claude Code 的内部消息类型更复杂，分布在 `src/utils/messages.ts` 等文件中。它除了普通 user/assistant，还要表示：

- progress message。
- system message。
- attachment message。
- tombstone message。
- compact boundary。
- tool use summary。
- API error message。

这些都是产品级需求。我们的教学项目先不需要。

### 4.8 定义模型客户端接口

接下来写 `src/mini_agent/model/client.py`。

不要让 Agent Engine 直接依赖某个 SDK。先定义接口：

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

这样做有三个好处。

第一，你可以先用 fake client，不需要 API key。

第二，将来可以切换模型供应商。

第三，测试 Agent 循环时，可以精确控制模型返回什么。

Claude Code 也有类似思想。`src/query/deps.ts` 把 `callModel`、`microcompact`、`autocompact`、`uuid` 抽成依赖。生产环境用真实实现，测试可以注入 fake。这种依赖注入对 Agent 非常重要，因为模型调用昂贵、慢、不稳定，不适合每个测试都真调。

### 4.9 Fake Model

写一个假模型：

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

这个模型没有智能，但能帮我们验证：

- CLI 能读输入。
- Engine 能保存消息。
- ModelClient 能被调用。
- 回复能打印出来。

新手做系统时，最重要的不是第一步就接入最强模型，而是让系统每层都能替换、能测试、能独立调试。

### 4.10 Agent Engine 第一版

写 `src/mini_agent/agent/engine.py`：

```python
from mini_agent.model.client import Message, ModelClient

class AgentEngine:
    def __init__(self, model: ModelClient) -> None:
        self.model = model
        self.messages: list[Message] = [
            Message(role="system", content="你是一个严谨的编程助手。")
        ]

    async def submit(self, user_text: str) -> Message:
        self.messages.append(Message(role="user", content=user_text))
        response = await self.model.complete(self.messages)
        self.messages.append(response.message)
        return response.message
```

这个类很小，但它已经有了 `QueryEngine` 的影子：

- 保存消息历史。
- 接收用户输入。
- 调用模型。
- 保存 assistant 回复。
- 返回结果。

Claude Code 的 `QueryEngine` 更复杂，因为它还要处理工具、权限、SDK 流式输出、用量统计、文件缓存、abort controller。但最核心的“会话状态 + submitMessage”思想是一样的。

### 4.11 CLI 第一版

写 `src/mini_agent/cli.py`：

```python
import asyncio

from mini_agent.agent.engine import AgentEngine
from mini_agent.model.fake_client import FakeModelClient

async def chat() -> None:
    engine = AgentEngine(FakeModelClient())

    while True:
        user_text = input("> ").strip()
        if user_text in {"exit", "quit"}:
            break
        message = await engine.submit(user_text)
        print(message.content)

def main() -> None:
    asyncio.run(chat())

if __name__ == "__main__":
    main()
```

运行后，你会看到：

```text
mini-agent started. Type /exit to quit.
> 你好
你刚才说：你好
> 今天我们学习 Agent
你刚才说：今天我们学习 Agent
```

这不是智能 Agent，但它是一个可靠起点。

### 4.12 为什么先写 Fake Client

很多人会觉得 fake client 没必要，直接接真实模型更快。但工程项目里，fake client 是非常有价值的。

原因一：节省成本。

调试 CLI、消息数组、循环逻辑时，不需要每次花 API 费用。

原因二：稳定。

真实模型输出不可预测，fake client 输出可预测。你可以写测试断言。

原因三：方便模拟工具调用。

下一章我们加入工具协议时，可以让 fake model 固定返回：

```json
{
  "type": "tool_use",
  "name": "read_file",
  "input": { "path": "README.md" }
}
```

这样就能测试工具循环，而不是祈祷真实模型刚好按你想的输出。

Claude Code 的 `src/query/deps.ts` 体现了同样思想。它把生产依赖集中起来，方便测试替换。

### 4.13 和 Claude Code 的入口对比

我们的入口：

```text
cli.py -> AgentEngine -> ModelClient
```

Claude Code 的入口：

```text
entrypoints/cli.ts -> main.tsx -> REPL/QueryEngine -> query.py -> services/api/claude.py
```

为什么 Claude Code 多这么多层？

因为它要支持：

- 版本命令。
- 非交互模式。
- React Ink TUI。
- MCP server。
- Chrome native host。
- remote control。
- daemon。
- 后台 session。
- 插件和技能。
- 企业策略。
- 会话恢复。

我们的教学项目不需要这些，所以先保持简单。真正重要的是保留可扩展方向：

- CLI 不直接调用模型，而是调用 Engine。
- Engine 不直接依赖 SDK，而是依赖 ModelClient 接口。
- 工具还没实现，但目录已经预留。
- 消息类型现在简单，但可以扩展为 content blocks。

### 4.14 本章练习

练习一：创建项目目录。

按照本章结构创建 `mini-agent`，并确保 `python -m mini_agent` 能启动。

练习二：给 CLI 增加 `/history` 命令。

当用户输入 `/history` 时，打印当前消息历史。例如：

```text
user: 你好
assistant: 你刚才说：你好
```

练习三：给 FakeModelClient 增加计数。

让它输出：

```text
第 1 次回复：你刚才说...
第 2 次回复：你刚才说...
```

这能帮助你确认 Engine 保存的是同一个会话，而不是每次新建。

练习四：思考一个问题。

如果下一章要加入工具调用，`Message` 类型应该怎么变？你可以先自己设计，再对照下一章。

### 4.15 本章小结

本章我们搭建了最小 CLI 项目。它只有纯聊天能力，但已经具备 Agent 系统的基本骨架：

1. CLI 负责输入输出。
2. AgentEngine 负责会话状态。
3. ModelClient 负责模型调用抽象。
4. FakeModelClient 负责本地可测试开发。
5. 消息类型为后续工具调用预留空间。

下一章我们会引入 Agent 最关键的协议：消息块、工具调用和工具结果。那时，我们的小程序会从“聊天机器人”开始变成真正的 Agent。

---
