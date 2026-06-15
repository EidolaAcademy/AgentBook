# 从零搭建 AI Agent：从 Claude Code 源码到工程实战

## 前言

很多人第一次听到 AI Agent，会以为它是某种很神秘的新模型。也有人以为只要把提示词写得足够长，让模型“自己思考、自己行动”，就已经拥有了 Agent。真正开始写代码之后，这种想象很快会被现实磨平：模型会忘记上下文，会生成错误参数，会调用不存在的工具，会把字符串当数字，会读取过大的文件，会在没有权限时试图修改系统文件，会在 shell 里执行你没有预料到的命令，也会在多轮工具调用之后把自己的状态弄乱。

这本书想解决的正是这个落差。

我们不会把 Agent 当成一个玄学概念来讨论，也不会停留在“写一个 prompt 调一个 API”的层面。我们会从最小可运行的命令行聊天程序开始，一步一步加入工具调用、文件读取、shell 执行、权限确认、上下文压缩、MCP、子 Agent、后台任务、会话恢复、日志、测试和产品化界面。每一步都会先讲为什么需要，再讲怎么实现，最后对照 Claude Code 源码看一个成熟产品是怎样处理同类问题的。

本书参考的源码是 `yasasbanukaofficial/claude-code` 仓库中的 Claude Code 源码快照。该仓库 README 说明这些文件来自 npm sourcemap 暴露后的源码备份，因此本书只把它作为工程学习材料：学习架构、抽象、边界处理和系统设计，而不是复制源码或鼓励把它当成可自由使用的开源项目。

如果你是新手，请不要被源码规模吓到。大型 Agent 项目看起来像一座城市：入口、道路、权限、通信、仓库、工厂、监控都在里面。我们不会从城市中心最复杂的十字路口开始，而是先搭一间小屋：一个能聊天的小程序。然后给它开门，让它能读文件；给它工具箱，让它能运行命令；给它规则，让它知道什么时候要停下来问用户；给它记忆，让它能长时间工作；最后才把它扩展成一个真正可用的 Agent 系统。

---

# 第 1 卷：入门与总览

## 第 1 章：AI Agent 到底是什么

### 1.1 本章目标

读完本章，你要能回答四个问题：

1. AI Agent 和普通聊天机器人有什么区别？
2. 一个 Agent 最少需要哪些组成部分？
3. 为什么工具调用是 Agent 的核心？
4. 为什么工程级 Agent 的大部分代码都在处理边界情况？

本章不会急着写代码。我们先建立一张清晰地图。很多初学者一上来就复制示例代码，结果很快卡在“模型为什么不调用工具”“工具结果为什么回不去”“为什么下一轮模型报错”“为什么上下文爆了”等问题上。先理解 Agent 的结构，会让后面的代码轻松很多。

### 1.2 普通聊天机器人是什么

最简单的聊天机器人只有三样东西：

1. 用户输入。
2. 模型调用。
3. 模型输出。

流程非常短：

```text
用户：帮我解释这段代码
  ↓
程序：把用户问题发给模型
  ↓
模型：生成解释
  ↓
程序：把解释打印出来
```

这类程序当然有价值。它可以回答问题、改写文本、生成代码片段、翻译、总结、做头脑风暴。但它有一个根本限制：它只能基于已经放进上下文里的信息回答。它不能自己打开文件，不能自己执行测试，不能自己搜索项目，不能自己修改代码，不能验证自己的答案。

举个例子，用户问：

```text
这个项目为什么测试失败？帮我修一下。
```

普通聊天机器人如果没有工具，只能做几种事：

1. 要求用户把错误日志贴出来。
2. 根据经验猜测原因。
3. 给出通用排查步骤。
4. 生成一段可能有用的代码。

它不能真正进入项目目录，不能运行 `npm test`，不能读取失败文件，不能修改代码，也不能再次运行测试确认修复。这样的系统更像“会说话的顾问”，不是“会干活的助手”。

### 1.3 Agent 的最小定义

在工程上，我们可以把 AI Agent 定义为：

```text
一个能在多轮循环中观察环境、调用工具、更新状态，并基于反馈继续行动的模型驱动程序。
```

这个定义里有几个关键词。

第一，多轮循环。Agent 不是问一次答一次，而是可以持续执行：

```text
思考 -> 调工具 -> 看结果 -> 再思考 -> 再调工具 -> 完成
```

第二，观察环境。环境可以是文件系统、终端、网页、数据库、API、MCP server、IDE、任务队列，也可以是用户的进一步反馈。

第三，调用工具。工具是模型影响外部世界的方式。没有工具，模型只能生成文本；有了工具，模型才能读文件、运行命令、搜索、写代码、发请求、创建任务。

第四，更新状态。Agent 要保存消息历史、工具结果、任务进度、文件读取缓存、权限决定、成本统计、上下文压缩摘要等。没有状态，Agent 就无法持续工作。

第五，基于反馈继续行动。工具结果不是给用户看的终点，而是给模型看的下一轮输入。模型读到结果后，决定下一步。

### 1.4 一个最小 Agent 的样子

先看一个非常简化的 Agent：

```text
用户：总结 README.md
  ↓
模型：我需要读取 README.md
  ↓
程序：执行 read_file("README.md")
  ↓
程序：把文件内容作为 tool_result 发回模型
  ↓
模型：根据 README 内容生成总结
  ↓
程序：输出总结给用户
```

这里已经出现 Agent 的核心循环：

```text
用户输入
  ↓
模型响应
  ↓
是否有工具调用？
  ↓ 是
执行工具
  ↓
把工具结果追加到消息历史
  ↓
再次调用模型
  ↓
直到模型不再调用工具
```

如果把这段逻辑写成伪代码，大概是：

```ts
while (true) {
  const assistantMessage = await callModel(messages, tools)
  messages.push(assistantMessage)

  const toolUses = extractToolUses(assistantMessage)
  if (toolUses.length === 0) {
    return assistantMessage
  }

  for (const toolUse of toolUses) {
    const result = await executeTool(toolUse)
    messages.push(makeToolResultMessage(toolUse.id, result))
  }
}
```

这就是 Agent 的心脏。后面几十章都在围绕这个循环做增强。

### 1.5 Agent 与自动化脚本的区别

有人会问：如果 Agent 只是循环调用工具，它和普通自动化脚本有什么区别？

区别在“决策位置”。

普通脚本的流程由程序员提前写死：

```text
读取文件 -> 运行测试 -> 如果失败就退出 -> 打印日志
```

Agent 的流程由模型在运行时决定：

```text
用户提出目标
  ↓
模型判断需要读哪些文件
  ↓
模型判断要运行哪些命令
  ↓
模型根据错误结果决定下一步
  ↓
模型决定修改哪个文件
  ↓
模型决定是否再次验证
```

程序员写的是“行动空间”和“规则边界”，不是完整步骤。工具、权限、上下文、状态机就是行动空间和规则边界。

这个区别非常重要。Agent 的灵活性来自模型决策，但风险也来自模型决策。模型不是传统程序，它可能误解目标、误判风险、生成错误参数、在上下文不足时猜测。因此工程系统必须给它护栏。

### 1.6 工程级 Agent 的四层能力模型

为了方便学习，我们把 Agent 能力分成四层。

第一层：对话能力。

这是最基础的能力。模型能理解用户目标，能解释、总结、规划、生成文本。没有这一层，就没有 Agent。

第二层：工具能力。

模型能通过结构化协议调用外部工具。例如：

- 读取文件。
- 搜索文件。
- 执行 shell。
- 写文件。
- 请求网页。
- 调用数据库。
- 调用 MCP 工具。

第三层：状态能力。

Agent 能维护长期任务状态。例如：

- 当前消息历史。
- 已读取文件缓存。
- 当前任务列表。
- 子 Agent 状态。
- 权限授权记录。
- 压缩后的摘要。
- 会话 transcript。

第四层：治理能力。

Agent 能安全、可控、可观测地运行。例如：

- 权限确认。
- 沙箱。
- 超时与取消。
- 错误恢复。
- 成本追踪。
- 日志。
- 测试。
- 策略配置。

很多教程只讲前两层，所以示例看起来很酷，一到真实项目就容易失控。Claude Code 这类工程级 Agent 最值得学习的地方，恰恰在第三层和第四层。

### 1.7 为什么 Claude Code 源码这么大

初看 Claude Code 源码，你可能会疑惑：不就是一个命令行 Agent 吗，为什么有这么多文件？

因为一个真正可用的 coding Agent 要处理的问题远超“调用模型”。

它要处理入口：

- 用户可能运行 `claude`。
- 用户可能运行 `claude --version`。
- 用户可能以非交互方式传入 prompt。
- 用户可能进入 remote-control。
- 用户可能启动后台任务。
- 用户可能连接 Chrome、IDE 或 MCP。

它要处理工具：

- Bash。
- 文件读写。
- 搜索。
- Web。
- Notebook。
- MCP。
- 子 Agent。
- 计划模式。
- 任务列表。

它要处理权限：

- 哪些命令能自动跑？
- 哪些文件能读？
- 哪些修改必须问用户？
- 后台 Agent 不能弹窗怎么办？
- 企业策略禁止某些能力怎么办？

它要处理上下文：

- 项目说明怎么注入？
- 用户偏好怎么注入？
- 工具结果太大怎么办？
- 长会话爆上下文怎么办？
- 压缩摘要不可靠怎么办？

它要处理产品体验：

- 终端 UI 如何实时显示流式输出？
- 工具进度如何展示？
- 用户如何中断？
- 会话如何恢复？
- 错误如何解释？

所以，工程级 Agent 的大部分代码不是在“让模型更聪明”，而是在“让模型安全可靠地工作”。

### 1.8 本章小结

本章我们建立了 Agent 的基本概念：

1. 普通聊天机器人只能生成文本。
2. Agent 能在循环中调用工具并根据结果继续行动。
3. 工具调用协议是 Agent 的核心。
4. Agent 的能力分为对话、工具、状态、治理四层。
5. 工程级 Agent 的复杂度主要来自边界、安全、上下文和产品化。

下一章我们开始进入第一个关键技术：从一次模型调用，扩展到真正的 Agent 循环。

---

## 第 2 章：从一次模型调用到 Agent 循环

### 2.1 本章目标

本章要把 Agent 的核心循环讲透。读完之后，你应该能自己写出一个最小 Agent runner，并能理解 Claude Code 源码中 `query()` 为什么是整个系统的心脏。

我们会先从普通模型调用开始，再加入 messages，再加入 tools，再加入 tool_result，最后形成完整循环。

### 2.2 一次普通模型调用

最简单的模型调用只有一个 prompt：

```ts
const answer = await model.generate("解释什么是闭包")
console.log(answer)
```

这类调用的问题是没有历史。用户下一句说“再举个例子”，模型不知道“再”指什么。于是我们引入消息历史：

```ts
const messages = [
  { role: "user", content: "解释什么是闭包" },
  { role: "assistant", content: "闭包是..." },
  { role: "user", content: "再举个例子" },
]
```

模型看到的是一段对话，而不是孤立问题。

### 2.3 System Prompt 的作用

除了用户消息，我们还需要 system prompt。它定义模型的身份、规则和工作方式。

例如：

```text
You are a careful coding assistant.
When you need information from the filesystem, use tools instead of guessing.
Do not modify files unless the user asks you to.
```

对 Agent 来说，system prompt 尤其重要，因为它要告诉模型：

1. 有哪些工具。
2. 什么时候该用工具。
3. 不要猜测外部世界。
4. 遇到权限限制时如何处理。
5. 输出格式和工作风格。

Claude Code 的 system prompt 并不是一个简单字符串，而是由多个部分拼接而成。源码中与 prompt 和 context 相关的逻辑分布在：

- `src/constants/prompts.js`
- `src/context.ts`
- `src/utils/queryContext.js`
- `src/utils/systemPrompt.ts`
- `src/query.ts`

这说明大型 Agent 的 prompt 是动态生成的：不同模型、不同模式、不同权限、不同项目上下文，都会影响最终 prompt。

### 2.4 Tools：让模型拥有行动选项

工具是结构化的。模型不是输出“我想读 README”，而是输出一个工具调用块：

```json
{
  "type": "tool_use",
  "id": "toolu_001",
  "name": "Read",
  "input": {
    "file_path": "/project/README.md"
  }
}
```

程序看到这个块后，执行本地工具：

```ts
const content = await readFile("/project/README.md", "utf8")
```

然后把结果包装成：

```json
{
  "type": "tool_result",
  "tool_use_id": "toolu_001",
  "content": "README 文件内容..."
}
```

再发回模型。模型读到工具结果，才能继续生成最终回答。

### 2.5 工具调用为什么必须结构化

初学者很容易想：让模型输出一段 JSON，然后我自己解析不就行了吗？

不够。

工具调用协议要解决的不只是 JSON 格式，还有：

1. 工具名称是否合法。
2. 参数 schema 是否正确。
3. 工具调用和工具结果如何配对。
4. 多个工具调用如何排序。
5. 工具失败如何表达。
6. 工具结果如何进入下一轮上下文。
7. 模型是否知道哪些工具可用。

如果没有正式协议，Agent 很快会出现奇怪问题。例如模型输出：

```text
CALL_TOOL read_file path=README.md
```

你可以写正则解析。但下一次它可能输出：

```text
I will read README.md now.
```

再下一次它可能输出：

```json
{"tool": "ReadFile", "filename": "README.md"}
```

这就变成了脆弱的文本解析。结构化工具调用让模型和程序之间有明确合同。

### 2.6 最小循环的状态机

Agent 循环可以画成状态机：

```text
[等待用户输入]
      ↓
[调用模型]
      ↓
模型是否请求工具？
      ↓ 否
[输出最终回答] -> [等待用户输入]
      ↓ 是
[执行工具]
      ↓
[追加工具结果]
      ↓
[调用模型]
```

注意，执行工具之后不是直接输出给用户，而是回到调用模型。因为工具结果往往只是原材料，模型还要解释、整合、决定下一步。

比如用户说“修复测试”。工具结果可能是：

```text
FAIL src/parser.test.ts
Expected "abc" but received "ab"
```

这个结果不是最终答案。模型要读它，判断失败原因，读取相关源文件，修改代码，再运行测试。

### 2.7 教学版 Agent Runner

下面是一个教学版 runner。它省略了真实 SDK 细节，但保留核心结构：

```ts
type ToolUse = {
  id: string
  name: string
  input: Record<string, unknown>
}

type Tool = {
  name: string
  call(input: Record<string, unknown>): Promise<string>
}

async function runAgent({
  userInput,
  tools,
}: {
  userInput: string
  tools: Tool[]
}) {
  const messages: any[] = [
    { role: "user", content: userInput },
  ]

  while (true) {
    const assistant = await callModel({ messages, tools })
    messages.push(assistant)

    const toolUses: ToolUse[] = extractToolUses(assistant)

    if (toolUses.length === 0) {
      return {
        finalMessage: assistant,
        messages,
      }
    }

    for (const toolUse of toolUses) {
      const tool = tools.find(t => t.name === toolUse.name)

      if (!tool) {
        messages.push(makeToolError(toolUse.id, `Unknown tool: ${toolUse.name}`))
        continue
      }

      try {
        const result = await tool.call(toolUse.input)
        messages.push(makeToolResult(toolUse.id, result))
      } catch (error) {
        messages.push(makeToolError(toolUse.id, String(error)))
      }
    }
  }
}
```

这段代码已经能表达 Agent 的核心。但是它还不适合真实项目，因为缺少：

- 参数校验。
- 权限检查。
- 并发控制。
- 超时取消。
- 上下文压缩。
- 流式响应。
- 工具结果大小限制。
- 日志和错误恢复。

Claude Code 的 `src/query.ts` 就是在这个最小循环上不断加工程层。

### 2.8 Claude Code 中的对应位置

核心函数是：

- `src/query.ts` 中的 `query()`
- `src/query.ts` 中的 `queryLoop()`

`query()` 是外层生成器，负责启动查询并在正常结束时标记命令生命周期完成。真正复杂逻辑在 `queryLoop()`。

`queryLoop()` 内部每轮都会做这些事：

1. 准备消息。
2. 做 memory prefetch。
3. 做 skill discovery prefetch。
4. 做 tool result budget。
5. 做 snip/microcompact/context collapse/autocompact。
6. 生成完整 system prompt。
7. 选择当前模型。
8. 调用 streaming API。
9. 收集 assistant messages 和 tool_use blocks。
10. 执行工具。
11. 将结果加入消息。
12. 判断是否继续。

你可以把它理解成一个加强版：

```ts
while (true) {
  messagesForQuery = prepareMessages(messages)
  messagesForQuery = await compactIfNeeded(messagesForQuery)

  const stream = callModelStreaming(messagesForQuery, tools)

  for await (const event of stream) {
    collectAssistantOutput(event)
    maybeStartStreamingToolExecution(event)
  }

  if (noToolUse) return done

  const toolResults = await runTools(toolUses)
  messages.push(...toolResults)
}
```

### 2.9 新手常见误区

误区一：把工具结果直接展示给用户。

工具结果应该先回给模型。用户真正需要的是模型基于工具结果整理后的答案。当然 UI 可以同时展示工具执行过程，但对话状态里必须有 tool_result。

误区二：工具失败就终止整个 Agent。

很多工具失败是可恢复的。文件不存在，模型可以换路径；测试失败，模型可以继续修；权限拒绝，模型可以解释原因或请求用户。

误区三：不保存完整消息历史。

没有历史，模型无法知道自己刚才调用了哪个工具。至少在一轮 Agent 执行中，assistant 的 tool_use 和 user 的 tool_result 必须保留。

误区四：工具参数不校验。

模型输出不等于可信输入。必须校验类型、必填字段、路径范围、安全约束。

### 2.10 本章小结

本章我们从一次普通模型调用出发，构造了 Agent 循环。你现在应该理解：

1. Messages 保存对话状态。
2. System prompt 定义规则。
3. Tools 定义行动空间。
4. Tool use 和 tool result 必须配对。
5. Agent 循环的核心是“模型 -> 工具 -> 模型”。
6. Claude Code 的 `query.ts` 是这个循环的工程级实现。

下一章我们进入源码地图，看看这套项目如何把入口、循环、工具、权限、MCP 和子 Agent 组织起来。

---

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

这不是一个完整带 `package.json` 的普通开源项目，更像是从 sourcemap 还原出的源码快照。因此我们分析它时不把重点放在“如何安装运行”，而是放在“如何学习架构”。

`src/` 是核心目录，里面包含 CLI、工具、服务、状态、组件、MCP、技能、命令等模块。

### 3.3 第一条主线：启动链路

启动链路从：

```text
src/entrypoints/cli.tsx
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
src/main.tsx
```

`main.tsx` 是重量级入口。它负责：

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
src/QueryEngine.ts
src/query.ts
```

`QueryEngine` 是面向“连续会话”的封装。它保存可变消息、读取缓存、总用量、abort controller 等。它的核心方法是 `submitMessage()`。

`query.ts` 是实际 Agent 循环。前面已经讲过，它负责：

- 上下文准备。
- 压缩。
- 模型调用。
- 工具收集。
- 工具执行。
- 继续下一轮。

如果只能读一个文件理解 Agent 心脏，就读 `src/query.ts`。

### 3.5 第三条主线：模型 API

模型 API 层在：

```text
src/services/api/claude.ts
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
src/Tool.ts
```

工具注册在：

```text
src/tools.ts
```

具体工具在：

```text
src/tools/
```

例如：

```text
src/tools/BashTool/
src/tools/FileReadTool/
src/tools/FileEditTool/
src/tools/FileWriteTool/
src/tools/AgentTool/
src/tools/MCPTool/
src/tools/WebFetchTool/
src/tools/TodoWriteTool/
```

学习工具系统时，推荐顺序是：

1. 先读 `Tool.ts`，理解一个工具必须提供哪些方法。
2. 再读 `tools.ts`，理解工具列表如何组合。
3. 然后读 `FileReadTool`，因为它相对容易理解。
4. 再读 `BashTool`，因为它最能体现安全复杂度。
5. 最后读 `AgentTool`，理解子 Agent。

### 3.7 第五条主线：工具执行

工具执行不是工具自己直接跑，而是经过统一执行层：

```text
src/services/tools/toolExecution.ts
src/services/tools/toolOrchestration.ts
src/services/tools/StreamingToolExecutor.ts
src/services/tools/toolHooks.ts
```

它们分别负责：

- `toolExecution.ts`：单个工具调用生命周期。
- `toolOrchestration.ts`：多个工具如何串行或并发执行。
- `StreamingToolExecutor.ts`：流式输出过程中提前执行工具。
- `toolHooks.ts`：pre/post hook。

如果你要自己写 Agent，不建议一开始就实现 streaming tool execution。先实现普通执行，再逐步升级。

### 3.8 第六条主线：权限系统

权限相关文件分布较广，关键入口是：

```text
src/hooks/useCanUseTool.tsx
src/utils/permissions/
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
src/tools/MCPTool/
src/tools/ListMcpResourcesTool/
src/tools/ReadMcpResourceTool/
```

其中 `src/services/mcp/client.ts` 很关键。它会连接 MCP server，拉取工具列表，并把 MCP tool 包装成内部 Tool。

MCP 的学习重点不是协议细节，而是“外部工具如何统一进入内部工具系统”。只要包装成同一个 Tool 接口，后面的权限、执行、日志、上下文都能复用。

### 3.10 第八条主线：子 Agent

子 Agent 相关文件是：

```text
src/tools/AgentTool/AgentTool.tsx
src/tools/AgentTool/runAgent.ts
src/tools/AgentTool/forkSubagent.ts
src/tools/AgentTool/loadAgentsDir.ts
src/tools/AgentTool/built-in/
```

`AgentTool` 是主 Agent 可调用的工具。模型调用它，就能启动另一个 Agent 来完成子任务。

`runAgent.ts` 则负责真正运行子 Agent，包括：

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
main.tsx -> QueryEngine.ts -> query.ts
```

问题二：模型如何调用工具？

阅读顺序：

```text
query.ts -> toolExecution.ts -> Tool.ts -> 某个具体工具
```

问题三：为什么有些命令需要确认？

阅读顺序：

```text
BashTool.tsx -> bashPermissions.ts -> useCanUseTool.tsx -> utils/permissions
```

问题四：子 Agent 怎么跑？

阅读顺序：

```text
AgentTool.tsx -> runAgent.ts -> tools.ts -> query.ts
```

问题五：MCP 工具怎么出现？

阅读顺序：

```text
services/mcp/client.ts -> MCPTool.ts -> tools.ts
```

### 3.12 本章小结

本章建立了 Claude Code 源码地图。你现在不需要理解每个文件的细节，只要记住几条主线：

1. `entrypoints/cli.tsx` 和 `main.tsx` 是启动入口。
2. `QueryEngine.ts` 和 `query.ts` 是对话与 Agent 循环。
3. `services/api/claude.ts` 是模型 API 适配。
4. `Tool.ts` 和 `tools.ts` 是工具抽象与注册。
5. `services/tools/*` 是工具执行层。
6. `hooks/useCanUseTool.tsx` 和 `utils/permissions/*` 是权限层。
7. `services/mcp/*` 是 MCP 外部能力。
8. `tools/AgentTool/*` 是子 Agent。

下一章我们开始真正动手，从零创建一个 `mini-agent` 项目。

---

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

### 4.2 为什么选择 TypeScript

本书示例主要使用 TypeScript，有几个原因。

第一，Claude Code 源码本身就是 TypeScript/React Ink 风格。用相近语言更容易对照学习。

第二，Agent 工程非常依赖结构化数据。消息、工具、权限、工具参数、工具结果、配置、状态都需要清晰类型。TypeScript 能帮我们尽早发现字段写错、类型不匹配、返回值遗漏等问题。

第三，前端、CLI、Node.js 生态里 TypeScript 工具链成熟，适合做命令行 Agent。

当然，Agent 思想不绑定 TypeScript。你也可以用 Python、Go、Rust 实现同样架构。但本书为了减少切换成本，会一直使用 TypeScript 做主线。

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
  package.json
  tsconfig.json
  src/
    cli.ts
    agent/
      engine.ts
      messages.ts
    model/
      client.ts
      fakeClient.ts
    tools/
      Tool.ts
      registry.ts
    utils/
      errors.ts
```

每个目录的职责：

`src/cli.ts` 是命令行入口，负责读取用户输入、打印输出、启动 engine。

`src/agent/engine.ts` 是 Agent 引擎。现在它只负责普通聊天，后面会加入工具循环。

`src/agent/messages.ts` 定义内部消息类型。

`src/model/client.ts` 定义模型客户端接口。真实模型和 fake model 都实现这个接口。

`src/model/fakeClient.ts` 是测试用假模型，方便我们不用 API key 也能跑通。

`src/tools/Tool.ts` 先预留工具接口。后面 Read、Bash、Edit 都会实现它。

`src/tools/registry.ts` 管理工具列表。

这个结构比 Claude Code 简单得多，但主线是一致的：

```text
cli -> engine -> model client -> tools
```

### 4.5 初始化 package.json

教学项目的 `package.json` 可以这样写：

```json
{
  "name": "mini-agent",
  "version": "0.1.0",
  "type": "module",
  "scripts": {
    "dev": "tsx src/cli.ts",
    "build": "tsc",
    "start": "node dist/cli.js"
  },
  "dependencies": {
    "@anthropic-ai/sdk": "^0.60.0",
    "zod": "^4.0.0"
  },
  "devDependencies": {
    "@types/node": "^24.0.0",
    "tsx": "^4.0.0",
    "typescript": "^5.0.0"
  }
}
```

版本号在你实际安装时可能不同，使用当前最新稳定版本即可。这里重点不是具体版本，而是依赖类别：

- `@anthropic-ai/sdk`：模型 API。
- `zod`：工具参数 schema 校验。
- `tsx`：开发阶段直接运行 TypeScript。
- `typescript`：类型检查和编译。

如果你还不想接真实模型，可以暂时不安装 SDK，只用 fake client。等工具循环跑通后再接真实 API。

### 4.6 tsconfig

一个简单的 `tsconfig.json`：

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "outDir": "dist",
    "rootDir": "src",
    "esModuleInterop": true,
    "skipLibCheck": true
  },
  "include": ["src"]
}
```

请打开 `strict`。Agent 项目里有大量结构化对象，关闭严格类型会让很多错误拖到运行时才暴露。

### 4.7 定义消息类型

先写 `src/agent/messages.ts`。

教学版可以从最小类型开始：

```ts
export type Role = "user" | "assistant" | "system"

export type TextMessage = {
  role: Role
  content: string
}

export type Message = TextMessage
```

这还不支持工具调用。没关系，先让聊天跑通。

后面我们会扩展成：

```ts
export type ToolUseBlock = {
  type: "tool_use"
  id: string
  name: string
  input: Record<string, unknown>
}

export type ToolResultBlock = {
  type: "tool_result"
  tool_use_id: string
  content: string
  is_error?: boolean
}

export type ContentBlock =
  | { type: "text"; text: string }
  | ToolUseBlock
  | ToolResultBlock
```

为什么不一开始就定义完整？因为新手最容易被类型吓住。先跑通纯文本，再引入 block，会更自然。

Claude Code 的内部消息类型更复杂，分布在 `src/types/message.js` 等文件中。它除了普通 user/assistant，还要表示：

- progress message。
- system message。
- attachment message。
- tombstone message。
- compact boundary。
- tool use summary。
- API error message。

这些都是产品级需求。我们的教学项目先不需要。

### 4.8 定义模型客户端接口

接下来写 `src/model/client.ts`。

不要让 Agent Engine 直接依赖某个 SDK。先定义接口：

```ts
import type { Message } from "../agent/messages.js"

export type ModelResponse = {
  message: Message
}

export type ModelClient = {
  complete(messages: Message[]): Promise<ModelResponse>
}
```

这样做有三个好处。

第一，你可以先用 fake client，不需要 API key。

第二，将来可以切换模型供应商。

第三，测试 Agent 循环时，可以精确控制模型返回什么。

Claude Code 也有类似思想。`src/query/deps.ts` 把 `callModel`、`microcompact`、`autocompact`、`uuid` 抽成依赖。生产环境用真实实现，测试可以注入 fake。这种依赖注入对 Agent 非常重要，因为模型调用昂贵、慢、不稳定，不适合每个测试都真调。

### 4.9 Fake Model

写一个假模型：

```ts
import type { Message } from "../agent/messages.js"
import type { ModelClient, ModelResponse } from "./client.js"

export class FakeModelClient implements ModelClient {
  async complete(messages: Message[]): Promise<ModelResponse> {
    const last = messages[messages.length - 1]

    return {
      message: {
        role: "assistant",
        content: `你刚才说：${last?.content ?? ""}`,
      },
    }
  }
}
```

这个模型没有智能，但能帮我们验证：

- CLI 能读输入。
- Engine 能保存消息。
- ModelClient 能被调用。
- 回复能打印出来。

新手做系统时，最重要的不是第一步就接入最强模型，而是让系统每层都能替换、能测试、能独立调试。

### 4.10 Agent Engine 第一版

写 `src/agent/engine.ts`：

```ts
import type { Message } from "./messages.js"
import type { ModelClient } from "../model/client.js"

export class AgentEngine {
  private messages: Message[] = []

  constructor(private readonly model: ModelClient) {}

  async submit(userText: string): Promise<Message> {
    this.messages.push({
      role: "user",
      content: userText,
    })

    const response = await this.model.complete(this.messages)

    this.messages.push(response.message)

    return response.message
  }

  getMessages(): readonly Message[] {
    return this.messages
  }
}
```

这个类很小，但它已经有了 `QueryEngine` 的影子：

- 保存消息历史。
- 接收用户输入。
- 调用模型。
- 保存 assistant 回复。
- 返回结果。

Claude Code 的 `QueryEngine` 更复杂，因为它还要处理工具、权限、SDK 流式输出、用量统计、文件缓存、abort controller。但最核心的“会话状态 + submitMessage”思想是一样的。

### 4.11 CLI 第一版

写 `src/cli.ts`：

```ts
import readline from "node:readline/promises"
import { stdin as input, stdout as output } from "node:process"
import { AgentEngine } from "./agent/engine.js"
import { FakeModelClient } from "./model/fakeClient.js"

async function main() {
  const rl = readline.createInterface({ input, output })
  const engine = new AgentEngine(new FakeModelClient())

  console.log("mini-agent started. Type /exit to quit.")

  while (true) {
    const text = await rl.question("> ")

    if (text.trim() === "/exit") {
      break
    }

    const message = await engine.submit(text)
    console.log(message.content)
  }

  rl.close()
}

main().catch(error => {
  console.error(error)
  process.exit(1)
})
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
cli.ts -> AgentEngine -> ModelClient
```

Claude Code 的入口：

```text
entrypoints/cli.tsx -> main.tsx -> REPL/QueryEngine -> query.ts -> services/api/claude.ts
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

按照本章结构创建 `mini-agent`，并确保 `npm run dev` 能启动。

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

## 第 5 章：消息协议与工具调用

### 5.1 本章目标

本章是从聊天机器人进入 Agent 的分水岭。上一章我们写了一个只会把用户输入发给模型的小 CLI。本章要给它引入工具调用协议。

读完本章，你应该能理解：

1. 为什么普通字符串消息不够用。
2. 什么是 content block。
3. 什么是 `tool_use`。
4. 什么是 `tool_result`。
5. 为什么工具调用和工具结果必须配对。
6. 工具失败时应该如何告诉模型。
7. mini-agent 的消息类型应该如何升级。
8. Claude Code 源码中哪些地方处理这些协议细节。

这章非常关键。很多 Agent 项目失败，不是因为模型不够强，而是因为消息协议处理错了。工具结果放错角色、工具 ID 对不上、错误直接 throw 出去、历史中留下半截工具调用，这些问题都会让 Agent 在多轮任务中变得不稳定。

### 5.2 为什么字符串消息不够用

上一章的消息类型是：

```ts
export type Message = {
  role: "user" | "assistant" | "system"
  content: string
}
```

这对普通聊天足够，但对 Agent 不够。因为 Agent 消息里不只有文本，还会有结构化动作。

模型可能输出：

```text
我需要读取 README.md。
```

这只是文本。程序无法可靠知道它是否真的要执行读取，也不知道路径字段在哪里。

Agent 需要模型输出结构化块：

```json
{
  "type": "tool_use",
  "id": "toolu_001",
  "name": "read_file",
  "input": {
    "path": "README.md"
  }
}
```

程序看到 `type: "tool_use"`，就知道这不是普通回答，而是一个需要执行的动作。

执行完成后，程序再追加：

```json
{
  "type": "tool_result",
  "tool_use_id": "toolu_001",
  "content": "README 文件内容..."
}
```

模型看到这个结果后，再继续生成总结。

这就是为什么 content 不能只是字符串。它需要支持多种 block。

### 5.3 Content Block 的概念

Content block 可以理解为“消息内容里的积木”。一条消息不再只有一段文本，而是由多个块组成。

例如 assistant 消息可以是：

```json
[
  {
    "type": "text",
    "text": "我先读取 README。"
  },
  {
    "type": "tool_use",
    "id": "toolu_001",
    "name": "read_file",
    "input": {
      "path": "README.md"
    }
  }
]
```

user 消息也可以是：

```json
[
  {
    "type": "tool_result",
    "tool_use_id": "toolu_001",
    "content": "# 项目说明..."
  }
]
```

注意这里的 `user` 不一定代表真人输入。工具结果在 Messages API 里通常作为 user role 的内容返回给模型。这一点初学者很容易困惑：明明工具是程序执行的，为什么 role 是 user？

可以这样理解：assistant 发起工具请求，外部环境把观察结果“交回”给 assistant。这个外部环境在对话协议里站在 user 侧，所以 tool_result 放在 user 消息中。

### 5.4 tool_use 的字段

一个工具调用块通常包含：

```ts
type ToolUseBlock = {
  type: "tool_use"
  id: string
  name: string
  input: Record<string, unknown>
}
```

字段含义：

`type` 表示这是工具调用。

`id` 是本次工具调用的唯一编号。后面的 tool_result 必须用它来配对。

`name` 是工具名，例如 `read_file`、`bash`、`edit_file`。

`input` 是工具参数。它必须符合工具 schema。

最重要的是 `id`。没有 ID，就无法在多个工具调用中知道哪个结果对应哪个请求。

例如模型同时请求两个文件：

```json
[
  {
    "type": "tool_use",
    "id": "toolu_read_a",
    "name": "read_file",
    "input": { "path": "a.ts" }
  },
  {
    "type": "tool_use",
    "id": "toolu_read_b",
    "name": "read_file",
    "input": { "path": "b.ts" }
  }
]
```

程序执行后必须返回：

```json
[
  {
    "type": "tool_result",
    "tool_use_id": "toolu_read_a",
    "content": "a.ts 内容"
  },
  {
    "type": "tool_result",
    "tool_use_id": "toolu_read_b",
    "content": "b.ts 内容"
  }
]
```

如果 ID 对错了，模型会把文件内容理解反。写代码时这类错误很隐蔽，必须通过类型和测试防住。

### 5.5 tool_result 的字段

工具结果块可以定义为：

```ts
type ToolResultBlock = {
  type: "tool_result"
  tool_use_id: string
  content: string
  is_error?: boolean
}
```

字段含义：

`type` 表示这是工具结果。

`tool_use_id` 对应前面的 `tool_use.id`。

`content` 是工具返回给模型的内容。

`is_error` 表示工具是否失败。

工具失败时，不应该总是直接终止程序。应该把错误作为工具结果交回模型。例如：

```json
{
  "type": "tool_result",
  "tool_use_id": "toolu_001",
  "is_error": true,
  "content": "Error: file README.md not found"
}
```

模型看到这个错误后，可能会搜索文件、换路径、询问用户，或者解释无法完成。

### 5.6 为什么错误也要进入消息历史

新手常犯的错误是：

```ts
try {
  const result = await tool.call(input)
} catch (error) {
  throw error
}
```

这样做会让整个 Agent 循环中断。对系统错误来说可以，但对工具执行错误来说通常不理想。

比如文件不存在：

```text
Error: ENOENT README.md
```

这不是 Agent 必须崩溃的理由。模型可以运行搜索工具找真实文件名，也可以告诉用户项目没有 README。

比如测试失败：

```text
Exit code 1
Expected 1 but received 2
```

这更不是崩溃理由。用户本来就是让 Agent 修测试，测试失败是有价值的观察。

所以我们要区分两类错误：

第一类，系统级错误。比如模型 API 认证失败、程序代码 bug、消息格式非法。这类可能需要终止。

第二类，任务级错误。比如文件不存在、命令退出码非零、权限拒绝。这类应该作为 `tool_result` 返回给模型。

Claude Code 的 `src/services/tools/toolExecution.ts` 就大量处理这种情况：工具不存在、schema 错误、validateInput 失败、权限拒绝、工具调用异常，都会被包装成模型可理解的工具结果。

### 5.7 工具调用配对规则

配对规则可以总结为：

```text
每个 assistant 的 tool_use，都必须有后续 user 的 tool_result。
```

更具体一点：

1. `tool_result.tool_use_id` 必须等于某个 `tool_use.id`。
2. 一个 `tool_use.id` 不应该对应多个成功结果。
3. 如果工具被取消，也要返回取消结果。
4. 如果工具失败，也要返回错误结果。
5. 不要在历史里留下没有结果的工具调用。

为什么这么严格？因为下一次 API 调用时，模型服务会验证消息结构。如果有 assistant 要求工具但没有结果，服务可能拒绝请求，或者模型会在逻辑上困惑。

Claude Code 在 `src/query.ts` 里有处理 missing tool result blocks 的逻辑。如果某些 assistant message 中有 tool_use 但缺少结果，它会生成中断消息补齐，避免下一轮上下文不合法。

教学版可以先写一个检查函数：

```ts
export function findUnresolvedToolUses(messages: Message[]): ToolUseBlock[] {
  const toolUses = new Map<string, ToolUseBlock>()
  const resolved = new Set<string>()

  for (const message of messages) {
    for (const block of message.content) {
      if (block.type === "tool_use") {
        toolUses.set(block.id, block)
      }

      if (block.type === "tool_result") {
        resolved.add(block.tool_use_id)
      }
    }
  }

  return [...toolUses.values()].filter(toolUse => !resolved.has(toolUse.id))
}
```

这个函数不完美，但可以帮你理解配对思想。

### 5.8 升级 mini-agent 的消息类型

现在我们把 `src/agent/messages.ts` 升级。

第一步，定义 block：

```ts
export type TextBlock = {
  type: "text"
  text: string
}

export type ToolUseBlock = {
  type: "tool_use"
  id: string
  name: string
  input: Record<string, unknown>
}

export type ToolResultBlock = {
  type: "tool_result"
  tool_use_id: string
  content: string
  is_error?: boolean
}

export type ContentBlock =
  | TextBlock
  | ToolUseBlock
  | ToolResultBlock
```

第二步，定义消息：

```ts
export type Role = "user" | "assistant" | "system"

export type Message = {
  role: Role
  content: ContentBlock[]
}
```

第三步，写辅助函数：

```ts
export function userText(text: string): Message {
  return {
    role: "user",
    content: [{ type: "text", text }],
  }
}

export function assistantText(text: string): Message {
  return {
    role: "assistant",
    content: [{ type: "text", text }],
  }
}

export function toolResult(
  toolUseId: string,
  content: string,
  isError = false,
): Message {
  return {
    role: "user",
    content: [
      {
        type: "tool_result",
        tool_use_id: toolUseId,
        content,
        ...(isError ? { is_error: true } : {}),
      },
    ],
  }
}
```

这样写的好处是，其他代码不需要手动拼对象，减少格式错误。

### 5.9 更新 FakeModelClient

上一章的 fake client 返回字符串，现在要返回 block 消息：

```ts
import { assistantText } from "../agent/messages.js"
import type { Message } from "../agent/messages.js"
import type { ModelClient, ModelResponse } from "./client.js"

function getText(message: Message): string {
  return message.content
    .filter(block => block.type === "text")
    .map(block => block.text)
    .join("")
}

export class FakeModelClient implements ModelClient {
  async complete(messages: Message[]): Promise<ModelResponse> {
    const last = messages[messages.length - 1]
    const text = last ? getText(last) : ""

    return {
      message: assistantText(`你刚才说：${text}`),
    }
  }
}
```

这一步看似麻烦，但很必要。因为从现在开始，模型回复可能不只有文本。我们必须养成遍历 content blocks 的习惯。

### 5.10 更新 CLI 输出

CLI 打印 assistant 消息时，也不能直接 `message.content`。要提取文本块：

```ts
function renderMessage(message: Message): string {
  return message.content
    .map(block => {
      if (block.type === "text") return block.text
      if (block.type === "tool_use") return `[tool_use: ${block.name}]`
      if (block.type === "tool_result") return `[tool_result: ${block.tool_use_id}]`
      return ""
    })
    .join("\n")
}
```

然后：

```ts
const message = await engine.submit(text)
console.log(renderMessage(message))
```

真实产品会把不同 block 渲染成不同 UI。Claude Code 使用 React Ink 组件渲染工具调用、工具结果、进度、diff、权限弹窗等。我们先用纯文本显示。

### 5.11 从模型响应中提取工具调用

写一个工具函数：

```ts
export function extractToolUses(message: Message): ToolUseBlock[] {
  if (message.role !== "assistant") return []

  return message.content.filter(
    (block): block is ToolUseBlock => block.type === "tool_use",
  )
}
```

这个函数后面会进入 AgentEngine：

```ts
const response = await this.model.complete(this.messages)
this.messages.push(response.message)

const toolUses = extractToolUses(response.message)

if (toolUses.length === 0) {
  return response.message
}

// 后面执行工具
```

现在我们还没有工具执行层，所以先只提取，不执行。

### 5.12 让 FakeModel 模拟工具调用

为了测试协议，可以让 fake model 在用户输入包含“读文件”时返回 tool_use：

```ts
export class FakeModelClient implements ModelClient {
  async complete(messages: Message[]): Promise<ModelResponse> {
    const last = messages[messages.length - 1]
    const text = last ? getText(last) : ""

    if (text.includes("读文件")) {
      return {
        message: {
          role: "assistant",
          content: [
            {
              type: "text",
              text: "我需要读取 README.md。",
            },
            {
              type: "tool_use",
              id: "fake_toolu_001",
              name: "read_file",
              input: {
                path: "README.md",
              },
            },
          ],
        },
      }
    }

    return {
      message: assistantText(`你刚才说：${text}`),
    }
  }
}
```

运行后，如果用户输入：

```text
请读文件
```

CLI 可以显示：

```text
我需要读取 README.md。
[tool_use: read_file]
```

下一章我们会真正执行这个工具。

### 5.13 消息协议和 Claude Code 的对应关系

Claude Code 中和消息协议相关的模块很多，主要包括：

```text
src/types/message.js
src/utils/messages.js
src/utils/messages/mappers.js
src/utils/messages/systemInit.js
src/services/api/claude.ts
src/query.ts
```

`src/utils/messages.js` 负责创建、规范化、裁剪、转换各种消息。

`src/services/api/claude.ts` 在发 API 前会把内部消息转换成 API 需要的格式，并确保工具结果配对。

`src/query.ts` 在 Agent 循环中收集 assistant message、tool_use block 和工具结果。

`src/services/tools/toolExecution.ts` 会把工具输出映射成 tool_result block。

这说明消息协议不是一个边缘细节，而是贯穿整个 Agent 系统的主干。

### 5.14 常见坑

坑一：把 `tool_result` 放进 assistant 消息。

工具结果应该作为 user role 内容返回。assistant 是请求工具的一方，不是报告工具执行结果的一方。

坑二：丢失 `tool_use.id`。

如果你执行工具时没有保存 ID，就无法生成正确 `tool_result.tool_use_id`。

坑三：工具失败时 throw。

任务级失败应该变成 `is_error: true` 的 tool_result。

坑四：把所有 content 当字符串。

一旦引入工具调用，content 就是 block 数组。渲染、日志、模型适配都要处理 block。

坑五：多个工具调用并发后结果顺序混乱。

即便并发执行，也要能正确按 ID 配对。Claude Code 的并发工具执行层会特别处理结果顺序和上下文修改。

### 5.15 本章练习

练习一：升级 `Message` 类型。

把上一章的字符串 content 改成 content blocks，并修复所有类型错误。

练习二：实现 `renderMessage()`。

要求：

- text block 打印文本。
- tool_use block 打印 `[tool_use: 工具名]`。
- tool_result block 打印 `[tool_result: ID]`。

练习三：实现 `extractToolUses()`。

要求只从 assistant 消息中提取工具调用。

练习四：让 FakeModelClient 返回工具调用。

当用户输入包含“读文件”时，返回 `read_file` 工具调用。

练习五：实现配对检查。

写一个函数找出没有结果的 tool_use。暂时只打印 warning，不中断程序。

### 5.16 本章小结

本章我们完成了从普通聊天消息到 Agent 消息协议的升级。你现在应该理解：

1. Agent 消息需要 content blocks。
2. `tool_use` 是 assistant 发出的动作请求。
3. `tool_result` 是外部环境返回给 assistant 的观察结果。
4. 两者必须通过 ID 配对。
5. 工具失败也应该作为 tool_result 回传。
6. mini-agent 已经准备好进入真正的工具执行阶段。

下一章我们会定义统一 Tool 接口，并实现第一个真正可执行的工具：读取文件。

---

# 第 2 卷：最小 Agent 实现

## 第 6 章：Tool 接口与第一个 Read 工具

### 6.1 本章目标

从这一章开始，我们的小程序会真正“动手”。第 5 章里，FakeModelClient 已经能返回一个 `tool_use` block，但 AgentEngine 还不会执行它。现在我们要补上工具系统的第一块拼图：统一 Tool 接口。

读完本章，你会完成：

1. 理解为什么所有工具都应该遵守同一个接口。
2. 学会用 Zod 定义工具参数 schema。
3. 实现 `Tool` 类型。
4. 实现工具注册表。
5. 实现 `runToolUse()`。
6. 实现第一个真实工具：`read_file`。
7. 把工具结果作为 `tool_result` 回到消息历史。
8. 对照 Claude Code 的 `src/Tool.ts` 和 `src/tools/FileReadTool/FileReadTool.ts` 理解工程级设计。

这一章是 Agent 从“会说话”到“会观察环境”的第一步。只要 `read_file` 跑通，后面的 `bash`、`write_file`、`grep`、`web_fetch` 本质上都是同一模式的扩展。

### 6.2 为什么需要统一 Tool 接口

先想一个最朴素的实现。模型返回：

```json
{
  "type": "tool_use",
  "id": "toolu_001",
  "name": "read_file",
  "input": {
    "path": "README.md"
  }
}
```

你当然可以直接写：

```ts
if (toolUse.name === "read_file") {
  const content = await fs.promises.readFile(toolUse.input.path, "utf8")
}
```

这在只有一个工具时能工作。但第二个工具来了怎么办？

```ts
if (toolUse.name === "read_file") {
  // ...
} else if (toolUse.name === "bash") {
  // ...
} else if (toolUse.name === "write_file") {
  // ...
} else if (toolUse.name === "grep") {
  // ...
}
```

很快你会遇到一堆重复问题：

1. 每个工具都要校验参数。
2. 每个工具都要处理错误。
3. 每个工具都要决定是否只读。
4. 每个工具都要决定是否能并发。
5. 每个工具都要生成给模型看的描述。
6. 每个工具都要把输出转换成字符串。
7. 每个工具都要被注册进工具列表。

如果没有统一接口，工具越多，代码越乱。最终你会得到一个巨大的 `switch`，里面混合了参数校验、权限、安全、执行、格式化和日志。

统一 Tool 接口的目标是：让 AgentEngine 不关心工具内部怎么实现。它只需要知道：

```text
根据名字找到工具 -> 校验 input -> 调用 tool.call -> 得到结果 -> 生成 tool_result
```

这和 Claude Code 的设计一致。源码里的 `src/Tool.ts` 定义了非常完整的 Tool 类型，而 `src/services/tools/toolExecution.ts` 只依赖这个统一接口执行工具。

### 6.3 一个工具应该回答哪些问题

新手版工具先回答六个问题：

1. 我叫什么？
2. 我给模型看的描述是什么？
3. 我的参数 schema 是什么？
4. 我是不是只读？
5. 我能不能和其他工具并发？
6. 调用我时怎么执行？

对应 TypeScript 类型：

```ts
import type { z } from "zod"

export type ToolContext = {
  cwd: string
}

export type Tool<Input, Output = string> = {
  name: string
  description: string
  inputSchema: z.ZodType<Input>
  isReadOnly(input: Input): boolean
  isConcurrencySafe(input: Input): boolean
  call(input: Input, context: ToolContext): Promise<Output>
  formatResult(output: Output): string
}
```

这里 `Input` 是工具参数类型，`Output` 是工具内部返回类型。

为什么还需要 `formatResult`？因为工具内部返回的数据结构不一定适合直接给模型。例如读取文件时，我们可能希望内部返回：

```ts
{
  path: "/abs/path/README.md",
  content: "...",
  lineCount: 120
}
```

但给模型的内容可以格式化成：

```text
File: /abs/path/README.md
Lines: 120

...
```

把执行和格式化拆开，会让后面处理大输出、截断、结构化结果更容易。

### 6.4 Claude Code 的 Tool 接口更复杂在哪里

Claude Code 的 `src/Tool.ts` 里，Tool 接口远比我们的教学版复杂。它除了 `call`、`inputSchema`、`isReadOnly` 这些核心字段，还包含：

- `prompt()`：给模型看的工具说明。
- `description()`：给用户或权限弹窗看的描述。
- `checkPermissions()`：工具自己的权限逻辑。
- `validateInput()`：schema 之外的业务校验。
- `mapToolResultToToolResultBlockParam()`：把工具输出转为 API tool_result。
- `renderToolUseMessage()`：终端 UI 渲染工具调用。
- `renderToolResultMessage()`：终端 UI 渲染工具结果。
- `getToolUseSummary()`：紧凑视图里的摘要。
- `isDestructive()`：是否破坏性操作。
- `isOpenWorld()`：是否访问开放外部世界。
- `preparePermissionMatcher()`：为 hook/权限规则准备匹配器。
- `toAutoClassifierInput()`：给安全分类器看的紧凑输入。

初学者不需要一开始实现这些。你只要明白一件事：复杂接口不是为了炫技，而是因为真实产品里工具同时服务于模型、用户、权限系统、UI、日志、测试、MCP 和子 Agent。

我们先实现最小接口，后面每章逐步加能力。

### 6.5 为什么需要 Zod

模型生成的工具参数不可信。这里的“不可信”不是说模型恶意，而是说它可能犯错。

例如工具 schema 要求：

```json
{
  "path": "README.md",
  "offset": 10,
  "limit": 20
}
```

模型可能生成：

```json
{
  "file": "README.md",
  "offset": "10",
  "limit": "twenty"
}
```

也可能生成：

```json
{
  "path": ["README.md"]
}
```

如果你直接把这些参数传给工具，工具内部会出现奇怪错误。更糟的是，对 Bash 或写文件工具来说，错误参数可能带来安全风险。

Zod 的作用是把“运行时输入”变成“经过验证的类型”。

比如：

```ts
import { z } from "zod"

const ReadFileInputSchema = z.object({
  path: z.string(),
  offset: z.number().int().nonnegative().optional(),
  limit: z.number().int().positive().optional(),
})

type ReadFileInput = z.infer<typeof ReadFileInputSchema>
```

校验：

```ts
const parsed = ReadFileInputSchema.safeParse(toolUse.input)

if (!parsed.success) {
  return toolError(toolUse.id, parsed.error.message)
}

const input = parsed.data
```

从 `parsed.data` 开始，TypeScript 知道 `input.path` 一定是 string，`offset` 如果存在就一定是非负整数。

Claude Code 中也使用 Zod。`src/Tool.ts` 的 Tool 类型中有 `inputSchema`，具体工具例如 `FileReadTool`、`BashTool` 都会定义自己的 schema。`src/services/tools/toolExecution.ts` 在执行工具前会先 `safeParse`，失败后返回 `InputValidationError` 给模型。

这就是工程级 Agent 的基本纪律：模型参数先校验，再执行。

### 6.6 创建 Tool.ts

在我们的 mini-agent 中，创建 `src/tools/Tool.ts`：

```ts
import type { z } from "zod"

export type ToolContext = {
  cwd: string
}

export type Tool<Input, Output = string> = {
  name: string
  description: string
  inputSchema: z.ZodType<Input>
  isReadOnly(input: Input): boolean
  isConcurrencySafe(input: Input): boolean
  call(input: Input, context: ToolContext): Promise<Output>
  formatResult(output: Output): string
}

export type AnyTool = Tool<any, any>
```

这里的 `AnyTool` 是为了工具注册表方便。严格来说，`any` 会损失类型精度，但工具注册表要存放不同 Input 类型的工具，很难用一个简单数组保持所有泛型。真实大型项目会用更复杂的类型技巧，Claude Code 的 `buildTool` 就做了很多类型层面的处理。

新手项目里，`AnyTool` 可以接受。关键是单个工具内部仍然有清晰 schema。

### 6.7 buildTool：给工具默认值

Claude Code 的 `buildTool` 很值得学习。它会给工具补上默认方法，例如默认 `isEnabled` 为 true，默认 `isConcurrencySafe` 为 false，默认 `isReadOnly` 为 false。

我们也可以做一个简化版：

```ts
import type { Tool } from "./Tool.js"

type ToolDef<Input, Output> = {
  name: string
  description: string
  inputSchema: Tool<Input, Output>["inputSchema"]
  call: Tool<Input, Output>["call"]
  formatResult?: Tool<Input, Output>["formatResult"]
  isReadOnly?: Tool<Input, Output>["isReadOnly"]
  isConcurrencySafe?: Tool<Input, Output>["isConcurrencySafe"]
}

export function buildTool<Input, Output = string>(
  def: ToolDef<Input, Output>,
): Tool<Input, Output> {
  return {
    name: def.name,
    description: def.description,
    inputSchema: def.inputSchema,
    call: def.call,
    formatResult: def.formatResult ?? ((output) => String(output)),
    isReadOnly: def.isReadOnly ?? (() => false),
    isConcurrencySafe: def.isConcurrencySafe ?? (() => false),
  }
}
```

为什么默认 `isReadOnly` 和 `isConcurrencySafe` 都是 false？

因为安全默认值要保守。如果一个工具没有明确声明自己只读，就把它当成可能写入。如果一个工具没有明确声明自己可并发，就串行执行。这样可能慢一点，但不容易出事故。

这和 Claude Code 的原则一致：不确定时 fail closed。

### 6.8 工具注册表

创建 `src/tools/registry.ts`：

```ts
import type { AnyTool } from "./Tool.js"

export class ToolRegistry {
  private tools = new Map<string, AnyTool>()

  register(tool: AnyTool) {
    if (this.tools.has(tool.name)) {
      throw new Error(`Tool already registered: ${tool.name}`)
    }

    this.tools.set(tool.name, tool)
  }

  get(name: string): AnyTool | undefined {
    return this.tools.get(name)
  }

  list(): AnyTool[] {
    return [...this.tools.values()]
  }
}
```

这个注册表做三件事：

1. 注册工具。
2. 根据名字查工具。
3. 列出所有工具给模型。

Claude Code 的 `src/tools.ts` 是更复杂的注册表。它不是简单 map，而是根据 feature flag、权限模式、MCP、环境变量、agent 类型动态组装工具池。但本质还是“当前会话有哪些工具可用”。

### 6.9 read_file 工具的需求

现在实现第一个工具。最小需求：

1. 输入 path。
2. 可选 offset 和 limit。
3. 只能读取当前工作目录下的文件。
4. 如果文件不存在，返回错误。
5. 如果是目录，返回错误。
6. 支持按行截取。
7. 返回文件路径、行数和内容。

为什么要限制当前工作目录？因为 Agent 不应该默认读取整个系统。即便是读操作，也可能读到密钥、浏览器数据、SSH key、系统配置。

Claude Code 的 FileReadTool 更复杂，它支持：

- 图片读取和压缩。
- PDF 页码范围。
- Notebook。
- 二进制检测。
- 大文件 token 限制。
- 设备文件阻止。
- 相似路径建议。
- 文件读取监听器。
- 权限规则。
- 技能触发。

我们先实现文本文件读取。

### 6.10 路径安全：不要相信模型给的 path

假设当前项目目录是：

```text
/Users/me/project
```

模型请求：

```json
{ "path": "../../.ssh/id_rsa" }
```

如果你直接 `readFile(path)`，就可能读取项目外的敏感文件。

因此要把路径解析为绝对路径，并检查它是否仍在 cwd 下：

```ts
import path from "node:path"

function resolveInsideCwd(cwd: string, inputPath: string): string {
  const resolved = path.resolve(cwd, inputPath)
  const relative = path.relative(cwd, resolved)

  if (relative.startsWith("..") || path.isAbsolute(relative)) {
    throw new Error(`Path is outside current workspace: ${inputPath}`)
  }

  return resolved
}
```

这个函数的意思是：

1. 用 `path.resolve` 得到绝对路径。
2. 计算它相对 cwd 的路径。
3. 如果相对路径以 `..` 开头，说明跑到 cwd 外面了。
4. 如果相对路径本身是绝对路径，也拒绝。

真实产品还要考虑 symlink、大小写文件系统、Windows 路径、额外授权目录等。Claude Code 的文件系统权限系统比这复杂得多。本章先掌握基本边界。

### 6.11 实现 read_file

创建 `src/tools/readFileTool.ts`：

```ts
import fs from "node:fs/promises"
import path from "node:path"
import { z } from "zod"
import { buildTool } from "./buildTool.js"
import type { ToolContext } from "./Tool.js"

const inputSchema = z.object({
  path: z.string(),
  offset: z.number().int().nonnegative().optional(),
  limit: z.number().int().positive().optional(),
})

type Input = z.infer<typeof inputSchema>

type Output = {
  path: string
  content: string
  startLine: number
  returnedLines: number
  totalLines: number
}

function resolveInsideCwd(cwd: string, inputPath: string): string {
  const resolved = path.resolve(cwd, inputPath)
  const relative = path.relative(cwd, resolved)

  if (relative.startsWith("..") || path.isAbsolute(relative)) {
    throw new Error(`Path is outside current workspace: ${inputPath}`)
  }

  return resolved
}

function sliceLines(
  text: string,
  offset = 0,
  limit?: number,
): {
  content: string
  startLine: number
  returnedLines: number
  totalLines: number
} {
  const lines = text.split(/\r?\n/)
  const selected = lines.slice(offset, limit ? offset + limit : undefined)

  return {
    content: selected.join("\n"),
    startLine: offset + 1,
    returnedLines: selected.length,
    totalLines: lines.length,
  }
}

async function readFile(input: Input, context: ToolContext): Promise<Output> {
  const filePath = resolveInsideCwd(context.cwd, input.path)
  const stat = await fs.stat(filePath)

  if (stat.isDirectory()) {
    throw new Error(`Path is a directory: ${input.path}`)
  }

  const text = await fs.readFile(filePath, "utf8")
  const sliced = sliceLines(text, input.offset, input.limit)

  return {
    path: filePath,
    ...sliced,
  }
}

export const readFileTool = buildTool<Input, Output>({
  name: "read_file",
  description: "Read a text file from the current workspace",
  inputSchema,
  isReadOnly: () => true,
  isConcurrencySafe: () => true,
  call: readFile,
  formatResult(output) {
    return [
      `File: ${output.path}`,
      `Lines: ${output.startLine}-${output.startLine + output.returnedLines - 1} of ${output.totalLines}`,
      "",
      output.content,
    ].join("\n")
  },
})
```

这个工具已经有了几个好习惯：

1. 用 schema 校验输入。
2. 限制路径在 cwd 内。
3. 区分目录和文件。
4. 支持 offset/limit。
5. 声明自己只读。
6. 声明自己可并发。
7. 内部输出和模型输出分离。

### 6.12 实现 runToolUse

现在我们需要把 `tool_use` block 变成 `tool_result` message。

创建 `src/tools/runToolUse.ts`：

```ts
import type { Message, ToolUseBlock } from "../agent/messages.js"
import { toolResult } from "../agent/messages.js"
import type { ToolContext } from "./Tool.js"
import type { ToolRegistry } from "./registry.js"

export async function runToolUse({
  toolUse,
  registry,
  context,
}: {
  toolUse: ToolUseBlock
  registry: ToolRegistry
  context: ToolContext
}): Promise<Message> {
  const tool = registry.get(toolUse.name)

  if (!tool) {
    return toolResult(
      toolUse.id,
      `Error: unknown tool: ${toolUse.name}`,
      true,
    )
  }

  const parsed = tool.inputSchema.safeParse(toolUse.input)

  if (!parsed.success) {
    return toolResult(
      toolUse.id,
      `InputValidationError: ${parsed.error.message}`,
      true,
    )
  }

  try {
    const output = await tool.call(parsed.data, context)
    return toolResult(toolUse.id, tool.formatResult(output), false)
  } catch (error) {
    const message = error instanceof Error ? error.message : String(error)
    return toolResult(toolUse.id, `Error: ${message}`, true)
  }
}
```

注意这段代码的错误处理策略：

1. 未知工具 -> 错误 tool_result。
2. 参数校验失败 -> 错误 tool_result。
3. 工具执行抛错 -> 错误 tool_result。

这些都是任务级错误，返回给模型，而不是让整个程序崩溃。

这正是 Claude Code 的 `toolExecution.ts` 思路。它会在找不到工具、Zod 校验失败、validateInput 失败、权限拒绝、call 抛错时生成模型可读的错误结果。

### 6.13 把工具注册进 CLI

在 `cli.ts` 或一个单独的 `createRegistry.ts` 中注册：

```ts
import { ToolRegistry } from "./tools/registry.js"
import { readFileTool } from "./tools/readFileTool.js"

export function createToolRegistry(): ToolRegistry {
  const registry = new ToolRegistry()
  registry.register(readFileTool)
  return registry
}
```

然后启动 Engine 时传进去：

```ts
const registry = createToolRegistry()
const engine = new AgentEngine({
  model: new FakeModelClient(),
  tools: registry,
  cwd: process.cwd(),
})
```

这意味着 `AgentEngine` 构造函数需要升级。

### 6.14 升级 AgentEngine

上一章的 Engine 只做：

```text
用户消息 -> 模型回复 -> 返回
```

现在要做：

```text
用户消息 -> 模型回复 -> 提取工具调用 -> 执行工具 -> 追加结果 -> 再次调用模型
```

先实现最简单版本：

```ts
import type { Message } from "./messages.js"
import { extractToolUses, userText } from "./messages.js"
import type { ModelClient } from "../model/client.js"
import type { ToolRegistry } from "../tools/registry.js"
import { runToolUse } from "../tools/runToolUse.js"

export class AgentEngine {
  private messages: Message[] = []

  constructor(
    private readonly options: {
      model: ModelClient
      tools: ToolRegistry
      cwd: string
    },
  ) {}

  async submit(userInput: string): Promise<Message> {
    this.messages.push(userText(userInput))

    while (true) {
      const response = await this.options.model.complete(this.messages)
      this.messages.push(response.message)

      const toolUses = extractToolUses(response.message)

      if (toolUses.length === 0) {
        return response.message
      }

      for (const toolUse of toolUses) {
        const resultMessage = await runToolUse({
          toolUse,
          registry: this.options.tools,
          context: {
            cwd: this.options.cwd,
          },
        })

        this.messages.push(resultMessage)
      }
    }
  }

  getMessages(): readonly Message[] {
    return this.messages
  }
}
```

这个循环已经是真正的 Agent 循环了。它会一直运行，直到模型返回不含 tool_use 的 assistant 消息。

### 6.15 防止无限循环

上面的循环有一个严重问题：如果模型一直调用工具，程序会永远跑下去。

比如 FakeModelClient 每次看到 tool_result 后又返回同一个 tool_use，就会无限循环。

所以必须加 `maxTurns`：

```ts
const maxTurns = 10
let turns = 0

while (true) {
  turns += 1

  if (turns > maxTurns) {
    throw new Error(`Agent exceeded max turns: ${maxTurns}`)
  }

  // call model...
}
```

但注意，这里 throw 是系统级保护。它不是某个工具失败，而是 Agent 循环失控。真实产品会更温和地返回一条 assistant error message 或中断状态。

Claude Code 的 `query.ts` 也支持 `maxTurns`。工程级 Agent 一定要有上限，包括：

- 最大轮数。
- 最大 token。
- 最大费用。
- 最大工具并发。
- 最大工具输出。
- 最大命令运行时间。

没有预算的 Agent 不是自动化助手，而是一台可能无限花钱和无限运行的机器。

### 6.16 让 FakeModel 完成一次读取流程

现在 FakeModel 要更聪明一点：

1. 用户说“读文件”时，返回 tool_use。
2. 收到 tool_result 后，返回最终文本。

示例：

```ts
function findLastToolResult(messages: Message[]) {
  for (let i = messages.length - 1; i >= 0; i--) {
    const message = messages[i]
    const result = message.content.find(block => block.type === "tool_result")
    if (result) return result
  }
}

export class FakeModelClient implements ModelClient {
  async complete(messages: Message[]): Promise<ModelResponse> {
    const lastToolResult = findLastToolResult(messages)

    if (lastToolResult) {
      return {
        message: assistantText(
          `我已经读取到文件内容，前 100 个字符是：\n${lastToolResult.content.slice(0, 100)}`,
        ),
      }
    }

    const last = messages[messages.length - 1]
    const text = last ? getText(last) : ""

    if (text.includes("读文件")) {
      return {
        message: {
          role: "assistant",
          content: [
            { type: "text", text: "我需要读取 README.md。" },
            {
              type: "tool_use",
              id: "fake_toolu_001",
              name: "read_file",
              input: { path: "README.md" },
            },
          ],
        },
      }
    }

    return {
      message: assistantText(`你刚才说：${text}`),
    }
  }
}
```

这不是完美模型，但它能模拟真实 Agent 的关键流程：

```text
用户请求 -> 模型发工具调用 -> 程序执行工具 -> 模型读结果 -> 模型回复
```

### 6.17 工具描述如何给模型

我们目前只是本地注册了工具，但真实模型需要知道工具有哪些、参数是什么、描述是什么。

在真实 API 请求中，你会把工具 schema 发给模型：

```json
{
  "name": "read_file",
  "description": "Read a text file from the current workspace",
  "input_schema": {
    "type": "object",
    "properties": {
      "path": { "type": "string" },
      "offset": { "type": "number" },
      "limit": { "type": "number" }
    },
    "required": ["path"]
  }
}
```

我们的 `Tool` 里已经有 `name`、`description` 和 `inputSchema`，但 Zod schema 不能直接发给模型。后面接真实 API 时，我们需要把 Zod 转成 JSON Schema，或者手写 JSON schema。

Claude Code 的 `src/utils/api.js` 中有 `toolToAPISchema` 一类逻辑，负责把内部 Tool 转成 API tool schema。MCP 工具则可能本身就提供 JSON Schema。

教学项目可以先手写：

```ts
export type ToolApiSchema = {
  name: string
  description: string
  input_schema: Record<string, unknown>
}
```

后面再逐步自动化。

### 6.18 与 Claude Code FileReadTool 对照

我们的 `read_file` 做了文本读取、路径限制、offset/limit。Claude Code 的 `FileReadTool` 做得更多。

它的输入 schema 包括：

- `file_path`：绝对路径。
- `offset`：起始行。
- `limit`：读取行数。
- `pages`：PDF 页码范围。

它还处理：

1. 图片文件：读取后转成模型可接收的 image block。
2. PDF：抽取指定页。
3. Notebook：按 cell 映射。
4. 二进制文件：避免按文本读取乱码。
5. 设备文件：阻止 `/dev/random` 等会卡住的路径。
6. token 限制：文件太大时要求 offset/limit。
7. 文件修改时间：辅助判断缓存是否过期。
8. 相似路径：文件不存在时给建议。
9. 权限检查：结合 filesystem permission rules。
10. 技能触发：读取特定文件后激活相关 skill。

为什么这么复杂？因为“读文件”在演示项目里是一行代码，在真实 coding Agent 里是高风险高频操作。

读文件可能带来：

- 隐私风险。
- 上下文爆炸。
- 程序卡死。
- 编码错误。
- 图片/PDF/Notebook 特殊格式。
- 路径误判。
- 权限越界。

这就是本书反复强调的：Agent 工程的复杂度来自边界情况。

### 6.19 常见坑

坑一：不校验路径范围。

模型可能生成 `../../secret`。即便模型不是恶意，用户也可能误导它。必须限制工作区。

坑二：直接读取超大文件。

一个 50MB 日志文件足以拖慢程序、撑爆上下文、浪费费用。后面我们会加入文件大小和 token 限制。

坑三：把工具内部错误当系统崩溃。

文件不存在应该作为 tool_result 返回给模型，而不是让 CLI 崩。

坑四：没有 maxTurns。

Agent 循环必须有上限。

坑五：把所有工具都当可并发。

Read 可以并发，但写文件、改目录、运行某些 shell 命令不一定可以。默认不要并发。

### 6.20 本章练习

练习一：实现 `Tool` 类型。

要求包含：

- `name`
- `description`
- `inputSchema`
- `call`
- `formatResult`
- `isReadOnly`
- `isConcurrencySafe`

练习二：实现 `buildTool()`。

要求：

- `formatResult` 默认 `String(output)`。
- `isReadOnly` 默认 false。
- `isConcurrencySafe` 默认 false。

练习三：实现 `ToolRegistry`。

要求：

- 支持注册工具。
- 支持按名字查找。
- 重复注册时报错。

练习四：实现 `read_file`。

要求：

- 只能读取 cwd 内的文本文件。
- 支持 offset/limit。
- 目录时报错。
- 文件不存在时返回错误 tool_result。

练习五：升级 `AgentEngine`。

要求：

- 提取 tool_use。
- 调用 `runToolUse()`。
- 把 tool_result 加入消息历史。
- 循环直到模型不再调用工具。
- 加 `maxTurns`。

### 6.21 本章小结

本章我们完成了 mini-agent 的第一个真实工具系统。现在它已经不只是聊天机器人，而是能通过工具读取工作区文件的 Agent。

你应该掌握：

1. 统一 Tool 接口能避免工具逻辑散落。
2. Zod 参数校验是工具执行前的必要步骤。
3. 工具失败应该以错误 tool_result 返回模型。
4. `read_file` 虽然简单，但已经涉及路径安全、文件类型、输出格式和上下文预算。
5. Claude Code 的 FileReadTool 是同一思想的产品级版本。

下一章我们会实现搜索工具。读取文件让 Agent 能看一个点，搜索工具让 Agent 能探索整个项目。

---

## 第 7 章：搜索工具，让 Agent 学会探索项目

### 7.1 本章目标

上一章我们实现了 `read_file`。这让 Agent 能读取一个已知路径的文件。但真实 coding 任务里，用户很少把所有路径都告诉 Agent。用户更常说：

```text
帮我找到登录逻辑在哪里。
```

或者：

```text
这个项目里哪里处理 API key？
```

或者：

```text
帮我分析为什么这个错误会出现。
```

这时 Agent 的第一步往往不是读取某个固定文件，而是搜索。它需要先发现项目结构，再定位相关代码，最后才读取具体文件。

本章要实现两个搜索能力：

1. `glob_files`：按文件名或路径模式找文件。
2. `grep_files`：按文本内容搜索代码。

读完本章，你会理解：

1. 为什么搜索工具是 coding Agent 的第二个核心工具。
2. glob 和 grep 的区别。
3. 为什么搜索结果必须限制数量和大小。
4. 如何把搜索工具做成只读、并发安全工具。
5. Claude Code 为什么同时有 `GlobTool`、`GrepTool`，也会在 `BashTool` 里识别 `rg`、`find`、`grep` 等命令。

### 7.2 读取文件与搜索文件的区别

`read_file` 的前提是：你已经知道路径。

比如：

```json
{
  "path": "src/auth/login.ts"
}
```

但搜索工具解决的是：你不知道路径，只知道线索。

例如：

```text
我想找所有包含 "login" 的文件。
```

或者：

```text
我想找所有 .tsx 文件。
```

这两类搜索不同。

第一类是路径搜索，也就是 glob：

```text
src/**/*.tsx
**/*auth*
**/*.test.ts
```

第二类是内容搜索，也就是 grep：

```text
login
process.env.API_KEY
function parseUser
```

在真实项目里，Agent 常常先 glob，再 grep，再 read：

```text
glob_files("src/**/*.ts")
  ↓
grep_files("login")
  ↓
read_file("src/auth/login.ts")
```

这是 coding Agent 最常见的探索路线。

### 7.3 为什么不用 Bash 直接跑 rg

既然 shell 里有 `find`、`grep`、`rg`，为什么还要单独做 `glob_files` 和 `grep_files` 工具？

有几个原因。

第一，专用工具更安全。

`grep_files` 只能搜索，不会修改文件。而 `bash` 可以做任何事。让模型用专用只读工具，权限系统更容易自动允许。

第二，专用工具的参数更结构化。

模型调用：

```json
{
  "pattern": "login",
  "include": "**/*.ts"
}
```

比生成 shell 命令：

```bash
rg "login" -g "**/*.ts"
```

更容易校验，也更容易跨平台。

第三，专用工具更容易控制输出。

搜索结果可能非常多。专用工具可以限制最多返回 100 条、每行最多 200 字符、总输出最多 20KB。Bash 命令如果不加限制，可能直接把巨大输出塞进上下文。

第四，专用工具更容易做 UI。

Claude Code 会把搜索/读取类工具折叠成更紧凑的显示。源码里 `BashTool` 甚至专门识别 `find`、`grep`、`rg`、`cat`、`head`、`ls` 等命令，判断它们是不是 search/read/list 操作。

所以最佳实践是：

```text
常见、安全、结构化的能力 -> 做成专用工具
开放、灵活、复杂的能力 -> 留给 Bash，但加权限
```

### 7.4 Claude Code 中的搜索工具

Claude Code 里和搜索相关的工具包括：

```text
src/tools/GlobTool/
src/tools/GrepTool/
src/tools/BashTool/BashTool.tsx
```

`GlobTool` 用于按路径模式找文件。

`GrepTool` 用于按内容搜索文件。

`BashTool` 中还有 `isSearchOrReadBashCommand()`，它会判断一个 Bash 命令是不是搜索、读取或列表操作。比如：

- `rg`
- `grep`
- `find`
- `cat`
- `head`
- `tail`
- `ls`
- `tree`

为什么 BashTool 也要识别？因为用户或模型仍然可能用 Bash 执行搜索。如果它能判断命令是只读搜索，就可以在 UI 上折叠显示，也可以在权限上更精细地处理。

我们的 mini-agent 会先实现专用工具，不急着让 Bash 承担搜索。

### 7.5 搜索工具的安全边界

搜索工具看起来是只读的，但仍然需要边界。

第一，路径必须限制在 cwd 内。

不能让 Agent 搜索整个 home 目录或系统目录。

第二，结果数量要限制。

如果一个项目有 10 万个文件，`glob_files("**/*")` 不能把所有路径都返回给模型。

第三，单条结果长度要限制。

内容搜索命中的一行可能非常长，例如 minified JS 或一整行 JSON。

第四，总输出大小要限制。

工具结果最终会进入模型上下文。搜索工具如果返回太多，会浪费 token，甚至导致 prompt too long。

第五，要尊重忽略规则。

真实项目通常应该忽略：

- `.git/`
- `node_modules/`
- `dist/`
- `build/`
- `.next/`
- 大型缓存目录

Claude Code 会更多依赖 `ripgrep` 等成熟工具来处理速度和忽略规则。教学项目先手写简化版，后面再讨论如何换成 `rg`。

### 7.6 glob_files 的输入设计

先设计 `glob_files`：

```ts
const inputSchema = z.object({
  pattern: z.string(),
  maxResults: z.number().int().positive().max(500).optional(),
})
```

字段含义：

`pattern` 是 glob 模式，例如 `src/**/*.ts`。

`maxResults` 是最多返回多少条，默认可以设为 100。

输出：

```ts
type Output = {
  pattern: string
  files: string[]
  truncated: boolean
}
```

`truncated` 表示结果是否因为数量限制被截断。这个字段很重要，因为模型看到截断后，可能会换更具体的搜索模式。

### 7.7 grep_files 的输入设计

再设计 `grep_files`：

```ts
const inputSchema = z.object({
  pattern: z.string(),
  include: z.string().optional(),
  maxResults: z.number().int().positive().max(500).optional(),
})
```

字段含义：

`pattern` 是要搜索的文本。

`include` 是可选的文件模式，例如 `**/*.ts`。

`maxResults` 限制命中数量。

输出：

```ts
type GrepMatch = {
  path: string
  line: number
  text: string
}

type Output = {
  pattern: string
  matches: GrepMatch[]
  truncated: boolean
}
```

这里先做普通字符串搜索，不做正则。原因很简单：新手版先降低复杂度。正则搜索有转义、性能、跨语言语法等问题，可以后面再加。

### 7.8 本章接下来要做什么

接下来几节会实现：

1. 忽略目录规则。
2. 文件遍历函数。
3. 简单 glob 匹配。
4. `glob_files` 工具。
5. `grep_files` 工具。
6. 搜索结果格式化。
7. AgentEngine 中并发执行多个只读搜索工具的准备工作。

搜索工具写完后，mini-agent 就能完成这样的流程：

```text
用户：找一下 README 在哪里
  ↓
模型：glob_files("**/README*")
  ↓
工具：返回 README.md
  ↓
模型：read_file("README.md")
  ↓
工具：返回文件内容
  ↓
模型：总结内容
```

这已经接近一个最小 coding Agent 的工作方式了。

### 7.9 忽略目录规则

搜索工具的第一步不是匹配，而是决定哪些目录根本不进入。

如果你递归遍历整个项目，很容易走进这些目录：

```text
.git/
node_modules/
dist/
build/
.next/
coverage/
.cache/
```

这些目录有几个问题。

第一，它们通常很大。`node_modules` 可能有几万个文件。

第二，它们大多不是用户要修改的源代码。搜索它们会产生大量噪音。

第三，它们可能包含生成物。Agent 如果读了生成物，容易误判真正源文件在哪里。

第四，搜索结果太多会撑爆上下文。

所以先定义忽略目录：

```ts
const DEFAULT_IGNORED_DIRS = new Set([
  ".git",
  "node_modules",
  "dist",
  "build",
  ".next",
  "coverage",
  ".cache",
  ".turbo",
  ".parcel-cache",
])
```

再写判断函数：

```ts
export function shouldIgnoreDir(name: string): boolean {
  return DEFAULT_IGNORED_DIRS.has(name)
}
```

真实项目里，忽略规则会更复杂。你可能需要读取 `.gitignore`，还要支持用户配置、插件缓存排除、隐藏目录策略等。Claude Code 里有 `src/utils/git/gitignore.ts`、`src/utils/plugins/orphanedPluginFilter.js` 等相关工具，也会优先使用 ripgrep 这类成熟搜索工具来尊重 ignore 规则。

教学项目先手写一小组默认规则，足够说明思路。

### 7.10 文件遍历函数

搜索工具需要遍历工作区。我们写一个异步递归函数。

创建 `src/tools/search/walkFiles.ts`：

```ts
import fs from "node:fs/promises"
import path from "node:path"

const DEFAULT_IGNORED_DIRS = new Set([
  ".git",
  "node_modules",
  "dist",
  "build",
  ".next",
  "coverage",
  ".cache",
  ".turbo",
  ".parcel-cache",
])

function shouldIgnoreDir(name: string): boolean {
  return DEFAULT_IGNORED_DIRS.has(name)
}

export async function walkFiles({
  cwd,
  maxFiles = 20_000,
}: {
  cwd: string
  maxFiles?: number
}): Promise<{
  files: string[]
  truncated: boolean
}> {
  const files: string[] = []
  let truncated = false

  async function visit(dir: string) {
    if (files.length >= maxFiles) {
      truncated = true
      return
    }

    const entries = await fs.readdir(dir, { withFileTypes: true })

    for (const entry of entries) {
      if (files.length >= maxFiles) {
        truncated = true
        return
      }

      if (entry.isDirectory()) {
        if (shouldIgnoreDir(entry.name)) continue
        await visit(path.join(dir, entry.name))
        continue
      }

      if (entry.isFile()) {
        const absolute = path.join(dir, entry.name)
        const relative = path.relative(cwd, absolute)
        files.push(relative)
      }
    }
  }

  await visit(cwd)

  return {
    files,
    truncated,
  }
}
```

这段代码有几个重要点。

第一，返回相对路径。给模型看相对路径通常更清晰，也更方便后续 `read_file`。

第二，有 `maxFiles`。即使忽略了常见目录，也不能假设项目一定小。

第三，返回 `truncated`。这告诉模型“我没有看完全部文件”。

第四，只处理普通文件。symlink、特殊设备、socket 等先不处理。

后面你可以继续增强：

- 支持 `.gitignore`。
- 支持隐藏文件开关。
- 支持只遍历某个子目录。
- 支持按扩展名过滤。
- 支持遇到权限错误时跳过而不是失败。

### 7.11 简化 glob 匹配

严格实现 glob 并不简单。`**/*.ts`、`*.{ts,tsx}`、`!ignore`、字符范围等都有细节。

生产项目建议使用成熟库，例如：

- `fast-glob`
- `minimatch`
- `picomatch`
- `glob`

但为了教学，我们先实现一个非常简化的 glob：

```ts
function escapeRegExp(text: string): string {
  return text.replace(/[|\\{}()[\]^$+?.]/g, "\\$&")
}

export function globToRegExp(pattern: string): RegExp {
  let source = ""
  let i = 0

  while (i < pattern.length) {
    const char = pattern[i]
    const next = pattern[i + 1]

    if (char === "*" && next === "*") {
      source += ".*"
      i += 2
      continue
    }

    if (char === "*") {
      source += "[^/]*"
      i += 1
      continue
    }

    source += escapeRegExp(char)
    i += 1
  }

  return new RegExp(`^${source}$`)
}
```

这个版本支持：

- `*`：匹配单个路径段中的任意字符。
- `**`：跨目录匹配。

例如：

```text
*.md          -> README.md
src/*.ts      -> src/index.ts
src/**/*.ts   -> src/a/b/c.ts
**/README*    -> docs/README.zh.md
```

它不支持完整 glob 语法，但足够让新手理解搜索工具的核心。

### 7.12 实现 glob_files 工具

创建 `src/tools/globFilesTool.ts`：

```ts
import { z } from "zod"
import { buildTool } from "./buildTool.js"
import { walkFiles } from "./search/walkFiles.js"
import { globToRegExp } from "./search/globToRegExp.js"

const inputSchema = z.object({
  pattern: z.string(),
  maxResults: z.number().int().positive().max(500).optional(),
})

type Input = z.infer<typeof inputSchema>

type Output = {
  pattern: string
  files: string[]
  truncated: boolean
}

export const globFilesTool = buildTool<Input, Output>({
  name: "glob_files",
  description: "Find files in the current workspace by glob pattern",
  inputSchema,
  isReadOnly: () => true,
  isConcurrencySafe: () => true,
  async call(input, context) {
    const maxResults = input.maxResults ?? 100
    const all = await walkFiles({
      cwd: context.cwd,
      maxFiles: 20_000,
    })

    const regex = globToRegExp(input.pattern)
    const matched = all.files.filter(file => regex.test(file))
    const files = matched.slice(0, maxResults)

    return {
      pattern: input.pattern,
      files,
      truncated: all.truncated || matched.length > files.length,
    }
  },
  formatResult(output) {
    if (output.files.length === 0) {
      return `No files matched pattern: ${output.pattern}`
    }

    const lines = [
      `Pattern: ${output.pattern}`,
      `Matched files: ${output.files.length}${output.truncated ? " (truncated)" : ""}`,
      "",
      ...output.files,
    ]

    if (output.truncated) {
      lines.push("")
      lines.push("Results were truncated. Use a more specific pattern.")
    }

    return lines.join("\n")
  },
})
```

注意格式化结果中的提示：

```text
Results were truncated. Use a more specific pattern.
```

这是给模型看的，不只是给用户看的。模型看到这句话后，下一步可能会换成更具体的模式，比如从 `**/*.ts` 改成 `src/**/*.ts`。

这类“面向模型的错误和提示”是 Agent 工程里非常重要的写作方式。

### 7.13 grep_files 的文件读取策略

`grep_files` 需要读取很多文件。这里有几个风险：

第一，不能读取二进制文件。二进制内容转字符串可能乱码，也可能非常大。

第二，不能读取超大文件。比如日志、bundle、lockfile 都可能很大。

第三，不能让一个文件读取失败导致整个搜索失败。某个文件权限错误时，可以跳过。

先写一个简单判断：

```ts
const TEXT_FILE_EXTENSIONS = new Set([
  ".ts",
  ".tsx",
  ".js",
  ".jsx",
  ".json",
  ".md",
  ".txt",
  ".css",
  ".html",
  ".yml",
  ".yaml",
  ".toml",
  ".py",
  ".go",
  ".rs",
  ".java",
  ".c",
  ".cpp",
  ".h",
])

export function looksLikeTextFile(file: string): boolean {
  const ext = path.extname(file).toLowerCase()
  return TEXT_FILE_EXTENSIONS.has(ext)
}
```

这不是完美判断。没有扩展名的文本文件会被跳过，某些扩展名文件也可能是二进制。但教学项目可以先这样。

生产级 FileReadTool 会更谨慎。Claude Code 的 `FileReadTool` 会检测二进制扩展、图片扩展、PDF、Notebook，并做不同处理。

### 7.14 实现 grep_files 工具

创建 `src/tools/grepFilesTool.ts`：

```ts
import fs from "node:fs/promises"
import path from "node:path"
import { z } from "zod"
import { buildTool } from "./buildTool.js"
import { walkFiles } from "./search/walkFiles.js"
import { globToRegExp } from "./search/globToRegExp.js"

const inputSchema = z.object({
  pattern: z.string(),
  include: z.string().optional(),
  maxResults: z.number().int().positive().max(500).optional(),
})

type Input = z.infer<typeof inputSchema>

type GrepMatch = {
  path: string
  line: number
  text: string
}

type Output = {
  pattern: string
  matches: GrepMatch[]
  truncated: boolean
}

const MAX_FILE_BYTES = 512 * 1024
const MAX_LINE_CHARS = 240

function truncateLine(line: string): string {
  if (line.length <= MAX_LINE_CHARS) return line
  return `${line.slice(0, MAX_LINE_CHARS)}...`
}

function looksLikeTextFile(file: string): boolean {
  const ext = path.extname(file).toLowerCase()

  return new Set([
    ".ts",
    ".tsx",
    ".js",
    ".jsx",
    ".json",
    ".md",
    ".txt",
    ".css",
    ".html",
    ".yml",
    ".yaml",
    ".toml",
    ".py",
    ".go",
    ".rs",
    ".java",
    ".c",
    ".cpp",
    ".h",
  ]).has(ext)
}

export const grepFilesTool = buildTool<Input, Output>({
  name: "grep_files",
  description: "Search text in files in the current workspace",
  inputSchema,
  isReadOnly: () => true,
  isConcurrencySafe: () => true,
  async call(input, context) {
    const maxResults = input.maxResults ?? 100
    const includeRegex = input.include
      ? globToRegExp(input.include)
      : undefined

    const walked = await walkFiles({
      cwd: context.cwd,
      maxFiles: 20_000,
    })

    const matches: GrepMatch[] = []
    let truncated = walked.truncated

    for (const relativePath of walked.files) {
      if (matches.length >= maxResults) {
        truncated = true
        break
      }

      if (includeRegex && !includeRegex.test(relativePath)) {
        continue
      }

      if (!looksLikeTextFile(relativePath)) {
        continue
      }

      const absolutePath = path.join(context.cwd, relativePath)

      try {
        const stat = await fs.stat(absolutePath)
        if (stat.size > MAX_FILE_BYTES) continue

        const text = await fs.readFile(absolutePath, "utf8")
        const lines = text.split(/\r?\n/)

        for (let i = 0; i < lines.length; i++) {
          if (matches.length >= maxResults) {
            truncated = true
            break
          }

          const line = lines[i]!

          if (line.includes(input.pattern)) {
            matches.push({
              path: relativePath,
              line: i + 1,
              text: truncateLine(line.trim()),
            })
          }
        }
      } catch {
        continue
      }
    }

    return {
      pattern: input.pattern,
      matches,
      truncated,
    }
  },
  formatResult(output) {
    if (output.matches.length === 0) {
      return `No matches found for: ${output.pattern}`
    }

    const lines = [
      `Pattern: ${output.pattern}`,
      `Matches: ${output.matches.length}${output.truncated ? " (truncated)" : ""}`,
      "",
      ...output.matches.map(
        match => `${match.path}:${match.line}: ${match.text}`,
      ),
    ]

    if (output.truncated) {
      lines.push("")
      lines.push("Results were truncated. Use include or a more specific pattern.")
    }

    return lines.join("\n")
  },
})
```

这段代码仍然是教学版，但已经体现出搜索工具的几个工程原则：

1. 结果数量有限制。
2. 单行长度有限制。
3. 大文件跳过。
4. 非文本扩展跳过。
5. 单文件读取失败不影响整体搜索。
6. 搜索结果带路径和行号。
7. 截断时明确告诉模型。

### 7.15 注册搜索工具

更新 `createToolRegistry()`：

```ts
import { ToolRegistry } from "./tools/registry.js"
import { readFileTool } from "./tools/readFileTool.js"
import { globFilesTool } from "./tools/globFilesTool.js"
import { grepFilesTool } from "./tools/grepFilesTool.js"

export function createToolRegistry(): ToolRegistry {
  const registry = new ToolRegistry()
  registry.register(readFileTool)
  registry.register(globFilesTool)
  registry.register(grepFilesTool)
  return registry
}
```

现在 mini-agent 有三个只读工具：

```text
read_file
glob_files
grep_files
```

这是 coding Agent 的基本观察能力：

- `glob_files` 发现文件。
- `grep_files` 定位内容。
- `read_file` 深入阅读。

### 7.16 更新 FakeModel：模拟搜索流程

为了测试，我们继续增强 FakeModelClient。

当用户说“找 README”时，先调用 `glob_files`：

```ts
if (text.includes("找 README")) {
  return {
    message: {
      role: "assistant",
      content: [
        { type: "text", text: "我先搜索 README 文件。" },
        {
          type: "tool_use",
          id: "fake_toolu_glob_readme",
          name: "glob_files",
          input: {
            pattern: "**/README*",
            maxResults: 20,
          },
        },
      ],
    },
  }
}
```

当用户说“搜索 login”时，调用 `grep_files`：

```ts
if (text.includes("搜索 login")) {
  return {
    message: {
      role: "assistant",
      content: [
        { type: "text", text: "我会搜索包含 login 的代码。" },
        {
          type: "tool_use",
          id: "fake_toolu_grep_login",
          name: "grep_files",
          input: {
            pattern: "login",
            include: "**/*.ts",
            maxResults: 50,
          },
        },
      ],
    },
  }
}
```

收到 tool_result 后，FakeModel 可以简单总结：

```ts
if (lastToolResult) {
  return {
    message: assistantText(
      `工具返回结果如下：\n${lastToolResult.content}`,
    ),
  }
}
```

真实模型会根据结果决定是否继续读取文件。FakeModel 只是帮助我们验证链路。

### 7.17 搜索结果为什么要为模型写得清楚

工具结果不是给机器看的纯数据，也不是给用户看的最终报告。它介于两者之间：模型要读它、理解它、基于它继续行动。

所以搜索结果格式非常重要。

不好的结果：

```text
["a.ts","b.ts","c.ts"]
```

模型不知道这是完整结果还是截断结果，也不知道搜索模式是什么。

更好的结果：

```text
Pattern: **/README*
Matched files: 2

README.md
docs/README.zh.md
```

如果截断：

```text
Pattern: **/*.ts
Matched files: 100 (truncated)

...

Results were truncated. Use a more specific pattern.
```

这会引导模型下一步缩小搜索范围。

同理，grep 结果采用传统格式：

```text
src/auth/login.ts:42: export function login(...)
```

这种格式对人和模型都友好，而且模型可以直接提取路径和行号。

### 7.18 搜索工具与并发

`glob_files` 和 `grep_files` 都是只读工具，所以我们声明：

```ts
isReadOnly: () => true
isConcurrencySafe: () => true
```

这意味着未来如果模型一次请求多个搜索：

```text
grep_files("login")
grep_files("logout")
glob_files("src/**/*.ts")
```

Agent 可以并发执行它们。

上一章我们的 AgentEngine 还是串行执行：

```ts
for (const toolUse of toolUses) {
  const resultMessage = await runToolUse(...)
  this.messages.push(resultMessage)
}
```

先串行没问题，简单可靠。后面讲并发时，我们会借鉴 Claude Code 的 `toolOrchestration.ts`：

1. 连续的并发安全工具可以放进一批。
2. 不安全工具单独执行。
3. 并发执行也要保证 tool_result 能正确配对。

现在只需要理解：工具接口里提前放 `isConcurrencySafe`，是为了未来调度器能做决定。

### 7.19 和 Claude Code 的 Glob/Grep 对照

Claude Code 的搜索工具比我们的版本更接近生产需求。

`GlobTool` 的重点：

- 快速查找路径。
- 尊重项目忽略规则。
- 控制返回数量。
- 用于让模型发现候选文件。

`GrepTool` 的重点：

- 快速按内容搜索。
- 通常基于 ripgrep。
- 返回路径、行号和匹配内容。
- 避免返回过多上下文。

`BashTool` 里还会识别搜索/读取/list 命令：

```text
find
grep
rg
ag
ack
locate
which
whereis
cat
head
tail
wc
stat
file
strings
ls
tree
du
```

这说明 Claude Code 不只关心“能不能执行”，还关心“这个执行在语义上是什么”。如果一个 Bash 命令本质上是搜索，就可以用更紧凑的 UI 展示，也可以被权限系统更精细地理解。

我们的教学版还没有 BashTool，所以先在专用工具中表达这种语义。

### 7.20 常见坑

坑一：搜索整个项目不设上限。

这会让大项目直接卡住，或者把海量结果塞给模型。

坑二：搜索 `node_modules`。

绝大多数 coding 任务不需要读依赖源码。默认跳过。

坑三：grep 读取二进制文件。

可能乱码、报错或产生巨大输出。

坑四：返回结果没有路径和行号。

模型后续无法精确读取相关文件。

坑五：截断但不告诉模型。

模型会误以为结果完整，做出错误判断。

坑六：一开始就实现完整正则和完整 glob。

新手项目可以先做普通字符串和简化 glob，理解主流程后再换成熟库。

### 7.21 本章练习

练习一：实现 `walkFiles()`。

要求：

- 递归遍历 cwd。
- 忽略 `.git`、`node_modules`、`dist` 等目录。
- 返回相对路径。
- 支持 `maxFiles` 和 `truncated`。

练习二：实现 `globToRegExp()`。

要求支持：

- `*`
- `**`

不用支持完整 glob 语法。

练习三：实现 `glob_files`。

要求：

- 按 pattern 过滤文件。
- 默认最多返回 100 条。
- 截断时提示模型缩小范围。

练习四：实现 `grep_files`。

要求：

- 普通字符串搜索。
- 返回 `path:line:text`。
- 跳过大文件和非文本文件。
- 限制结果数量和单行长度。

练习五：增强 FakeModelClient。

要求：

- 输入“找 README”时调用 `glob_files`。
- 输入“搜索 login”时调用 `grep_files`。
- 收到 tool_result 后输出结果摘要。

### 7.22 本章小结

本章我们让 mini-agent 从“能读已知文件”升级为“能探索项目”。

现在它拥有三种观察能力：

1. `glob_files`：发现文件。
2. `grep_files`：定位内容。
3. `read_file`：深入阅读。

这三个工具组合起来，已经能支持很多真实 coding 任务的前半段：探索、定位、理解。

下一章我们会进入最危险也最强大的工具：Shell。它能运行测试、查看 git 状态、执行项目脚本，但也需要更严格的权限和安全边界。

---

## 第 8 章：Shell 工具，最强大也最危险的能力

### 8.1 本章目标

前两章我们实现了文件读取和搜索。Agent 现在能观察项目，但还不能验证项目。一个 coding Agent 如果不能运行测试、查看 git 状态、执行构建命令，它就很难真正完成“修复问题”。

Shell 工具就是为了解决这个问题。

它能做很多有用的事：

```bash
npm test
pnpm lint
git status
git diff
ls -la
cat package.json
rg "TODO"
```

但它也能做非常危险的事：

```bash
rm -rf .
curl bad.example/script.sh | sh
git push --force
npm publish
chmod -R 777 /
```

所以 Shell 工具是 Agent 工程里的分水岭。做得好，它让 Agent 拥有真实工作能力；做得差，它会让 Agent 变成高风险自动执行器。

本章目标：

1. 理解 Shell 工具为什么必要。
2. 理解 Shell 工具的风险类别。
3. 设计最小 `shell` 工具输入输出。
4. 实现命令执行、超时、输出截断。
5. 初步区分只读命令和写入命令。
6. 为下一章权限系统做准备。
7. 对照 Claude Code 的 `BashTool` 理解生产级复杂度。

### 8.2 为什么需要 Shell 工具

搜索和读取只能回答“代码现在是什么样”。Shell 可以回答“代码运行起来怎么样”。

举例：

用户说：

```text
修复测试失败。
```

一个没有 shell 的 Agent 只能读代码、猜测修改。它不能知道当前测试失败在哪里，也不能确认修复后是否通过。

有 shell 后，Agent 可以：

```text
运行测试 -> 读取错误 -> 定位文件 -> 修改代码 -> 再运行测试
```

这就是 coding Agent 的基本闭环。

再比如用户说：

```text
看看这个项目怎么启动。
```

Agent 可以：

```text
read_file package.json
shell npm run
read_file README.md
```

Shell 工具让 Agent 能利用项目已有工具链，而不是把所有能力都重写成专用工具。

### 8.3 Shell 工具为什么危险

Shell 的危险来自它的开放性。`read_file` 只能读文件，`grep_files` 只能搜索文本，但 shell 几乎可以做任何事。

风险一：删除或覆盖文件。

```bash
rm -rf src
echo "" > package.json
git checkout -- .
```

风险二：修改系统或权限。

```bash
chmod -R 777 .
sudo rm ...
```

风险三：网络执行。

```bash
curl https://example.com/install.sh | sh
wget ... && bash ...
```

风险四：发布或推送。

```bash
npm publish
git push --force
```

风险五：无限运行。

```bash
while true; do echo hi; done
npm run dev
tail -f log.txt
```

风险六：巨大输出。

```bash
cat huge.log
find /
yes
```

风险七：命令注入和 shell 解析复杂性。

比如：

```bash
echo safe && rm -rf src
cat file | xargs rm
VAR=1 git push
```

你不能只看命令开头是不是 `echo` 或 `cat`。Shell 是一种语言，有管道、重定向、变量、子命令、条件执行、别名、函数、通配符。生产级工具必须非常谨慎。

Claude Code 的 `BashTool` 之所以很复杂，就是因为它要处理这些风险。

### 8.4 先做最小安全版

本书不会一上来实现完整 Bash 安全解析。我们先做一个最小安全版：

1. 命令必须在 cwd 中运行。
2. 命令有超时。
3. stdout/stderr 有最大长度。
4. 默认只自动允许明显只读命令。
5. 非只读命令先返回“需要权限”，后续章节再实现交互确认。

这不是生产级安全，但比裸跑 shell 好很多。

本章的目标不是“让 Shell 工具完美安全”，而是建立结构：

```text
输入 schema
  ↓
命令分类
  ↓
权限决策
  ↓
执行命令
  ↓
超时/取消
  ↓
截断输出
  ↓
tool_result
```

后面我们会逐步增强权限和沙箱。

### 8.5 Shell 工具输入设计

先设计输入：

```ts
const inputSchema = z.object({
  command: z.string(),
  timeoutMs: z.number().int().positive().max(120_000).optional(),
})
```

字段含义：

`command` 是要执行的 shell 命令。

`timeoutMs` 是超时时间，最大限制 120 秒。

为什么要限制最大 timeout？因为模型可能请求：

```json
{ "timeoutMs": 999999999 }
```

这会让命令长时间占用进程。schema 不只是类型校验，也是安全边界。

后面可以加入：

```ts
description: z.string().optional()
runInBackground: z.boolean().optional()
```

Claude Code 的 BashTool 输入就更丰富，支持描述、超时、后台运行等。我们先保持最小。

### 8.6 Shell 工具输出设计

输出需要包含：

```ts
type Output = {
  command: string
  exitCode: number | null
  stdout: string
  stderr: string
  timedOut: boolean
  truncated: boolean
}
```

字段含义：

`command`：实际运行的命令。

`exitCode`：退出码。超时或信号终止时可能没有正常退出码。

`stdout`：标准输出。

`stderr`：错误输出。

`timedOut`：是否超时。

`truncated`：输出是否被截断。

为什么要把 `exitCode` 给模型？因为命令失败不一定是工具失败。比如 `npm test` 退出码 1，说明测试失败，这正是模型需要分析的信息。它应该作为普通工具结果返回，而不是 `is_error: true`。

工具执行本身失败，例如 shell 启动失败、cwd 不存在、权限系统拒绝，才更适合作为工具错误。

这是一个重要区别：

```text
命令退出码非零 -> 任务观察结果
工具无法执行命令 -> 工具错误
```

Claude Code 的 BashTool 也会解释命令结果，并把 stdout/stderr、exit code、是否 interrupted 等信息映射给模型。

### 8.7 只读命令的第一版判断

我们先定义一组明显只读命令：

```ts
const READ_ONLY_COMMANDS = new Set([
  "ls",
  "pwd",
  "cat",
  "head",
  "tail",
  "grep",
  "rg",
  "find",
  "wc",
  "git",
])
```

但 `git` 很特殊：

```bash
git status
git diff
git log
```

是只读。

```bash
git commit
git push
git reset --hard
```

不是只读。

所以最小版可以写：

```ts
function isObviouslyReadOnly(command: string): boolean {
  const trimmed = command.trim()

  if (trimmed === "") return false

  const [base, subcommand] = trimmed.split(/\s+/)

  if (base === "git") {
    return new Set(["status", "diff", "log", "show", "branch"]).has(
      subcommand ?? "",
    )
  }

  return new Set([
    "ls",
    "pwd",
    "cat",
    "head",
    "tail",
    "grep",
    "rg",
    "find",
    "wc",
    "stat",
    "file",
  ]).has(base ?? "")
}
```

这个函数很不完美。

例如：

```bash
cat README.md > copy.md
```

开头是 `cat`，但它写文件。

```bash
ls && rm -rf src
```

开头是 `ls`，但后半段删除。

所以我们必须明确告诉读者：这是教学版，不是生产级安全判断。下一章权限系统会进一步改进；后面讲 Bash 安全时还会专门讨论 shell 解析。

Claude Code 不会只靠这种简单判断。它有 `bashPermissions.ts`、`readOnlyValidation.ts`、`commandSemantics.ts`、`bashSecurity.ts` 等模块，并且会解析命令结构。

### 8.8 本章接下来要做什么

接下来几节会实现：

1. `shell` 工具的 Zod schema。
2. `execCommand()`：基于 Node `child_process` 执行命令。
3. 超时控制。
4. stdout/stderr 收集和截断。
5. `shellTool` 的 `call()`。
6. 非只读命令的临时拒绝策略。
7. 和 AgentEngine 的集成。

本章写完后，mini-agent 将能完成：

```text
用户：运行测试
  ↓
模型：shell("npm test")
  ↓
工具：执行命令并返回 stdout/stderr/exitCode
  ↓
模型：解释测试结果
```

这会让我们的 Agent 第一次具备“验证”能力。

### 8.9 为什么选择 spawn 而不是 exec

Node.js 里执行命令常见有两个选择：

```ts
import { exec, spawn } from "node:child_process"
```

`exec` 用起来很简单：

```ts
exec("npm test", (error, stdout, stderr) => {
  // ...
})
```

但它有一个特点：会把 stdout/stderr 缓存在内存里，等命令结束后一次性回调。对小命令没问题，对 Agent Shell 工具不够理想。

为什么？

第一，命令可能输出很多内容。

比如：

```bash
npm test
find .
cat large.log
```

如果全部缓存，可能占用大量内存。

第二，我们希望未来支持实时进度。

比如命令运行时不断输出测试进度，TUI 应该能显示。`spawn` 可以监听 data event，边输出边处理。

第三，我们需要更精细地截断输出。

当 stdout 超过上限时，我们可以停止继续积累，而不是等全部结束。

所以教学项目也采用 `spawn`。先不做实时 UI，但保留结构。

### 8.10 输出截断器

先写一个小工具，限制字符串最大长度。

创建 `src/utils/OutputAccumulator.ts`：

```ts
export class OutputAccumulator {
  private chunks: string[] = []
  private size = 0
  private didTruncate = false

  constructor(private readonly maxChars: number) {}

  append(text: string) {
    if (this.didTruncate) return

    const remaining = this.maxChars - this.size

    if (remaining <= 0) {
      this.didTruncate = true
      return
    }

    if (text.length > remaining) {
      this.chunks.push(text.slice(0, remaining))
      this.size += remaining
      this.didTruncate = true
      return
    }

    this.chunks.push(text)
    this.size += text.length
  }

  toString(): string {
    return this.chunks.join("")
  }

  get truncated(): boolean {
    return this.didTruncate
  }
}
```

这个类非常简单，但意义很大。工具输出必须有预算。无论是 shell、grep、web fetch，还是 MCP 工具，都可能返回巨大内容。如果你不限制，模型上下文、内存和成本都会失控。

Claude Code 中有更成熟的机制，比如：

- BashTool 有输出截断和大输出持久化。
- `toolResultStorage` 会把过大的工具结果保存到磁盘，只给模型预览。
- `applyToolResultBudget` 会在查询前进一步限制历史工具结果总量。

我们的 mini-agent 先把 stdout/stderr 截断到固定长度。

### 8.11 execCommand 第一版

创建 `src/tools/shell/execCommand.ts`：

```ts
import { spawn } from "node:child_process"
import { OutputAccumulator } from "../../utils/OutputAccumulator.js"

export type ExecCommandResult = {
  command: string
  exitCode: number | null
  stdout: string
  stderr: string
  timedOut: boolean
  truncated: boolean
}

export async function execCommand({
  command,
  cwd,
  timeoutMs,
  maxOutputChars = 20_000,
}: {
  command: string
  cwd: string
  timeoutMs: number
  maxOutputChars?: number
}): Promise<ExecCommandResult> {
  return await new Promise((resolve, reject) => {
    const stdout = new OutputAccumulator(maxOutputChars)
    const stderr = new OutputAccumulator(maxOutputChars)
    let timedOut = false
    let settled = false

    const child = spawn(command, {
      cwd,
      shell: true,
      stdio: ["ignore", "pipe", "pipe"],
    })

    const timer = setTimeout(() => {
      timedOut = true
      child.kill("SIGTERM")
    }, timeoutMs)

    child.stdout.setEncoding("utf8")
    child.stderr.setEncoding("utf8")

    child.stdout.on("data", chunk => {
      stdout.append(String(chunk))
    })

    child.stderr.on("data", chunk => {
      stderr.append(String(chunk))
    })

    child.on("error", error => {
      if (settled) return
      settled = true
      clearTimeout(timer)
      reject(error)
    })

    child.on("close", code => {
      if (settled) return
      settled = true
      clearTimeout(timer)

      resolve({
        command,
        exitCode: code,
        stdout: stdout.toString(),
        stderr: stderr.toString(),
        timedOut,
        truncated: stdout.truncated || stderr.truncated,
      })
    })
  })
}
```

这段代码做了几件事：

1. 在指定 cwd 中运行命令。
2. 使用 shell 执行命令。
3. 忽略 stdin，避免命令等待输入。
4. 收集 stdout/stderr。
5. 超时后发送 SIGTERM。
6. 截断输出。
7. 返回 exitCode、timedOut、truncated。

这里有一个重要选择：

```ts
shell: true
```

这让用户可以运行：

```bash
npm test && npm run lint
rg "foo" | head
```

但也意味着 shell 语法会生效，风险更高。生产系统要非常谨慎。Claude Code 的 BashTool 需要解析命令、分类命令、判断权限，就是因为 shell 是完整语言。

教学项目先保留 `shell: true`，但后面权限系统会默认拒绝非只读命令。

### 8.12 shell 工具 schema

创建 `src/tools/shellTool.ts`：

```ts
import { z } from "zod"
import { buildTool } from "./buildTool.js"
import { execCommand } from "./shell/execCommand.js"

const inputSchema = z.object({
  command: z.string().min(1),
  timeoutMs: z.number().int().positive().max(120_000).optional(),
})

type Input = z.infer<typeof inputSchema>

type Output = {
  command: string
  exitCode: number | null
  stdout: string
  stderr: string
  timedOut: boolean
  truncated: boolean
}
```

这里 `command` 使用 `.min(1)`，避免空命令。

`timeoutMs` 最大 120 秒。默认值可以放在 call 里：

```ts
const DEFAULT_TIMEOUT_MS = 30_000
```

### 8.13 第一版只读判断

在同一个文件中加入：

```ts
const READ_ONLY_COMMANDS = new Set([
  "ls",
  "pwd",
  "cat",
  "head",
  "tail",
  "grep",
  "rg",
  "find",
  "wc",
  "stat",
  "file",
])

const READ_ONLY_GIT_SUBCOMMANDS = new Set([
  "status",
  "diff",
  "log",
  "show",
  "branch",
])

function firstWords(command: string): string[] {
  return command.trim().split(/\s+/)
}

export function isObviouslyReadOnly(command: string): boolean {
  const [base, subcommand] = firstWords(command)

  if (!base) return false

  if (base === "git") {
    return READ_ONLY_GIT_SUBCOMMANDS.has(subcommand ?? "")
  }

  return READ_ONLY_COMMANDS.has(base)
}
```

再强调一次：这只是教学版。它不能识别重定向、管道、`&&`、`;` 等危险组合。

为了让教学版更保守，我们可以加一个粗略拒绝：

```ts
function containsShellOperator(command: string): boolean {
  return /[;&|<>`$]/.test(command)
}
```

然后：

```ts
export function isObviouslyReadOnly(command: string): boolean {
  if (containsShellOperator(command)) return false

  const [base, subcommand] = firstWords(command)
  // ...
}
```

这会拒绝很多本来安全的命令，例如：

```bash
rg foo | head
```

但教学版宁可保守。后面学完权限和 Bash 解析，再逐步放宽。

### 8.14 临时权限策略

我们还没有实现交互式权限确认，所以本章先做一个临时策略：

```text
只读命令：允许执行
非只读命令：返回错误 tool_result，提示需要权限系统
```

也就是说，如果模型调用：

```bash
npm test
```

严格说 `npm test` 不一定只读，因为测试脚本可以写文件、启动服务、访问网络。但它是 coding Agent 很常用的验证命令。

教学版可以有两个选择。

选择一：保守，拒绝 `npm test`，等权限系统完成后再允许。

选择二：把 `npm test` 当成需要用户明确允许的命令，但目前没有 UI，所以先拒绝。

本章选择保守策略。下一章权限系统实现后，用户可以确认执行。

### 8.15 实现 shellTool

完整工具：

```ts
const DEFAULT_TIMEOUT_MS = 30_000

export const shellTool = buildTool<Input, Output>({
  name: "shell",
  description: "Run a shell command in the current workspace",
  inputSchema,
  isReadOnly(input) {
    return isObviouslyReadOnly(input.command)
  },
  isConcurrencySafe(input) {
    return isObviouslyReadOnly(input.command)
  },
  async call(input, context) {
    if (!isObviouslyReadOnly(input.command)) {
      throw new Error(
        `Command requires permission and cannot run yet: ${input.command}`,
      )
    }

    return await execCommand({
      command: input.command,
      cwd: context.cwd,
      timeoutMs: input.timeoutMs ?? DEFAULT_TIMEOUT_MS,
      maxOutputChars: 20_000,
    })
  },
  formatResult(output) {
    const parts = [
      `Command: ${output.command}`,
      `Exit code: ${output.exitCode ?? "unknown"}`,
    ]

    if (output.timedOut) {
      parts.push("Timed out: true")
    }

    if (output.truncated) {
      parts.push("Output truncated: true")
    }

    if (output.stdout.trim()) {
      parts.push("")
      parts.push("STDOUT:")
      parts.push(output.stdout.trimEnd())
    }

    if (output.stderr.trim()) {
      parts.push("")
      parts.push("STDERR:")
      parts.push(output.stderr.trimEnd())
    }

    return parts.join("\n")
  },
})
```

注意这里的策略：

```ts
if (!isObviouslyReadOnly(input.command)) {
  throw new Error(...)
}
```

因为 `runToolUse()` 会捕获工具错误并返回错误 tool_result，所以模型会看到：

```text
Error: Command requires permission and cannot run yet: npm test
```

下一章加入权限系统后，这里不应该直接 throw，而应该交给 `checkPermission()`。

### 8.16 注册 shell 工具

更新工具注册：

```ts
import { shellTool } from "./tools/shellTool.js"

export function createToolRegistry(): ToolRegistry {
  const registry = new ToolRegistry()
  registry.register(readFileTool)
  registry.register(globFilesTool)
  registry.register(grepFilesTool)
  registry.register(shellTool)
  return registry
}
```

现在工具池是：

```text
read_file
glob_files
grep_files
shell
```

这已经很接近最小 coding Agent：

- 搜索文件。
- 读取文件。
- 查看状态。
- 运行只读命令。

真正写文件和运行测试还要等权限系统。

### 8.17 更新 FakeModel：模拟 shell 调用

让 FakeModel 支持：

```text
列文件
查看 git 状态
运行测试
```

示例：

```ts
if (text.includes("列文件")) {
  return {
    message: {
      role: "assistant",
      content: [
        { type: "text", text: "我会列出当前目录文件。" },
        {
          type: "tool_use",
          id: "fake_toolu_shell_ls",
          name: "shell",
          input: {
            command: "ls",
          },
        },
      ],
    },
  }
}

if (text.includes("git 状态")) {
  return {
    message: {
      role: "assistant",
      content: [
        { type: "text", text: "我会查看 git status。" },
        {
          type: "tool_use",
          id: "fake_toolu_shell_git_status",
          name: "shell",
          input: {
            command: "git status",
          },
        },
      ],
    },
  }
}

if (text.includes("运行测试")) {
  return {
    message: {
      role: "assistant",
      content: [
        { type: "text", text: "我想运行测试，但这需要权限。" },
        {
          type: "tool_use",
          id: "fake_toolu_shell_test",
          name: "shell",
          input: {
            command: "npm test",
          },
        },
      ],
    },
  }
}
```

运行结果应该是：

- `ls` 可以执行。
- `git status` 可以执行。
- `npm test` 会被拒绝，返回错误 tool_result。

这正好给下一章权限系统埋下伏笔。

### 8.18 命令退出码不是工具错误

这一点非常重要，再单独强调。

假设命令是：

```bash
grep "not-exist-pattern" README.md
```

`grep` 没找到时可能返回 exit code 1。这不是工具执行失败，而是命令结果。

再比如：

```bash
npm test
```

测试失败返回 exit code 1。这正是 Agent 需要观察的信息。

所以 `execCommand()` 不应该因为 exit code 非零就 reject。它应该正常 resolve：

```ts
resolve({
  exitCode: code,
  stdout,
  stderr,
})
```

只有这些情况才 reject：

- 无法启动进程。
- cwd 不存在。
- Node spawn 自身报错。

Claude Code 的 BashTool 也会把命令语义和工具错误区分开。它还会用 `interpretCommandResult` 判断某些命令的非零退出码是否应被解释为错误、成功或特殊状态。

### 8.19 输出截断的模型提示

如果输出被截断，模型应该知道。

我们的 `formatResult()` 里有：

```text
Output truncated: true
```

还可以更明确一点：

```ts
if (output.truncated) {
  parts.push(
    "Note: output was truncated. Run a more specific command if you need more detail.",
  )
}
```

这类提示会影响模型行为。

如果模型看到：

```text
Output truncated: true
```

它可能继续运行：

```bash
npm test -- parser.test.ts
```

或者：

```bash
grep -n "specific error" log.txt
```

这就是 Agent 工具结果设计的技巧：不仅要返回数据，还要帮助模型知道下一步怎么缩小范围。

### 8.20 和 Claude Code BashTool 对照

Claude Code 的 BashTool 比我们的版本复杂很多。它处理：

1. 命令 schema。
2. 默认超时和最大超时。
3. 后台运行。
4. shell 输出流式进度。
5. cwd 变化检测和限制。
6. 只读命令判断。
7. sed in-place edit 的特殊处理。
8. 沙箱判断。
9. 权限规则匹配。
10. Bash classifier。
11. git 操作追踪。
12. 输出截断。
13. 大输出持久化。
14. 图片输出。
15. 前台/后台任务管理。
16. 中断和取消。
17. UI 渲染。

为什么这么多？因为 shell 是 Agent 能力中最接近“任意代码执行”的部分。每一个小细节都可能影响安全和体验。

例如，Claude Code 会把搜索/读取命令识别出来，作为 collapsible display 的依据。它还会区分 `isReadOnly` 和 `isConcurrencySafe`。只读搜索可以并发，写入命令必须串行。

我们的 Shell 工具目前只是学习版。但它已经有几个正确方向：

- schema 校验。
- cwd 限制。
- 超时。
- 输出截断。
- 非只读默认拒绝。
- exit code 作为观察结果。

### 8.21 常见坑

坑一：用 `exec` 不限制输出。

大输出会占内存，也会撑爆上下文。

坑二：非零 exit code 直接 throw。

测试失败、grep 无匹配、lint 报错都可能是有用观察，不应该自动变成工具异常。

坑三：没有超时。

`npm run dev`、`tail -f`、死循环都会卡住 Agent。

坑四：用命令开头判断安全。

`cat file > other` 和 `ls && rm -rf src` 都能绕过简单判断。

坑五：没有权限系统就运行写命令。

Shell 工具应该默认保守。先拒绝，再逐步引入用户确认。

### 8.22 本章练习

练习一：实现 `OutputAccumulator`。

要求：

- 支持最大字符数。
- 超过后截断。
- 暴露 `truncated`。

练习二：实现 `execCommand()`。

要求：

- 使用 `spawn`。
- 支持 cwd。
- 支持 timeout。
- 收集 stdout/stderr。
- exit code 非零也 resolve。

练习三：实现 `shellTool`。

要求：

- schema 包含 `command` 和 `timeoutMs`。
- 只允许明显只读命令。
- 输出包含 command、exitCode、stdout、stderr、timedOut、truncated。

练习四：测试命令。

测试：

```bash
ls
git status
npm test
```

确认：

- `ls` 可运行。
- `git status` 可运行。
- `npm test` 暂时被拒绝。

练习五：思考安全问题。

下面命令为什么危险？

```bash
cat README.md > copy.md
ls && rm -rf src
curl https://example.com/install.sh | sh
```

### 8.23 本章小结

本章我们实现了 Shell 工具的最小安全版。它还不能运行所有命令，但已经建立了正确的工程结构：

1. 命令参数必须 schema 校验。
2. 命令必须在 cwd 内执行。
3. 命令必须有超时。
4. 输出必须截断。
5. 退出码是观察结果，不等于工具错误。
6. 非只读命令先拒绝，等待权限系统。

下一章我们就来实现权限系统，让 Agent 在遇到写入、测试、构建等命令时，可以询问用户，而不是只能拒绝。

---

## 第 9 章：权限系统，给 Agent 装上刹车

### 9.1 本章目标

到目前为止，mini-agent 有了四类工具：

```text
read_file
glob_files
grep_files
shell
```

前三个是只读工具。`shell` 则非常特殊：它既可以只读，也可以写入；既可以安全，也可以危险。上一章我们用一个临时策略处理它：明显只读就运行，不明显只读就拒绝。

这不够。

一个真正可用的 coding Agent 必须能在用户允许时运行测试、安装依赖、生成文件、执行脚本。它也必须能在危险操作前停下来，让用户决定。

这就是权限系统的作用。

本章目标：

1. 理解为什么权限系统是 Agent 的核心，不是附加功能。
2. 设计 `PermissionDecision`。
3. 给 Tool 接口加入 `checkPermission()`。
4. 实现三种权限模式：default、plan、bypass。
5. 实现 CLI 中的用户确认。
6. 把 Shell 工具接入权限系统。
7. 对照 Claude Code 的 `useCanUseTool` 和权限上下文。

### 9.2 权限系统解决什么问题

权限系统解决的是“模型想做”和“系统允许做”之间的边界。

模型可能想做：

```bash
npm test
```

这通常可以允许，但最好让用户知道。

模型可能想做：

```bash
rm -rf dist
```

这可能是合理清理，也可能误删。

模型可能想做：

```bash
git reset --hard
```

这非常危险，可能丢失用户未提交修改。

模型可能想做：

```bash
curl https://example.com/install.sh | sh
```

这涉及网络和执行远程代码，必须高度谨慎。

如果没有权限系统，你只有两个极端：

1. 什么都不让做，Agent 很弱。
2. 什么都自动做，Agent 很危险。

权限系统提供中间道路：

```text
明显安全 -> 自动允许
明显危险 -> 自动拒绝
不确定 -> 问用户
```

这就是 Agent 的刹车。

### 9.3 权限决策类型

先定义权限决策。

创建 `src/permissions/types.ts`：

```ts
export type PermissionDecision =
  | {
      behavior: "allow"
      reason?: string
    }
  | {
      behavior: "deny"
      reason: string
    }
  | {
      behavior: "ask"
      reason: string
    }
```

三种行为：

`allow`：允许执行。

`deny`：拒绝执行，不询问用户。

`ask`：需要用户确认。

为什么要有 `reason`？

因为权限系统必须可解释。用户看到“拒绝”或“询问”时，需要知道为什么。模型收到拒绝结果时，也需要知道下一步怎么做。

例如：

```ts
{
  behavior: "deny",
  reason: "Plan mode does not allow shell commands that may modify files."
}
```

这比简单返回 `false` 好得多。

Claude Code 的权限结果更复杂，会记录 decision reason、rule source、classifier、hook、permission prompt 等信息。我们先从最小可解释决策开始。

### 9.4 权限模式

再定义权限模式：

```ts
export type PermissionMode =
  | "default"
  | "plan"
  | "bypass"
```

三种模式：

`default`：只读工具自动允许，写入或不确定操作询问用户。

`plan`：计划模式。只允许观察，不允许修改和执行不确定命令。

`bypass`：绕过权限。所有操作允许。只应该在用户明确选择、受信环境中使用。

为什么需要 plan？

很多时候用户只是让 Agent 分析、规划、阅读代码，而不是立刻修改。计划模式可以防止模型在没有批准方案前直接动手。

为什么需要 bypass？

在某些自动化环境里，用户可能明确希望 Agent 不停询问，直接执行。但这是高风险模式，必须显式开启。

Claude Code 中权限模式更多，例如 default、acceptEdits、plan、bypassPermissions、auto 等。我们的三种模式足够覆盖教学项目。

### 9.5 PermissionContext

权限判断需要上下文。

创建：

```ts
export type PermissionContext = {
  mode: PermissionMode
}
```

后面可以扩展：

```ts
export type PermissionContext = {
  mode: PermissionMode
  alwaysAllow: string[]
  alwaysDeny: string[]
  cwd: string
}
```

Claude Code 的 `ToolPermissionContext` 非常丰富，包括：

- 当前模式。
- 额外工作目录。
- always allow rules。
- always deny rules。
- always ask rules。
- 是否可用 bypass。
- 是否 auto mode 可用。
- 后台 Agent 是否应避免权限弹窗。
- plan mode 前的模式。

这些都是从真实产品需求长出来的。我们先从 mode 开始。

### 9.6 给 Tool 接口加入 checkPermission

上一章的 Tool 接口没有权限方法。现在升级：

```ts
import type { PermissionContext, PermissionDecision } from "../permissions/types.js"

export type ToolContext = {
  cwd: string
  permission: PermissionContext
}

export type Tool<Input, Output = string> = {
  name: string
  description: string
  inputSchema: z.ZodType<Input>
  isReadOnly(input: Input): boolean
  isConcurrencySafe(input: Input): boolean
  checkPermission(
    input: Input,
    context: ToolContext,
  ): Promise<PermissionDecision>
  call(input: Input, context: ToolContext): Promise<Output>
  formatResult(output: Output): string
}
```

然后更新 `buildTool()` 默认值：

```ts
checkPermission:
  def.checkPermission ??
  (async (input) => {
    if (def.isReadOnly?.(input) ?? false) {
      return { behavior: "allow", reason: "Tool is read-only." }
    }

    return { behavior: "ask", reason: "Tool may modify state." }
  }),
```

注意：默认策略不能无脑 allow。只读允许，非只读询问。

这和前面讲的“安全默认值保守”一致。

### 9.7 通用权限判断

可以写一个通用函数：

```ts
export async function checkToolPermission<Input>({
  tool,
  input,
  context,
}: {
  tool: Tool<Input, unknown>
  input: Input
  context: ToolContext
}): Promise<PermissionDecision> {
  if (context.permission.mode === "bypass") {
    return {
      behavior: "allow",
      reason: "Bypass mode allows all tools.",
    }
  }

  if (context.permission.mode === "plan" && !tool.isReadOnly(input)) {
    return {
      behavior: "deny",
      reason: "Plan mode only allows read-only tools.",
    }
  }

  return await tool.checkPermission(input, context)
}
```

这里有两层：

第一层是全局模式。

第二层是工具自己的权限逻辑。

例如 `read_file` 在 default 模式下自动允许，因为它只读；`shell` 要根据命令判断。

Claude Code 也是类似分层，只是层数更多：

- permission mode。
- allow/deny/ask rules。
- tool-specific checkPermissions。
- hooks。
- classifier。
- interactive prompt。
- coordinator/swarm worker 特殊处理。

### 9.8 更新 runToolUse

现在 `runToolUse()` 在 call 前要检查权限：

```ts
const permission = await checkToolPermission({
  tool,
  input: parsed.data,
  context,
})

if (permission.behavior === "deny") {
  return toolResult(
    toolUse.id,
    `Permission denied: ${permission.reason}`,
    true,
  )
}

if (permission.behavior === "ask") {
  return toolResult(
    toolUse.id,
    `Permission required: ${permission.reason}`,
    true,
  )
}
```

这只是第一版。它还不会真的问用户，只会告诉模型需要权限。

下一节我们会让 CLI 提供确认能力。

### 9.9 为什么 ask 不能在工具内部直接 question

你可能想在 `shellTool.call()` 里直接写：

```ts
const answer = await rl.question("Allow command?")
```

不建议。

工具本身应该只描述“我需要权限”，不要知道 UI 怎么问用户。原因：

1. 交互式 CLI 可以问用户。
2. 非交互模式不能问，只能拒绝。
3. 未来 TUI 可能用弹窗。
4. SDK 模式可能把权限请求发给宿主应用。
5. 子 Agent 或后台任务可能不能显示 UI。

所以权限询问应该由运行环境处理，而不是工具内部处理。

Claude Code 的 `useCanUseTool.tsx` 正是这个思想。工具和通用权限逻辑先得出 allow/deny/ask，交互式场景再把 ask 放入确认队列，UI 渲染权限弹窗。

### 9.10 PermissionPrompter

在 mini-agent 中，我们可以定义一个简单接口：

```ts
export type PermissionPrompter = {
  ask(question: string): Promise<boolean>
}
```

CLI 实现：

```ts
import readline from "node:readline/promises"

export class CliPermissionPrompter implements PermissionPrompter {
  constructor(private readonly rl: readline.Interface) {}

  async ask(question: string): Promise<boolean> {
    const answer = await this.rl.question(`${question} [y/N] `)
    return answer.trim().toLowerCase() === "y"
  }
}
```

然后 AgentEngine 持有：

```ts
permissionPrompter?: PermissionPrompter
```

非交互模式可以不传，遇到 ask 就拒绝。

### 9.11 runToolUse 支持 ask

更新 `runToolUse()` 参数：

```ts
export async function runToolUse({
  toolUse,
  registry,
  context,
  permissionPrompter,
}: {
  toolUse: ToolUseBlock
  registry: ToolRegistry
  context: ToolContext
  permissionPrompter?: PermissionPrompter
}): Promise<Message> {
  // ...
}
```

处理 ask：

```ts
if (permission.behavior === "ask") {
  if (!permissionPrompter) {
    return toolResult(
      toolUse.id,
      `Permission required but no interactive prompt is available: ${permission.reason}`,
      true,
    )
  }

  const allowed = await permissionPrompter.ask(permission.reason)

  if (!allowed) {
    return toolResult(
      toolUse.id,
      `Permission denied by user: ${permission.reason}`,
      true,
    )
  }
}
```

如果用户允许，就继续执行工具。

这里还有一个产品问题：用户是否只允许这一次，还是以后都允许？Claude Code 支持更丰富的规则来源和持久化。mini-agent 先只做“一次性允许”。

### 9.12 shellTool 接入权限

上一章 shellTool 在 `call()` 里直接拒绝非只读命令。现在改成 `checkPermission()`：

```ts
export const shellTool = buildTool<Input, Output>({
  name: "shell",
  description: "Run a shell command in the current workspace",
  inputSchema,
  isReadOnly(input) {
    return isObviouslyReadOnly(input.command)
  },
  isConcurrencySafe(input) {
    return isObviouslyReadOnly(input.command)
  },
  async checkPermission(input) {
    if (isObviouslyReadOnly(input.command)) {
      return {
        behavior: "allow",
        reason: "Command appears to be read-only.",
      }
    }

    return {
      behavior: "ask",
      reason: `Allow shell command: ${input.command}`,
    }
  },
  async call(input, context) {
    return await execCommand({
      command: input.command,
      cwd: context.cwd,
      timeoutMs: input.timeoutMs ?? DEFAULT_TIMEOUT_MS,
      maxOutputChars: 20_000,
    })
  },
  formatResult(output) {
    // same as before
  },
})
```

现在 `npm test` 不会自动拒绝，而是 ask。CLI 用户输入 `y` 后就会执行。

### 9.13 plan 模式

如何让用户进入 plan 模式？先用 CLI 命令：

```text
/mode plan
/mode default
/mode bypass
```

AgentEngine 保存：

```ts
private permissionMode: PermissionMode = "default"

setPermissionMode(mode: PermissionMode) {
  this.permissionMode = mode
}
```

构造 ToolContext 时：

```ts
context: {
  cwd: this.options.cwd,
  permission: {
    mode: this.permissionMode,
  },
}
```

CLI 里处理：

```ts
if (text.startsWith("/mode ")) {
  const mode = text.slice("/mode ".length).trim()

  if (mode === "default" || mode === "plan" || mode === "bypass") {
    engine.setPermissionMode(mode)
    console.log(`Permission mode: ${mode}`)
    continue
  }

  console.log("Unknown mode. Use default, plan, or bypass.")
  continue
}
```

现在 plan 模式下，非只读工具会被拒绝。

### 9.14 bypass 模式的警告

`bypass` 很危险。不要悄悄开启。

CLI 可以要求用户明确输入：

```text
/mode bypass
```

并打印：

```text
Warning: bypass mode allows all tools without asking.
```

更严格一点，可以要求二次确认。教学项目先打印警告即可。

Claude Code 的 bypass permissions 也非常敏感。它会检查是否可用、是否被策略限制，也会有用户确认和配置迁移。

### 9.15 权限结果如何反馈给模型

当用户拒绝命令：

```bash
npm test
```

工具结果应该类似：

```text
Permission denied by user: Allow shell command: npm test
```

模型看到后应该停止尝试执行同一命令，转而解释：

```text
我无法运行测试，因为你拒绝了权限。你可以手动运行 npm test，或允许我执行该命令。
```

这就是为什么权限拒绝也要作为 tool_result 回到模型，而不是只在 UI 打印。

Claude Code 中权限拒绝会变成模型可见的工具结果，例如 rejected tool use message。这样模型可以调整计划。

### 9.16 常见坑

坑一：权限判断散落在工具 call 内部。

这样 UI、非交互模式、子 Agent 都不好统一处理。

坑二：ask 时工具自己读 stdin。

这会把工具和 CLI 绑定死，未来无法迁移到 TUI 或 SDK。

坑三：用户拒绝后不告诉模型。

模型会以为工具丢失或系统错误，可能重复尝试。

坑四：bypass 默认开启。

高风险。必须显式开启。

坑五：plan 模式仍允许写操作。

计划模式的价值就是先观察和规划，不应修改。

### 9.17 和 Claude Code 权限系统对照

Claude Code 的权限系统关键文件包括：

```text
src/Tool.ts
src/hooks/useCanUseTool.tsx
src/utils/permissions/
src/hooks/toolPermission/
src/components/permissions/
```

它比 mini-agent 多很多能力：

1. 多种权限模式。
2. allow/deny/ask 规则。
3. 规则来源区分：session、user settings、project settings、policy settings。
4. Bash classifier 自动判断。
5. hooks 可以参与权限。
6. 交互式 UI 队列。
7. coordinator 和 swarm worker 特殊处理。
8. 后台 Agent 避免权限弹窗。
9. 权限决定日志。
10. 企业策略限制。

但核心思想和我们一样：

```text
工具请求 -> 校验参数 -> 权限判断 -> allow/deny/ask -> 执行或返回拒绝结果
```

学会 mini-agent 的权限系统后，再看 Claude Code 的实现，就不会觉得它是魔法，只是把更多真实场景加进来了。

### 9.18 本章练习

练习一：定义 `PermissionDecision`。

支持：

- allow
- deny
- ask

练习二：给 Tool 加 `checkPermission()`。

默认策略：

- 只读工具 allow。
- 非只读工具 ask。

练习三：实现三种模式。

要求：

- default：按工具判断。
- plan：非只读 deny。
- bypass：全部 allow。

练习四：实现 CLI 确认。

当 shell 请求 `npm test` 时，询问用户：

```text
Allow shell command: npm test [y/N]
```

练习五：测试权限结果。

测试：

```text
/mode plan
运行测试

/mode default
运行测试

/mode bypass
运行测试
```

观察三种模式的不同结果。

### 9.19 本章小结

本章我们给 Agent 装上了第一版刹车。

你现在应该理解：

1. 权限系统不是附加功能，而是 Agent 安全运行的核心。
2. 权限结果应该是 allow、deny、ask，而不是简单 boolean。
3. 工具不应该自己处理 UI 询问。
4. plan/default/bypass 是三种基础权限模式。
5. 权限拒绝也要反馈给模型。

下一章我们会实现写文件和编辑文件工具。到那时，权限系统会真正发挥作用，因为写入工具默认必须询问用户。

---

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
    "path": "src/index.ts",
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
    "path": "src/index.ts",
    "oldString": "const port = 3000",
    "newString": "const port = 4000"
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

```ts
const inputSchema = z.object({
  path: z.string(),
  content: z.string(),
  overwrite: z.boolean().optional(),
})
```

字段含义：

`path`：目标路径。

`content`：完整文件内容。

`overwrite`：是否允许覆盖已有文件。

为什么需要 `overwrite`？

因为创建新文件和覆盖已有文件风险不同。默认情况下，如果文件已存在但模型没明确说 overwrite，就应该拒绝。

### 10.5 edit_file 输入设计

`edit_file` schema：

```ts
const inputSchema = z.object({
  path: z.string(),
  oldString: z.string(),
  newString: z.string(),
})
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

```ts
function resolveInsideCwd(cwd: string, inputPath: string): string
```

现在应该把它抽到公共文件：

```ts
// src/utils/paths.ts
import path from "node:path"

export function resolveInsideCwd(cwd: string, inputPath: string): string {
  const resolved = path.resolve(cwd, inputPath)
  const relative = path.relative(cwd, resolved)

  if (relative.startsWith("..") || path.isAbsolute(relative)) {
    throw new Error(`Path is outside current workspace: ${inputPath}`)
  }

  return resolved
}
```

读、写、编辑、搜索都应该复用同一个路径边界函数。不要每个工具手写一份，否则很容易出现不一致。

### 10.7 实现 write_file

创建 `src/tools/writeFileTool.ts`：

```ts
import fs from "node:fs/promises"
import path from "node:path"
import { z } from "zod"
import { buildTool } from "./buildTool.js"
import { resolveInsideCwd } from "../utils/paths.js"

const inputSchema = z.object({
  path: z.string(),
  content: z.string(),
  overwrite: z.boolean().optional(),
})

type Input = z.infer<typeof inputSchema>

type Output = {
  path: string
  created: boolean
  bytes: number
}

export const writeFileTool = buildTool<Input, Output>({
  name: "write_file",
  description: "Create or overwrite a text file in the current workspace",
  inputSchema,
  isReadOnly: () => false,
  isConcurrencySafe: () => false,
  async checkPermission(input) {
    return {
      behavior: "ask",
      reason: `Allow writing file: ${input.path}`,
    }
  },
  async call(input, context) {
    const filePath = resolveInsideCwd(context.cwd, input.path)

    let exists = false
    try {
      await fs.stat(filePath)
      exists = true
    } catch {
      exists = false
    }

    if (exists && !input.overwrite) {
      throw new Error(
        `File already exists: ${input.path}. Set overwrite: true to replace it.`,
      )
    }

    await fs.mkdir(path.dirname(filePath), { recursive: true })
    await fs.writeFile(filePath, input.content, "utf8")

    return {
      path: filePath,
      created: !exists,
      bytes: Buffer.byteLength(input.content, "utf8"),
    }
  },
  formatResult(output) {
    return [
      output.created ? "File created." : "File overwritten.",
      `Path: ${output.path}`,
      `Bytes: ${output.bytes}`,
    ].join("\n")
  },
})
```

注意：

1. `isReadOnly` 是 false。
2. `isConcurrencySafe` 是 false。
3. `checkPermission` 总是 ask。
4. 已存在文件默认不覆盖。
5. 自动创建父目录。

### 10.8 实现 edit_file

创建 `src/tools/editFileTool.ts`：

```ts
import fs from "node:fs/promises"
import { z } from "zod"
import { buildTool } from "./buildTool.js"
import { resolveInsideCwd } from "../utils/paths.js"

const inputSchema = z.object({
  path: z.string(),
  oldString: z.string().min(1),
  newString: z.string(),
})

type Input = z.infer<typeof inputSchema>

type Output = {
  path: string
  replacements: number
}

function countOccurrences(text: string, search: string): number {
  if (search === "") return 0

  let count = 0
  let index = 0

  while (true) {
    const found = text.indexOf(search, index)
    if (found === -1) break
    count += 1
    index = found + search.length
  }

  return count
}

export const editFileTool = buildTool<Input, Output>({
  name: "edit_file",
  description: "Replace an exact text snippet in a file",
  inputSchema,
  isReadOnly: () => false,
  isConcurrencySafe: () => false,
  async checkPermission(input) {
    return {
      behavior: "ask",
      reason: `Allow editing file: ${input.path}`,
    }
  },
  async call(input, context) {
    const filePath = resolveInsideCwd(context.cwd, input.path)
    const original = await fs.readFile(filePath, "utf8")
    const occurrences = countOccurrences(original, input.oldString)

    if (occurrences === 0) {
      throw new Error(`oldString was not found in file: ${input.path}`)
    }

    if (occurrences > 1) {
      throw new Error(
        `oldString appears ${occurrences} times in file. Use a more specific oldString.`,
      )
    }

    const updated = original.replace(input.oldString, input.newString)
    await fs.writeFile(filePath, updated, "utf8")

    return {
      path: filePath,
      replacements: 1,
    }
  },
  formatResult(output) {
    return [
      "File edited.",
      `Path: ${output.path}`,
      `Replacements: ${output.replacements}`,
    ].join("\n")
  },
})
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

```ts
replaceAll: boolean
```

但必须配合 diff 和用户确认。

### 10.10 注册写入工具

更新注册表：

```ts
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

```ts
if (text.includes("创建 hello 文件")) {
  return {
    message: {
      role: "assistant",
      content: [
        { type: "text", text: "我会创建 hello.txt。" },
        {
          type: "tool_use",
          id: "fake_toolu_write_hello",
          name: "write_file",
          input: {
            path: "hello.txt",
            content: "hello\n",
          },
        },
      ],
    },
  }
}
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
Allow editing file: src/index.ts
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

## 第 11 章：把工具串成完整任务闭环

### 11.1 本章目标

前面几章我们逐个实现了工具：

```text
read_file
glob_files
grep_files
shell
write_file
edit_file
```

如果只看单个工具，它们都不复杂。Agent 真正的价值在于把工具串起来，完成一个目标。

例如：

```text
用户：把 README 里的端口 3000 改成 4000
  ↓
Agent 搜索 README
  ↓
Agent 读取 README
  ↓
Agent 编辑 README
  ↓
Agent 报告完成
```

再复杂一点：

```text
用户：修复 login 测试
  ↓
Agent 搜索 login 测试
  ↓
Agent 读取测试和源码
  ↓
Agent 编辑源码
  ↓
Agent 运行测试
  ↓
Agent 根据结果继续修或报告完成
```

本章目标：

1. 理解完整 Agent 任务闭环。
2. 设计多轮工具调用的消息历史。
3. 改进 `AgentEngine`，支持多轮工具调用和 maxTurns。
4. 理解工具结果如何影响下一步。
5. 用 FakeModel 模拟一个完整任务。
6. 对照 Claude Code 的 `query.ts` 主循环。

### 11.2 完整闭环是什么

完整闭环不是“调用一个工具”，而是：

```text
目标
  ↓
观察
  ↓
判断
  ↓
行动
  ↓
验证
  ↓
继续或结束
```

对 coding Agent 来说：

观察可能是：

- 搜索文件。
- 读取文件。
- 查看 git diff。
- 运行测试。

行动可能是：

- 写文件。
- 编辑文件。
- 运行格式化命令。

验证可能是：

- 再次读取文件。
- 运行测试。
- 运行 lint。
- 查看 diff。

Agent 的主循环必须允许模型在每一步根据结果决定下一步。

### 11.3 消息历史示例

以“修改 README 端口”为例，消息历史可能是：

```text
user:
  把 README 里的端口 3000 改成 4000

assistant:
  我先查找 README 文件。
  tool_use glob_files {"pattern":"**/README*"}

user:
  tool_result glob_files:
  README.md

assistant:
  我读取 README.md。
  tool_use read_file {"path":"README.md"}

user:
  tool_result read_file:
  ... port 3000 ...

assistant:
  我将替换端口。
  tool_use edit_file {"path":"README.md","oldString":"port 3000","newString":"port 4000"}

user:
  tool_result edit_file:
  File edited.

assistant:
  已完成修改。
```

这里每个工具结果都会成为下一次模型调用的输入。模型不是靠隐藏状态记住工具返回，而是靠 messages 看到结果。

### 11.4 AgentEngine 当前的问题

我们之前的 Engine 已经有 while 循环，但还比较粗糙：

```ts
while (true) {
  const response = await model.complete(messages)
  messages.push(response.message)

  const toolUses = extractToolUses(response.message)

  if (toolUses.length === 0) {
    return response.message
  }

  for (const toolUse of toolUses) {
    const result = await runToolUse(...)
    messages.push(result)
  }
}
```

它还缺几个东西：

1. maxTurns。
2. 对连续工具调用的日志。
3. 对权限 prompter 的传递。
4. 对最终消息的渲染。
5. 对工具结果顺序的清晰保证。

先加 maxTurns 和 prompter。

### 11.5 改进 AgentEngine 配置

定义：

```ts
type AgentEngineOptions = {
  model: ModelClient
  tools: ToolRegistry
  cwd: string
  maxTurns?: number
  permissionPrompter?: PermissionPrompter
}
```

Engine：

```ts
export class AgentEngine {
  private messages: Message[] = []
  private permissionMode: PermissionMode = "default"

  constructor(private readonly options: AgentEngineOptions) {}

  setPermissionMode(mode: PermissionMode) {
    this.permissionMode = mode
  }

  async submit(userInput: string): Promise<Message> {
    this.messages.push(userText(userInput))

    const maxTurns = this.options.maxTurns ?? 10

    for (let turn = 1; turn <= maxTurns; turn++) {
      const response = await this.options.model.complete(this.messages)
      this.messages.push(response.message)

      const toolUses = extractToolUses(response.message)

      if (toolUses.length === 0) {
        return response.message
      }

      for (const toolUse of toolUses) {
        const resultMessage = await runToolUse({
          toolUse,
          registry: this.options.tools,
          context: {
            cwd: this.options.cwd,
            permission: {
              mode: this.permissionMode,
            },
          },
          permissionPrompter: this.options.permissionPrompter,
        })

        this.messages.push(resultMessage)
      }
    }

    return assistantText(
      `I stopped because the agent reached maxTurns (${maxTurns}).`,
    )
  }
}
```

这里有一个设计选择：达到 maxTurns 后返回 assistantText，而不是 throw。

为什么？

因为 maxTurns 是用户任务层面的停止，不一定是程序 bug。返回一条消息给用户更友好。

生产系统可能会更细：

- SDK 返回 structured status。
- UI 显示“已达到最大轮数”。
- transcript 记录停止原因。

Claude Code 的 `query.ts` 会返回 terminal reason，并且有更多预算控制。

### 11.6 工具调用顺序

当前 Engine 对多个工具调用串行执行：

```ts
for (const toolUse of toolUses) {
  await runToolUse(...)
}
```

这保证了顺序简单、上下文清晰。

如果模型一次请求：

```text
read_file A
read_file B
```

串行执行没问题，只是慢一点。

如果模型一次请求：

```text
edit_file A
read_file A
```

串行顺序就很重要。先编辑再读取，和先读取再编辑结果不同。

所以新手版先全部串行是合理的。后面讲并发时，再根据 `isConcurrencySafe` 分批。

Claude Code 的 `toolOrchestration.ts` 就是做这个事情：连续并发安全工具可以并行，不安全工具串行。

### 11.7 FakeModel 模拟完整任务

真实模型会自己根据工具结果决定下一步。FakeModel 没有智能，所以我们写一个小状态机模拟。

目标：用户输入：

```text
把 README 里的 3000 改成 4000
```

FakeModel 流程：

1. 如果还没有 glob 结果，调用 `glob_files`。
2. 如果有 glob 结果但还没 read，调用 `read_file`。
3. 如果有 read 结果但还没 edit，调用 `edit_file`。
4. 如果 edit 成功，输出完成。

为了实现这个，我们需要根据历史判断已经做过什么。

示例辅助函数：

```ts
function hasToolUse(messages: Message[], name: string): boolean {
  return messages.some(message =>
    message.content.some(
      block => block.type === "tool_use" && block.name === name,
    ),
  )
}

function lastToolResultText(messages: Message[]): string | undefined {
  for (let i = messages.length - 1; i >= 0; i--) {
    const message = messages[i]!
    const block = message.content.find(block => block.type === "tool_result")
    if (block?.type === "tool_result") {
      return block.content
    }
  }
}
```

然后：

```ts
if (text.includes("README") && text.includes("3000") && text.includes("4000")) {
  if (!hasToolUse(messages, "glob_files")) {
    return {
      message: {
        role: "assistant",
        content: [
          { type: "text", text: "我先查找 README 文件。" },
          {
            type: "tool_use",
            id: "toolu_demo_glob",
            name: "glob_files",
            input: { pattern: "**/README*", maxResults: 10 },
          },
        ],
      },
    }
  }

  if (!hasToolUse(messages, "read_file")) {
    return {
      message: {
        role: "assistant",
        content: [
          { type: "text", text: "我读取 README.md。" },
          {
            type: "tool_use",
            id: "toolu_demo_read",
            name: "read_file",
            input: { path: "README.md" },
          },
        ],
      },
    }
  }

  if (!hasToolUse(messages, "edit_file")) {
    return {
      message: {
        role: "assistant",
        content: [
          { type: "text", text: "我把 3000 替换成 4000。" },
          {
            type: "tool_use",
            id: "toolu_demo_edit",
            name: "edit_file",
            input: {
              path: "README.md",
              oldString: "3000",
              newString: "4000",
            },
          },
        ],
      },
    }
  }

  return {
    message: assistantText("README 已经修改完成。"),
  }
}
```

这个 FakeModel 很笨，但它模拟了多轮工具调用。

### 11.8 FakeModel 的局限

你可能已经发现一个问题：上面的 FakeModel 硬编码了 `README.md`。如果 glob 结果是 `docs/README.md`，它仍然会读 `README.md`。

真实模型会从 tool_result 里读取路径，再决定下一步。FakeModel 要做到这一点也可以，但会越来越像一个手写规则系统。

这提醒我们：FakeModel 的目的不是替代真实模型，而是测试 AgentEngine 的控制流。

它适合测试：

- tool_use 是否被执行。
- tool_result 是否追加。
- maxTurns 是否生效。
- 权限询问是否触发。

不适合测试：

- 模型是否能智能选择文件。
- 模型是否能正确修 bug。
- 模型是否能理解复杂错误。

### 11.9 真实模型接入时要注意什么

当你接真实模型时，最重要的是把工具 schema 传给模型。

模型需要知道：

- 工具名。
- 工具描述。
- 参数 JSON Schema。

否则它不知道该怎么调用。

教学项目目前的 Tool 使用 Zod schema。你可以：

1. 手写每个工具的 JSON Schema。
2. 使用 zod-to-json-schema 之类工具转换。
3. 为 Tool 增加 `inputJsonSchema` 字段。

Claude Code 的 Tool 支持 `inputSchema` 和 `inputJSONSchema`，MCP 工具可以直接提供 JSON Schema。

### 11.10 工具描述对模型行为的影响

工具描述不是文档装饰，而是模型决策的一部分。

不好的描述：

```text
Run stuff.
```

好的描述：

```text
Read a text file from the current workspace. Use this when you know the file path and need its contents.
```

再比如 `grep_files`：

```text
Search for a plain text pattern in workspace files. Returns file path, line number, and matching line. Use this before read_file when you know a symbol or string but not the file path.
```

模型会根据描述决定何时使用工具。

Claude Code 的工具都有 prompt/description，并且有些工具还有 searchHint，用于工具搜索。

### 11.11 完整任务中的权限体验

当完整任务进入写入阶段时，用户会看到：

```text
Allow editing file: README.md [y/N]
```

这时候用户输入 `y`，Agent 才执行 edit。

这就是人与 Agent 协作的关键时刻。Agent 不应该悄悄改文件；用户也不应该被每个只读操作打断。好的权限体验是：

```text
观察类操作自动执行
修改类操作请求确认
危险操作明确警告
```

Claude Code 在这方面做了大量 UI 工作，包括 diff 展示、权限弹窗、计划模式等。

### 11.12 本章练习

练习一：给 AgentEngine 加 maxTurns。

要求：

- 默认 10。
- 达到上限后返回 assistant 消息。
- 不要无限循环。

练习二：写一个 FakeModel 多步任务。

模拟：

```text
glob_files -> read_file -> edit_file -> final answer
```

练习三：测试权限。

在 default 模式下，确认 edit_file 会询问用户。

练习四：测试 plan 模式。

在 plan 模式下，同样任务应该在 edit_file 时被拒绝。

练习五：测试 maxTurns。

让 FakeModel 永远返回同一个 tool_use，确认 Engine 会停止。

### 11.13 本章小结

本章我们把工具串成了完整任务闭环。

你现在应该理解：

1. Agent 的价值来自多轮工具调用，而不是单个工具。
2. 消息历史是模型理解前一步工具结果的唯一依据。
3. maxTurns 是必要的安全预算。
4. FakeModel 适合测试控制流，不适合测试智能。
5. 工具描述会影响真实模型如何选择工具。

下一章我们会开始接入真实模型 API，让 mini-agent 从 FakeModel 走向真正的 LLM Agent。

---

## 第 12 章：接入真实模型 API

### 12.1 本章目标

到目前为止，我们一直使用 FakeModel。它帮助我们搭建了 Agent 骨架，但它不具备真实语言理解能力。真正的 Agent 需要接入模型 API，让模型自己根据用户目标和工具结果决定下一步。

本章目标：

1. 理解 ModelClient 抽象的价值。
2. 实现一个真实模型客户端。
3. 把内部 Message 转成 API messages。
4. 把 Tool 转成 API tools。
5. 处理模型返回的 text 和 tool_use。
6. 理解非流式和流式的区别。
7. 对照 Claude Code 的 `services/api/claude.ts`。

### 12.2 为什么先有 ModelClient 抽象

我们很早就定义了：

```ts
export type ModelClient = {
  complete(messages: Message[]): Promise<ModelResponse>
}
```

当时看起来有点多余。现在它的价值出现了。

FakeModelClient 和真实模型客户端都实现同一个接口：

```text
AgentEngine -> ModelClient
```

AgentEngine 不需要知道背后是真模型还是假模型。

这带来几个好处：

1. 测试时用 FakeModel。
2. 开发时可以切换模型供应商。
3. 失败时可以记录请求和响应。
4. 未来可以加入重试、fallback、成本统计，而不改 AgentEngine。

Claude Code 的 `src/query/deps.ts` 就是同样思想。`query()` 不直接写死所有依赖，而是可以注入 `callModel`、`microcompact` 等。

### 12.3 选择非流式先实现

真实产品通常使用 streaming，因为用户体验更好，工具调用也能更早开始。

但教学项目先用非流式。

原因：

1. 代码更短。
2. 更容易理解。
3. 工具调用块一次性返回，不需要拼接 partial input。
4. 方便调试。

等非流式跑通后，再学习流式。Claude Code 使用复杂的 streaming 逻辑，是为了性能和体验；新手不应该第一步就跳进去。

### 12.4 API 消息格式和内部消息格式

我们的内部 Message：

```ts
type Message = {
  role: "user" | "assistant" | "system"
  content: ContentBlock[]
}
```

API 也有 messages，但具体字段可能略有不同。我们需要写 mapper。

原则：

```text
内部结构不要直接等于外部 API 结构
```

为什么？

第一，外部 API 可能变化。

第二，你可能支持多个模型供应商。

第三，内部消息还有 UI、进度、系统事件等，不一定都发给模型。

Claude Code 有大量 mapper，例如 `src/utils/messages/mappers.js`、`normalizeMessagesForAPI` 等，就是为了处理内部消息和 API 消息之间的差异。

### 12.5 转换 text block

先写简单转换：

```ts
function toApiContent(blocks: ContentBlock[]) {
  return blocks.map(block => {
    if (block.type === "text") {
      return {
        type: "text",
        text: block.text,
      }
    }

    if (block.type === "tool_use") {
      return {
        type: "tool_use",
        id: block.id,
        name: block.name,
        input: block.input,
      }
    }

    if (block.type === "tool_result") {
      return {
        type: "tool_result",
        tool_use_id: block.tool_use_id,
        content: block.content,
        is_error: block.is_error,
      }
    }
  })
}
```

再转换 message：

```ts
function toApiMessages(messages: Message[]) {
  return messages
    .filter(message => message.role !== "system")
    .map(message => ({
      role: message.role,
      content: toApiContent(message.content),
    }))
}
```

system message 可以单独放到 API 的 `system` 参数里。

### 12.6 Tool 转 API Schema

我们的 Tool 目前只有 Zod schema，没有 JSON Schema。教学项目可以先给 Tool 增加一个字段：

```ts
inputJsonSchema: Record<string, unknown>
```

更新 Tool：

```ts
export type Tool<Input, Output = string> = {
  name: string
  description: string
  inputSchema: z.ZodType<Input>
  inputJsonSchema: Record<string, unknown>
  // ...
}
```

然后每个工具手写。

例如 read_file：

```ts
inputJsonSchema: {
  type: "object",
  properties: {
    path: {
      type: "string",
      description: "Path to the file, relative to the workspace root.",
    },
    offset: {
      type: "number",
      description: "Zero-based line offset.",
    },
    limit: {
      type: "number",
      description: "Maximum number of lines to read.",
    },
  },
  required: ["path"],
  additionalProperties: false,
}
```

转换：

```ts
function toApiTools(registry: ToolRegistry) {
  return registry.list().map(tool => ({
    name: tool.name,
    description: tool.description,
    input_schema: tool.inputJsonSchema,
  }))
}
```

手写 JSON Schema 有重复，但对新手清晰。后面可以用自动转换。

### 12.7 AnthropicModelClient 示例

下面是概念代码。实际 SDK 版本可能略有差异，按你安装的 SDK 文档调整。

```ts
import Anthropic from "@anthropic-ai/sdk"
import type { Message } from "../agent/messages.js"
import { assistantText } from "../agent/messages.js"
import type { ModelClient, ModelResponse } from "./client.js"
import type { ToolRegistry } from "../tools/registry.js"

export class AnthropicModelClient implements ModelClient {
  private client: Anthropic

  constructor(
    private readonly options: {
      apiKey: string
      model: string
      tools: ToolRegistry
      system: string
    },
  ) {
    this.client = new Anthropic({
      apiKey: options.apiKey,
    })
  }

  async complete(messages: Message[]): Promise<ModelResponse> {
    const response = await this.client.messages.create({
      model: this.options.model,
      max_tokens: 4096,
      system: this.options.system,
      messages: toApiMessages(messages) as any,
      tools: toApiTools(this.options.tools) as any,
    })

    return {
      message: fromApiMessage(response),
    }
  }
}
```

这里用了 `as any`，是为了避免教学代码被 SDK 具体类型淹没。真实项目应该认真对齐 SDK 类型。

### 12.8 fromApiMessage

把 API 响应转回内部 Message：

```ts
function fromApiMessage(response: any): Message {
  return {
    role: "assistant",
    content: response.content.map((block: any) => {
      if (block.type === "text") {
        return {
          type: "text",
          text: block.text,
        }
      }

      if (block.type === "tool_use") {
        return {
          type: "tool_use",
          id: block.id,
          name: block.name,
          input: block.input,
        }
      }

      throw new Error(`Unsupported response block: ${block.type}`)
    }),
  }
}
```

assistant 响应通常不会包含 `tool_result`，因为 tool_result 是客户端执行工具后再发回模型的。

### 12.9 system prompt 第一版

先写一个简短 system prompt：

```ts
const system = `
You are mini-agent, a careful coding assistant.

You can use tools to inspect and modify the current workspace.
Use read_file when you know the path.
Use glob_files when you need to find files by path.
Use grep_files when you need to search file contents.
Use shell for command-line inspection and validation.
Use write_file or edit_file only when changing files is necessary.

Do not guess file contents. Use tools.
When a tool result is truncated, narrow your search.
`
```

这个 prompt 传达了几个重要规则：

1. 不要猜文件内容。
2. 搜索、读取、shell、写入分别何时用。
3. 截断时缩小范围。

Claude Code 的 system prompt 远比这复杂，会动态加入上下文、工具说明、模式、环境信息、项目记忆等。但核心目标一样：指导模型如何安全有效地使用工具。

### 12.10 环境变量

CLI 中读取 API key：

```ts
const apiKey = process.env.ANTHROPIC_API_KEY

if (!apiKey) {
  console.error("Missing ANTHROPIC_API_KEY")
  process.exit(1)
}
```

然后：

```ts
const registry = createToolRegistry()

const model = new AnthropicModelClient({
  apiKey,
  model: "claude-sonnet-4-5",
  tools: registry,
  system,
})

const engine = new AgentEngine({
  model,
  tools: registry,
  cwd: process.cwd(),
  permissionPrompter: new CliPermissionPrompter(rl),
})
```

模型名请根据当前可用模型调整。真实项目里应该把模型放进配置，而不是写死。

Claude Code 有复杂模型选择逻辑，包括默认模型、用户指定模型、fallback model、fast mode、thinking config 等。mini-agent 先写死一个模型即可。

### 12.11 第一次真实运行

你可以试：

```text
总结 README
```

理想流程：

```text
assistant -> glob_files 或 read_file
tool -> 返回内容
assistant -> 总结
```

再试：

```text
搜索 login
```

理想流程：

```text
assistant -> grep_files
tool -> 返回命中
assistant -> 解释结果或继续 read_file
```

再试：

```text
创建 hello.txt
```

理想流程：

```text
assistant -> write_file
permission prompt -> y/N
tool -> 写入
assistant -> 报告完成
```

如果模型不调用工具，而是直接编造文件内容，说明 system prompt 或工具描述不够明确。

### 12.12 常见接入问题

问题一：模型不调用工具。

可能原因：

- tools 没传给 API。
- tool schema 格式不对。
- system prompt 没强调使用工具。
- 用户请求不需要工具。

问题二：模型调用工具但参数错误。

这是正常现象。schema 校验会返回错误 tool_result，模型下一轮可能修正。

问题三：tool_use 和 tool_result 配对错误。

检查 tool_use id 是否保存，tool_result 是否使用同一个 id。

问题四：API 报 messages 格式错误。

通常是 content block 格式不符合 SDK 类型，或 tool_result 放错 role。

问题五：上下文太长。

现在 mini-agent 还没有压缩。长任务可能很快遇到限制。后面会讲上下文工程。

### 12.13 和 Claude Code API 层对照

Claude Code 的 `src/services/api/claude.ts` 做的事情包括：

1. 内部消息转 API 消息。
2. 工具转 API schema。
3. 处理 prompt caching。
4. 处理 thinking。
5. 处理 beta headers。
6. 处理 streaming。
7. 处理 retry 和 fallback。
8. 处理 usage、cost、quota。
9. 处理 max output tokens。
10. 处理工具搜索和 deferred tools。

我们的 mini-agent 只做了最小 API 调用。但主线一样：

```text
internal messages + tools -> API request -> assistant content blocks -> internal message
```

理解这个主线后，再看 Claude Code 的 API 层，会发现复杂度来自产品需求，而不是核心概念不同。

### 12.14 本章练习

练习一：给 Tool 增加 `inputJsonSchema`。

为所有工具手写 JSON Schema。

练习二：实现 `toApiMessages()`。

支持：

- text
- tool_use
- tool_result

练习三：实现 `toApiTools()`。

把 ToolRegistry 转成 API tools。

练习四：实现真实 ModelClient。

用你选择的模型 SDK 完成一次非流式调用。

练习五：测试真实工具调用。

让模型完成：

```text
总结 README
```

确认它会调用 `read_file` 或 `glob_files`。

### 12.15 本章小结

本章我们把 mini-agent 从 FakeModel 推向真实模型。

你应该理解：

1. ModelClient 抽象让假模型和真模型可替换。
2. 内部消息需要映射到 API 消息。
3. Tool 需要映射到 API schema。
4. system prompt 会强烈影响模型是否正确使用工具。
5. 非流式先跑通，流式后优化。

下一章我们会加入流式响应，让用户不用等完整模型回复结束才看到输出。

---

## 第 13 章：流式响应，让 Agent 更像实时助手

### 13.1 本章目标

上一章我们实现了非流式模型调用。非流式的体验是：

```text
用户输入
  ↓
等待模型完整生成
  ↓
一次性显示结果
```

如果模型回答很短，这没问题。但 coding Agent 经常会：

- 思考较长时间。
- 输出较长解释。
- 生成工具调用。
- 等待工具执行。
- 再继续生成。

用户如果一直看不到反馈，会以为程序卡住了。

流式响应可以让模型边生成边显示：

```text
用户输入
  ↓
模型开始输出 token
  ↓
CLI 实时显示
  ↓
如果出现 tool_use，执行工具
  ↓
继续下一轮
```

本章目标：

1. 理解流式响应的价值。
2. 实现文本流式显示。
3. 理解工具调用流式输出为什么复杂。
4. 设计 mini-agent 的简化流式接口。
5. 对照 Claude Code 的 streaming 实现。

### 13.2 非流式的问题

非流式最大的问题是“沉默”。

例如用户说：

```text
分析这个项目的架构。
```

模型可能需要几秒甚至几十秒。如果 CLI 没有任何输出，用户不知道：

- 模型是否收到请求。
- 网络是否卡住。
- 程序是否崩溃。
- Agent 是否正在调用工具。

流式输出可以缓解这个问题。即使只显示：

```text
我先查看项目结构...
```

用户也会感觉系统活着。

Claude Code 作为终端工具，非常重视流式体验。它不仅流式显示文本，还流式显示工具进度、Bash 输出、权限状态、spinner、后台任务等。

### 13.3 最简单的流式接口

先定义模型流事件：

```ts
export type ModelStreamEvent =
  | {
      type: "text_delta"
      text: string
    }
  | {
      type: "message_complete"
      message: Message
    }
```

ModelClient 增加：

```ts
export type StreamingModelClient = {
  stream(messages: Message[]): AsyncGenerator<ModelStreamEvent>
}
```

为什么用 AsyncGenerator？

因为模型输出是一串异步事件：

```ts
for await (const event of model.stream(messages)) {
  // handle event
}
```

这和 Claude Code 的 `query()` 很像。`query()` 本身就是 async generator，会不断 yield stream event、message、tool result summary 等。

### 13.4 CLI 显示文本 delta

AgentEngine 可以先支持纯文本流：

```ts
for await (const event of model.stream(this.messages)) {
  if (event.type === "text_delta") {
    process.stdout.write(event.text)
  }

  if (event.type === "message_complete") {
    this.messages.push(event.message)
    finalMessage = event.message
  }
}

process.stdout.write("\n")
```

这已经能让用户看到模型实时输出。

但是工具调用还没处理。我们需要在 message_complete 后提取 tool_use，再执行工具。也就是说第一版流式仍然是：

```text
流式显示文本
  ↓
等完整 assistant message
  ↓
执行工具
```

这比非流式体验好，但还不是最高性能。

### 13.5 为什么工具调用流式更复杂

模型生成 tool_use 时，input JSON 可能不是一次到达。

流式事件可能类似：

```text
content_block_start: tool_use name=read_file id=...
input_json_delta: {"pa
input_json_delta: th":"READ
input_json_delta: ME.md"}
content_block_stop
```

客户端要把这些 delta 拼起来，等 JSON 完整后才能执行工具。

更复杂的是，有些系统会在工具 input 完整时就提前执行工具，而不是等整个 assistant 消息结束。Claude Code 的 `StreamingToolExecutor` 就是做这个优化。

它要处理：

- 工具块流式到达。
- 参数是否完整。
- 并发安全。
- 非并发工具排队。
- 工具进度立即 yield。
- streaming fallback 时丢弃旧结果。
- 用户中断时取消工具。

所以教学项目先不做“流式工具提前执行”。我们先做“文本流式 + message complete 后执行工具”。

### 13.6 实现 stream 后的 AgentEngine

可以给 Engine 增加一个方法：

```ts
async submitStreaming(userInput: string): Promise<Message> {
  this.messages.push(userText(userInput))

  const maxTurns = this.options.maxTurns ?? 10

  for (let turn = 1; turn <= maxTurns; turn++) {
    let finalMessage: Message | undefined

    for await (const event of this.options.model.stream(this.messages)) {
      if (event.type === "text_delta") {
        this.options.onTextDelta?.(event.text)
      }

      if (event.type === "message_complete") {
        finalMessage = event.message
      }
    }

    if (!finalMessage) {
      return assistantText("Model stream ended without a message.")
    }

    this.messages.push(finalMessage)

    const toolUses = extractToolUses(finalMessage)

    if (toolUses.length === 0) {
      return finalMessage
    }

    for (const toolUse of toolUses) {
      const resultMessage = await runToolUse({
        toolUse,
        registry: this.options.tools,
        context: {
          cwd: this.options.cwd,
          permission: { mode: this.permissionMode },
        },
        permissionPrompter: this.options.permissionPrompter,
      })

      this.messages.push(resultMessage)
    }
  }

  return assistantText(`I stopped because the agent reached maxTurns.`)
}
```

`onTextDelta` 可以由 CLI 传入：

```ts
onTextDelta(text) {
  process.stdout.write(text)
}
```

### 13.7 流式事件和最终消息都要保留

一个常见错误是：流式打印了文本，但没有保存完整 assistant message。

这会导致下一轮模型看不到上一轮回答，也看不到 tool_use。

正确做法：

```text
delta 用于 UI
完整 message 用于历史
```

Claude Code 也区分 stream event 和 message。UI 可以实时显示 partial，但 transcript 和下一轮 API 需要规范化后的完整消息。

### 13.8 工具进度也应该流式

模型文本可以流式，工具也应该流式。

例如 shell 命令：

```bash
npm test
```

可能持续输出：

```text
running test A
running test B
failed test C
```

如果工具要等全部结束才显示，用户体验仍然不好。

我们的 `execCommand()` 现在内部监听 stdout/stderr，但没有向外报告进度。可以扩展：

```ts
onOutput?: (chunk: string) => void
```

然后：

```ts
child.stdout.on("data", chunk => {
  stdout.append(String(chunk))
  onOutput?.(String(chunk))
})
```

Tool 接口也可以支持 progress callback。

Claude Code 的 Tool `call()` 支持 `onProgress`，工具执行层会把 progress message yield 给 UI。BashTool、AgentTool 等都会利用这个能力。

### 13.9 本章先不实现的高级能力

为了保持 mini-agent 可理解，本章不实现：

1. partial tool input JSON 拼接。
2. 工具提前执行。
3. thinking block 流式处理。
4. streaming fallback。
5. tombstone partial message。
6. 多工具流式并发。

这些都是 Claude Code 级别的能力。我们会在后面工程化章节讲原理。

### 13.10 和 Claude Code streaming 对照

Claude Code 的流式链路大致是：

```text
query.ts
  ↓
services/api/claude.ts queryModelWithStreaming
  ↓
Anthropic streaming API
  ↓
stream events
  ↓
assistant messages / tool_use blocks
  ↓
StreamingToolExecutor
  ↓
tool progress / tool_result
```

其中 `StreamingToolExecutor` 是关键优化。它让工具在模型输出期间就可以开始执行，从而减少总耗时。

举例：

```text
模型生成 tool_use read_file README.md
  ↓
客户端发现 tool_use 完整
  ↓
立刻读取文件
  ↓
模型可能还在输出后续文本
```

这对快速工具很有价值。但复杂度也很高，所以 mini-agent 先不做。

### 13.11 本章练习

练习一：定义 `ModelStreamEvent`。

支持：

- text_delta
- message_complete

练习二：给 ModelClient 增加 `stream()`。

FakeModel 可以模拟流式：把字符串拆成多个字符或词逐个 yield。

练习三：实现 `submitStreaming()`。

要求：

- delta 实时输出。
- complete message 保存进历史。
- message_complete 后执行工具。

练习四：给 shell exec 加 `onOutput`。

先只在 CLI 打印输出，不必进入消息历史。

练习五：比较非流式和流式体验。

运行同一个长回答，看用户等待感有什么不同。

### 13.12 本章小结

本章我们让 mini-agent 具备了实时响应的雏形。

你应该理解：

1. streaming 主要改善用户体验和延迟。
2. delta 用于 UI，完整 message 用于历史。
3. 工具调用流式比文本流式复杂得多。
4. 生产级 Agent 会流式显示模型输出和工具进度。
5. Claude Code 的 StreamingToolExecutor 是性能优化层，不是 Agent 最小必需层。

下一章我们会处理一个越来越明显的问题：上下文会不断增长。Agent 工作越久，messages 越长，最终一定需要压缩和预算。

---

## 第 14 章：上下文预算，避免 Agent 被历史淹没

### 14.1 本章目标

Agent 每一轮都会把新消息加入历史：

```text
user message
assistant message
tool_result
assistant message
tool_result
...
```

短任务没问题。长任务中，messages 会越来越长。工具结果尤其危险：一次 `read_file` 可能返回几百行，一次 `shell` 可能返回几千行，一次 `grep_files` 可能返回几百条命中。

最终会出现几个问题：

1. API 请求超过模型上下文限制。
2. 成本越来越高。
3. 延迟越来越大。
4. 模型被大量旧信息干扰。
5. 工具结果重复占用上下文。

本章目标：

1. 理解上下文窗口是什么。
2. 理解 token 预算为什么重要。
3. 实现粗略 token 估算。
4. 实现消息裁剪。
5. 实现工具结果截断和替换。
6. 理解自动压缩的基本思路。
7. 对照 Claude Code 的 compact、microcompact 和 tool result budget。

### 14.2 上下文窗口是什么

模型不是无限记忆。每次 API 调用都有上下文窗口限制。你发给模型的 system prompt、messages、tools、tool results，都要放进这个窗口。

如果超过限制，API 可能报错：

```text
prompt is too long
```

或者模型表现变差。

上下文窗口可以理解为模型本次“能看到的材料总量”。Agent 的挑战是：既要让模型看到足够信息，又不能把所有历史都无限塞进去。

### 14.3 哪些内容占上下文

Agent 请求里通常包含：

1. system prompt。
2. 工具 schema。
3. 用户消息。
4. assistant 消息。
5. tool_use。
6. tool_result。
7. 项目上下文。
8. 记忆。
9. 压缩摘要。

初学者常常只关注用户消息，忽略工具 schema 和工具结果。实际上工具结果可能是最大来源。

比如：

```text
read_file package-lock.json
```

如果工具返回整个 lockfile，可能一下子占掉大量上下文。

所以 FileReadTool、BashTool、GrepTool 都必须限制输出。

### 14.4 粗略 token 估算

精确 token 计算需要 tokenizer。教学项目先用粗略估算：

```ts
export function estimateTokens(text: string): number {
  return Math.ceil(text.length / 4)
}
```

这是非常粗略的英文估算。中文、代码、JSON 都会有偏差。但作为预算保护，比完全没有好。

估算消息：

```ts
function messageToText(message: Message): string {
  return message.content
    .map(block => {
      if (block.type === "text") return block.text
      if (block.type === "tool_use") return JSON.stringify(block)
      if (block.type === "tool_result") return block.content
      return ""
    })
    .join("\n")
}

export function estimateMessageTokens(message: Message): number {
  return estimateTokens(messageToText(message))
}

export function estimateMessagesTokens(messages: Message[]): number {
  return messages.reduce(
    (sum, message) => sum + estimateMessageTokens(message),
    0,
  )
}
```

真实项目应该使用模型对应 tokenizer 或 API token count。Claude Code 里有 token estimation 服务和上下文 warning 逻辑。

### 14.5 最简单的裁剪策略

第一版策略：

```text
如果消息超过预算，只保留最近 N 条。
```

代码：

```ts
export function trimMessagesToRecent({
  messages,
  maxMessages,
}: {
  messages: Message[]
  maxMessages: number
}): Message[] {
  if (messages.length <= maxMessages) return messages
  return messages.slice(messages.length - maxMessages)
}
```

这个策略简单，但有危险：

1. 可能删掉任务目标。
2. 可能删掉工具调用对应结果。
3. 可能删掉重要文件内容。
4. 可能让模型失去上下文。

所以只能作为临时保护。

### 14.6 不能拆散 tool_use 和 tool_result

裁剪消息时，最重要的是不要留下半截工具调用。

错误历史：

```text
assistant: tool_use read_file id=1
```

但对应的：

```text
user: tool_result id=1
```

被裁掉了。

这会导致下一次 API 请求不合法或模型困惑。

所以裁剪时应该从边界开始，确保保留区域内没有未配对 tool_use。

简化策略：

```ts
export function hasUnresolvedToolUse(messages: Message[]): boolean {
  const toolUses = new Set<string>()
  const results = new Set<string>()

  for (const message of messages) {
    for (const block of message.content) {
      if (block.type === "tool_use") toolUses.add(block.id)
      if (block.type === "tool_result") results.add(block.tool_use_id)
    }
  }

  for (const id of toolUses) {
    if (!results.has(id)) return true
  }

  return false
}
```

裁剪后如果有 unresolved，就继续往前包含更多消息，直到配对完整。

Claude Code 中相关逻辑更复杂，因为还要处理 thinking blocks、compact boundary、tombstone 等。

### 14.7 工具结果预算

比裁剪整条消息更好的办法是限制工具结果。

比如一个 shell 结果：

```text
STDOUT:
... 20000 行 ...
```

我们可以保留前一部分，并提示：

```text
Tool result was truncated. Use a more specific command.
```

我们之前已经在各工具中做了局部截断。但长会话中，即便每次工具结果都不大，累计起来也会很大。

可以在每次模型调用前统一处理：

```ts
export function shrinkToolResults(
  messages: Message[],
  maxToolResultChars: number,
): Message[] {
  return messages.map(message => ({
    ...message,
    content: message.content.map(block => {
      if (block.type !== "tool_result") return block
      if (block.content.length <= maxToolResultChars) return block

      return {
        ...block,
        content:
          block.content.slice(0, maxToolResultChars) +
          "\n\n[Tool result truncated in history.]",
      }
    }),
  }))
}
```

这会损失信息，但能保护上下文。

Claude Code 的 `toolResultStorage` 更高级：过大的工具结果会保存到磁盘，模型看到预览和文件路径。如果需要完整内容，可以再用 Read 工具读取。

### 14.8 自动摘要压缩

更高级的做法是摘要旧消息。

流程：

```text
旧 messages
  ↓
调用小模型或当前模型总结
  ↓
生成 summary message
  ↓
保留 summary + 最近消息
```

示例 summary：

```text
Conversation summary:
- User asked to fix login tests.
- We found src/auth/login.ts and tests/login.test.ts.
- Test failed because expired tokens were treated as valid.
- We edited login.ts to reject expired tokens.
- Need to rerun npm test.
```

然后消息变成：

```text
user: Conversation summary...
最近 10 条消息
```

这比简单裁剪好，因为保留了任务脉络。

但摘要也有风险：

1. 摘要可能遗漏细节。
2. 摘要可能写错。
3. 摘要会消耗额外 API 成本。
4. 摘要不能破坏工具配对。

Claude Code 的 autoCompact 就是这类机制，但更复杂。它会根据 token warning state 触发，生成 summary messages，并处理 compact boundary。

### 14.9 mini-agent 的第一版 compact

先实现一个接口：

```ts
export type Summarizer = {
  summarize(messages: Message[]): Promise<string>
}
```

FakeSummarizer：

```ts
export class FakeSummarizer implements Summarizer {
  async summarize(messages: Message[]): Promise<string> {
    return `Summary of ${messages.length} earlier messages.`
  }
}
```

真实 Summarizer 可以调用模型。

compact：

```ts
export async function compactMessagesIfNeeded({
  messages,
  maxEstimatedTokens,
  keepRecent,
  summarizer,
}: {
  messages: Message[]
  maxEstimatedTokens: number
  keepRecent: number
  summarizer: Summarizer
}): Promise<Message[]> {
  const tokens = estimateMessagesTokens(messages)

  if (tokens <= maxEstimatedTokens) {
    return messages
  }

  const old = messages.slice(0, Math.max(0, messages.length - keepRecent))
  const recent = messages.slice(Math.max(0, messages.length - keepRecent))
  const summary = await summarizer.summarize(old)

  return [
    userText(`Conversation summary so far:\n${summary}`),
    ...recent,
  ]
}
```

这个版本不完美，但能展示结构。

### 14.10 compact 应该在什么时候运行

应该在调用模型前运行：

```ts
const messagesForModel = await compactMessagesIfNeeded({
  messages: this.messages,
  maxEstimatedTokens: 80_000,
  keepRecent: 20,
  summarizer,
})

const response = await model.complete(messagesForModel)
```

注意：内部完整 transcript 可以保留，发给模型的是 compact 后的 view。

Claude Code 也有类似区分：UI/会话可能保留更完整历史，但 API 请求前会投影、压缩、预算化。

### 14.11 上下文工程的原则

原则一：工具输出先限制。

不要等上下文爆了才处理。每个工具都应该控制输出。

原则二：历史请求前再预算。

每次 API 调用前检查当前 messages。

原则三：不要破坏协议。

tool_use/tool_result、thinking blocks、compact boundary 都要小心。

原则四：摘要不是事实源。

如果需要精确代码内容，应该重新读文件，而不是相信摘要。

原则五：告诉模型信息被截断或压缩。

模型要知道自己看到的不是完整历史。

### 14.12 和 Claude Code 对照

Claude Code 中上下文工程相关模块包括：

```text
src/query.ts
src/services/compact/autoCompact.ts
src/services/compact/microCompact.ts
src/services/compact/compact.ts
src/utils/toolResultStorage.ts
src/services/tokenEstimation.ts
src/memdir/
src/services/SessionMemory/
```

它的层次大致是：

1. 工具自己限制输出。
2. 大工具结果可持久化到磁盘。
3. query 前应用 tool result budget。
4. microcompact 做局部压缩。
5. autoCompact 在接近上下文限制时总结历史。
6. memory 系统保存长期信息。

这是工程级 Agent 必须有的能力。没有上下文工程，Agent 越工作越笨，越工作越贵，最后直接失败。

### 14.13 本章练习

练习一：实现 token 粗略估算。

用字符数除以 4。

练习二：实现 `shrinkToolResults()`。

要求：

- tool_result 超过上限时截断。
- 加入提示。

练习三：实现 `compactMessagesIfNeeded()`。

要求：

- 超过预算时总结旧消息。
- 保留最近 N 条。

练习四：确保不拆散工具配对。

写测试构造 tool_use/tool_result，检查裁剪后仍配对。

练习五：观察上下文增长。

打印每轮 estimated tokens，看长任务中增长趋势。

### 14.14 本章小结

本章我们给 mini-agent 加入了上下文预算意识。

你应该理解：

1. Agent 历史会无限增长，必须控制。
2. 工具结果是上下文膨胀的主要来源。
3. 简单裁剪有风险，不能破坏工具配对。
4. 摘要压缩能保留任务脉络，但可能遗漏细节。
5. Claude Code 的 compact 系统是产品级上下文工程。

下一章我们会讲会话持久化：如何把 messages 保存下来，让 Agent 能恢复之前的工作。

---

## 第 15 章：会话持久化与恢复

### 15.1 本章目标

到目前为止，mini-agent 的消息历史都在内存里。程序一退出，历史就消失。

这对玩具项目没问题，但对 coding Agent 不够。真实任务可能持续很久，用户可能中途关闭终端，或者希望回看 Agent 做过什么。我们需要把会话保存下来。

本章目标：

1. 理解 transcript 的作用。
2. 设计会话文件格式。
3. 实现消息保存。
4. 实现会话恢复。
5. 理解持久化和上下文压缩的关系。
6. 对照 Claude Code 的 session storage。

### 15.2 为什么需要 transcript

Transcript 可以理解为“会话记录”。

它记录：

- 用户说了什么。
- 模型回答了什么。
- 模型调用了哪些工具。
- 工具返回了什么。
- 哪些权限被允许或拒绝。
- 哪些文件被修改。

它有几个用途。

第一，恢复会话。

用户可以第二天继续昨天的任务。

第二，调试问题。

如果 Agent 做错了事，可以回看它为什么做出那个决定。

第三，审计安全。

写文件、运行命令、权限决定都应该可追溯。

第四，生成总结。

长任务可以从 transcript 中提取摘要。

Claude Code 会把会话保存在项目相关目录中，并支持 `/resume`、session recovery、agent transcript 等能力。

### 15.3 JSONL 格式

会话文件可以用 JSONL：一行一个 JSON 对象。

示例：

```jsonl
{"type":"message","message":{"role":"user","content":[{"type":"text","text":"总结 README"}]}}
{"type":"message","message":{"role":"assistant","content":[{"type":"tool_use","id":"1","name":"read_file","input":{"path":"README.md"}}]}}
{"type":"message","message":{"role":"user","content":[{"type":"tool_result","tool_use_id":"1","content":"..."}]}}
```

为什么用 JSONL？

1. 追加方便。
2. 文件损坏时可以恢复前面完整行。
3. 大会话不用每次重写整个 JSON 数组。
4. 很适合日志。

也可以用 JSON 数组，但每次追加都要读写整个文件，不适合长会话。

### 15.4 定义 TranscriptEvent

创建 `src/session/transcript.ts`：

```ts
import type { Message } from "../agent/messages.js"

export type TranscriptEvent =
  | {
      type: "message"
      message: Message
      timestamp: string
    }
  | {
      type: "permission"
      toolName: string
      decision: "allow" | "deny"
      reason: string
      timestamp: string
    }
```

先记录 message 和 permission。后面可以加入：

- tool_progress
- compact_boundary
- error
- cost
- model_request

Claude Code 的消息类型更丰富，也会记录 API usage、tool result replacement、agent metadata 等。

### 15.5 保存消息

创建 `TranscriptStore`：

```ts
import fs from "node:fs/promises"
import path from "node:path"
import type { Message } from "../agent/messages.js"
import type { TranscriptEvent } from "./transcript.js"

export class TranscriptStore {
  constructor(private readonly filePath: string) {}

  async append(event: TranscriptEvent): Promise<void> {
    await fs.mkdir(path.dirname(this.filePath), { recursive: true })
    await fs.appendFile(this.filePath, JSON.stringify(event) + "\n", "utf8")
  }

  async appendMessage(message: Message): Promise<void> {
    await this.append({
      type: "message",
      message,
      timestamp: new Date().toISOString(),
    })
  }
}
```

AgentEngine 每次 push message 时，也 append：

```ts
this.messages.push(message)
await this.options.transcript?.appendMessage(message)
```

为了避免到处手写，可以封装：

```ts
private async addMessage(message: Message) {
  this.messages.push(message)
  await this.options.transcript?.appendMessage(message)
}
```

然后所有地方都用 `addMessage()`。

### 15.6 恢复消息

读取 JSONL：

```ts
export async function loadMessages(filePath: string): Promise<Message[]> {
  let text: string

  try {
    text = await fs.readFile(filePath, "utf8")
  } catch {
    return []
  }

  const messages: Message[] = []

  for (const line of text.split(/\r?\n/)) {
    if (!line.trim()) continue

    try {
      const event = JSON.parse(line) as TranscriptEvent
      if (event.type === "message") {
        messages.push(event.message)
      }
    } catch {
      continue
    }
  }

  return messages
}
```

这里遇到坏行直接跳过。JSONL 的好处是，一行坏了不影响整个文件。

启动时：

```ts
const initialMessages = await loadMessages(sessionPath)

const engine = new AgentEngine({
  model,
  tools,
  cwd,
  initialMessages,
  transcript,
})
```

Engine 构造：

```ts
private messages: Message[]

constructor(options: AgentEngineOptions) {
  this.options = options
  this.messages = [...(options.initialMessages ?? [])]
}
```

### 15.7 会话路径设计

可以把会话存在：

```text
.mini-agent/sessions/<timestamp>.jsonl
```

例如：

```ts
function createSessionPath(cwd: string): string {
  const safeTime = new Date().toISOString().replace(/[:.]/g, "-")
  return path.join(cwd, ".mini-agent", "sessions", `${safeTime}.jsonl`)
}
```

也可以支持：

```bash
mini-agent --resume .mini-agent/sessions/xxx.jsonl
```

教学项目先手写路径即可。

真实产品要考虑：

- 多项目隔离。
- session id。
- 自定义标题。
- 最近会话列表。
- 删除和清理。
- 隐私和权限。

Claude Code 有项目路径映射、session id、resume conversation、agent sidechain transcript 等复杂机制。

### 15.8 持久化与 compact 的关系

一个重要设计：保存到 transcript 的应该是完整事件，发给模型的可以是压缩视图。

也就是说：

```text
transcript: 尽量完整，方便审计和恢复
model context: 预算化、压缩、裁剪
```

不要因为上下文压缩就把原始 transcript 删掉。否则用户无法回看完整过程。

Claude Code 也区分 UI/history/transcript 和 API context projection。压缩是为了发给模型，不等于历史消失。

### 15.9 恢复后的注意事项

恢复会话后，工具配对仍然要合法。

如果 transcript 最后停在：

```text
assistant: tool_use ...
```

但没有 tool_result，说明程序可能在工具执行中崩溃。

恢复时可以：

1. 自动补一个错误 tool_result。
2. 丢弃最后未完成 assistant message。
3. 提示用户会话不完整。

教学项目可以先检测：

```ts
const unresolved = findUnresolvedToolUses(messages)

if (unresolved.length > 0) {
  console.warn("Warning: session has unresolved tool uses.")
}
```

Claude Code 有更完整的 conversation recovery 逻辑，会处理不完整消息、工具结果、compact boundary 等。

### 15.10 权限记录

权限决定也应该记录。

当用户允许：

```text
Allow shell command: npm test
```

记录：

```json
{
  "type": "permission",
  "toolName": "shell",
  "decision": "allow",
  "reason": "Allow shell command: npm test",
  "timestamp": "..."
}
```

这对审计很有用。

本书当前 mini-agent 可以先只记录 message。后面安全章节再完善权限日志。

### 15.11 本章练习

练习一：实现 TranscriptStore。

要求：

- append JSONL。
- 自动创建目录。

练习二：Engine 使用 `addMessage()`。

要求所有消息写入内存时也写 transcript。

练习三：实现 `loadMessages()`。

要求：

- 读取 JSONL。
- 跳过坏行。
- 只恢复 message event。

练习四：实现 `--resume`。

从指定 session 文件恢复。

练习五：检测未完成工具调用。

恢复后如果存在 unresolved tool_use，打印 warning。

### 15.12 本章小结

本章我们让 mini-agent 有了记忆文件。

你应该理解：

1. Transcript 是 Agent 的审计日志。
2. JSONL 适合长会话追加。
3. 恢复会话要检查消息合法性。
4. Transcript 应该尽量完整，模型上下文可以压缩。
5. Claude Code 的 session storage 是同一思想的产品级版本。

下一章我们会讲测试。Agent 系统如果没有测试，很容易在工具协议、权限、上下文裁剪中悄悄出错。

---

## 第 16 章：测试 Agent，不要相信“看起来能跑”

### 16.1 本章目标

Agent 项目很容易出现一种错觉：你手动试了几个问题，模型回答不错，于是觉得系统没问题。

这很危险。

Agent 系统里有很多结构性约束：

- tool_use 和 tool_result 必须配对。
- 工具参数必须校验。
- 权限拒绝必须反馈给模型。
- plan 模式不能写文件。
- 输出截断必须提示模型。
- 会话恢复不能留下坏消息。
- maxTurns 必须阻止无限循环。

这些不应该靠手动测试。

本章目标：

1. 理解 Agent 测试和普通应用测试的区别。
2. 学会测试 Tool。
3. 学会测试 runToolUse。
4. 学会用 FakeModel 测试 AgentEngine。
5. 学会测试权限模式。
6. 学会测试上下文裁剪和会话恢复。
7. 对照 Claude Code 中依赖注入和测试友好设计。

### 16.2 Agent 测试的难点

普通函数测试通常是：

```text
输入 -> 输出
```

Agent 测试有几类特殊难点。

第一，模型不稳定。

真实模型同一个输入可能输出不同结果。不能让所有测试依赖真实模型。

第二，工具有副作用。

写文件、运行 shell、改目录都可能影响测试环境。

第三，消息历史复杂。

测试不只看最终回答，还要看中间是否调用了正确工具。

第四，权限有交互。

测试不能真的等人输入，需要 fake prompter。

第五，上下文裁剪可能破坏协议。

这类 bug 不一定立刻显现，但下一次 API 会失败。

所以测试策略要分层。

### 16.3 测试分层

建议分五层：

第一层：纯函数测试。

例如：

- `globToRegExp()`
- `estimateTokens()`
- `resolveInsideCwd()`
- `countOccurrences()`

第二层：工具测试。

例如：

- `read_file` 能读文件。
- `edit_file` oldString 不存在时报错。
- `shell` 超时能停止。

第三层：工具执行层测试。

例如：

- 未知工具返回错误 tool_result。
- 参数 schema 错误返回 InputValidationError。
- 权限 deny 返回错误 tool_result。

第四层：AgentEngine 测试。

用 FakeModel 模拟 tool_use，确认 Engine 会执行工具并追加结果。

第五层：持久化和恢复测试。

确认 transcript 能保存、加载，并检测坏消息。

### 16.4 测试工具：临时目录

文件工具测试不要在真实项目目录乱写。用临时目录。

```ts
import fs from "node:fs/promises"
import os from "node:os"
import path from "node:path"

export async function createTempWorkspace(): Promise<string> {
  return await fs.mkdtemp(path.join(os.tmpdir(), "mini-agent-test-"))
}
```

测试结束可以删除。注意删除命令要谨慎，确保路径是测试创建的临时目录。

### 16.5 read_file 测试

示例：

```ts
import fs from "node:fs/promises"
import path from "node:path"
import { readFileTool } from "../src/tools/readFileTool.js"
import { createTempWorkspace } from "./helpers.js"

test("read_file reads a text file", async () => {
  const cwd = await createTempWorkspace()
  await fs.writeFile(path.join(cwd, "README.md"), "hello\nworld\n")

  const output = await readFileTool.call(
    { path: "README.md" },
    { cwd, permission: { mode: "default" } },
  )

  expect(output.content).toContain("hello")
  expect(output.totalLines).toBe(3)
})
```

再测路径逃逸：

```ts
test("read_file rejects paths outside cwd", async () => {
  const cwd = await createTempWorkspace()

  await expect(
    readFileTool.call(
      { path: "../secret.txt" },
      { cwd, permission: { mode: "default" } },
    ),
  ).rejects.toThrow("outside current workspace")
})
```

这类测试非常重要。路径边界不应该靠人工记忆。

### 16.6 edit_file 测试

测试 oldString 不存在：

```ts
test("edit_file fails when oldString is missing", async () => {
  const cwd = await createTempWorkspace()
  await fs.writeFile(path.join(cwd, "a.txt"), "hello\n")

  await expect(
    editFileTool.call(
      {
        path: "a.txt",
        oldString: "missing",
        newString: "new",
      },
      { cwd, permission: { mode: "bypass" } },
    ),
  ).rejects.toThrow("oldString was not found")
})
```

测试 oldString 多次出现：

```ts
test("edit_file fails when oldString is not unique", async () => {
  const cwd = await createTempWorkspace()
  await fs.writeFile(path.join(cwd, "a.txt"), "x\nx\n")

  await expect(
    editFileTool.call(
      {
        path: "a.txt",
        oldString: "x",
        newString: "y",
      },
      { cwd, permission: { mode: "bypass" } },
    ),
  ).rejects.toThrow("appears 2 times")
})
```

这能防止误改多处。

### 16.7 runToolUse 测试

runToolUse 是协议关键层。

测试未知工具：

```ts
test("runToolUse returns error for unknown tool", async () => {
  const registry = new ToolRegistry()

  const result = await runToolUse({
    toolUse: {
      type: "tool_use",
      id: "1",
      name: "missing",
      input: {},
    },
    registry,
    context: {
      cwd: process.cwd(),
      permission: { mode: "default" },
    },
  })

  const block = result.content[0]
  expect(block.type).toBe("tool_result")
  expect(block.is_error).toBe(true)
})
```

测试 schema 错误：

```ts
test("runToolUse returns validation error", async () => {
  const registry = new ToolRegistry()
  registry.register(readFileTool)

  const result = await runToolUse({
    toolUse: {
      type: "tool_use",
      id: "1",
      name: "read_file",
      input: { path: 123 },
    },
    registry,
    context: {
      cwd: process.cwd(),
      permission: { mode: "default" },
    },
  })

  expect(result.content[0]?.type).toBe("tool_result")
})
```

这些测试保证模型生成坏参数时，Agent 不会崩。

### 16.8 FakePrompter

权限测试需要 fake prompter：

```ts
export class FakePermissionPrompter {
  constructor(private readonly decision: boolean) {}

  async ask(_question: string): Promise<boolean> {
    return this.decision
  }
}
```

测试用户允许：

```ts
const permissionPrompter = new FakePermissionPrompter(true)
```

测试用户拒绝：

```ts
const permissionPrompter = new FakePermissionPrompter(false)
```

这样测试不会卡住等待 stdin。

### 16.9 AgentEngine 测试

用 FakeModel 模拟：

```ts
class OneToolUseModel implements ModelClient {
  private called = false

  async complete(): Promise<ModelResponse> {
    if (!this.called) {
      this.called = true
      return {
        message: {
          role: "assistant",
          content: [
            {
              type: "tool_use",
              id: "1",
              name: "read_file",
              input: { path: "README.md" },
            },
          ],
        },
      }
    }

    return {
      message: assistantText("done"),
    }
  }
}
```

测试：

```ts
test("AgentEngine executes tool and continues", async () => {
  const cwd = await createTempWorkspace()
  await fs.writeFile(path.join(cwd, "README.md"), "hello")

  const registry = new ToolRegistry()
  registry.register(readFileTool)

  const engine = new AgentEngine({
    model: new OneToolUseModel(),
    tools: registry,
    cwd,
  })

  const final = await engine.submit("read README")

  expect(renderMessage(final)).toContain("done")
  expect(engine.getMessages().some(hasToolResult)).toBe(true)
})
```

这个测试不需要真实模型，却能验证 Agent 循环。

### 16.10 maxTurns 测试

模型永远返回工具调用：

```ts
class InfiniteToolModel implements ModelClient {
  async complete(): Promise<ModelResponse> {
    return {
      message: {
        role: "assistant",
        content: [
          {
            type: "tool_use",
            id: crypto.randomUUID(),
            name: "glob_files",
            input: { pattern: "**/*" },
          },
        ],
      },
    }
  }
}
```

测试 Engine 会停：

```ts
const engine = new AgentEngine({
  model: new InfiniteToolModel(),
  tools: registry,
  cwd,
  maxTurns: 3,
})

const final = await engine.submit("loop")
expect(renderMessage(final)).toContain("maxTurns")
```

这能防止无限循环回归。

### 16.11 上下文裁剪测试

构造一组消息：

```ts
const messages = [
  userText("start"),
  {
    role: "assistant",
    content: [
      { type: "tool_use", id: "1", name: "read_file", input: { path: "a" } },
    ],
  },
  toolResult("1", "content"),
]
```

测试裁剪后不留下 unresolved tool_use。

这类测试很容易被忽视，但非常关键。上下文裁剪 bug 往往不是当场报错，而是在下一次 API 请求时爆炸。

### 16.12 Transcript 测试

测试保存和恢复：

```ts
test("TranscriptStore saves and loads messages", async () => {
  const cwd = await createTempWorkspace()
  const file = path.join(cwd, "session.jsonl")
  const store = new TranscriptStore(file)

  await store.appendMessage(userText("hello"))

  const messages = await loadMessages(file)
  expect(messages).toHaveLength(1)
  expect(renderMessage(messages[0]!)).toContain("hello")
})
```

再测试坏行：

```ts
await fs.writeFile(file, "{bad json}\n")
const messages = await loadMessages(file)
expect(messages).toHaveLength(0)
```

### 16.13 不要把真实模型作为单元测试依赖

真实模型测试可以有，但不要放在普通单元测试里。

原因：

1. 慢。
2. 花钱。
3. 不稳定。
4. 需要 API key。
5. 输出不完全可预测。

可以把真实模型测试放到单独命令：

```bash
npm run test:integration
```

并且默认不在 CI 跑。

Claude Code 这类项目也会通过依赖注入、VCR、fake deps 等方式避免所有测试都打真实 API。

### 16.14 本章练习

练习一：为纯函数写测试。

至少测试：

- `globToRegExp`
- `resolveInsideCwd`
- `countOccurrences`

练习二：为 read/edit/write 工具写测试。

覆盖成功和失败。

练习三：为 runToolUse 写测试。

覆盖：

- unknown tool
- schema error
- permission denied
- tool success

练习四：为 AgentEngine 写 FakeModel 测试。

覆盖多轮工具调用和 maxTurns。

练习五：为 transcript 写测试。

覆盖保存、恢复、坏行跳过。

### 16.15 本章小结

本章我们建立了 mini-agent 的测试意识。

你应该理解：

1. Agent 测试不能依赖“手动试一试”。
2. FakeModel 是测试控制流的关键。
3. FakePrompter 是测试权限的关键。
4. 工具和协议层必须有单元测试。
5. 真实模型测试应该单独放在集成测试里。

下一章我们会回到 Claude Code 源码，把 mini-agent 已经实现的模块和 Claude Code 的对应模块做一次系统对照。

---

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
src/cli.ts
```

Claude Code：

```text
src/entrypoints/cli.tsx
src/main.tsx
src/replLauncher.tsx
src/screens/REPL.tsx
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
src/agent/engine.ts
```

Claude Code：

```text
src/QueryEngine.ts
src/query.ts
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
src/tools/Tool.ts
src/tools/buildTool.ts
```

Claude Code：

```text
src/Tool.ts
```

mini-agent Tool：

```ts
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
src/tools/runToolUse.ts
```

Claude Code：

```text
src/services/tools/toolExecution.ts
src/services/tools/toolOrchestration.ts
src/services/tools/StreamingToolExecutor.ts
src/services/tools/toolHooks.ts
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

但骨架一样。理解 mini-agent 的 `runToolUse()`，再读 `toolExecution.ts` 会轻松很多。

### 17.6 文件工具映射

mini-agent：

```text
readFileTool
writeFileTool
editFileTool
```

Claude Code：

```text
src/tools/FileReadTool/FileReadTool.ts
src/tools/FileWriteTool/FileWriteTool.ts
src/tools/FileEditTool/FileEditTool.ts
src/tools/NotebookEditTool/
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
src/tools/BashTool/BashTool.tsx
src/tools/BashTool/bashPermissions.ts
src/tools/BashTool/readOnlyValidation.ts
src/tools/BashTool/commandSemantics.ts
src/tools/BashTool/bashSecurity.ts
src/tools/BashTool/shouldUseSandbox.ts
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
src/hooks/useCanUseTool.tsx
src/utils/permissions/
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
src/services/api/claude.ts
src/services/api/client.ts
src/utils/api.js
src/utils/messages.js
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
src/utils/toolResultStorage.ts
src/services/tokenEstimation.ts
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
src/utils/sessionStorage.ts
src/utils/conversationRecovery.ts
src/commands/resume/
src/tools/AgentTool/runAgent.ts
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

- 想看入口，读 `entrypoints/cli.tsx` 和 `main.tsx`。
- 想看 Agent 循环，读 `QueryEngine.ts` 和 `query.ts`。
- 想看工具接口，读 `Tool.ts`。
- 想看工具执行，读 `services/tools/*`。
- 想看文件工具，读 `FileReadTool`、`FileEditTool`、`FileWriteTool`。
- 想看 Shell，读 `BashTool`。
- 想看权限，读 `useCanUseTool` 和 `utils/permissions`。
- 想看上下文，读 `services/compact` 和 `toolResultStorage`。

下一卷我们会更深入地工程化工具系统，重点讲 schema、权限、并发、结果预算、MCP 工具包装和 ToolSearch。

---

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
7. 对照 Claude Code 的 `Tool.ts`、`toolToAPISchema` 和 deferred tool schema。

### 18.2 Schema 不是类型注解那么简单

在普通 TypeScript 项目里，类型主要帮助开发者：

```ts
type ReadFileInput = {
  path: string
}
```

但运行时并没有这个类型。如果外部传进来：

```json
{ "path": 123 }
```

TypeScript 不会自动阻止。

Agent 工具参数来自模型，是运行时数据。必须用运行时 schema 校验。

所以我们需要：

```ts
const inputSchema = z.object({
  path: z.string(),
})
```

它不仅是给 TypeScript 看，也是给运行时看。

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

```ts
z.object({
  data: z.string(),
})
```

问题：

1. `data` 太泛。
2. 模型不知道应该放路径、内容还是命令。
3. 错误时用户也看不懂。

更好的 schema：

```ts
z.object({
  path: z
    .string()
    .describe("Path to the file, relative to the workspace root."),
  content: z
    .string()
    .describe("Full text content to write to the file."),
  overwrite: z
    .boolean()
    .optional()
    .describe("Set true to replace an existing file."),
})
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

```ts
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

```ts
z.object({
  path: z.string(),
  offset: z.number().optional(),
  limit: z.number().optional(),
})
```

模型可能不知道何时使用 offset/limit。工具描述要补充：

```text
Only provide offset and limit when reading a portion of a large file.
```

Claude Code 的 FileReadTool prompt 中也会告诉模型，文件太大时使用 offset/limit。

### 18.7 枚举比自由字符串更安全

如果字段只有几个值，用 enum。

例如权限模式：

```ts
z.enum(["default", "plan", "bypass"])
```

比：

```ts
z.string()
```

更安全。

模型如果输出 `"planning"`，schema 会拒绝，并告诉模型合法值是什么。

### 18.8 严格对象

Zod 默认对象可能允许额外字段，取决于写法。工具参数建议严格：

```ts
z.strictObject({
  path: z.string(),
})
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

```ts
if (!parsed.success) {
  return toolResult(
    toolUse.id,
    [
      `InputValidationError for ${tool.name}:`,
      parsed.error.message,
      "",
      "Check the tool schema and retry with valid arguments.",
    ].join("\n"),
    true,
  )
}
```

Claude Code 的 `toolExecution.ts` 会格式化 Zod 错误，并在 deferred tool schema 未发送时给特殊提示。

### 18.10 JSON Schema 和 Zod Schema 的区别

Zod schema 用于程序运行时校验：

```ts
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

Claude Code 的 `services/mcp/client.ts` 中 `fetchToolsForClient` 就做了这类转换。它会给 MCP 工具加 `mcp__server__tool` 前缀，避免命名冲突。

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

```ts
test("read_file schema accepts valid input", () => {
  expect(
    readFileTool.inputSchema.safeParse({ path: "README.md" }).success,
  ).toBe(true)
})

test("read_file schema rejects invalid path", () => {
  expect(
    readFileTool.inputSchema.safeParse({ path: 123 }).success,
  ).toBe(false)
})
```

还可以测试 JSON Schema 字段：

```ts
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
