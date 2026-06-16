# 附录 C：mini-agent 完整 Python 代码清单

前面章节为了讲解概念，很多地方只展示局部结构，例如 `Message`、`TranscriptEntry`、`ToolUse` 或某个工具函数。局部代码适合解释一个概念，但不适合新手直接照着运行。

本附录给出一个完整、单文件、可运行的 Python mini-agent。它不是 Claude Code 的复刻，而是把本书主线里的核心思想压缩成一个教学版本：

1. 消息结构。
2. 工具协议。
3. 工具注册表。
4. Read、Grep、Bash、Edit 工具。
5. 权限判断。
6. transcript JSONL 持久化。
7. Agent loop。
8. fake model。
9. CLI 入口。

运行方式：

```bash
python mini_agent_full.py
```

这个版本默认使用 `FakeModel`，不需要 API key。你可以先用它理解 Agent loop，再把 `FakeModel.complete()` 换成真实模型调用。

## C.1 完整单文件实现

将下面代码保存为 `mini_agent_full.py`：

```python
from __future__ import annotations

import asyncio
import json
import re
import shlex
import subprocess
import uuid
from dataclasses import asdict, dataclass, field
from datetime import datetime, timezone
from pathlib import Path
from typing import Any, Literal, Protocol


Role = Literal["system", "user", "assistant", "tool"]
PermissionDecision = Literal["allow", "deny", "ask"]
Content = list[dict[str, Any]] | str


def utc_now() -> str:
    return datetime.now(timezone.utc).isoformat()


@dataclass
class ToolUse:
    id: str
    name: str
    input: dict[str, Any]


@dataclass
class ToolResult:
    tool_use_id: str
    content: str
    is_error: bool = False


@dataclass
class Message:
    role: Role
    content: Content = ""
    tool_uses: list[ToolUse] = field(default_factory=list)
    tool_result: ToolResult | None = None


def content_to_text(content: Content) -> str:
    if isinstance(content, str):
        return content
    parts: list[str] = []
    for block in content:
        block_type = block.get("type")
        if block_type == "text":
            parts.append(str(block.get("text", "")))
        elif block_type == "tool_result":
            parts.append(str(block.get("content", "")))
    return "\n".join(part for part in parts if part)


def message_text(message: Message) -> str:
    if message.tool_result is not None:
        return message.tool_result.content
    return content_to_text(message.content)


@dataclass
class TranscriptEntry:
    kind: str
    session_id: str
    message: Message
    uuid: str = field(default_factory=lambda: str(uuid.uuid4()))
    parent_uuid: str | None = None
    created_at: str = field(default_factory=utc_now)


@dataclass
class ToolDefinition:
    name: str
    description: str
    input_schema: dict[str, Any]
    read_only: bool = True


class Tool(Protocol):
    definition: ToolDefinition

    async def call(self, tool_input: dict[str, Any], context: "RunContext") -> str:
        ...


@dataclass
class PermissionResult:
    decision: PermissionDecision
    reason: str = ""
    rule: str | None = None


@dataclass
class RunContext:
    cwd: Path
    session_id: str
    transcript: "TranscriptStore"
    permissions: "PermissionEngine"


class TranscriptStore:
    def __init__(self, path: Path) -> None:
        self.path = path
        self.path.parent.mkdir(parents=True, exist_ok=True)

    def append(self, entry: TranscriptEntry) -> None:
        with self.path.open("a", encoding="utf-8") as file:
            file.write(json.dumps(asdict(entry), ensure_ascii=False) + "\n")

    def load(self) -> list[dict[str, Any]]:
        if not self.path.exists():
            return []
        return [
            json.loads(line)
            for line in self.path.read_text(encoding="utf-8").splitlines()
            if line.strip()
        ]


def resolve_inside_workspace(cwd: Path, user_path: str) -> Path:
    root = cwd.resolve()
    target = (root / user_path).resolve()
    if target != root and root not in target.parents:
        raise ValueError(f"路径越界，拒绝访问工作区之外的文件: {user_path}")
    return target


class ReadTool:
    definition = ToolDefinition(
        name="Read",
        description="读取当前工作区内的 UTF-8 文本文件。",
        input_schema={
            "type": "object",
            "properties": {
                "file_path": {"type": "string"},
                "offset": {"type": "integer"},
                "limit": {"type": "integer"},
            },
            "required": ["file_path"],
        },
        read_only=True,
    )

    async def call(self, tool_input: dict[str, Any], context: RunContext) -> str:
        file_path = resolve_inside_workspace(context.cwd, str(tool_input["file_path"]))
        offset = int(tool_input.get("offset", 0))
        limit = int(tool_input.get("limit", 200))
        text = file_path.read_text(encoding="utf-8")
        lines = text.splitlines()
        selected = lines[offset : offset + limit]
        numbered = [f"{offset + index + 1:>4} | {line}" for index, line in enumerate(selected)]
        suffix = ""
        if offset + limit < len(lines):
            suffix = f"\n[已截断：共 {len(lines)} 行，仅显示 {offset + 1}-{offset + len(selected)} 行]"
        return "\n".join(numbered) + suffix


class GrepTool:
    definition = ToolDefinition(
        name="Grep",
        description="在当前工作区搜索文本，返回匹配文件和行号。",
        input_schema={
            "type": "object",
            "properties": {
                "pattern": {"type": "string"},
                "path": {"type": "string"},
                "max_results": {"type": "integer"},
            },
            "required": ["pattern"],
        },
        read_only=True,
    )

    ignored_dirs = {".git", ".venv", "__pycache__", ".pytest_cache", "dist", "build"}

    async def call(self, tool_input: dict[str, Any], context: RunContext) -> str:
        pattern = re.compile(str(tool_input["pattern"]))
        start = resolve_inside_workspace(context.cwd, str(tool_input.get("path", ".")))
        max_results = int(tool_input.get("max_results", 50))
        results: list[str] = []

        for file_path in start.rglob("*"):
            if len(results) >= max_results:
                break
            if not file_path.is_file():
                continue
            if any(part in self.ignored_dirs for part in file_path.parts):
                continue
            try:
                lines = file_path.read_text(encoding="utf-8").splitlines()
            except UnicodeDecodeError:
                continue
            for line_number, line in enumerate(lines, 1):
                if pattern.search(line):
                    rel = file_path.relative_to(context.cwd)
                    results.append(f"{rel}:{line_number}: {line}")
                    if len(results) >= max_results:
                        break

        if not results:
            return "未找到匹配结果。"
        return "\n".join(results)


class BashTool:
    definition = ToolDefinition(
        name="Bash",
        description="在当前工作区执行 shell 命令。危险命令会被权限系统拒绝或询问。",
        input_schema={
            "type": "object",
            "properties": {
                "command": {"type": "string"},
                "timeout": {"type": "number"},
            },
            "required": ["command"],
        },
        read_only=False,
    )

    async def call(self, tool_input: dict[str, Any], context: RunContext) -> str:
        command = str(tool_input["command"])
        timeout = float(tool_input.get("timeout", 30.0))
        process = await asyncio.create_subprocess_shell(
            command,
            cwd=str(context.cwd),
            stdout=asyncio.subprocess.PIPE,
            stderr=asyncio.subprocess.PIPE,
        )
        try:
            stdout, stderr = await asyncio.wait_for(process.communicate(), timeout=timeout)
        except asyncio.TimeoutError:
            process.kill()
            return f"命令超时：{command}"

        return (
            f"command: {command}\n"
            f"exit_code: {process.returncode}\n"
            f"stdout:\n{stdout.decode(errors='replace')[:12000]}\n"
            f"stderr:\n{stderr.decode(errors='replace')[:12000]}"
        )


class EditTool:
    definition = ToolDefinition(
        name="Edit",
        description="在文件中用 new_string 替换唯一匹配的 old_string。",
        input_schema={
            "type": "object",
            "properties": {
                "file_path": {"type": "string"},
                "old_string": {"type": "string"},
                "new_string": {"type": "string"},
            },
            "required": ["file_path", "old_string", "new_string"],
        },
        read_only=False,
    )

    async def call(self, tool_input: dict[str, Any], context: RunContext) -> str:
        file_path = resolve_inside_workspace(context.cwd, str(tool_input["file_path"]))
        old = str(tool_input["old_string"])
        new = str(tool_input["new_string"])
        text = file_path.read_text(encoding="utf-8")
        count = text.count(old)
        if count == 0:
            raise ValueError("old_string 没有匹配到任何内容。")
        if count > 1:
            raise ValueError(f"old_string 匹配了 {count} 次；为避免误改，拒绝编辑。")
        file_path.write_text(text.replace(old, new), encoding="utf-8")
        return f"已编辑文件：{file_path.relative_to(context.cwd)}"


class ToolRegistry:
    def __init__(self) -> None:
        self._tools: dict[str, Tool] = {}

    def register(self, tool: Tool) -> None:
        self._tools[tool.definition.name] = tool

    def get(self, name: str) -> Tool:
        if name not in self._tools:
            raise KeyError(f"未知工具：{name}")
        return self._tools[name]

    def definitions(self) -> list[ToolDefinition]:
        return [tool.definition for tool in self._tools.values()]


class PermissionEngine:
    dangerous_prefixes = ("rm -rf", "git push", "python -m build", "twine upload")

    def check(self, tool: Tool, tool_input: dict[str, Any]) -> PermissionResult:
        name = tool.definition.name
        if tool.definition.read_only:
            return PermissionResult("allow", rule=f"{name}(read_only)")

        if name == "Bash":
            command = str(tool_input.get("command", "")).strip()
            if command.startswith(self.dangerous_prefixes):
                return PermissionResult("deny", f"拒绝危险命令：{command}", "dangerous_bash")
            if command.startswith(("pytest", "ruff", "mypy", "git status", "git diff")):
                return PermissionResult("allow", rule="safe_validation_command")
            return PermissionResult("ask", f"是否允许执行命令：{command}", "bash_requires_confirmation")

        if name == "Edit":
            return PermissionResult("ask", "是否允许修改文件？", "edit_requires_confirmation")

        return PermissionResult("ask", f"是否允许执行工具：{name}")


def ask_user(question: str) -> bool:
    answer = input(f"{question} [y/N] ").strip().lower()
    return answer in {"y", "yes"}


class FakeModel:
    async def complete(self, messages: list[Message], tools: list[ToolDefinition]) -> Message:
        last_user = next((message_text(m) for m in reversed(messages) if m.role == "user"), "")
        lower = last_user.lower()

        if lower.startswith("read "):
            return Message(
                role="assistant",
                content="我将读取文件。",
                tool_uses=[ToolUse(str(uuid.uuid4()), "Read", {"file_path": last_user[5:].strip()})],
            )
        if lower.startswith("grep "):
            return Message(
                role="assistant",
                content="我将搜索文本。",
                tool_uses=[ToolUse(str(uuid.uuid4()), "Grep", {"pattern": last_user[5:].strip()})],
            )
        if lower.startswith("bash "):
            return Message(
                role="assistant",
                content="我将执行命令。",
                tool_uses=[ToolUse(str(uuid.uuid4()), "Bash", {"command": last_user[5:].strip()})],
            )
        if lower.startswith("edit "):
            return Message(
                role="assistant",
                content="Edit 示例格式：edit path | old | new",
                tool_uses=[],
            )
        return Message(
            role="assistant",
            content=(
                "FakeModel 已收到问题。可用示例：\n"
                "- read README.md\n"
                "- grep class Agent\n"
                "- bash pytest\n"
            ),
        )


class AgentEngine:
    def __init__(
        self,
        model: FakeModel,
        registry: ToolRegistry,
        context: RunContext,
        max_turns: int = 8,
    ) -> None:
        self.model = model
        self.registry = registry
        self.context = context
        self.max_turns = max_turns
        self.messages: list[Message] = [
            Message(role="system", content="你是一个谨慎的本地代码 Agent。")
        ]

    def record(self, kind: str, message: Message) -> None:
        self.context.transcript.append(
            TranscriptEntry(
                kind=kind,
                session_id=self.context.session_id,
                message=message,
            )
        )

    async def submit(self, user_text: str) -> str:
        user_message = Message(role="user", content=user_text)
        self.messages.append(user_message)
        self.record("user", user_message)

        final_text = ""
        for _ in range(self.max_turns):
            assistant = await self.model.complete(self.messages, self.registry.definitions())
            self.messages.append(assistant)
            self.record("assistant", assistant)
            final_text = message_text(assistant)

            if not assistant.tool_uses:
                return final_text

            for tool_use in assistant.tool_uses:
                tool_result = await self.run_tool(tool_use)
                tool_message = Message(role="tool", tool_result=tool_result)
                self.messages.append(tool_message)
                self.record("tool", tool_message)
                final_text = tool_result.content

        return f"达到最大轮数 {self.max_turns}，任务停止。最后结果：\n{final_text}"

    async def run_tool(self, tool_use: ToolUse) -> ToolResult:
        try:
            tool = self.registry.get(tool_use.name)
            permission = self.context.permissions.check(tool, tool_use.input)
            if permission.decision == "deny":
                return ToolResult(tool_use.id, f"权限拒绝：{permission.reason}", True)
            if permission.decision == "ask" and not ask_user(permission.reason):
                return ToolResult(tool_use.id, f"用户拒绝：{permission.reason}", True)
            output = await tool.call(tool_use.input, self.context)
            return ToolResult(tool_use.id, output)
        except Exception as error:
            return ToolResult(tool_use.id, f"工具执行失败：{error}", True)


def create_registry() -> ToolRegistry:
    registry = ToolRegistry()
    registry.register(ReadTool())
    registry.register(GrepTool())
    registry.register(BashTool())
    registry.register(EditTool())
    return registry


async def repl() -> None:
    cwd = Path.cwd()
    session_id = str(uuid.uuid4())
    transcript = TranscriptStore(cwd / ".mini-agent" / "sessions" / f"{session_id}.jsonl")
    context = RunContext(
        cwd=cwd,
        session_id=session_id,
        transcript=transcript,
        permissions=PermissionEngine(),
    )
    engine = AgentEngine(FakeModel(), create_registry(), context)

    print("mini-agent 已启动。输入 exit 退出。")
    print("示例：read README.md | grep class Agent | bash pytest")

    while True:
        user_text = input("> ").strip()
        if user_text in {"exit", "quit"}:
            break
        answer = await engine.submit(user_text)
        print(answer)


def main() -> None:
    asyncio.run(repl())


if __name__ == "__main__":
    main()
```

## C.2 这个完整版本对应哪些章节

这份代码可以和前面章节这样对应：

1. `Message`、`ToolUse`、`ToolResult` 对应第 5 章。
2. `ToolDefinition`、`Tool`、`ToolRegistry` 对应第 6 章和第 18 章。
3. `ReadTool` 对应第 6 章。
4. `GrepTool` 对应第 7 章。
5. `BashTool` 对应第 8 章和第 22 章。
6. `PermissionEngine` 对应第 9 章、第 25 章、第 26 章。
7. `EditTool` 对应第 10 章。
8. `TranscriptStore` 对应第 15 章和第 38 章。
9. `AgentEngine.submit()` 对应第 2 章、第 11 章、第 19 章。
10. `FakeModel` 对应第 4 章和第 16 章。

## C.3 为什么这份代码仍然是教学版

它已经能运行，但还不是生产级 Agent。它缺少：

1. 真实模型 API。
2. 流式响应。
3. 严格 JSON Schema 校验。
4. 工具并发调度。
5. 上下文 compact。
6. 子 Agent。
7. MCP。
8. 完整 eval runner。
9. 完整 sandbox。
10. telemetry 和 profiler。

但它已经具备最重要的闭环：

```text
用户输入 -> 模型决定工具 -> 权限判断 -> 工具执行 -> 工具结果写入 transcript -> 返回用户
```

这才是新手应该先跑通的核心。
