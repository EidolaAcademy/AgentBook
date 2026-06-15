# 第 1 卷：入门与总览

## 第 3 章：Claude Code 源码地图

### 3.1 本章目标

本章不会逐行解释所有源码，而是先建立目录级地图。大型项目最怕“打开文件就迷路”。我们要先知道每个区域大概负责什么，再沿着主线深入。

本章读完后，你应该能回答：

1. CLI 从哪里启动？
2. 用户输入在哪里进入 Agent 循环？
3. 模型 API 调用在哪里？
4. 工具在哪里注册？
5. 单个工具在哪里执行？
6. 权限在哪里判断？
7. 子 Agent 在哪里创建？
8. MCP 工具在哪里接入？

### 3.2 仓库基本情况

当前仓库根目录主要包含：

```text
README.md
assets/
src/
```

这不是一个完整带 `pyproject.toml` 的普通开源项目，更像是从 sourcemap 还原出的源码快照。因此我们分析它时不把重点放在“如何安装运行”，而是放在“如何学习架构”。

`src/` 是核心目录，里面包含 CLI、工具、服务、状态、组件、MCP、技能、命令等模块。

### 3.3 第一条主线：启动链路

启动链路从：

```text
src/entrypoints/cli.python
```

开始。

这个文件的职责不是直接运行完整 Agent，而是做轻量分发。它会先检查一些特殊参数：

- `--version`
- `--dump-system-prompt`
- `--claude-in-chrome-mcp`
- `--chrome-native-host`
- `remote-control`
- `daemon`
- `ps/logs/attach/kill`

如果命中 fast path，就只加载必要模块并执行。这样可以减少启动时间。

如果没有命中特殊路径，才进入完整 CLI 主程序，也就是：

```text
src/main.python
```

`main.python` 是重量级入口。它负责：

1. 启动 profiler。
2. 读取 MDM 和 keychain。
3. 加载配置。
4. 初始化模型、认证、策略、插件、技能。
5. 加载命令和工具。
6. 初始化 MCP。
7. 创建 AppState。
8. 启动 React Ink 界面或非交互模式。

新手读这个文件时不要试图一次读完。正确方法是只追一条线：工具是在哪里拿到的，用户消息最后交给谁。

### 3.4 第二条主线：对话生命周期

对话生命周期的重要文件是：

```text
src/QueryEngine.py
src/query.py
```

`QueryEngine` 是面向“连续会话”的封装。它保存可变消息、读取缓存、总用量、abort controller 等。它的核心方法是 `submitMessage()`。

`query.py` 是实际 Agent 循环。前面已经讲过，它负责：

- 上下文准备。
- 压缩。
- 模型调用。
- 工具收集。
- 工具执行。
- 继续下一轮。

如果只能读一个文件理解 Agent 心脏，就读 `src/query.py`。

### 3.5 第三条主线：模型 API

模型 API 层在：

```text
src/services/api/claude.py
```

这个文件负责把内部结构转成 Anthropic Messages API 请求。它处理：

- model 名称规范化。
- messages 规范化。
- system prompt。
- tools schema。
- thinking。
- beta headers。
- prompt caching。
- retry。
- fallback。
- streaming。
- usage 和 cost。

你可以把它看作“内部 Agent 世界”和“外部模型 API 世界”之间的适配器。

### 3.6 第四条主线：工具定义与注册

工具接口在：

```text
src/tool.py
```

工具注册在：

```text
src/tools.py
```

具体工具在：

```text
src/mini_agent/tools/
```

例如：

```text
src/mini_agent/tools/BashTool/
src/mini_agent/tools/FileReadTool/
src/mini_agent/tools/FileEditTool/
src/mini_agent/tools/FileWriteTool/
src/mini_agent/tools/AgentTool/
src/mini_agent/tools/MCPTool/
src/mini_agent/tools/WebFetchTool/
src/mini_agent/tools/TodoWriteTool/
```

学习工具系统时，推荐顺序是：

1. 先读 `tool.py`，理解一个工具必须提供哪些方法。
2. 再读 `tools.py`，理解工具列表如何组合。
3. 然后读 `FileReadTool`，因为它相对容易理解。
4. 再读 `BashTool`，因为它最能体现安全复杂度。
5. 最后读 `AgentTool`，理解子 Agent。

### 3.7 第五条主线：工具执行

工具执行不是工具自己直接跑，而是经过统一执行层：

```text
src/services/tools/toolExecution.py
src/services/tools/toolOrchestration.py
src/services/tools/StreamingToolExecutor.py
src/services/tools/toolHooks.py
```

它们分别负责：

- `toolExecution.py`：单个工具调用生命周期。
- `toolOrchestration.py`：多个工具如何串行或并发执行。
- `StreamingToolExecutor.py`：流式输出过程中提前执行工具。
- `toolHooks.py`：pre/post hook。

如果你要自己写 Agent，不建议一开始就实现 streaming tool execution。先实现普通执行，再逐步升级。

### 3.8 第六条主线：权限系统

权限相关文件分布较广，关键入口是：

```text
src/hooks/useCanUseTool.python
src/mini_agent/utils/permissions/
src/hooks/toolPermission/
src/components/permissions/
```

权限系统不只是弹窗，它要决定：

- 当前模式是否允许工具。
- 规则是否匹配。
- 是否需要用户确认。
- 后台 Agent 是否能请求确认。
- 自动 classifier 是否能批准。
- hook 是否覆盖决定。
- 拒绝时如何反馈给模型。

对于新手项目，可以先实现三种模式：

```text
default：只读自动允许，写入询问
plan：禁止写入
bypass：全部允许，但只用于受信环境
```

以后再加复杂规则。

### 3.9 第七条主线：MCP

MCP 相关文件集中在：

```text
src/services/mcp/
src/mini_agent/tools/MCPTool/
src/mini_agent/tools/ListMcpResourcesTool/
src/mini_agent/tools/ReadMcpResourceTool/
```

其中 `src/services/mcp/client.py` 很关键。它会连接 MCP server，拉取工具列表，并把 MCP tool 包装成内部 Tool。

MCP 的学习重点不是协议细节，而是“外部工具如何统一进入内部工具系统”。只要包装成同一个 Tool 接口，后面的权限、执行、日志、上下文都能复用。

### 3.10 第八条主线：子 Agent

子 Agent 相关文件是：

```text
src/mini_agent/tools/AgentTool/AgentTool.python
src/mini_agent/tools/AgentTool/runAgent.py
src/mini_agent/tools/AgentTool/forkSubagent.py
src/mini_agent/tools/AgentTool/loadAgentsDir.py
src/mini_agent/tools/AgentTool/built-in/
```

`AgentTool` 是主 Agent 可调用的工具。模型调用它，就能启动另一个 Agent 来完成子任务。

`runAgent.py` 则负责真正运行子 Agent，包括：

- 构造子 Agent 初始消息。
- 选择模型。
- 构造工具池。
- 调整权限模式。
- 设置上下文。
- 处理 transcript。
- 返回结果或进度。

子 Agent 是高级能力，不建议初学者一开始就实现。但理解它很重要，因为很多复杂任务都需要拆分。

### 3.11 源码阅读策略

读大型源码不要从第一个文件读到最后一个文件。建议用“问题驱动”。

问题一：用户输入后发生什么？

阅读顺序：

```text
main.python -> QueryEngine.py -> query.py
```

问题二：模型如何调用工具？

阅读顺序：

```text
query.py -> toolExecution.py -> tool.py -> 某个具体工具
```

问题三：为什么有些命令需要确认？

阅读顺序：

```text
BashTool.python -> bashPermissions.py -> useCanUseTool.python -> utils/permissions
```

问题四：子 Agent 怎么跑？

阅读顺序：

```text
AgentTool.python -> runAgent.py -> tools.py -> query.py
```

问题五：MCP 工具怎么出现？

阅读顺序：

```text
services/mcp/client.py -> MCPtool.py -> tools.py
```

### 3.12 本章小结

本章建立了 Claude Code 源码地图。你现在不需要理解每个文件的细节，只要记住几条主线：

1. `entrypoints/cli.python` 和 `main.python` 是启动入口。
2. `QueryEngine.py` 和 `query.py` 是对话与 Agent 循环。
3. `services/api/claude.py` 是模型 API 适配。
4. `tool.py` 和 `tools.py` 是工具抽象与注册。
5. `services/tools/*` 是工具执行层。
6. `hooks/useCanUseTool.python` 和 `utils/permissions/*` 是权限层。
7. `services/mcp/*` 是 MCP 外部能力。
8. `tools/AgentTool/*` 是子 Agent。

下一章我们开始真正动手，从零创建一个 `mini-agent` 项目。

---
