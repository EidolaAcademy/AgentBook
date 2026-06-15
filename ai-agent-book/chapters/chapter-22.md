# 第 3 卷：工具系统工程化

## 第 22 章：Bash 安全，不能只看命令开头

### 22.1 本章目标

mini-agent 的 Shell 工具目前用简单函数判断只读命令：

```python
isObviouslyReadOnly(command)
```

这只能用于教学。真实 Shell 安全远比这个复杂。

本章目标：

1. 理解为什么 Shell 是最高风险工具。
2. 理解字符串前缀判断为什么不可靠。
3. 学会识别危险 shell 语法。
4. 设计多层 Bash 安全策略。
5. 区分只读、写入、破坏性、开放网络命令。
6. 理解沙箱的作用和边界。
7. 对照 Claude Code 的 BashTool 安全模块。

### 22.2 Shell 是完整语言

很多人把 shell 命令当成“一个可执行程序加参数”：

```bash
ls src
```

但 shell 实际上是一门语言。它有：

- 管道。
- 重定向。
- 条件执行。
- 变量。
- 子命令。
- 通配符。
- 函数。
- alias。
- here document。

例如：

```bash
ls src && rm -rf dist
```

开头是 `ls`，但后面删除文件。

```bash
cat README.md > copy.md
```

开头是 `cat`，但它写文件。

```bash
echo $(rm -rf tmp)
```

看起来是 `echo`，但子命令会执行删除。

所以安全判断不能只看第一个单词。

### 22.3 危险语法清单

教学版可以先拒绝包含以下模式的命令，让它们进入 ask 或 deny：

```text
;
&&
||
|
>
>>
<
`
$(
sudo
curl ... | sh
wget ... | sh
rm
mv
chmod
chown
git reset --hard
git push --force
python -m build && twine upload dist/*
```

注意：这些模式不都是绝对危险。例如管道 `rg foo | head` 很常见，也可以安全。但在没有 shell parser 前，保守拒绝是合理的。

安全系统可以分阶段：

```text
阶段 1：保守拒绝复杂语法
阶段 2：引入 shell parser
阶段 3：基于 AST 判断只读/写入
阶段 4：沙箱执行部分命令
```

Claude Code 已经处于后面阶段：它有 bash parser、安全解析、读写约束、命令语义判断等。

### 22.4 命令分类

Shell 命令可以粗略分四类。

第一类：只读观察。

```bash
ls
pwd
cat README.md
rg "login"
git status
git diff
```

第二类：普通写入。

```bash
pip install
ruff format .
touch file
mkdir dir
```

第三类：破坏性操作。

```bash
rm -rf
git reset --hard
git clean -fd
```

第四类：开放世界/网络/发布。

```bash
curl
wget
python -m build && twine upload dist/*
git push
ssh
scp
```

权限策略可以不同：

```text
只读观察 -> default 自动允许
普通写入 -> ask
破坏性操作 -> 强警告或 deny
开放网络 -> ask 或 deny，取决于策略
```

### 22.5 简单危险检测

先写一个保守检测：

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

在 `shellTool.checkPermission()` 中：

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

这不是完整安全，但能挡住明显高危操作。

### 22.6 复杂语法检测

可以检测 shell operators：

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

如果复杂：

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

不要自动允许。

### 22.7 只读判断第二版

只读判断应该先排除复杂语法和危险模式：

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

这比第一版更保守。

### 22.8 为什么需要 shell parser

字符串检测永远有漏洞。

例如：

```bash
rm{IFS}-rf{IFS}dist
```

或者：

```bash
command rm -rf dist
```

或者 alias：

```bash
alias ls='rm -rf dist'; ls
```

真正要理解 shell，需要 parser 把命令变成 AST。然后分析每个 command、operator、redirect、substitution。

Claude Code 的 BashTool 中有 `parseForSecurity`、`splitCommandWithOperators`、`readOnlyValidation` 等逻辑，说明它不是只靠正则。

mini-agent 可以把 parser 作为后续增强，不在本章实现。

### 22.9 沙箱的作用

沙箱是另一层防护。

即使命令被允许，也可以放进受限环境：

```text
只能访问 workspace
不能访问 home
不能联网
不能写特定目录
```

沙箱不是权限系统的替代，而是第二道防线。

权限系统回答：

```text
是否允许尝试执行？
```

沙箱回答：

```text
即使执行，也限制它能影响哪里。
```

Claude Code 有 sandbox adapter 和 sandbox UI utils。不同平台沙箱能力不同，因此实现很复杂。

### 22.10 shellTool 权限策略升级

可以升级：

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

plan 模式仍然会在全局权限层拒绝非只读命令。

bypass 模式会允许所有命令，但是否应该绕过 dangerous deny？这是设计选择。

更安全的设计：

```text
bypass 绕过 ask，但不绕过 policy deny。
```

也就是说，用户可以少确认，但组织策略和硬安全规则仍然生效。

### 22.11 命令语义

有些命令退出码需要语义解释。

例如：

```bash
grep pattern file
```

exit code 1 可能只是没匹配，不是错误。

```bash
test -f file
```

exit code 1 也是正常条件结果。

Claude Code 的 `commandSemantics.py` 会解释命令结果，帮助 UI 和模型理解。

mini-agent 暂时只返回 exit code，但后续可以加入：

```python
interpretCommandResult(command, exitCode, stdout, stderr)
```

返回：

```python
from typing import Any

def example(context: dict[str, Any]) -> dict[str, Any]:
    return {"ok": True, "context": context}
```

### 22.12 长时间运行命令

有些命令本来就不会结束：

```bash
python -m mini_agent
tail -f log.txt
watch ...
```

Agent 如果直接运行，会等到超时。

策略：

1. 检测明显长期命令，要求后台运行。
2. 给 shell 工具加 `runInBackground`。
3. 提供 task output 工具读取后台任务输出。

Claude Code 的 BashTool 支持后台任务和任务输出路径。mini-agent 后面可以实现。

### 22.13 本章练习

练习一：实现 `detectDangerousShellCommand()`。

至少检测：

- `rm -rf`
- `git reset --hard`
- `git push`
- `python -m build && twine upload dist/*`
- `curl | sh`

练习二：实现 `hasComplexShellSyntax()`。

检测：

- 管道。
- 重定向。
- `&&`
- `;`
- command substitution。

练习三：升级 `isReadOnlyShellCommand()`。

先排除危险和复杂语法，再判断只读命令。

练习四：升级 shellTool 权限策略。

危险命令 deny，非只读 ask，只读 allow。

练习五：写测试。

确保：

```text
ls -> read-only
cat README.md > copy.md -> not read-only
rm -rf dist -> dangerous
git status -> read-only
git push -> dangerous or ask
```

### 22.14 本章小结

本章我们深入 Shell 安全。

你应该理解：

1. Shell 是完整语言，不能只看第一个词。
2. 只读判断要保守。
3. 危险命令应被 deny 或强警告。
4. 复杂 shell 语法应进入 ask。
5. 沙箱是权限之外的第二道防线。
6. 生产级 BashTool 需要 parser、语义分析、权限规则和沙箱组合。

下一章我们会进入 MCP 工具包装。MCP 让 Agent 能接入外部工具，但也带来命名、schema、权限和信任边界问题。
