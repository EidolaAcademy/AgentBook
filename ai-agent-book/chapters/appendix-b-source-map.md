# 附录 B：Claude Code 源码阅读索引与学习路线

本附录给读者一张源码阅读地图。

前面的章节已经围绕 Agent 工程把 Claude Code 的关键设计拆开讲了。但真正学习源码时，很多新手会遇到一个问题：文件很多，不知道从哪里读起。

源码阅读不能从第一个文件一路顺序读到最后一个文件。大型项目不是小说，而是城市。你需要先知道主干道路在哪里，再进入街区。

本附录的目标是：

1. 帮你找到入口文件。
2. 帮你区分核心逻辑和 UI 逻辑。
3. 帮你按主题阅读工具、权限、会话、MCP、子 Agent、hooks、memory、plugins。
4. 帮你把源码阅读和本书章节对应起来。
5. 帮你形成自己的源码笔记方法。

说明：源码项目可能继续变化，文件路径可能随版本调整。本附录基于当前工作区中 `repo/src` 的结构整理。读者学习时应结合实际仓库版本确认。

## B.1 阅读源码前的心态

读 Agent 源码不要期待第一遍就全部看懂。

你应该分三遍读。

第一遍：读结构。

目标是知道有哪些模块、入口在哪里、核心循环在哪里。不追求理解每个函数。

第二遍：读链路。

目标是沿着一次用户请求走完整流程：输入、query、模型、工具、权限、结果、UI、持久化。

第三遍：读细节。

目标是深入某个主题，比如权限规则、Bash 安全、ToolSearch、子 Agent、MCP、session storage。

不要从 UI 组件开始读，也不要从所有 utils 开始读。那会很快迷路。

## B.2 第一组入口：主程序与 query

优先阅读：

```text
repo/src/main.python
repo/src/query.py
repo/src/QueryEngine.py
repo/src/context.py
repo/src/setup.py
```

这些文件回答的问题：

1. 程序如何启动？
2. 用户输入如何进入 Agent？
3. query 函数如何组织一次请求？
4. 上下文如何加载？
5. 模型调用前做了哪些准备？
6. 工具结果如何回到循环？

阅读方法：

先读 `main.python`，找到 CLI/应用启动入口。然后读 `query.py`，寻找关键的 query 执行流程。再看 `QueryEngine.py`，理解 query 被封装成什么抽象。最后读 `context.py`，了解上下文组装逻辑。

你不需要第一次就理解所有条件分支。先抓住一条主线：

```text
用户输入 -> query -> 组装上下文 -> 调模型 -> 处理响应 -> 工具执行 -> 递归/继续
```

对应章节：

1. 第 2 章：从一次模型调用到 Agent 循环。
2. 第 12 章：接入真实模型 API。
3. 第 13 章：流式响应。
4. 第 14 章：上下文预算。
5. 第 38 章：query profiling。

## B.3 第二组入口：工具抽象

优先阅读：

```text
repo/src/tool.py
repo/src/tools.py
repo/src/constants/tools.py
repo/src/mini_agent/tools/
```

这些文件回答的问题：

1. Tool 接口长什么样？
2. 工具如何声明 name、description、schema？
3. 工具如何被合并和暴露给模型？
4. 工具结果如何格式化？
5. 内置工具有哪些？

阅读方法：

先读 `tool.py`。这是理解整个工具系统的关键。你要重点找：

1. 工具定义字段。
2. 输入 schema 类型。
3. 执行函数签名。
4. 权限相关字段。
5. 结果返回类型。

然后读 `tools.py` 和 `constants/tools.py`，看工具如何集中导出、分组或命名。

最后进入 `repo/src/mini_agent/tools/`，按工具逐个看。不要一口气看完所有工具。建议顺序：

1. Read 类工具。
2. Grep/Glob 类工具。
3. Bash 工具。
4. Edit/Write 类工具。
5. AgentTool。
6. MCP 包装工具。

对应章节：

1. 第 5 章：消息协议与工具调用。
2. 第 6 章：Read 工具。
3. 第 7 章：搜索工具。
4. 第 8 章：Shell 工具。
5. 第 10 章：写文件与编辑文件。
6. 第 18 章：Tool Schema。
7. 第 19 章：工具执行生命周期。

## B.4 第三组入口：权限系统

优先阅读：

```text
repo/src/commands/permissions/
repo/src/components/permissions/
repo/src/hooks/toolPermission/PermissionContext.py
repo/src/hooks/toolPermission/permissionLogging.py
repo/src/components/permissions/shellPermissionHelpers.python
repo/src/components/permissions/utils.py
```

这些文件回答的问题：

1. 权限规则如何展示和配置？
2. 工具执行前如何请求权限？
3. allow、deny、ask 如何在 UI 中表现？
4. shell 权限如何解释？
5. 权限决策如何记录？

阅读方法：

先读 `PermissionContext.py`，理解权限上下文如何传递。然后读 `permissionLogging.py`，看权限决策如何被记录。接着读 `components/permissions` 下的 UI 组件，理解用户批准和拒绝如何呈现。

对于 Bash，重点看 `shellPermissionHelpers.python`。Shell 权限不是简单工具名判断，它还要解释命令风险、命令前缀、沙箱提示。

对应章节：

1. 第 9 章：权限系统。
2. 第 22 章：Bash 安全。
3. 第 25 章：权限、安全与沙箱总览。
4. 第 26 章：权限规则。
5. 第 29 章：审计、日志与安全可解释性。

## B.5 第四组入口：会话、历史与 transcript

优先阅读：

```text
repo/src/assistant/sessionHistory.py
repo/src/commands/session/
repo/src/history.py
repo/src/mini_agent/utils/sessionStorage.py
repo/src/hooks/useAssistantHistory.py
```

这些文件回答的问题：

1. 会话历史如何存储？
2. transcript 文件在哪里？
3. session id 如何生成和兼容？
4. resume 如何加载历史？
5. UI 如何展示历史消息？

阅读方法：

先读 `sessionHistory.py` 和 `history.py`，理解历史消息如何被管理。然后读 `utils/sessionStorage.py`，这是 transcript 相关逻辑的重点。你要关注：

1. transcript path。
2. session id。
3. JSONL 读写。
4. 哪些消息会持久化。
5. 旧格式如何兼容。
6. sidechain 如何记录。

最后读 `commands/session`，看用户如何通过命令管理 session。

对应章节：

1. 第 15 章：会话持久化与恢复。
2. 第 32 章：子 Agent 上下文隔离。
3. 第 38 章：Transcript、sidechain、replay。

## B.6 第五组入口：子 Agent 与多 Agent

优先阅读：

```text
repo/src/cli/handlers/agents.py
repo/src/commands/agents/
repo/src/components/agents/
repo/src/mini_agent/tools/AgentTool/
repo/src/mini_agent/utils/agentToolUtils.py
```

如果某些路径在版本中不同，可以用：

```bash
rg "AgentTool" repo/src
rg "agentId" repo/src
rg "sidechain" repo/src
```

这些文件回答的问题：

1. AgentDefinition 如何定义？
2. 自定义 Agent 如何加载？
3. AgentTool 如何启动子 Agent？
4. 子 Agent 如何限制工具？
5. 子 Agent 如何返回结果？
6. sidechain transcript 如何记录？

阅读方法：

先读 CLI handler 和 command，理解用户如何管理 agents。再读 `components/agents`，理解编辑器和列表界面。最后读 AgentTool 和 agentToolUtils，理解真正执行子 Agent 的链路。

读子 Agent 代码时要盯住四个字段：

1. agentId。
2. tools。
3. maxTurns。
4. transcript/sidechain。

对应章节：

1. 第 30 章：子 Agent 总览。
2. 第 31 章：深入 AgentTool。
3. 第 32 章：上下文隔离。
4. 第 34 章：多 Agent 编排。
5. 第 37 章：自定义 Agent、插件 Agent。

## B.7 第六组入口：MCP

优先阅读：

```text
repo/src/entrypoints/mcp.py
repo/src/cli/handlers/mcp.python
repo/src/commands/mcp/
repo/src/components/mcp/
```

这些文件回答的问题：

1. MCP 入口如何启动？
2. MCP server 如何添加、配置、展示？
3. MCP tool 如何展示详情？
4. MCP 权限和连接状态如何处理？
5. MCP 和本地工具如何融合？

阅读方法：

先读 `entrypoints/mcp.py`，了解 MCP 入口。然后读 `commands/mcp`，看命令行如何管理 MCP server。再读 `components/mcp`，看 UI 如何展示 server、tool list、capabilities、reconnect。

最后在工具合并逻辑中找 MCP tool adapter，理解外部工具如何变成模型可调用的工具。

对应章节：

1. 第 23 章：MCP 工具包装。
2. 第 24 章：ToolSearch。
3. 第 25 章：权限、安全与沙箱总览。

## B.8 第七组入口：Hooks

优先阅读：

```text
repo/src/commands/hooks/
repo/src/components/hooks/
repo/src/hooks/useDeferredHookMessages.py
```

这些文件回答的问题：

1. hooks 如何配置？
2. hook event 有哪些？
3. matcher 如何选择？
4. hook 执行结果如何展示？
5. hook 消息如何进入 UI？

阅读方法：

先看 commands，理解用户如何配置 hooks。然后看 components，理解事件、matcher、hook mode 的 UI。最后看运行时 hook 消息如何被延迟展示或插入。

对应章节：

1. 第 28 章：Hooks 与外部策略。
2. 第 29 章：审计、日志与安全可解释性。

## B.9 第八组入口：Memory

优先阅读：

```text
repo/src/commands/memory/
repo/src/components/memory/
repo/src/hooks/useMemoryUsage.py
```

这些文件回答的问题：

1. memory 如何查看和编辑？
2. memory 文件如何选择？
3. memory 更新如何通知用户？
4. memory 对上下文有什么影响？

阅读方法：

先读 `commands/memory`，看命令入口。再读 `components/memory`，看 UI 如何展示 memory 文件选择和更新通知。最后查 memory 如何进入上下文。

对应章节：

1. 第 36 章：Agent memory、skills preload 与长期协作能力。
2. 第 39 章：路线图中的 skills 和 memory 阶段。

## B.10 第九组入口：Skills 与插件

优先搜索：

```bash
rg "skill" repo/src
rg "plugin" repo/src
rg "marketplace" repo/src
```

重点路径通常包括：

```text
repo/src/hooks/useSkillsChange.py
repo/src/hooks/useManagePlugins.py
repo/src/hooks/notifs/usePluginInstallationStatus.python
repo/src/hooks/notifs/usePluginAutoupdateNotification.python
repo/src/commands/install.python
repo/src/components/agents/
```

这些文件回答的问题：

1. skills 如何变化和加载？
2. plugin 如何安装和更新？
3. plugin agent 如何进入 agents 列表？
4. 插件状态如何通知用户？
5. 插件能力如何受权限约束？

对应章节：

1. 第 36 章：memory、skills preload。
2. 第 37 章：自定义 Agent、插件 Agent 与分发机制。

## B.11 第十组入口：UI 与交互

优先阅读：

```text
repo/src/components/App.python
repo/src/components/AgentProgressLine.python
repo/src/components/CompactSummary.python
repo/src/components/ContextVisualization.python
repo/src/components/FileEditToolDiff.python
repo/src/components/FallbackToolUseErrorMessage.python
repo/src/components/FallbackToolUseRejectedMessage.python
repo/src/components/DiagnosticsDisplay.python
```

这些文件回答的问题：

1. 主界面如何组织？
2. Agent 进度如何展示？
3. compact summary 如何展示？
4. 上下文可视化如何呈现？
5. 文件 diff 如何展示？
6. 工具错误和拒绝如何展示？

UI 不是 Agent 核心，但它决定用户能否理解 Agent 正在做什么。

对应章节：

1. 第 13 章：流式响应。
2. 第 21 章：工具结果预算。
3. 第 29 章：审计与可解释性。
4. 第 38 章：Task progress 和调试。

## B.12 第十一组入口：CLI commands

优先阅读：

```text
repo/src/commands.py
repo/src/commands/
repo/src/cli/
```

命令目录里能看到很多功能入口：

```text
repo/src/commands/review.py
repo/src/commands/security-review.py
repo/src/commands/commit.py
repo/src/commands/commit-push-pr.py
repo/src/commands/init.py
repo/src/commands/statusline.python
repo/src/commands/version.py
```

这些文件回答的问题：

1. 斜杠命令或 CLI 命令如何注册？
2. review/security-review 如何作为专门工作流存在？
3. commit/push/pr 这类高风险命令如何设计？
4. 命令如何和 Agent 主循环交互？

对应章节：

1. 第 4 章：CLI。
2. 第 11 章：完整任务闭环。
3. 第 35 章：teammate 和团队式协作。

## B.13 第十二组入口：Bridge 与远程会话

优先阅读：

```text
repo/src/bridge/
repo/src/hooks/useRemoteSession.py
repo/src/hooks/useMailboxBridge.py
repo/src/components/BridgeDialog.python
repo/src/components/DesktopHandoff.python
```

这些文件回答的问题：

1. 本地和远程如何桥接？
2. session 如何创建和运行？
3. inbound messages 如何处理？
4. remote IO 如何进入本地 UI？
5. handoff 如何展示？

这一组对新手不是第一优先级。等你理解本地 Agent loop 后再读。

对应章节：

1. 第 15 章：会话持久化与恢复。
2. 第 34 章：任务取消、恢复与冲突处理。
3. 第 35 章：团队式协作。

## B.14 第十三组入口：成本与统计

优先阅读：

```text
repo/src/cost-tracker.py
repo/src/costHook.py
repo/src/components/CostThresholdDialog.python
repo/src/hooks/useMemoryUsage.py
repo/src/mini_agent/utils/queryProfiler.py
```

这些文件回答的问题：

1. token 和成本如何统计？
2. 成本阈值如何提示用户？
3. query profiler 如何记录 checkpoint？
4. memory usage 如何展示？

对应章节：

1. 第 14 章：上下文预算。
2. 第 21 章：工具结果预算。
3. 第 38 章：生产可观测性。

## B.15 推荐源码阅读顺序

如果你只有一天时间，按这个顺序：

1. `repo/src/tool.py`
2. `repo/src/query.py`
3. `repo/src/tools.py`
4. `repo/src/context.py`
5. `repo/src/hooks/toolPermission/permissionLogging.py`
6. `repo/src/mini_agent/utils/sessionStorage.py`
7. `repo/src/mini_agent/utils/queryProfiler.py`

如果你有三天时间：

第一天：主循环和工具。

1. `main.python`
2. `query.py`
3. `QueryEngine.py`
4. `tool.py`
5. `tools.py`
6. `repo/src/mini_agent/tools/`

第二天：权限、会话、上下文。

1. `components/permissions/`
2. `hooks/toolPermission/`
3. `utils/sessionStorage.py`
4. `context.py`
5. `components/CompactSummary.python`
6. `components/ContextVisualization.python`

第三天：子 Agent、MCP、观测。

1. `commands/agents/`
2. `components/agents/`
3. `agentToolUtils.py`
4. `commands/mcp/`
5. `components/mcp/`
6. `utils/queryProfiler.py`

如果你有两周时间：

第一阶段：从入口到 Agent loop。

第二阶段：从 Tool 接口到具体工具。

第三阶段：从权限 UI 到权限审计。

第四阶段：从 sessionStorage 到 replay 思路。

第五阶段：从 AgentTool 到 sidechain。

第六阶段：从 MCP 到 ToolSearch。

第七阶段：从 queryProfiler 到 eval 设计。

## B.16 每读一个文件要问的五个问题

读源码时不要只是看代码。每个文件都问五个问题：

1. 这个文件负责什么？
2. 它的输入是什么？
3. 它的输出是什么？
4. 它依赖哪些模块？
5. 它被哪些模块调用？

对于 Agent 代码，再加五个问题：

1. 它是否影响模型上下文？
2. 它是否影响工具执行？
3. 它是否影响权限判断？
4. 它是否影响持久化？
5. 它是否影响用户可见行为？

这样读源码会更有方向。

## B.17 如何做源码笔记

建议用表格：

| 文件 | 责任 | 关键函数/类型 | 关联章节 | 我的理解 |
| --- | --- | --- | --- | --- |
| `tool.py` | 工具抽象 | Tool | 第 18 章 | 工具是模型和程序的合同 |
| `query.py` | 主请求流程 | query | 第 2 章 | Agent loop 的核心链路 |
| `sessionStorage.py` | 会话持久化 | transcript | 第 15 章 | JSONL 保存事实链 |

每读完一个模块，写三句话：

1. 它解决什么问题。
2. 它怎么解决。
3. 如果我做 mini-agent，先实现其中哪 20%。

这种笔记比复制代码有用。

## B.18 如何把源码转成自己的 mini-agent

不要照抄全部源码。

应该做“降维实现”：

1. 从 `tool.py` 学接口，但只实现 4 个工具。
2. 从 `query.py` 学循环，但先不做所有边缘分支。
3. 从权限系统学 allow/deny/ask，但先用简单规则。
4. 从 sessionStorage 学 JSONL，但先不做所有兼容逻辑。
5. 从 AgentTool 学 sidechain，但先只做一个子 Agent。
6. 从 queryProfiler 学 checkpoint，但先记录 8 个关键点。

这就是本书一直强调的：理解生产系统，构建教学版 mini-agent，再逐步演进。

## B.19 读源码时的常见误区

第一，先读 UI。

UI 文件很多，容易让你误以为系统就是组件堆叠。Agent 核心不在 UI，先读 query 和 Tool。

第二，追每一个 import。

这会让你陷入无穷跳转。先读主链路，再补依赖。

第三，忽略类型定义。

Python 项目里，类型定义往往就是架构说明。`Tool`、`AgentDefinition`、`PermissionDecision` 这类类型比很多实现细节更重要。

第四，把生产复杂度照搬到学习项目。

Claude Code 要处理大量真实场景。mini-agent 不需要一开始就处理所有兼容、UI、远程桥接和插件生态。

第五，只看 happy path。

Agent 工程真正难的是错误、权限、取消、超时、截断、恢复、审计。

## B.20 本附录小结

读 Claude Code 源码的路线可以概括为：

1. 先读入口和 query，找到 Agent loop。
2. 再读 Tool，理解模型如何调用程序能力。
3. 再读权限，理解工具为什么不能随便执行。
4. 再读 sessionStorage，理解 transcript 和 resume。
5. 再读 AgentTool，理解子 Agent。
6. 再读 MCP、hooks、memory、plugins，理解扩展能力。
7. 最后读 profiler、events、UI，理解生产可观测性和用户体验。

源码不是用来崇拜的，而是用来学习取舍的。

你真正要带走的不是某个函数写法，而是这些工程判断：

1. 能力要通过工具边界暴露。
2. 工具要有 schema。
3. 执行要有权限。
4. 历史要有 transcript。
5. 大输出要有预算。
6. 子任务要隔离。
7. 失败要能回放。
8. 生产要能观测。

读懂这些，再回去做自己的 mini-agent，你会清楚得多。
