# 第 2 卷：最小 Agent 实现

## 第 17 章：从 mini-agent 映射回 Claude Code 源码

### 17.1 本章目标

前面我们没有一开始硬啃 Claude Code，而是从零做了一个 mini-agent。现在它已经具备：

- CLI。
- 消息协议。
- Tool 接口。
- read/glob/grep/shell/write/edit 工具。
- 权限模式。
- 真实模型客户端。
- 流式响应雏形。
- 上下文预算。
- transcript。
- 测试策略。

现在回头看 Claude Code 源码，会清晰很多。

本章目标：

1. 把 mini-agent 模块映射到 Claude Code 文件。
2. 理解真实工程为什么拆得更细。
3. 区分“核心概念”和“产品级增强”。
4. 为下一卷深入工具系统工程化做准备。

### 17.2 入口映射

mini-agent：

```text
src/mini_agent/cli.py
```

Claude Code：

```text
src/entrypoints/cli.python
src/main.python
src/replLauncher.python
src/screens/REPL.python
```

mini-agent 的入口只做：

- 创建 readline。
- 创建 registry。
- 创建 model。
- 创建 engine。
- 循环读取用户输入。

Claude Code 的入口要做更多：

- fast path。
- remote-control。
- daemon。
- background session。
- MCP server。
- 插件和技能初始化。
- React Ink TUI。
- 登录和配置。
- 策略和企业设置。

核心思想一样：

```text
入口负责把运行环境准备好，然后把用户输入交给 Agent 引擎。
```

### 17.3 Engine 映射

mini-agent：

```text
src/mini_agent/agent/engine.py
```

Claude Code：

```text
src/QueryEngine.py
src/query.py
```

mini-agent 的 Engine 做：

- 保存 messages。
- submit user input。
- 调模型。
- 提取 tool_use。
- 执行工具。
- 追加 tool_result。
- maxTurns。

Claude Code 的 QueryEngine/query 做更多：

- SDK 消息输出。
- permission denial tracking。
- file read cache。
- usage/cost。
- memory。
- compact。
- streaming。
- fallback。
- stop hooks。
- task budget。
- agent source。
- snip/context collapse。

但核心循环仍然是：

```text
model -> tool_use -> run tools -> tool_result -> model
```

### 17.4 Tool 接口映射

mini-agent：

```text
src/mini_agent/tools/tool.py
src/mini_agent/tools/buildtool.py
```

Claude Code：

```text
src/tool.py
```

mini-agent Tool：

```python
name
description
inputSchema
inputJsonSchema
isReadOnly
isConcurrencySafe
checkPermission
call
formatResult
```

Claude Code Tool 增加了：

- prompt。
- renderToolUseMessage。
- renderToolResultMessage。
- mapToolResultToToolResultBlockParam。
- validateInput。
- isDestructive。
- isOpenWorld。
- requiresUserInteraction。
- MCP metadata。
- ToolSearch defer。
- progress rendering。
- permission matcher。

可以这样理解：

```text
mini-agent Tool = 模型和执行层需要的最小合同
Claude Code Tool = 模型、执行层、权限、UI、MCP、日志、SDK 全部需要的合同
```

### 17.5 工具执行映射

mini-agent：

```text
src/mini_agent/tools/runToolUse.py
```

Claude Code：

```text
src/services/tools/toolExecution.py
src/services/tools/toolOrchestration.py
src/services/tools/StreamingToolExecutor.py
src/services/tools/toolHooks.py
```

mini-agent 的执行流程：

```text
find tool
  ↓
schema parse
  ↓
permission
  ↓
tool.call
  ↓
format result
```

Claude Code 的执行流程还包括：

- telemetry。
- hook。
- progress。
- MCP auth error。
- tool result storage。
- permission denied hook。
- classifier。
- concurrent orchestration。
- streaming execution。
- contextModifier。

但骨架一样。理解 mini-agent 的 `runToolUse()`，再读 `toolExecution.py` 会轻松很多。

### 17.6 文件工具映射

mini-agent：

```text
readFileTool
writeFileTool
editFileTool
```

Claude Code：

```text
src/mini_agent/tools/FileReadTool/FileReadtool.py
src/mini_agent/tools/FileWriteTool/FileWritetool.py
src/mini_agent/tools/FileEditTool/FileEdittool.py
src/mini_agent/tools/NotebookEditTool/
```

mini-agent 文件工具处理：

- cwd 边界。
- 文本读取。
- offset/limit。
- oldString 唯一匹配。
- 写入前 ask。

Claude Code 额外处理：

- 图片。
- PDF。
- Notebook。
- 二进制。
- 文件大小。
- token 限制。
- diff UI。
- IDE 通知。
- 文件历史。
- sed edit。
- 权限规则。

核心原则一致：

```text
读要限制上下文，写要确认，编辑要精确。
```

### 17.7 Shell 映射

mini-agent：

```text
shellTool
execCommand
OutputAccumulator
```

Claude Code：

```text
src/mini_agent/tools/BashTool/BashTool.python
src/mini_agent/tools/BashTool/bashPermissions.py
src/mini_agent/tools/BashTool/readOnlyValidation.py
src/mini_agent/tools/BashTool/commandSemantics.py
src/mini_agent/tools/BashTool/bashSecurity.py
src/mini_agent/tools/BashTool/shouldUseSandbox.py
```

mini-agent Shell：

- spawn。
- timeout。
- output truncation。
- 粗略只读判断。
- 权限 ask。

Claude Code Shell：

- 更完整命令解析。
- 沙箱。
- 后台任务。
- 权限规则。
- classifier。
- git 操作追踪。
- sed 特殊处理。
- UI progress。
- 大输出持久化。
- search/read/list 分类。

Shell 是两者差距最大的模块之一。原因不是概念不同，而是生产安全要求高得多。

### 17.8 权限系统映射

mini-agent：

```text
PermissionDecision
PermissionMode
PermissionPrompter
checkToolPermission
```

Claude Code：

```text
src/hooks/useCanUseTool.python
src/mini_agent/utils/permissions/
src/hooks/toolPermission/
src/components/permissions/
```

mini-agent 权限模式：

- default。
- plan。
- bypass。

Claude Code 权限系统：

- 多模式。
- allow/deny/ask 规则。
- session/user/project/policy 来源。
- auto mode。
- classifier。
- hooks。
- coordinator/swarm 特殊处理。
- UI queue。
- SDK permission callback。

核心原则一致：

```text
工具请求必须先经过权限决策，ask 应由运行环境处理。
```

### 17.9 模型 API 映射

mini-agent：

```text
AnthropicModelClient
toApiMessages
toApiTools
fromApiMessage
```

Claude Code：

```text
src/services/api/claude.py
src/services/api/client.py
src/mini_agent/utils/api.js
src/mini_agent/utils/messages.js
```

mini-agent API 层：

- 非流式调用。
- system prompt。
- messages。
- tools。
- response mapping。

Claude Code API 层：

- streaming。
- retry。
- fallback。
- prompt cache。
- thinking。
- beta headers。
- usage/cost。
- quota。
- deferred tools。
- context management。

核心仍然是：

```text
内部消息和工具 -> API 请求 -> API 响应 -> 内部消息
```

### 17.10 上下文工程映射

mini-agent：

```text
estimateTokens
shrinkToolResults
compactMessagesIfNeeded
```

Claude Code：

```text
src/services/compact/
src/mini_agent/utils/toolResultStorage.py
src/services/tokenEstimation.py
src/memdir/
src/services/SessionMemory/
```

mini-agent 只做粗略估算和简单摘要。

Claude Code 做：

- token warning。
- auto compact。
- micro compact。
- cached microcompact。
- tool result disk storage。
- memory prefetch。
- session memory。
- context collapse。

核心原则一致：

```text
发给模型的上下文必须预算化，不能无限增长。
```

### 17.11 会话映射

mini-agent：

```text
TranscriptStore
loadMessages
```

Claude Code：

```text
src/mini_agent/utils/sessionStorage.py
src/mini_agent/utils/conversationRecovery.py
src/commands/resume/
src/mini_agent/tools/AgentTool/runAgent.py
```

mini-agent 保存 JSONL。

Claude Code 还处理：

- session id。
- project mapping。
- resume UI。
- transcript search。
- agent metadata。
- content replacement records。
- sidechain agent transcript。

核心原则一致：

```text
完整历史用于审计和恢复，模型上下文可以是压缩视图。
```

### 17.12 为什么真实工程会复杂这么多

现在你应该能看出：Claude Code 不是因为“Agent 概念本身很神秘”才复杂，而是因为它要在真实用户环境中可靠运行。

复杂度来自：

1. 多平台。
2. 终端 UI。
3. 权限安全。
4. 长会话。
5. 大项目。
6. MCP 外部工具。
7. 子 Agent。
8. 插件。
9. 企业策略。
10. 观测和调试。

mini-agent 是骨架。Claude Code 是骨架加上真实世界的肌肉、神经和免疫系统。

### 17.13 本章小结

本章把 mini-agent 映射回 Claude Code。

你现在应该能带着问题读源码：

- 想看入口，读 `entrypoints/cli.python` 和 `main.python`。
- 想看 Agent 循环，读 `QueryEngine.py` 和 `query.py`。
- 想看工具接口，读 `tool.py`。
- 想看工具执行，读 `services/tools/*`。
- 想看文件工具，读 `FileReadTool`、`FileEditTool`、`FileWriteTool`。
- 想看 Shell，读 `BashTool`。
- 想看权限，读 `useCanUseTool` 和 `utils/permissions`。
- 想看上下文，读 `services/compact` 和 `toolResultStorage`。

下一卷我们会更深入地工程化工具系统，重点讲 schema、权限、并发、结果预算、MCP 工具包装和 ToolSearch。

---

# 第 3 卷：工具系统工程化
