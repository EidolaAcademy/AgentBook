# 第 29 章：审计、日志与安全可解释性

一个 Agent 做出动作后，用户迟早会问：

```text
刚才它为什么这么做？
谁允许它这么做？
它到底执行了什么？
结果是什么？
有没有改文件？
有没有访问网络？
有没有被某条规则拒绝？
```

如果系统回答不了这些问题，就不是真正可用于工程场景的 Agent。

安全系统不只是阻止危险行为，还要让行为可解释、可追踪、可复盘。

本章讲审计、日志与安全可解释性。

## 29.1 为什么 Agent 需要审计

传统程序一般有明确调用路径：

```text
用户点击按钮 -> 后端接口 -> 数据库写入
```

Agent 的路径更复杂：

```text
用户自然语言 -> 模型推理 -> 工具调用 -> 权限判断 -> hook -> 沙箱 -> 工具结果 -> 下一轮推理
```

中间每一步都可能影响最终行为。

如果没有审计记录，你只能看到结果：

```text
文件被改了。
命令执行了。
消息发出去了。
```

却不知道：

- 是哪轮模型决定的。
- 工具输入是什么。
- 是否经过用户确认。
- 命中了哪条权限规则。
- 是否被 hook 修改过 input。
- 是否在沙箱内执行。
- 执行失败是否因沙箱限制。
- 结果有没有被 hook 修改。

这会让用户不信任 Agent，也会让开发者无法调试。

## 29.2 三类记录：Transcript、Analytics、Telemetry

成熟 Agent 通常会有三类记录。

第一类：Transcript。

它是会话记录。主要用于：

- 用户回看。
- 恢复会话。
- 调试上下文。
- 展示工具调用和结果。

第二类：Analytics。

它是产品分析事件。主要用于：

- 统计功能使用。
- 了解错误率。
- 统计权限批准/拒绝漏斗。
- 衡量性能。

第三类：Telemetry。

它是工程观测数据。主要用于：

- 跟踪耗时。
- 关联请求。
- 监控工具执行。
- 排查线上问题。

这三类不要混在一起。

```text
Transcript 可以包含较多上下文，但通常本地存储或受用户控制。
Analytics 应该尽量不包含敏感内容。
Telemetry 可以包含结构化细节，但要受开关和脱敏控制。
```

Claude Code 源码里也能看到这种分工：

- `useLogMessages` 记录 transcript。
- `logEvent` 记录 analytics。
- `logOTelEvent` 记录 telemetry。
- `logPermissionDecision` 统一记录权限决策，并分发到多个系统。

## 29.3 Transcript：会话的事实记录

Transcript 是用户最容易理解的记录。

它应该回答：

```text
这次会话发生了什么？
```

典型 transcript 包含：

- 用户消息。
- assistant 消息。
- tool_use。
- tool_result。
- 系统消息。
- compact boundary。
- hook 附加消息。
- 权限决策提示。
- 工具错误。

Claude Code 的 `useLogMessages` 会把新消息追加记录到 transcript，而且处理了 compaction、增量记录、消息 UUID、父子关系等问题。

为什么需要 UUID 和 parent？

因为工具调用和工具结果不是普通文本流。你需要知道：

```text
这个 tool_result 对应哪个 tool_use。
这条消息属于哪个 session。
这条消息是否在 compact 后仍然有效。
```

教学版 transcript 结构可以这样：

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

这个版本简单，但已经有关键字段。

## 29.4 toolUseID 是审计链条的核心

每次工具调用都应该有一个唯一 ID。

例如：

```text
toolu_123
```

它连接：

- 模型输出的 tool_use。
- 权限决策。
- hook 结果。
- 工具执行。
- 工具结果。
- telemetry span。
- UI 展示。

没有 toolUseID，就很难回答：

```text
这个错误结果对应哪个工具请求？
这次权限确认批准的是哪个输入？
这个 hook 阻止的是哪次调用？
```

教学版：

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

每个工具调用创建一条记录，后续阶段不断补充。

## 29.5 权限决策要集中记录

权限决策非常重要，不应该散落在各个 UI 组件里。

Claude Code 有一个集中函数：

```text
logPermissionDecision()
```

它负责：

- 记录 analytics event。
- 记录 OTel tool_decision。
- 更新 code edit counter。
- 把 decision 存到 toolUseContext.toolDecisions。

这就是好设计。

教学版也应该做一个统一入口：

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

不要让每个权限分支自己随手写日志。

否则迟早出现：

- 某些 allow 有记录，某些没有。
- 某些 reject 记录了原因，某些没有。
- analytics 和 transcript 不一致。
- UI 显示的来源和 telemetry 来源不一致。

## 29.6 决策来源比决策本身更重要

只记录：

```text
allow
```

不够。

你要记录：

```text
allow by config
allow by user once
allow by user permanent
allow by hook
allow by classifier
deny by policy rule
deny by user reject
deny by safety check
deny by sandbox override policy
```

Claude Code 里会把来源映射成 OTel source：

```text
config
hook
user_permanent
user_temporary
user_reject
```

教学版可以更细：

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

审计时用户想知道的不是“通过了”，而是“谁让它通过的”。

## 29.7 decisionReason 要结构化

上一卷我们已经讲过 `PermissionDecisionReason`。在审计里，它尤其重要。

不要只写：

```text
reason: "denied"
```

应该结构化：

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

结构化 reason 的好处：

- UI 可以友好展示。
- telemetry 可以聚合统计。
- 测试可以断言。
- 规则来源可以追踪。
- 后续自动解释可以基于类型生成。

例如：

```python
def explainDecision(reason: PermissionDecisionReason):
    def switch(self, reason.type):
        case "rule":
        return "Matched {reason.rule.source} {reason.rule.behavior} rule {formatRule(reason.rule)}"
        case "hook":
        return "Decided by hook {reason.hookName}: {reason.reason  or  "no reason provided"}"
        case "mode":
        return "Decided by permission mode {reason.mode}"
        case "safetyCheck":
        return "Blocked by safety check: {reason.reason}"
        "default":
        return reason.reason
```

## 29.8 用户等待时间也值得记录

权限提示会打断用户。

如果用户每次都等很久才确认，说明：

- 提示不够清楚。
- 规则建议不够好。
- Agent 动作太频繁。
- 默认权限策略太保守。

Claude Code 的 permission logging 会记录：

```text
waiting_for_user_permission_ms
```

教学版也可以记录：

```python
started = time.time()
choice = await ui.askPermission(request)
waitMs = time.time() - started
```

这不是为了监控用户，而是为了优化产品体验。

如果某类工具平均等待很久，可能要改 UI 文案或规则建议。

## 29.9 Analytics 不能随便记录敏感内容

Agent 处理的是代码、文件路径、命令、环境、外部 API 数据。这些都可能敏感。

Analytics 事件尤其要谨慎，因为它们可能发送到远端。

Claude Code 的 analytics metadata 里有明显的隐私意识：

- tool name 会 sanitize。
- tool input 遥测需要开关。
- 长字符串会截断。
- 深层对象会变成 `<nested>`。
- 内部 `_` 字段会跳过。
- 文件扩展名也会做长度限制，避免敏感 hash 泄露。

教学版要有同样意识。

不要这样：

```python
analytics.track("tool_called",:
    "command": input.command
    "fullFileContent": content
    "env": process.env
```

更合理：

```python
analytics.track("tool_called",:
    toolName
    "commandType": getCommandType(input.command)
    "fileExtension": getSafeExtension(filePath)
    success
    durationMs
```

敏感内容放 transcript 本地记录也要谨慎，更不要默认发到远程 analytics。

## 29.10 Telemetry 细节要受开关控制

Telemetry 用于工程排查，可能需要更多细节。

例如：

- 工具输入。
- 耗时。
- token 数。
- 模型 request id。
- tool result 大小。

但这些细节也可能敏感。

所以可以提供开关：

```text
OTEL_LOG_TOOL_DETAILS=true
```

只有开启时才记录工具输入摘要。

即使开启，也要：

- 截断长字符串。
- 限制数组长度。
- 限制对象深度。
- 去掉内部字段。
- 脱敏 token。

教学版：

```python
import json

def extractToolInputForTelemetry(input: Any):
    if (not settings.telemetry.logToolDetails) return None
    return json.dumps(truncateAndRedact(input))
```

这能在可观测性和隐私之间保持平衡。

## 29.11 审计记录应该包含哪些字段

一条完整工具审计记录可以包含：

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

这不是必须一次写全，但方向要对。

尤其是：

```text
toolUseId
toolName
inputPreview
permission source
decision reason
execution result
```

这几个字段非常关键。

## 29.12 输出预览和完整结果分离

工具输出可能很大。

例如：

```bash
pytest
cat large.log
grep -R ...
```

审计记录不能无限保存完整输出。

更好的方式：

```text
Transcript 保存模型看到的结果或摘要。
本地文件保存大输出。
审计记录保存 output path、大小、hash、preview。
```

教学版：

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

这样既能复盘，又不会把 transcript 撑爆。

## 29.13 Hook 审计

上一章讲 hooks。Hooks 也必须审计。

每个 hook 至少记录：

- hook 名称。
- hook 事件。
- 输入摘要。
- 输出摘要。
- 是否超时。
- 是否报错。
- 耗时。
- 是否修改 input。
- 是否做出权限决策。

否则，当 hook 阻止工具时，用户只会看到：

```text
Execution stopped by hook.
```

但不知道哪个 hook、为什么。

教学版：

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

Hook 是扩展点，越是扩展点越要有审计。

## 29.14 沙箱审计

沙箱也要记录。

每次 Bash 命令可以记录：

```text
是否 sandboxed。
为什么 sandboxed / 为什么未 sandboxed。
是否使用 dangerouslyDisableSandbox。
是否命中 excludedCommands。
是否出现 violation。
violation 类型和目标。
```

特别是无沙箱执行，要有清楚来源：

```text
用户明确要求。
命令命中 excludedCommands。
沙箱不可用。
策略允许绕过。
```

教学版：

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

当出现安全事故时，这些字段能快速回答：

```text
为什么这个命令没有进沙箱？
```

## 29.15 审计 UI 怎么展示

给用户看的审计不应该是一坨 JSON。

可以做成时间线：

```text
12:01:03  Assistant requested Bash
          command: pytest

12:01:03  Permission allowed
          source: session rule Bash(pytest)

12:01:03  Sandbox enabled
          write: project, $TMPDIR
          network: none

12:01:04  Command finished
          exit code: 1
          duration: 1.2s

12:01:04  PostToolUseFailure hook added context
          hook: test-report-summary
```

这种展示比：

```json
{ ...巨大对象... }
```

更适合人类。

但底层仍然应该保存结构化数据，方便搜索和导出。

## 29.16 审计和模型上下文不是一回事

不要把所有审计信息都塞给模型。

模型需要的是和下一步任务相关的信息。  
审计系统需要的是完整复盘信息。

例如：

```text
用户等待权限 1423ms
OTel source=user_temporary
analytics event=tengu_tool_use_granted_in_prompt_temporary
```

这些对模型没什么帮助，不应该进入上下文。

但：

```text
命令被拒绝，因为路径在工作目录之外。
```

这对模型有帮助，应该作为 tool_result 或系统消息告诉模型。

区分：

```text
For model
For user
For audit
For telemetry
```

这是成熟 Agent 的基本能力。

## 29.17 错误日志要对模型有恢复价值

工具错误不仅要给用户看，也要帮助模型修正。

坏错误：

```text
Error.
```

好错误：

```text
Denied by project rule Bash(rm *).
Try a narrower non-destructive command, or ask the user to approve deletion.
```

坏错误：

```text
Validation failed.
```

好错误：

```text
Input validation failed: field "paths" must be an array. This tool was deferred and its schema was not loaded. Call ToolSearch with query "select:mcp__x__y" first.
```

日志和错误信息不是装饰。它们直接影响 Agent 的自我修复能力。

## 29.18 审计存储和保留策略

Transcript 和审计记录不能无限保存。

需要保留策略：

```text
保存最近 N 天。
保存最近 N 个会话。
用户可清除。
敏感项目可禁用 transcript。
企业可配置保留策略。
```

Claude Code 设置里也有 transcript retention 相关提示。

教学版可以先做：

```python
import json
from pathlib import Path
from typing import Any

def read_json(path: Path) -> dict[str, Any]:
    return json.loads(path.read_text(encoding="utf-8"))

def read_jsonl(path: Path) -> list[dict[str, Any]]:
    if not path.exists():
        return []
    return [json.loads(line) for line in path.read_text(encoding="utf-8").splitlines() if line.strip()]

def load_agent_task(task_id: str, root: Path) -> dict[str, Any]:
    metadata = read_json(root / ".agent" / "tasks" / task_id / "metadata.json")
    transcript = read_jsonl(root / ".agent" / "subagents" / f"{task_id}.jsonl")
    return {"id": task_id, "metadata": metadata, "transcript": transcript}
```

启动时清理过期：

```python
async def cleanupOldTranscripts(policy: RetentionPolicy):
    if (not policy.enabled) return deleteAllTranscripts()
    // 删除超过 maxDays 的记录
```

安全工具必须给用户可控性。

## 29.19 审计数据的安全

审计数据本身也敏感。

它可能包含：

- 文件路径。
- 命令。
- 错误输出。
- 外部服务对象 ID。
- 部分代码片段。
- 用户输入。

所以审计存储要考虑：

- 存在哪里。
- 文件权限。
- 是否加密。
- 是否上传。
- 如何清除。
- 如何导出。
- 谁能访问。

不要因为“这是日志”就降低安全标准。

## 29.20 教学版 AuditStore

我们实现一个简单 AuditStore。

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

执行工具时：

```python
audit.record(:
    "type": "tool_requested"
    toolUseId
    "toolName": tool.name
    "inputPreview": sanitizeForAudit(input)
    "timestamp": time.time()
```

权限决策时：

```python
audit.record(:
    "type": "permission_decision"
    toolUseId
    "decision": "accept"
    "source": "user_once"
    "reason": "User approved in prompt"
    "timestamp": time.time()
```

工具结束时：

```python
audit.record(:
    "type": "tool_finished"
    toolUseId
    "success": True
    durationMs
    outputPreview
    "timestamp": time.time()
```

第一版可以内存存储。之后再写入本地 JSONL。

## 29.21 JSONL 是简单可靠的本地审计格式

本地审计可以用 JSONL。

每行一个事件：

```json
{"type":"tool_requested","toolUseId":"1","toolName":"Bash","timestamp":...}
{"type":"permission_decision","toolUseId":"1","decision":"accept","source":"session_rule","timestamp":...}
{"type":"tool_finished","toolUseId":"1","success":true,"durationMs":812,"timestamp":...}
```

优点：

- 追加写简单。
- 崩溃时不容易损坏整个文件。
- 可以用命令行检索。
- 易于导入数据库。

写入时要注意：

- 追加模式。
- 每行独立 JSON。
- 敏感字段脱敏。
- 大字段不要直接写。

## 29.22 审计测试清单

最低测试：

1. 每次 tool_use 都生成 tool_requested。
2. allow 决策记录 source。
3. deny 决策记录 reason。
4. 用户确认记录 waitMs。
5. hook 阻止工具时记录 hookName。
6. hook 修改 input 时记录 updatedInput=true。
7. Bash 执行记录 sandboxed。
8. 沙箱违规记录 target。
9. 大输出只记录 preview 和 path。
10. analytics 不包含完整命令中的敏感 token。
11. telemetry 开关关闭时不记录工具输入。
12. transcript 能通过 toolUseId 找到 tool_result。
13. compaction 后 transcript 仍能恢复关键状态。
14. 清理策略会删除过期 transcript。

审计测试不是形式主义。它能防止你以后改执行流程时漏记关键事件。

## 29.23 常见错误

错误一：只记录工具结果，不记录权限来源。

正确做法：记录 decision、source、reason。

错误二：analytics 记录完整输入。

正确做法：远程 analytics 只记录脱敏聚合字段。

错误三：transcript 和 telemetry 混在一起。

正确做法：区分用户可见记录、产品分析、工程观测。

错误四：没有 toolUseID。

正确做法：每次工具调用都有唯一 ID，并贯穿全链路。

错误五：hook 决策没有审计。

正确做法：记录 hookName、event、duration、输出摘要。

错误六：沙箱绕过没有原因。

正确做法：记录 overrideRequested、approvedBy、reason。

错误七：大输出全部写进日志。

正确做法：preview + path + hash。

错误八：日志里保存环境变量。

正确做法：默认不记录 env，必要时白名单。

## 29.24 本章练习

练习一：实现 `AuditStore`。

支持：

- record
- forToolUse
- exportJsonl

练习二：为工具执行流程加入三类事件：

- tool_requested
- permission_decision
- tool_finished

练习三：实现 `sanitizeForAnalytics()`。

要求：

- 去掉 `_` 开头字段。
- 截断长字符串。
- 限制对象深度。

练习四：实现 `PermissionDecisionReason` 的解释函数。

练习五：实现 JSONL 审计文件写入。

练习六：实现 transcript 查看命令。

根据 toolUseId 展示完整时间线。

练习七：实现大输出审计。

输出超过 20KB 时保存到文件，审计只保存 preview 和 path。

练习八：实现 telemetry 开关。

关闭时不记录 tool input。

练习九：写测试：用户临时允许和永久允许在日志里 source 不同。

练习十：思考题：

```text
哪些信息应该给模型看？哪些只应该进本地审计？哪些可以进远程 analytics？
```

## 29.25 本章小结

本章我们讲了审计、日志与安全可解释性。

你应该理解：

1. 安全系统不仅要做决策，还要能解释决策。
2. Transcript、analytics、telemetry 是三类不同记录。
3. toolUseID 是工具调用审计链条的核心。
4. 权限决策必须集中记录。
5. 决策来源和 decisionReason 比 allow/deny 本身更有价值。
6. Analytics 不能默认记录敏感内容。
7. Telemetry 细节要受开关和脱敏控制。
8. Hook、沙箱、大输出都要审计。
9. 审计数据本身也需要安全和保留策略。
10. 好错误信息能帮助模型自我修复。

到这里，第 4 卷“权限、安全与沙箱”就形成了完整闭环：规则决定能不能做，沙箱限制做的范围，hooks 允许外部策略参与，审计让一切可复盘。

下一章我们进入第 5 卷：子 Agent、任务分解与多 Agent 协作。
