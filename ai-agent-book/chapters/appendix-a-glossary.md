# 附录 A：AI Agent 工程术语表

本附录整理全书反复出现的术语。它不是严格学术词典，而是面向工程实践的解释。你可以把它当作阅读本书和搭建 mini-agent 时的速查表。

## A.1 Agent

Agent 是能围绕目标进行多轮决策和行动的程序系统。

它通常包含：

1. 模型。
2. 消息上下文。
3. 工具。
4. 工具调用循环。
5. 权限系统。
6. 状态记录。
7. 终止条件。

普通聊天机器人主要生成文本。Agent 不只生成文本，还会请求工具、读取信息、执行命令、修改文件、根据结果继续推理。

最小 Agent 循环：

```text
用户目标 -> 模型 -> 工具调用 -> 工具结果 -> 模型 -> 最终答案
```

## A.2 Agent Loop

Agent Loop 是 Agent 的核心执行循环。

典型流程：

1. 把用户输入和历史消息发送给模型。
2. 模型返回文本或 tool_use。
3. 如果返回文本并不再请求工具，任务结束。
4. 如果返回 tool_use，程序执行工具。
5. 把 tool_result 加回 messages。
6. 再次调用模型。

Agent Loop 必须有 `maxTurns`，否则模型可能不断请求工具，导致无限循环。

## A.3 Model

Model 指大语言模型。它负责根据上下文生成下一步响应。

在 Agent 中，模型不直接读文件、不直接执行命令、不直接修改代码。它只是产生意图，例如：

```json
{
  "type": "tool_use",
  "name": "Read",
  "input": {
    "file_path": "src/app.py"
  }
}
```

真正执行的是宿主程序。

## A.4 Model Client

Model Client 是对模型供应商 SDK 的封装。

它的作用是隔离具体供应商差异，让 Agent 核心逻辑不直接依赖某个 SDK。

推荐接口：

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

以后更换模型时，只需要换 client 实现。

## A.5 Message

Message 是模型上下文中的一条消息。

常见角色：

1. system：系统指令。
2. user：用户输入或工具结果。
3. assistant：模型输出。

Agent 的 messages 不只是聊天记录，还包含工具调用和工具结果。

## A.6 System Prompt

System Prompt 是给模型的高优先级行为说明。

它可以包含：

1. Agent 身份。
2. 工具使用规则。
3. 安全约束。
4. 输出风格。
5. 任务流程。

System Prompt 不应该无限增长。太长会占用上下文预算，也可能让模型忽略关键指令。

## A.7 User Prompt

User Prompt 是用户输入的任务请求。

例如：

```text
修复这个项目中失败的测试，并说明你修改了什么。
```

Agent 应该保留用户原始目标，尤其是在 compact 后仍要保存。

## A.8 Tool

Tool 是 Agent 可以调用的程序能力。

例如：

1. Read：读取文件。
2. Grep：搜索文本。
3. Bash：执行命令。
4. Edit：编辑文件。
5. Write：写文件。
6. Agent：启动子 Agent。
7. MCP tool：外部服务器提供的工具。

Tool 必须有清晰 schema、权限检查和稳定结果格式。

## A.9 Tool Definition

Tool Definition 是工具暴露给模型的说明。

它通常包含：

1. name。
2. description。
3. inputSchema。

模型根据 Tool Definition 判断什么时候调用工具、如何构造输入。

如果 description 写得模糊，模型就容易误用工具。

## A.10 Tool Schema

Tool Schema 描述工具输入结构。

常用 JSON Schema 风格：

```json
{
  "type": "object",
  "properties": {
    "file_path": {
      "type": "string"
    }
  },
  "required": ["file_path"]
}
```

Schema 是模型和程序之间的合同。程序不能相信模型永远给出合法输入，所以执行前仍要校验。

## A.11 Tool Use

Tool Use 是模型请求调用工具的消息块。

它通常包含：

1. id。
2. name。
3. input。

示例：

```json
{
  "id": "toolu_01",
  "name": "Read",
  "input": {
    "file_path": "pyproject.toml"
  }
}
```

程序收到 tool_use 后，找到对应工具并执行。

## A.12 Tool Result

Tool Result 是工具执行结果。

它必须关联 tool_use id，这样模型才能知道结果对应哪次调用。

示例：

```json
{
  "tool_use_id": "toolu_01",
  "content": "{ \"scripts\": { \"test\": \"pytest\" } }"
}
```

Tool Result 不是给用户看的最终答案，而是给模型继续推理的上下文。

## A.13 Tool Registry

Tool Registry 是工具注册表。

职责：

1. 注册工具。
2. 返回工具定义。
3. 根据 tool name 查找工具。
4. 执行工具。

它让 Agent Loop 不需要写一大堆 if else。

## A.14 Read Tool

Read Tool 用于读取文件。

关键要求：

1. 限制路径在 workspace 内。
2. 限制文件大小和行数。
3. 对二进制文件友好失败。
4. 文件不存在时返回清晰错误。
5. 最好带行号。

Read 是代码 Agent 最基础的工具。

## A.15 Grep Tool

Grep Tool 用于搜索文本。

关键要求：

1. 支持 pattern。
2. 支持 path 限制。
3. 限制结果数量。
4. 忽略 `.git`、`node_modules`、`dist` 等目录。
5. 返回文件路径和匹配行。

Agent 不知道文件在哪里时，应该先 Grep 再 Read。

## A.16 Bash Tool

Bash Tool 用于执行命令。

它是最强也最危险的工具。

必须具备：

1. timeout。
2. stdout/stderr 捕获。
3. exitCode。
4. 输出截断。
5. 权限检查。
6. 工作目录限制。

不要让 Bash 绕过权限系统。

## A.17 Edit Tool

Edit Tool 用于局部修改文件。

常见设计是 old_string/new_string 替换：

```json
{
  "file_path": "src/app.py",
  "old_string": "x = 1",
  "new_string": "x = 2"
}
```

关键要求：

1. old_string 必须唯一匹配。
2. 修改前最好读过文件。
3. 修改后返回 diff 摘要。
4. 不允许路径越界。

## A.18 Write Tool

Write Tool 用于创建或覆盖文件。

它比 Edit 更危险，因为可能覆盖完整文件。

建议：

1. 创建新文件可以按策略 allow 或 ask。
2. 覆盖已有文件必须 ask。
3. 禁止写 `.env` 等敏感文件，除非用户明确授权。
4. 写入后记录 changed files。

## A.19 Permission

Permission 是工具执行前的权限判断。

常见决策：

1. allow：允许。
2. deny：拒绝。
3. ask：询问用户。

权限系统要根据工具名和输入一起判断。只看工具名是不够的。

## A.20 Permission Rule

Permission Rule 是权限匹配规则。

示例：

```text
Read(*) -> allow
Bash(pytest*) -> allow
Bash(git push*) -> ask
Bash(rm -rf*) -> deny
Edit(*) -> ask
```

规则应该可审计、可解释、可测试。

## A.21 Permission Audit

Permission Audit 是权限决策记录。

它应该记录：

1. toolName。
2. toolUseId。
3. decision。
4. source。
5. matchedRule。
6. reason。
7. sanitized input summary。

权限审计用于安全复盘和合规解释。

## A.22 Sandbox

Sandbox 是限制程序执行边界的环境。

它可以限制：

1. 文件读写范围。
2. 网络访问。
3. 命令执行。
4. 环境变量。
5. 进程能力。

权限系统回答“应不应该执行”，沙箱回答“即使想执行，系统最多允许到哪里”。

两者都需要。

## A.23 Hook

Hook 是在某些生命周期节点触发的外部逻辑。

例如：

1. 工具执行前。
2. 工具执行后。
3. 用户提交前。
4. Agent 完成后。

Hook 可以用于组织策略、安全检查、审计、格式化、通知。

Hook 也可能带来风险，所以来源和权限要受控。

## A.24 MCP

MCP 是 Model Context Protocol。

它让外部工具服务器把能力提供给 Agent。

Agent 可以通过 MCP：

1. 发现工具。
2. 调用工具。
3. 获取资源。
4. 接入外部系统。

MCP 工具应该被包装成本地统一 Tool，并经过权限系统。

## A.25 ToolSearch

ToolSearch 是工具太多时的延迟加载机制。

当系统有很多工具时，不应该每轮都把所有 schema 注入模型。可以先给模型一个搜索工具，让它按需查找相关工具。

ToolSearch 解决的是上下文预算问题。

## A.26 Context

Context 是模型本次调用能看到的信息。

包括：

1. system prompt。
2. user message。
3. assistant message。
4. tool_result。
5. memory。
6. skills。
7. tool schema。

上下文不是越多越好。无关信息会增加成本，也会干扰模型。

## A.27 Context Window

Context Window 是模型一次调用能处理的最大上下文长度。

即使模型支持很长上下文，也不意味着应该塞满。长上下文会增加成本、延迟和注意力噪声。

## A.28 Context Budget

Context Budget 是你给不同上下文部分分配的预算。

例如：

1. system prompt：10%。
2. tool schema：20%。
3. conversation：40%。
4. tool results：30%。

预算不是严格比例，而是一种工程意识：每类信息都不能无限增长。

## A.29 Compaction

Compaction 是上下文压缩。

当历史太长时，系统把旧消息总结成摘要，保留关键事实，删除重复细节。

好的 compact 要保留：

1. 用户目标。
2. 明确限制。
3. 已读关键文件。
4. 已修改文件。
5. 已运行命令。
6. 当前结论。
7. 下一步。

## A.30 Transcript

Transcript 是会话事实账本。

它记录 user、assistant、tool_result 等可恢复事实。

用途：

1. resume。
2. replay。
3. audit。
4. fork。
5. sidechain。

Transcript 不应该混入所有 UI progress。

## A.31 JSONL

JSONL 是一行一个 JSON 对象的文件格式。

适合 transcript：

```jsonl
{"type":"user","content":"hello"}
{"type":"assistant","content":"hi"}
```

优点：

1. 可追加写入。
2. 单行损坏影响较小。
3. 方便流式处理。
4. 方便命令行查看。

## A.32 Resume

Resume 是恢复历史会话继续执行。

它依赖 transcript。

恢复时要重建：

1. messages。
2. sessionId。
3. parentUuid。
4. 工具结果。
5. compact 摘要。

## A.33 Replay

Replay 是回放一次 Agent 运行。

模式：

1. view mode：只看历史。
2. deterministic replay：按历史工具结果重放。
3. model replay：重新调模型，但工具结果 mock。
4. live rerun：重新调模型并执行工具。

审计历史时不要把 live rerun 当作事实。

## A.34 Sidechain

Sidechain 是子 Agent 的独立 transcript。

主 Agent 只接收子 Agent 摘要，但系统保存子 Agent 的完整执行链。

用途：

1. 审计子任务。
2. 回放子任务。
3. 防止主上下文被污染。
4. 分析多 Agent 行为。

## A.35 Sub Agent

Sub Agent 是被主 Agent 委托执行特定任务的 Agent。

例如：

1. code-reviewer。
2. security-reviewer。
3. test-runner。
4. docs-writer。

子 Agent 应该有明确任务、工具限制、maxTurns 和输出格式。

## A.36 AgentTool

AgentTool 是把“启动子 Agent”包装成一个工具。

主 Agent 可以像调用 Read、Grep 一样调用 AgentTool：

```json
{
  "name": "Agent",
  "input": {
    "agentId": "code-reviewer",
    "task": "检查认证模块风险"
  }
}
```

AgentTool 内部负责创建子 Agent、运行、收集结果、记录 sidechain。

## A.37 Worktree Isolation

Worktree Isolation 是用 git worktree 给子任务创建隔离工作区。

它适合多 Agent 并行修改代码。

优点：

1. 子 Agent 不直接污染主工作区。
2. 可以单独查看 diff。
3. 可以选择是否合并。
4. 冲突更可控。

## A.38 Agent Definition

Agent Definition 是一个 Agent 的结构化定义。

通常包含：

1. name。
2. description。
3. prompt。
4. tools。
5. model。
6. maxTurns。
7. skills。
8. isolation。

自定义 Agent 通常用 Markdown frontmatter 描述这些字段。

## A.39 Custom Agent

Custom Agent 是用户、项目或插件定义的专用 Agent。

例如项目可以定义：

```text
frontend-reviewer
database-migration-helper
security-auditor
```

Custom Agent 要有加载优先级和安全边界。

## A.40 Plugin Agent

Plugin Agent 是插件分发的 Agent。

它必须命名空间化，避免覆盖内置或项目 Agent。

例如：

```text
plugin:docs/docs-writer
plugin:test/test-runner
```

插件 Agent 不应该静默获得高权限。

## A.41 Skill

Skill 是一组可加载的任务说明、流程、模板或工具使用知识。

它和 Agent Definition 不同：

1. Agent Definition 定义一个角色。
2. Skill 定义一项能力或流程。

Agent 可以按任务加载相关 skill。

## A.42 Memory

Memory 是跨会话保存的长期信息。

适合保存：

1. 用户偏好。
2. 项目约定。
3. 常用命令。
4. 重要背景。

不适合保存：

1. secret。
2. 所有聊天记录。
3. 不确定事实。
4. 临时噪声。

## A.43 Eval

Eval 是评估 Agent 行为的测试。

好的 eval 不只看最终答案，也看：

1. 工具轨迹。
2. 文件变更。
3. 命令执行。
4. 权限行为。
5. 成本。
6. 耗时。
7. 安全约束。

## A.44 Fixture

Fixture 是 eval 使用的固定输入环境。

例如一个小型项目，里面故意放一个失败测试。

固定 fixture 能让不同版本的 Agent 可比较。

## A.45 Golden Transcript

Golden Transcript 是人工认可的优秀执行轨迹。

它可以作为评估参考，检查新版本 Agent 是否明显偏离合理路径。

不要求完全一致，但可以比较：

1. 是否先探索再修改。
2. 是否读取关键文件。
3. 是否运行测试。
4. 是否避免危险工具。

## A.46 Profiler

Profiler 用于记录一次 query 的性能时间线。

它回答：

1. 慢在哪里？
2. 模型首 token 慢不慢？
3. 工具执行慢不慢？
4. compact 慢不慢？
5. schema build 慢不慢？

Profiler 是调试性能问题的重要工具。

## A.47 Checkpoint

Checkpoint 是 profiler 中的时间点。

例如：

1. query_user_input_received。
2. context_loading_start。
3. api_request_sent。
4. first_chunk_received。
5. tool_execution_start。
6. query_end。

通过 checkpoint 可以计算阶段耗时。

## A.48 Trace

Trace 是一次任务的树状执行记录。

它包含多个 span：

1. task span。
2. model span。
3. tool span。
4. permission span。
5. child agent span。

Trace 适合分析复杂多 Agent 任务。

## A.49 Span

Span 是 trace 中的一段操作。

它有：

1. spanId。
2. parentSpanId。
3. startedAt。
4. endedAt。
5. status。
6. attributes。

Span 能表达嵌套关系，比如某个 tool span 属于某个 model turn。

## A.50 Structured Event

Structured Event 是结构化事件日志。

它不是普通字符串日志，而是有字段的对象：

```json
{
  "eventName": "tool_execution_completed",
  "toolName": "Bash",
  "status": "error",
  "errorType": "timeout"
}
```

结构化事件方便统计、筛选、报警和分析。

## A.51 Task Progress

Task Progress 是展示给用户的任务进度状态。

它可以包含：

1. status。
2. toolUseCount。
3. tokenCount。
4. recentActivities。
5. updatedAt。

Progress 是 UI 状态，不应该直接污染 transcript。

## A.52 MaxTurns

MaxTurns 是 Agent loop 的最大轮数。

它防止无限循环。

如果任务达到 maxTurns，Agent 应该停止，并告诉用户：

1. 已完成什么。
2. 卡在哪里。
3. 下一步需要什么。

## A.53 Token

Token 是模型处理文本的基本单位。

中文、英文、代码都会被切成 token。

Agent 成本通常和 input token、output token、工具结果 token、compact token 有关。

## A.54 Token Budget

Token Budget 是对 token 使用的预算控制。

例如：

1. 单次工具结果最多 12000 字符。
2. 单次任务最多 20 轮。
3. 子 Agent 最多 8 轮。
4. compact 后摘要不超过指定长度。

Token Budget 控制成本和稳定性。

## A.55 Tool Result Truncation

Tool Result Truncation 是截断过大的工具结果。

截断时要告诉模型：

1. 已截断。
2. 原始大小。
3. 当前保留范围。
4. 如何获取更多信息。

不要悄悄截断，否则模型会误以为结果完整。

## A.56 Streaming

Streaming 是流式响应。

模型边生成边返回 chunk，用户能更早看到输出。

Agent 中 streaming 要小心 tool_use。普通文本可以即时显示，但工具调用要等结构完整后再执行。

## A.57 TTFT

TTFT 是 Time To First Token。

它表示从请求模型到收到第一个输出 chunk 的时间。

TTFT 高，用户会觉得系统没有反应。

## A.58 Pre-request Overhead

Pre-request Overhead 是发起模型请求前的开销。

包括：

1. 加载上下文。
2. 构建工具 schema。
3. 处理附件。
4. compact。
5. message normalization。

如果这部分很高，即使模型很快，用户也会觉得慢。

## A.59 Message Normalization

Message Normalization 是把内部消息转换成模型 API 需要的格式。

它要处理：

1. 文本。
2. tool_use。
3. tool_result。
4. attachments。
5. compact summary。

格式不对会导致模型 API 报错。

## A.60 Attachment

Attachment 是附加到会话中的文件、图片或其他输入。

代码 Agent 中 attachment 可能是用户拖入的文件、截图、日志。

Attachment 应该进入上下文预算管理，不要无限注入。

## A.61 Diff

Diff 是文件修改前后的差异。

Agent 修改代码后，应该能展示 diff。

Diff 用于：

1. 用户审查。
2. eval 检查。
3. 事故复盘。
4. 合并子 Agent 结果。

## A.62 Dirty Worktree

Dirty Worktree 指 git 工作区存在未提交修改。

Agent 修改文件前应注意 dirty worktree，因为这些修改可能是用户的。

安全原则：

1. 不要覆盖用户未提交修改。
2. 任务结束列出 changed files。
3. 高风险修改前询问。

## A.63 GitHub Pages

GitHub Pages 是 GitHub 提供的静态站点托管服务。

它适合发布：

1. 项目文档。
2. 教程。
3. 个人主页。
4. 静态手册。

本书使用 GitHub Pages 发布 Markdown 文档站。

## A.64 GitHub Actions

GitHub Actions 是 GitHub 的 CI/CD 系统。

在本书项目中，它负责：

1. checkout 仓库。
2. 构建 Jekyll 站点。
3. 上传 artifact。
4. 部署 Pages。

## A.65 Jekyll

Jekyll 是静态站点生成器。

GitHub Pages 原生支持 Jekyll。它可以把 Markdown 文件构建成 HTML。

本书使用简单 Jekyll 配置，不依赖复杂主题。

## A.66 README

README 是仓库入口文档。

它应该告诉读者：

1. 项目是什么。
2. 如何阅读。
3. 如何运行或部署。
4. 目录在哪里。

## A.67 index.md

`index.md` 是文档站首页。

在 GitHub Pages 中，它会被渲染为站点根路径页面。

## A.68 Deployment

Deployment 是发布过程。

对本书来说，部署流程是：

1. git add。
2. git commit。
3. git remote add origin。
4. git push。
5. GitHub Actions 构建。
6. GitHub Pages 发布。

## A.69 CI

CI 是持续集成。

对 Agent 项目，CI 可以做：

1. 类型检查。
2. 单元测试。
3. eval。
4. lint。
5. 文档构建。

CI 能防止本地能跑、远程坏掉。

## A.70 Regression

Regression 是回归，即新版本破坏了旧能力。

Agent 很容易回归，因为 prompt、工具 schema、权限策略、模型版本都会影响行为。

Eval 是发现 regression 的核心手段。

## A.71 Deterministic Check

Deterministic Check 是确定性检查。

例如：

1. 文件是否改变。
2. 命令是否运行。
3. 测试是否通过。
4. 禁止命令是否出现。

它比模型 judge 更稳定，应该优先使用。

## A.72 Model Judge

Model Judge 是用模型评价模型输出。

它适合评价语义质量，但不应该替代确定性检查。

使用时必须给 rubric，并提供事实依据，比如 diff 和测试结果。

## A.73 Rubric

Rubric 是评分标准。

好的 rubric 明确每项得分条件，避免 judge 随意发挥。

## A.74 Safety

Safety 是安全性。

Agent 安全包括：

1. 权限安全。
2. 文件安全。
3. 命令安全。
4. 数据隐私。
5. 工具边界。
6. 子 Agent 隔离。

安全不是上线前补丁，而是架构的一部分。

## A.75 Privacy

Privacy 是隐私保护。

Agent 可能接触源代码、密钥、路径、日志、用户输入。可观测性系统必须避免不必要上传敏感内容。

默认策略：本地可以更完整，远程 telemetry 要最小化。

## A.76 Telemetry

Telemetry 是远程观测数据。

适合上传：

1. 耗时。
2. 错误类型。
3. token 数。
4. 工具名。
5. 决策类型。
6. 截断状态。

不应默认上传：

1. 完整代码。
2. 完整 prompt。
3. secret。
4. 完整命令输出。

## A.77 Sanitization

Sanitization 是清洗敏感数据。

例如把 API key 替换为 `[REDACTED]`。

清洗不是万能的。更好的策略是默认不收集原文。

## A.78 Source of Truth

Source of Truth 是事实来源。

在 Agent 系统中：

1. transcript 是会话事实来源。
2. permission audit 是权限事实来源。
3. git diff 是文件修改事实来源。
4. profiler 是性能事实来源。

不要让多个地方保存互相矛盾的事实。

## A.79 Project Policy

Project Policy 是项目级策略。

例如：

1. 禁止修改 migrations。
2. Bash 只能运行测试。
3. Edit 配置文件需要 ask。
4. 只允许特定 MCP server。

项目策略比用户临时偏好更适合团队协作。

## A.80 Organization Policy

Organization Policy 是组织级策略。

它通常比项目策略更高优先级。

例如：

1. 禁止上传 telemetry 原文。
2. 禁止执行部署命令。
3. 禁止访问某些路径。
4. 强制启用审计。

## A.81 Frontmatter

Frontmatter 是 Markdown 文件开头的结构化元数据。

示例：

```md
---
name: code-reviewer
tools:
  - Read
  - Grep
maxTurns: 8
---
```

Custom Agent 常用 frontmatter 定义 name、description、tools 等字段。

## A.82 Namespace

Namespace 是命名空间。

插件 Agent 需要命名空间，避免和内置 Agent 冲突。

例如：

```text
plugin:security/security-reviewer
```

## A.83 Schema Version

Schema Version 是 schema 版本。

Tool schema、event schema、transcript schema 都可能演进。记录版本能帮助兼容旧数据。

## A.84 Backward Compatibility

Backward Compatibility 是向后兼容。

例如旧 transcript 里有 progress entry，新版本加载时可以识别并桥接，但新版本不再写入这种格式。

这是系统演化中非常重要的能力。

## A.85 Human-in-the-loop

Human-in-the-loop 是人在环。

Agent 遇到高风险操作时，应让用户参与决策。

例如：

1. 执行部署。
2. 删除文件。
3. 覆盖配置。
4. 推送代码。
5. 访问敏感文件。

## A.86 Approval

Approval 是用户批准。

Ask 权限会产生 approval 流程。

Approval 应该有范围，例如：

1. 只批准这一次。
2. 本会话批准。
3. 对某个命令前缀批准。

不要把一次批准扩大成永久无限授权。

## A.87 Denial Recovery

Denial Recovery 是权限拒绝后的恢复策略。

好的 Agent 在工具被拒绝后会：

1. 解释限制。
2. 尝试低风险替代方案。
3. 告诉用户需要什么授权。
4. 不重复请求同一危险操作。

## A.88 Artifact

Artifact 是任务产物。

例如：

1. 修改后的文件。
2. diff。
3. eval report。
4. transcript。
5. profile。
6. 文档站。

Agent 不只生成文本，也生成工程产物。

## A.89 Local-first

Local-first 是本地优先。

代码 Agent 很适合本地优先，因为代码、git、测试、权限都在用户机器上。

本地优先不等于没有云端，而是敏感操作先在用户控制范围内发生。

## A.90 本附录小结

如果你是新手，不需要一次记住所有术语。建议先掌握这十个：

1. Agent Loop。
2. Tool。
3. Tool Schema。
4. Tool Use。
5. Tool Result。
6. Permission。
7. Transcript。
8. Context Budget。
9. Eval。
10. Replay。

这十个概念连起来，就是 AI Agent 工程的骨架。

等你开始做多 Agent、MCP、plugins、observability，再回来查其他术语，会更容易理解。
