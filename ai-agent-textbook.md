# 从 Claude Code 源码学习 AI Agent 搭建

> 说明：本教材基于 `yasasbanukaofficial/claude-code` 仓库中的源码快照做架构学习。仓库 README 称这些文件来自 npm sourcemap 暴露的源码备份，因此本文只做工程模式、架构思想和可复刻实现讲解，不大段复制原项目源码，也不建议把这套源码当作可直接商用的开源库。

---

## 一、学习计划

这本教程的目标不是让你“背诵 Claude Code 的源码”，而是让你从零学会搭建一个能真实工作的 AI Agent，并理解工程级 Agent 为什么会长成现在这个样子。

### 阶段 0：建立全局地图

学习目标：

1. 知道 CLI Agent 的基本组成：入口、模型调用、消息历史、工具系统、权限系统、UI、持久化。
2. 能从源码中定位一条完整链路：用户输入 -> 模型响应 -> 工具调用 -> 工具结果 -> 下一轮模型。
3. 明白“Agent”不是一个神秘模块，而是一套循环、协议和安全边界。

重点阅读文件：

- `src/entrypoints/cli.tsx`：轻量启动器和特殊命令 fast path。
- `src/main.tsx`：CLI/TUI 主入口，负责配置、命令、工具、MCP、状态初始化。
- `src/QueryEngine.ts`：对话生命周期封装，面向 SDK/headless 场景。
- `src/query.ts`：核心 Agent 循环。
- `src/Tool.ts`：工具接口定义。
- `src/tools.ts`：工具注册和过滤。

### 阶段 1：实现一个最小可用 Agent

学习目标：

1. 学会用 Messages API 的 `messages`、`system`、`tools`、`tool_result` 组织一次 Agent 循环。
2. 实现两个工具：`read_file` 和 `run_shell`。
3. 实现最简单的“模型要求调用工具 -> 本地执行 -> 结果回传模型”循环。

你会得到：

- 一个没有 UI、没有权限复杂度、但能完成简单任务的 CLI Agent。
- 对工具调用协议的直觉。

### 阶段 2：把最小 Agent 工程化

学习目标：

1. 用 schema 校验模型生成的工具参数。
2. 把工具抽象成统一接口。
3. 区分只读工具和写入工具。
4. 加入权限确认、超时、取消、错误恢复。
5. 支持并发执行安全的工具。

对应源码：

- `src/Tool.ts`
- `src/services/tools/toolExecution.ts`
- `src/services/tools/toolOrchestration.ts`
- `src/services/tools/StreamingToolExecutor.ts`
- `src/hooks/useCanUseTool.tsx`

### 阶段 3：上下文工程

学习目标：

1. 理解系统提示词、用户上下文、项目上下文、记忆、附件如何进入模型。
2. 理解为什么长会话会爆上下文。
3. 学习自动压缩、微压缩、工具结果预算、历史裁剪等策略。

对应源码：

- `src/query.ts`
- `src/services/compact/*`
- `src/utils/toolResultStorage.ts`
- `src/memdir/*`
- `src/services/SessionMemory/*`

### 阶段 4：多工具与 MCP

学习目标：

1. 学会把内置工具和外部 MCP 工具统一成同一种 Tool。
2. 学会从 MCP server 拉取 tools/list。
3. 学会给外部工具加命名空间、权限、只读标记、描述和 schema。

对应源码：

- `src/services/mcp/client.ts`
- `src/tools/MCPTool/MCPTool.ts`
- `src/tools/ListMcpResourcesTool/*`
- `src/tools/ReadMcpResourceTool/*`

### 阶段 5：子 Agent 与任务分解

学习目标：

1. 理解为什么需要 subagent：隔离任务、并行探索、降低主线程上下文污染。
2. 实现一个简单的 `AgentTool`：主 Agent 可以委派任务给子 Agent。
3. 理解子 Agent 的权限继承、工具限制、工作目录隔离、后台任务、进度上报。

对应源码：

- `src/tools/AgentTool/AgentTool.tsx`
- `src/tools/AgentTool/runAgent.ts`
- `src/tools/AgentTool/forkSubagent.ts`
- `src/tasks/LocalAgentTask/*`

### 阶段 6：产品化

学习目标：

1. CLI/TUI 输入输出体验。
2. 会话恢复、日志、成本追踪、遥测、错误报告。
3. 安全默认值、沙箱、策略配置。
4. 插件、技能、命令、快捷键等扩展系统。

对应源码：

- `src/components/*`
- `src/ink/*`
- `src/services/plugins/*`
- `src/skills/*`
- `src/commands/*`
- `src/utils/sessionStorage.ts`

---

## 二、源码总体架构

这套代码可以理解成一个“终端里的 Agent 操作系统”。它不是只有一个聊天函数，而是由多个层组成：

```text
用户输入
  ↓
CLI/TUI 层：Commander + React Ink
  ↓
会话层：QueryEngine / REPL / AppState
  ↓
Agent 主循环：query() / queryLoop()
  ↓
模型 API 层：Anthropic Messages API streaming
  ↓
工具调度层：toolExecution / toolOrchestration / StreamingToolExecutor
  ↓
工具实现层：Bash、Read、Edit、Write、Grep、Web、MCP、AgentTool
  ↓
权限与安全层：permission mode、rules、hooks、classifier、sandbox
  ↓
上下文工程：memory、attachments、compact、tool result budget
  ↓
持久化与观测：transcript、cost、usage、analytics、logs
```

最重要的认知是：Agent 的核心不是“让模型自由发挥”，而是让模型在一套可验证、可恢复、可审计的协议里行动。

---

## 三、入口：CLI 如何启动

源码里有两个入口层。

第一层是 `src/entrypoints/cli.tsx`。它是一个轻量 bootstrap，做几件事：

1. 处理 `--version` 这种不需要加载完整应用的 fast path。
2. 根据参数进入不同模式，例如 Chrome MCP、computer-use MCP、daemon、remote-control、background sessions。
3. 通过动态 import 延迟加载重模块，减少启动时间。
4. 最后才进入完整主程序。

第二层是 `src/main.tsx`。这个文件非常大，职责也非常多：

1. 初始化配置、遥测、用户认证、模型配置。
2. 读取项目上下文和用户上下文。
3. 加载命令、工具、MCP server、插件、技能。
4. 初始化权限上下文。
5. 启动 React Ink TUI 或非交互/headless 模式。
6. 把用户输入交给后续查询引擎。

这给我们一个工程启发：成熟 CLI Agent 要非常重视启动性能。很多逻辑都应该延迟加载，尤其是只有少数命令会用到的功能。

如果你从零实现，可以先这样分层：

```ts
// cli.ts
async function main() {
  const args = process.argv.slice(2)

  if (args.includes("--version")) {
    console.log("my-agent 0.1.0")
    return
  }

  const { runApp } = await import("./app")
  await runApp(args)
}

main().catch(err => {
  console.error(err)
  process.exit(1)
})
```

然后在 `app.ts` 里做完整初始化：

```ts
export async function runApp(args: string[]) {
  const config = await loadConfig()
  const tools = buildTools(config)
  const state = createSessionState()

  await repl({
    config,
    tools,
    state,
  })
}
```

不要一开始就做成巨型入口。先让每层职责清楚，后面再优化。

---

## 四、Agent 主循环：真正的心脏

源码的核心循环在 `src/query.ts`。

简化后，它做的是：

```text
while true:
  1. 准备本轮 messages
  2. 注入 system prompt、user context、tools
  3. 必要时压缩上下文
  4. 调用模型流式接口
  5. 收集 assistant 消息和 tool_use block
  6. 如果没有工具调用，结束本轮
  7. 如果有工具调用，执行工具
  8. 把 tool_result 加入 messages
  9. 继续下一轮
```

这是所有工具型 Agent 的基本范式。

### 4.1 最小 Agent 循环

先写一个教学版循环：

```ts
type Message =
  | { role: "user"; content: string }
  | { role: "assistant"; content: any[] }
  | { role: "user"; content: any[] }

async function runAgent(userText: string) {
  const messages: Message[] = [{ role: "user", content: userText }]

  while (true) {
    const response = await callModel({
      system: "You are a coding assistant.",
      messages,
      tools: [readFileTool, shellTool],
    })

    messages.push({
      role: "assistant",
      content: response.content,
    })

    const toolUses = response.content.filter(block => block.type === "tool_use")

    if (toolUses.length === 0) {
      return response
    }

    for (const toolUse of toolUses) {
      const result = await runTool(toolUse)
      messages.push({
        role: "user",
        content: [{
          type: "tool_result",
          tool_use_id: toolUse.id,
          content: result,
        }],
      })
    }
  }
}
```

这个版本能工作，但很脆弱。Claude Code 的源码围绕这个小循环加了很多工程保护。

### 4.2 Claude Code 的增强点

`src/query.ts` 的真实循环比上面复杂，主要增强在这些地方：

1. `messagesForQuery` 不是简单等于全部历史，而是会经过 compact、microcompact、snip、tool result budget。
2. `fullSystemPrompt` 是主 system prompt 加系统上下文后的组合。
3. 模型调用走 `deps.callModel`，生产环境对应 `queryModelWithStreaming`，测试可以替换假实现。
4. 工具执行可以在模型流式输出工具参数时就提前开始，而不是等整条 assistant 消息结束。
5. 对 prompt too long、max output tokens、streaming fallback、API retry 都有恢复逻辑。
6. 每轮都有 query chain tracking，用于日志、调试、性能分析。

这就是教学版 Agent 和产品级 Agent 的差距。

---

## 五、消息协议：Agent 的状态机

工具型 Agent 最重要的数据结构是消息历史。

模型看到的大概是：

```json
[
  { "role": "user", "content": "请读 package.json" },
  {
    "role": "assistant",
    "content": [
      {
        "type": "tool_use",
        "id": "toolu_123",
        "name": "Read",
        "input": { "file_path": "/repo/package.json" }
      }
    ]
  },
  {
    "role": "user",
    "content": [
      {
        "type": "tool_result",
        "tool_use_id": "toolu_123",
        "content": "{...文件内容...}"
      }
    ]
  }
]
```

工具调用必须成对出现：

- assistant 发 `tool_use`
- user 回 `tool_result`

如果丢失一半，下一次 API 调用就可能失败。因此源码里有很多“配对保护”和“缺失工具结果补偿”逻辑。例如 `src/query.ts` 有处理 missing tool result blocks 的逻辑，`src/services/api/claude.ts` 也会在发 API 前规范化消息。

你自己实现时要记住：

1. 不要把工具结果作为 assistant 消息。
2. 不要改变已经发给模型的 thinking/tool_use 块。
3. 工具失败也要回 `tool_result`，只是 `is_error: true`。
4. 每个 `tool_result.tool_use_id` 必须对应前面的 `tool_use.id`。

---

## 六、模型 API 层

`src/services/api/claude.ts` 负责把内部消息、工具和配置变成 Anthropic API 请求。

它做了很多事情：

1. 把内部 Tool 转成 API tool schema。
2. 设置 model、messages、system、tools、tool_choice。
3. 配置 thinking、max_tokens、temperature、beta headers、context management。
4. 使用 streaming API。
5. 处理 retry、fallback model、错误分类、成本和用量统计。
6. 把流式事件转成内部 `AssistantMessage`、`StreamEvent` 等。

简化版模型调用可以这样写：

```ts
async function callModel({ client, model, system, messages, tools }) {
  const stream = await client.messages.create({
    model,
    system,
    messages,
    tools,
    max_tokens: 4096,
    stream: true,
  })

  const content = []

  for await (const event of stream) {
    // 生产代码要处理 content_block_start、delta、stop 等事件
    // 教学版可以先用 SDK 的高层接口拼出最终 message
  }

  return { content }
}
```

产品级实现必须解决三个问题。

第一，流式工具参数不是一次性来的。模型可能一点点输出 JSON input。Claude Code 为了性能，选择绕过某些高层 partial JSON parsing，自己处理流式块。

第二，API 失败不一定应该结束。网络错误、529、fallback、max output tokens、prompt too long 都可能触发不同恢复路径。

第三，工具 schema 会变得很大。如果你有几十个工具和 MCP 工具，全部塞进 prompt 会浪费上下文，所以源码里有 ToolSearch 和 deferred tools 的设计。

---

## 七、工具系统：把世界变成模型可调用的函数

工具抽象定义在 `src/Tool.ts`。

一个工具至少要回答这些问题：

1. 叫什么名字？
2. 参数 schema 是什么？
3. 如何描述给模型？
4. 如何执行？
5. 结果如何映射成 `tool_result`？
6. 是否只读？
7. 是否可以并发？
8. 是否需要权限？
9. 是否会破坏性修改？
10. UI 怎么展示？

教学版接口可以这样设计：

```ts
import { z } from "zod"

export type Tool<Input, Output> = {
  name: string
  description: string
  inputSchema: z.ZodType<Input>
  isReadOnly(input: Input): boolean
  isConcurrencySafe(input: Input): boolean
  checkPermission(input: Input, ctx: ToolContext): Promise<PermissionDecision>
  call(input: Input, ctx: ToolContext): Promise<Output>
  formatResult(output: Output): string
}
```

Claude Code 的 `Tool` 接口更复杂，因为它还服务于：

- React Ink 渲染
- MCP 兼容
- 自动权限分类
- 工具搜索
- 结果压缩
- transcript search
- agent 子任务
- hook 匹配
- interruption behavior

### 7.1 buildTool：给工具安全默认值

源码中 `buildTool` 会给工具补默认方法。这个设计很值得学习。

默认策略大体是：

- `isEnabled` 默认 true。
- `isConcurrencySafe` 默认 false。
- `isReadOnly` 默认 false。
- `isDestructive` 默认 false。
- `checkPermissions` 默认 allow，但还有外层通用权限系统。
- `toAutoClassifierInput` 默认空字符串。

这体现了一个原则：安全相关的默认值要保守。

比如并发安全默认 false，因为一个工具如果会写文件、改状态、启动进程，就不能随便并发执行。

---

## 八、工具注册：有哪些工具给模型看

工具注册在 `src/tools.ts`。

这个文件维护“所有可能可用的工具”，再根据环境、权限、feature flag、MCP、模式做过滤。

常见内置工具包括：

- `AgentTool`：启动子 Agent。
- `BashTool`：执行 shell 命令。
- `FileReadTool`：读取文件。
- `FileEditTool`：编辑文件。
- `FileWriteTool`：写文件。
- `GlobTool` / `GrepTool`：搜索文件。
- `WebFetchTool` / `WebSearchTool`：网络检索。
- `TodoWriteTool`：维护任务列表。
- `MCPTool` 系列：调用外部 MCP 工具。
- `EnterPlanModeTool` / `ExitPlanModeTool`：计划模式切换。

你的最小 Agent 可以只注册三个工具：

```ts
const tools = [
  readFileTool,
  writeFileTool,
  shellTool,
]
```

但随着工具越来越多，你需要引入工具池：

```ts
function getTools(permissionContext: PermissionContext): Tool[] {
  return getAllTools()
    .filter(tool => tool.isEnabled())
    .filter(tool => !isDeniedByPolicy(tool, permissionContext))
}
```

Claude Code 还会处理一个高级问题：MCP 工具和内置工具可能非常多，不能全部塞给模型。于是它有 ToolSearch，让模型先搜索工具，再加载目标工具 schema。

---

## 九、工具执行链路

工具执行核心在 `src/services/tools/toolExecution.ts`。

一条工具调用会经历：

```text
1. 根据 tool_use.name 找到工具
2. 用 Zod 校验 input schema
3. 调用工具自己的 validateInput
4. 运行权限判断 canUseTool
5. 执行 pre-tool hooks
6. 调用 tool.call
7. 执行 post-tool hooks
8. 把输出映射成 tool_result
9. 记录日志、用量、错误、耗时
```

教学版可以先实现前五步：

```ts
async function runToolUse(toolUse, ctx) {
  const tool = ctx.tools.find(t => t.name === toolUse.name)
  if (!tool) {
    return errorResult(toolUse.id, `Unknown tool: ${toolUse.name}`)
  }

  const parsed = tool.inputSchema.safeParse(toolUse.input)
  if (!parsed.success) {
    return errorResult(toolUse.id, parsed.error.message)
  }

  const permission = await tool.checkPermission(parsed.data, ctx)
  if (permission.behavior !== "allow") {
    return errorResult(toolUse.id, "Permission denied")
  }

  try {
    const output = await tool.call(parsed.data, ctx)
    return successResult(toolUse.id, tool.formatResult(output))
  } catch (err) {
    return errorResult(toolUse.id, String(err))
  }
}
```

关键点：模型生成的参数不可信。即便模型“很聪明”，也会把数字写成字符串、漏掉字段、写错路径、调用不存在的工具。schema 校验是 Agent 工程的地基。

---

## 十、并发执行：哪些工具可以一起跑

`src/services/tools/toolOrchestration.ts` 解决并发问题。

它把工具调用分成批次：

- 只读且并发安全的工具可以放在同一批并发执行。
- 非只读或不确定安全的工具单独串行执行。

比如：

```text
Read(a.ts), Grep("foo"), Glob("*.ts")  -> 可以并发
Edit(a.ts)                            -> 必须串行
Bash("npm test")                      -> 通常串行，除非判断为安全
```

教学版分批算法：

```ts
function partitionToolCalls(toolUses, tools) {
  const batches = []

  for (const toolUse of toolUses) {
    const tool = findTool(toolUse.name)
    const safe = tool?.isConcurrencySafe(toolUse.input) ?? false
    const last = batches[batches.length - 1]

    if (safe && last?.safe) {
      last.items.push(toolUse)
    } else {
      batches.push({ safe, items: [toolUse] })
    }
  }

  return batches
}
```

更高级的 `StreamingToolExecutor` 甚至能在模型还没完全结束输出时，发现一个完整的工具调用就开始执行。这样可以降低延迟。它还处理：

- 并发安全判断。
- 非安全工具独占执行。
- 进度消息。
- 工具错误后取消兄弟工具。
- 用户中断。
- streaming fallback 后丢弃旧结果。
- 保持结果按工具出现顺序返回。

这是产品级 Agent 的典型优化：不是改变能力，而是改变响应速度和稳定性。

---

## 十一、BashTool：最危险也最有用的工具

`src/tools/BashTool/BashTool.tsx` 是最值得学习的工具之一。

它解决的问题包括：

1. 参数 schema：命令、描述、超时、是否后台运行等。
2. 只读判断：`ls`、`cat`、`grep` 等可以视为只读，`rm`、`mv`、`npm install` 等不是。
3. 权限匹配：支持类似 `Bash(git status)` 的规则。
4. 安全解析：不能只用字符串包含判断，要解析 shell 命令结构。
5. 沙箱：某些命令需要放进 sandbox。
6. 超时和中断：长命令要可取消。
7. 后台任务：长时间运行的命令不能阻塞主对话。
8. 输出截断和持久化：命令输出太大时保存到文件，只给模型预览。
9. 图像输出：某些 shell 输出可能是图片或可视内容。
10. UI 展示：实时显示进度、输出、错误和后台提示。

教学版 BashTool：

```ts
const shellTool = {
  name: "shell",
  description: "Run a shell command",
  inputSchema: z.object({
    command: z.string(),
    timeoutMs: z.number().optional(),
  }),

  isReadOnly(input) {
    return /^(ls|cat|pwd|grep|rg|find|wc|head|tail)\b/.test(input.command)
  },

  isConcurrencySafe(input) {
    return this.isReadOnly(input)
  },

  async checkPermission(input, ctx) {
    if (this.isReadOnly(input)) return { behavior: "allow" }
    return await ctx.askUser(`Allow command: ${input.command}?`)
  },

  async call(input, ctx) {
    return await execWithTimeout(input.command, input.timeoutMs ?? 30_000)
  },
}
```

但这个简化版不够安全。真实场景至少要注意：

- `cat file > other` 不是只读。
- `grep pattern file | xargs rm` 不是只读。
- `VAR=1 git push` 仍然是 git push。
- `alias`、subshell、command substitution 都可能绕过简单规则。
- `sed -i` 是编辑。
- `npm test` 可能执行任意脚本。

所以源码使用了 bash parser、安全语义分析、权限规则和 classifier 组合处理。

---

## 十二、FileReadTool：看似简单，细节很多

`src/tools/FileReadTool/FileReadTool.ts` 说明读取文件也不是一句 `fs.readFile`。

它处理：

1. 绝对路径要求。
2. 文件不存在时给相似路径建议。
3. 二进制文件检测。
4. 大文件按 offset/limit 读取。
5. token 限制。
6. 图片压缩和转 base64。
7. PDF 页码范围和抽取。
8. Notebook 读取。
9. 特殊设备文件阻止，例如 `/dev/random`。
10. 权限规则。
11. 读取后触发技能或记忆。

教学版 ReadTool：

```ts
const readFileTool = {
  name: "read_file",
  inputSchema: z.object({
    path: z.string(),
    offset: z.number().optional(),
    limit: z.number().optional(),
  }),

  isReadOnly() {
    return true
  },

  isConcurrencySafe() {
    return true
  },

  async call(input) {
    const abs = path.resolve(input.path)
    assertInsideWorkspace(abs)
    assertNotBlockedDevice(abs)

    const text = await fs.promises.readFile(abs, "utf8")
    return sliceLines(text, input.offset, input.limit)
  },
}
```

这里的原则是：读操作也会带来风险。它可能泄露密钥、卡死进程、把超大文件塞爆上下文、读取用户不希望暴露的路径。

---

## 十三、权限系统：Agent 的刹车

权限链路大致由三部分组成：

1. 工具自身的 `checkPermissions`。
2. 通用权限规则 `hasPermissionsToUseTool`。
3. UI/非交互场景的 `canUseTool` 决策。

`src/hooks/useCanUseTool.tsx` 展示了交互式权限处理：

- 如果配置明确 allow，直接允许。
- 如果配置明确 deny，拒绝。
- 如果需要 ask，就生成描述，进入确认队列。
- 对 coordinator、swarm worker、interactive UI 有不同处理。
- 对 Bash classifier 有自动批准或拒绝逻辑。
- 被取消时要返回 cancel 并 abort。

一个简化权限模型：

```ts
type PermissionDecision =
  | { behavior: "allow"; updatedInput?: unknown }
  | { behavior: "deny"; reason: string }
  | { behavior: "ask"; reason: string }

async function canUseTool(tool, input, ctx) {
  const rule = matchRule(ctx.permissionRules, tool, input)

  if (rule?.behavior === "allow") return { behavior: "allow" }
  if (rule?.behavior === "deny") return { behavior: "deny", reason: "Denied by rule" }

  if (tool.isReadOnly(input)) return { behavior: "allow" }

  if (ctx.nonInteractive) {
    return { behavior: "deny", reason: "No permission prompt available" }
  }

  return await ctx.promptUser(tool, input)
}
```

权限系统的关键设计：

1. 默认不要给写权限。
2. 后台 Agent 如果不能弹窗，就不能无限等待用户确认。
3. 权限规则要可解释，方便用户知道为什么允许或拒绝。
4. 工具执行前后都可以跑 hooks。
5. 权限和参数校验要在工具真实执行前完成。

---

## 十四、上下文工程：让 Agent 不被自己的历史淹没

长会话 Agent 必然遇到上下文问题。源码里有多种手段：

1. `autoCompact`：接近上下文限制时自动总结历史。
2. `microCompact`：对部分工具结果或消息做更细粒度压缩。
3. `toolResultBudget`：工具输出太大时替换成预览和文件路径。
4. `snip` / `contextCollapse`：在特定 feature 下进一步裁剪或投影历史。
5. memory：把长期信息放进可检索/可注入的记忆，而不是每轮都塞全部。

简化版上下文压缩：

```ts
async function compactIfNeeded(messages, maxTokens) {
  const tokens = estimateTokens(messages)
  if (tokens < maxTokens * 0.8) return messages

  const summary = await summarize(messages.slice(0, -10))

  return [
    {
      role: "user",
      content: `Conversation summary so far:\n${summary}`,
    },
    ...messages.slice(-10),
  ]
}
```

但产品级压缩要更精细：

- 不能破坏 tool_use/tool_result 配对。
- 不能改写带签名的 thinking block。
- 不能删掉当前任务必须依赖的文件上下文。
- 压缩本身也会消耗 token 和费用。
- 压缩失败要有 circuit breaker，不能无限重试。

源码里的 `query.ts` 把 compact 放在每次 API 调用前，这是合理位置：它能基于最新消息决定是否压缩，又不会影响工具执行中的中间状态。

---

## 十五、MCP：把外部能力接进同一套工具协议

MCP 的核心价值是：外部服务也能暴露工具，Agent 客户端把它们当成本地 Tool 一样调用。

`src/services/mcp/client.ts` 中的 `fetchToolsForClient` 做了关键转换：

```text
MCP tools/list 返回的 tool
  ↓
清洗和规范化
  ↓
加 mcp__server__tool 命名空间
  ↓
转换为内部 Tool
  ↓
接入权限、只读、破坏性、open world、searchHint 等字段
```

教学版 MCP 工具包装：

```ts
function wrapMcpTool(serverName, mcpTool, client): Tool<any, any> {
  return {
    name: `mcp__${serverName}__${mcpTool.name}`,
    description: mcpTool.description ?? "",
    inputSchema: jsonSchemaToZod(mcpTool.inputSchema),

    isReadOnly() {
      return mcpTool.annotations?.readOnlyHint === true
    },

    isConcurrencySafe(input) {
      return this.isReadOnly(input)
    },

    async checkPermission(input, ctx) {
      if (this.isReadOnly(input)) return { behavior: "allow" }
      return await ctx.askUser(`Allow MCP tool ${mcpTool.name}?`)
    },

    async call(input) {
      return await client.callTool({
        name: mcpTool.name,
        arguments: input,
      })
    },
  }
}
```

注意：外部 MCP server 不一定可信。它给的描述、schema、metadata 都要清洗。源码里会处理 unicode sanitize、名称 normalize、description 截断、prefix、auth 等问题。

---

## 十六、子 Agent：让 Agent 学会委派

`src/tools/AgentTool/AgentTool.tsx` 和 `src/tools/AgentTool/runAgent.ts` 是多 Agent 的核心。

子 Agent 的价值：

1. 把探索任务从主上下文隔离出去。
2. 并行处理多个方向。
3. 给不同任务不同工具集和权限。
4. 让主 Agent 只接收总结结果，而不是所有中间噪音。

AgentTool 的输入大致包括：

- `description`：短任务描述。
- `prompt`：交给子 Agent 的完整任务。
- `subagent_type`：选择专门 Agent，例如 Explore、Plan、general-purpose。
- `model`：子 Agent 模型。
- `run_in_background`：是否后台运行。
- `isolation`：是否用 worktree 隔离。
- `cwd`：子 Agent 工作目录。

教学版 AgentTool：

```ts
const agentTool = {
  name: "agent",
  inputSchema: z.object({
    description: z.string(),
    prompt: z.string(),
  }),

  async call(input, parentCtx) {
    const childCtx = createChildContext(parentCtx, {
      tools: parentCtx.tools.filter(t => t.name !== "agent"),
      permissionMode: "read-only",
    })

    const result = await runAgentLoop({
      messages: [{ role: "user", content: input.prompt }],
      ctx: childCtx,
      maxTurns: 10,
    })

    return summarizeChildResult(result)
  },
}
```

产品级子 Agent 还要考虑：

1. 是否继承父 Agent 的上下文。
2. 是否共享文件读取 cache。
3. 是否允许写操作。
4. 是否能弹权限确认。
5. 子 Agent 的 transcript 存在哪里。
6. 子 Agent 被取消时如何清理后台任务。
7. 子 Agent 是否用独立 git worktree。
8. 子 Agent 输出太长时如何总结。

源码里的 `runAgent` 会重新构造子 Agent 的 system prompt、user context、system context、tool pool、permission context 和 abort controller。这是一个很好的设计：子 Agent 应该是“受控的独立会话”，不是简单函数调用。

---

## 十七、计划模式与任务列表

成熟 coding Agent 不能总是直接改文件。计划模式的意义是：

1. 先探索，再提出方案。
2. 用户批准后再执行。
3. 避免 Agent 在不确定需求时直接写代码。

源码里有：

- `EnterPlanModeTool`
- `ExitPlanModeTool`
- `TodoWriteTool`
- `TaskListTool` 等任务工具

你可以先实现一个简单约束：

```ts
if (ctx.permissionMode === "plan" && tool.isDestructive(input)) {
  return {
    behavior: "deny",
    reason: "Plan mode forbids destructive tool calls",
  }
}
```

然后引入 `todo_write`：

```ts
type Todo = {
  content: string
  status: "pending" | "in_progress" | "completed"
}
```

任务列表不是装饰。它能让 Agent：

- 把复杂任务外化。
- 减少遗漏。
- 向用户展示进展。
- 在中断恢复后知道做到哪里。

---

## 十八、会话状态与 QueryEngine

`src/QueryEngine.ts` 把对话生命周期封装成类。

它保存：

- `mutableMessages`
- `abortController`
- `permissionDenials`
- `readFileState`
- `totalUsage`
- skill discovery 状态
- nested memory paths

并提供 `submitMessage()` 作为每轮用户输入入口。

这和简单函数式 `runAgent(prompt)` 的区别是：真实产品有连续会话。每次用户输入不是新世界，而是在同一个会话状态上继续。

教学版：

```ts
class QueryEngine {
  private messages: Message[] = []
  private abortController = new AbortController()

  constructor(private config: EngineConfig) {}

  async submitMessage(text: string) {
    this.messages.push({ role: "user", content: text })

    const result = await runAgentLoop({
      messages: this.messages,
      tools: this.config.tools,
      signal: this.abortController.signal,
    })

    this.messages = result.messages
    return result.finalAnswer
  }

  abort(reason = "user") {
    this.abortController.abort(reason)
  }
}
```

一旦你支持 SDK、非交互模式、远程会话或 UI，就需要类似 QueryEngine 的抽象。

---

## 十九、错误恢复

Agent 工程最容易被低估的是错误恢复。

Claude Code 处理的错误类型包括：

1. 工具不存在。
2. 工具参数 schema 不匹配。
3. 工具 validateInput 失败。
4. 权限拒绝。
5. 用户取消。
6. shell 命令失败。
7. 并发工具中一个失败导致兄弟工具取消。
8. API 网络错误。
9. API fallback。
10. prompt too long。
11. max output tokens。
12. compact 失败。
13. MCP auth 失败。
14. streaming fallback 导致已产生的 partial message 失效。

一条重要原则：错误也要进入模型可理解的协议。

比如工具失败不要直接 throw 到最外层，而应该返回：

```json
{
  "type": "tool_result",
  "tool_use_id": "toolu_123",
  "is_error": true,
  "content": "Error: file not found"
}
```

这样模型可以自我修正，例如换路径、缩小读取范围、请求用户授权。

---

## 二十、从零搭建路线：一个可完成的项目

下面是一条建议路线。每一步都可以跑通，不需要一口气做完。

### 第 1 步：纯聊天 CLI

功能：

- 用户输入文本。
- 调模型。
- 打印回复。
- 保存 messages。

验收：

- 可以连续聊天 5 轮。

### 第 2 步：加入 Read 工具

功能：

- 模型可以请求读取文件。
- 本地执行读取。
- 把结果作为 tool_result 回传。

验收：

- 用户说“总结 README”，Agent 能自己读文件再总结。

### 第 3 步：加入 Shell 工具

功能：

- 支持只读命令自动执行。
- 写命令需要用户确认。
- 命令有超时。

验收：

- 用户说“看看这个项目的文件结构”，Agent 能运行 `ls` 或 `find`。

### 第 4 步：加入 Write/Edit 工具

功能：

- 写文件前展示 diff 或说明。
- 需要用户确认。
- 保存变更记录。

验收：

- 用户说“创建一个 hello.ts”，Agent 能写文件。

### 第 5 步：加入任务列表

功能：

- Agent 可以维护 pending/in_progress/completed。
- UI 或 CLI 展示当前任务。

验收：

- 复杂任务中能看到进度变化。

### 第 6 步：加入上下文压缩

功能：

- 超过 token 阈值时总结早期消息。
- 保留最近 N 轮。
- 不破坏工具配对。

验收：

- 长会话不会因为上下文超限直接崩。

### 第 7 步：加入子 Agent

功能：

- 主 Agent 可委派探索任务给子 Agent。
- 子 Agent 只返回总结。

验收：

- 用户说“分析这个项目架构”，主 Agent 能派一个子 Agent 搜索文件，最后汇总。

### 第 8 步：加入 MCP

功能：

- 连接一个 MCP server。
- 拉取工具列表。
- 包装为内部 Tool。

验收：

- Agent 能调用外部 MCP 工具。

### 第 9 步：产品化

功能：

- session 保存与恢复。
- 日志和成本统计。
- 权限规则配置。
- 插件/技能目录。
- 更好的 TUI。

验收：

- 可以作为日常 coding assistant 使用。

---

## 二十一、关键工程原则总结

1. Agent 是循环，不是单次调用。
2. 工具调用是协议，不是随便执行函数。
3. schema 校验必须存在。
4. 权限系统必须默认保守。
5. 只读和写入要严格区分。
6. 并发只给确定安全的工具。
7. 工具失败也要回传给模型。
8. 上下文要预算化，不能无限增长。
9. 子 Agent 要隔离，不要共享所有权限。
10. MCP 工具要清洗和命名空间隔离。
11. UI、日志、持久化不是锦上添花，是调试 Agent 的基础设施。
12. 产品级 Agent 的大部分代码都在处理边界情况。

---

## 二十二、源码导读索引

建议按下面顺序读源码：

1. `src/entrypoints/cli.tsx`
   - 看启动 fast path 和动态 import。

2. `src/main.tsx`
   - 看工具、命令、MCP、配置、状态如何汇合。

3. `src/QueryEngine.ts`
   - 看连续会话如何封装。

4. `src/query.ts`
   - 看 Agent 主循环、压缩、模型调用、工具执行、恢复路径。

5. `src/services/api/claude.ts`
   - 看内部消息如何变成 API 请求，流式响应如何还原。

6. `src/Tool.ts`
   - 看工具接口和 `buildTool` 默认值。

7. `src/tools.ts`
   - 看工具池和过滤。

8. `src/services/tools/toolExecution.ts`
   - 看单个工具调用的完整生命周期。

9. `src/services/tools/toolOrchestration.ts`
   - 看工具批处理和并发。

10. `src/services/tools/StreamingToolExecutor.ts`
    - 看流式工具执行优化。

11. `src/tools/BashTool/BashTool.tsx`
    - 看 shell 工具的安全与工程复杂度。

12. `src/tools/FileReadTool/FileReadTool.ts`
    - 看读取文件如何处理大文件、图片、PDF、权限。

13. `src/hooks/useCanUseTool.tsx`
    - 看交互式权限确认。

14. `src/services/mcp/client.ts`
    - 看 MCP 工具如何转换为内部 Tool。

15. `src/tools/AgentTool/AgentTool.tsx`
    - 看主 Agent 如何启动子 Agent。

16. `src/tools/AgentTool/runAgent.ts`
    - 看子 Agent 的上下文、权限、工具和模型如何隔离。

---

## 二十三、练习：实现你的第一个工程级 Agent

### 练习 1：实现 Tool 接口

要求：

- 用 TypeScript 和 Zod。
- 支持 `name`、`inputSchema`、`call`、`isReadOnly`、`isConcurrencySafe`。
- 写 `buildTool` 给默认值。

验收：

- `read_file` 工具可以通过统一 `runToolUse` 调用。

### 练习 2：实现工具配对检查

要求：

- 扫描 messages。
- 找出 assistant 里的 tool_use。
- 检查后续 user 消息里是否有对应 tool_result。

验收：

- 缺失时能生成错误 tool_result 或阻止 API 调用。

### 练习 3：实现权限规则

要求：

- 支持 `allow: ["Read", "Bash(ls *)"]`。
- 支持 `deny: ["Bash(rm *)"]`。
- 写命令默认 ask。

验收：

- `ls` 自动允许，`rm -rf` 自动拒绝。

### 练习 4：实现上下文压缩

要求：

- 超过估算 token 阈值时总结旧消息。
- 保留最近 10 条消息。
- 不拆散 tool_use/tool_result。

验收：

- 造 100 轮消息，压缩后仍能继续调用模型。

### 练习 5：实现子 Agent

要求：

- 父 Agent 有 `agent` 工具。
- 子 Agent 只能用 read/search 工具。
- 子 Agent 最后返回总结。

验收：

- 父 Agent 可委派“搜索项目中所有 TODO”给子 Agent。

---

## 二十四、下一步扩写方向

这份教材第一版已经覆盖主干。后续可以继续扩成完整书稿：

1. 每章加入完整可运行代码。
2. 增加一个 `mini-claude-code` 示例项目。
3. 为每个工具写单元测试。
4. 加入 React Ink TUI 教程。
5. 加入 MCP server 从零实现。
6. 加入安全沙箱和命令解析专题。
7. 加入成本统计、日志、会话恢复专题。
8. 加入多 Agent 调度和 worktree 隔离专题。

如果继续写成“书”，建议目录如下：

```text
第 1 章  Agent 是什么
第 2 章  Messages API 与工具协议
第 3 章  从零实现 CLI Agent
第 4 章  工具抽象与 Schema
第 5 章  文件工具
第 6 章  Shell 工具与安全
第 7 章  权限系统
第 8 章  Agent 主循环
第 9 章  流式响应与流式工具执行
第 10 章 上下文工程
第 11 章 记忆系统
第 12 章 MCP 扩展
第 13 章 子 Agent 与任务委派
第 14 章 CLI/TUI 产品化
第 15 章 测试、观测与调试
第 16 章 安全、沙箱与策略
第 17 章 完整项目实战
```

