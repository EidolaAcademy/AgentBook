# 第 33 章：worktree isolation 与并行实现策略

上一章我们讲了子 Agent 的上下文隔离。上下文隔离解决的是“消息、权限、工具状态不要互相污染”。但如果子 Agent 会修改代码，仅有上下文隔离还不够。因为它们最终操作的是同一个文件系统。

这一章讨论一个更硬的工程问题：多个 Agent 同时改代码时，怎样避免互相覆盖？

Claude Code 的答案之一是 worktree isolation。简单说，就是给某个子 Agent 创建一个独立的 git worktree，让它在自己的目录里工作。这样它可以读文件、改文件、跑测试，而不会直接影响主工作区。任务结束后，系统检查它有没有改动：如果没有改动，就自动清理；如果有改动，就保留 worktree 路径和分支，让用户或父 Agent 决定如何合并。

这章会从零讲清楚：

1. 为什么共享工作区会让多 Agent 变危险。
2. git worktree 是什么。
3. Agent worktree 和普通 session worktree 有什么区别。
4. 如何创建、使用、清理 worktree。
5. 如何处理大目录、gitignored 文件、本地设置和 hooks。
6. 如何检测子 Agent 是否产生修改。
7. 如何把隔离修改合并回主工作区。
8. 如何设计并行 Agent 的冲突处理策略。

如果你想做能真正改代码的多 Agent 系统，这一章非常关键。

## 33.1 共享工作区为什么危险

先看一个场景。父 Agent 同时启动两个子 Agent：

- Agent A：修复 `src/auth/token.py` 中的刷新逻辑。
- Agent B：修复 `src/auth/session.py` 中的并发登录问题。

听起来它们改的是不同文件，好像没问题。但真实项目中，很可能两个任务都会触碰同一个工具函数、同一个测试文件、同一个类型定义。

如果它们共享工作区，会出现这些问题。

第一，读取到的文件状态不一致。

Agent A 在 10:00 读取 `token.py`。Agent B 在 10:01 修改了同一个文件。Agent A 在 10:02 基于旧内容写回。结果 Agent B 的修改被覆盖。

第二，测试结果不可归因。

Agent A 修改后运行测试，刚好 Agent B 也改了文件。测试失败到底是谁导致的？很难判断。

第三，diff 混在一起。

两个 Agent 都改完后，主工作区只有一坨混合 diff。你不知道每个改动来自哪个任务。

第四，回滚困难。

如果 Agent B 的方案不好，你想只撤销 B 的修改。但修改已经和 A 的修改混在一起，撤销会很痛苦。

第五，父 Agent 失去控制。

父 Agent 原本想比较多个方案，最后选择一个。但如果多个方案都直接写进主工作区，就没有“比较”这个阶段了。

所以，一旦允许并行写代码，隔离工作区就不是高级功能，而是安全基础。

## 33.2 git worktree 是什么

`git worktree` 是 Git 自带功能。它允许同一个仓库有多个工作目录，每个工作目录可以检出不同分支。

假设主仓库在：

```txt
/repo/my-app
```

你可以创建一个新 worktree：

```bash
git worktree add ../my-app-agent-1 -b agent-1
```

此时会出现：

```txt
/repo/my-app
/repo/my-app-agent-1
```

两个目录共享同一个 `.git` 对象数据库，但工作区文件是独立的。

这意味着：

- 不需要完整复制整个仓库历史。
- 每个 worktree 可以有自己的工作区文件。
- 每个 worktree 可以在不同分支。
- 一个 worktree 的未提交修改不会直接出现在另一个 worktree。

对 Agent 来说，这正合适。我们可以为每个会写代码的子 Agent 创建一个临时 worktree：

```txt
主工作区：
  /repo/my-app

子 Agent A：
  /repo/my-app/.claude/worktrees/agent-a1b2c3d4

子 Agent B：
  /repo/my-app/.claude/worktrees/agent-b5c6d7e8
```

每个子 Agent 在自己的目录里运行。

## 33.3 worktree isolation 在 AgentTool 中的位置

在 Claude Code 的 `AgentTool` 中，worktree isolation 大致发生在这个位置：

```txt
选择 selectedAgent
        |
        v
计算 effectiveIsolation
        |
        v
如果 isolation === "worktree"
        |
        v
创建 agent worktree
        |
        v
用 worktreePath 包装子 Agent 执行 cwd
        |
        v
子 Agent 完成
        |
        v
检测 worktree 是否有修改
        |
        v
无修改：删除 worktree
有修改：保留 worktreePath / worktreeBranch
```

教学版可以写成：

```python
from dataclasses import dataclass
from pathlib import Path
import subprocess

@dataclass
class AgentWorktree:
    path: Path
    branch: str

def create_agent_worktree(repo: Path, branch: str, target: Path) -> AgentWorktree:
    subprocess.run(["git", "worktree", "add", str(target), branch], cwd=repo, check=True)
    return AgentWorktree(path=target, branch=branch)

def remove_agent_worktree(repo: Path, worktree: AgentWorktree) -> None:
    subprocess.run(["git", "worktree", "remove", str(worktree.path)], cwd=repo, check=True)
```

注意一个细节：worktree 是子 Agent 的工作目录，不是父 Agent 的工作目录。父 Agent 不应该突然 `chdir` 到 worktree；只应该让子 Agent 在 worktree 中运行。

## 33.4 Agent worktree 与 session worktree 的区别

源码中有两类 worktree 思路。

第一类是 session worktree。它面向整个会话。用户进入一个隔离工作区后，整个 Claude Code 会话都在这个 worktree 中运行。

第二类是 agent worktree。它只给某个子 Agent 使用，不改变全局 session 状态。

区别如下：

```txt
session worktree：
- 改变当前会话 cwd
- 需要保存 activeWorktreeSession
- 用户可能长期在里面工作
- 清理或保留由会话生命周期决定

agent worktree：
- 不改变全局 cwd
- 只传给子 Agent 作为 cwd override
- 通常是临时的
- 无修改自动删除，有修改保留
```

教学版一开始建议只实现 agent worktree。因为它更适合多 Agent 并行，且不会影响用户当前工作区。

## 33.5 最小可用 createAgentWorktree

我们先实现一个最小版本。

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

这已经能工作，但还不够安全。接下来我们逐步补。

## 33.6 slug 校验：防止路径逃逸

创建 worktree 时，我们会把 slug 拼到路径里：

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

如果 slug 是：

```txt
../../../somewhere
```

就可能逃出 worktrees 目录。生产系统必须校验 slug。

可以实现：

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

Claude Code 源码里也做了类似事情。这个细节很重要，因为 Agent 名称、任务 ID、分支名、路径名很容易来自动态输入。任何会进入文件路径的字符串都必须校验。

## 33.7 为什么要 flatten slug

如果允许 slug 中出现 `/`，例如：

```txt
user/feature-foo
```

路径上会变成嵌套目录：

```txt
.claude/worktrees/user/feature-foo
```

分支名可能变成：

```txt
worktree-user/feature-foo
```

这会带来两个问题。

第一，Git ref 可能出现文件/目录冲突。比如已经有 `worktree-user` 分支，再创建 `worktree-user/feature` 就会冲突。

第二，目录嵌套可能导致清理父 worktree 时误删子 worktree。

源码中的做法是把 `/` 替换成 `+`：

```python
def flattenSlug(slug: str):
    return slug.replaceAll('/', '+')

def worktreeBranchName(slug: str):
    return "worktree-{flattenSlug(slug)}"
```

这样：

```txt
user/feature-foo
```

变成：

```txt
user+feature-foo
```

这个映射很聪明，因为 `+` 不在 slug segment 的允许字符里，所以不会和原始输入混淆。

新手版本如果不需要嵌套 slug，可以更简单：直接禁止 `/`。但你要理解源码为什么允许再 flatten，这是为了兼顾可读性和安全性。

## 33.8 worktree 应该放在哪里

一种做法是放到仓库外面：

```txt
../my-app-agent-1
```

另一种做法是放到仓库内部的隐藏目录：

```txt
.claude/worktrees/agent-a1b2c3d4
```

Claude Code 采用类似后者的方式。好处是：

- 所有临时 worktree 都集中在项目下。
- 易于清理。
- 易于恢复。
- 易于在会话列表中关联项目。

但要注意，这个目录应该被 gitignore 忽略。否则你会把 worktree 目录加入仓库，这是灾难。

可以在 `.gitignore` 中加入：

```gitignore
.agent/worktrees/
.claude/worktrees/
```

如果你做自己的 Agent 框架，也建议固定一个隔离目录，例如：

```txt
.agent/worktrees/
```

## 33.9 创建 worktree 时用哪个 base

最简单的是从当前 HEAD 创建：

```bash
git worktree add -B worktree-agent-a1b2c3d4 .agent/worktrees/agent-a1b2c3d4 HEAD
```

但生产系统会遇到更多选择：

1. 从当前 HEAD 创建。
2. 从默认远程分支创建。
3. 从某个 PR 创建。
4. 从某个指定 commit 创建。

Claude Code 的源码中会尝试使用默认分支的远程引用。如果本地已有 `origin/<defaultBranch>`，就尽量避免 fetch，因为 fetch 可能慢，也可能因认证提示卡住。如果没有远程引用，才尝试 fetch，失败则退回 HEAD。

这个设计很实用。Agent 系统不能随便运行一个可能卡住的网络命令。

教学版可以先简单：

```python
baseRef = options.baseRef  or  'HEAD'
```

然后逐步增强：

```python
async def resolveWorktreeBase(gitRoot: str):
    defaultBranch = await getDefaultBranch(gitRoot)
    originRef = "origin/{defaultBranch}"

    def if(self, await refExists(gitRoot, originRef)):
        return originRef

    return 'HEAD'
```

原则是：不要为了“最新”牺牲可靠性。一个稍微旧一点但能快速创建的 worktree，通常比一个卡在 fetch 的 worktree 更有用。

## 33.10 禁止 Git 交互式提示

创建 worktree 时，系统可能执行 `git fetch`。如果远程需要凭据，Git 可能弹出交互提示或等待输入。这对 Agent 来说非常危险，因为后台任务可能永远挂住。

源码中会设置类似：

```python
GIT_NO_PROMPT_ENV =:
    "GIT_TERMINAL_PROMPT": '0'
    "GIT_ASKPASS": ''
```

教学版也应该这样：

```python
await execFileAsync(
'git'
['fetch', 'origin', defaultBranch]

    "cwd": gitRoot
    "env"::
        ...process.env
        "GIT_TERMINAL_PROMPT": '0'
        "GIT_ASKPASS": ''
)
```

Agent 系统里的外部命令有个基本原则：不能依赖交互式输入。需要输入就应该明确失败，并把错误返回给父 Agent 或用户。

## 33.11 post-creation setup：创建完 worktree 后还没结束

很多新手以为 `git worktree add` 完就可以了。但真实项目里，一个新的 worktree 常常缺东西：

- 本地配置。
- gitignored 的环境文件。
- hooks。
- node_modules 或其他依赖目录。
- 构建缓存。

Claude Code 的 `performPostCreationSetup` 做了几类事情：

1. 复制本地 settings。
2. 配置 git hooks。
3. 按设置 symlink 大目录。
4. 按 `.worktreeinclude` 复制 gitignored 文件。

我们逐个讲。

## 33.12 复制本地设置

Agent 在 worktree 中运行时，仍然可能需要项目本地配置。例如：

```txt
.claude/settings.local.json
.agent/settings.local.json
```

如果这些文件没有复制，子 Agent 的权限、MCP、hook 或项目规则可能和主工作区不同。

教学版：

```python
from dataclasses import dataclass
from pathlib import Path
import subprocess

@dataclass
class AgentWorktree:
    path: Path
    branch: str

def create_agent_worktree(repo: Path, branch: str, target: Path) -> AgentWorktree:
    subprocess.run(["git", "worktree", "add", str(target), branch], cwd=repo, check=True)
    return AgentWorktree(path=target, branch=branch)

def remove_agent_worktree(repo: Path, worktree: AgentWorktree) -> None:
    subprocess.run(["git", "worktree", "remove", str(worktree.path)], cwd=repo, check=True)
```

注意，这类配置可能包含敏感信息。复制前要明确它是本地工作所需，并且 worktree 目录不应该被提交。

## 33.13 处理 gitignored 文件：.worktreeinclude

有些项目需要 gitignored 文件才能运行。例如：

```txt
.env.local
config/dev.secret.json
certs/local.pem
```

这些文件不会被 Git 检出到新 worktree。子 Agent 如果需要跑测试，可能会失败。

一种解决方式是 `.worktreeinclude`：

```txt
.env.local
config/dev.secret.json
certs/*.pem
```

创建 worktree 后，系统读取 `.worktreeinclude`，把匹配的 gitignored 文件复制过去。

教学版：

```python
from dataclasses import dataclass
from pathlib import Path
import subprocess

@dataclass
class AgentWorktree:
    path: Path
    branch: str

def create_agent_worktree(repo: Path, branch: str, target: Path) -> AgentWorktree:
    subprocess.run(["git", "worktree", "add", str(target), branch], cwd=repo, check=True)
    return AgentWorktree(path=target, branch=branch)

def remove_agent_worktree(repo: Path, worktree: AgentWorktree) -> None:
    subprocess.run(["git", "worktree", "remove", str(worktree.path)], cwd=repo, check=True)
```

生产版要支持 gitignore 语法、目录折叠、glob、性能优化和路径安全。新手版先支持明确文件路径即可。

## 33.14 symlink 大目录，避免磁盘膨胀

如果每个 worktree 都复制 `node_modules`，磁盘会爆炸。git worktree 本身不会复制未跟踪目录，但某些 setup 可能需要依赖目录存在。

Claude Code 支持通过设置把某些目录 symlink 到 worktree：

```json
{
  "worktree": {
    "symlinkDirectories": [".venv", ".pytest_cache"]
  }
}
```

教学版：

```python
from dataclasses import dataclass
from pathlib import Path
import subprocess

@dataclass
class AgentWorktree:
    path: Path
    branch: str

def create_agent_worktree(repo: Path, branch: str, target: Path) -> AgentWorktree:
    subprocess.run(["git", "worktree", "add", str(target), branch], cwd=repo, check=True)
    return AgentWorktree(path=target, branch=branch)

def remove_agent_worktree(repo: Path, worktree: AgentWorktree) -> None:
    subprocess.run(["git", "worktree", "remove", str(worktree.path)], cwd=repo, check=True)
```

这里要小心路径穿越，不能让配置写成：

```txt
../../somewhere
```

另外，symlink 共享目录可能带来副作用。例如子 Agent 如果修改 `node_modules`，其实会影响主工作区。所以它适合依赖缓存，不适合会被任务修改的目录。

## 33.15 hooks：worktree 中也要遵守项目规则

很多项目通过 Git hooks 做提交检查、格式化、commit message 校验等。如果 worktree 没有正确配置 hooks，Agent 在 worktree 中的行为可能绕过项目规则。

Claude Code 会尝试配置 worktree 使用主仓库 hooks，例如 `.husky` 或 `.git/hooks`。还会处理某些 hooksPath 被重新设置的问题。

教学版可以先做简单版：

```python
import asyncio
from dataclasses import dataclass
from typing import Any

@dataclass
class HookResult:
    status: str
    message: str = ""

async def run_hook(name: str, payload: dict[str, Any], timeout: float = 5.0) -> HookResult:
    try:
        await asyncio.wait_for(asyncio.sleep(0), timeout=timeout)
        return HookResult("ok", f"hook 已执行: {name}")
    except asyncio.TimeoutError:
        return HookResult("timeout", f"hook 超时: {name}")
```

但要知道，`core.hooksPath` 在多 worktree 场景下可能写入共享 git config，不一定是 worktree-local。生产系统要更谨慎。

## 33.16 cwd override：让子 Agent 在 worktree 中运行

创建 worktree 后，系统必须确保子 Agent 的所有文件和 shell 操作都在 worktree 中执行。

不要这样：

```python
process.chdir(worktreePath)
await runChildAgent()
process.chdir(originalCwd)
```

全局 `chdir` 在并发系统里非常危险。另一个 Agent 或工具可能同时运行，突然发现 cwd 变了。

更好的方式是 cwd override。也就是在上下文中记录 cwd：

```python
childContext =:
    ...parentContext
    "cwd": worktreePath
```

工具执行时使用这个 cwd：

```python
async def runBash(command: str, ctx: ToolUseContext):
    return execFileAsync('bash', ['-lc', command],:
        "cwd": ctx.cwd
```

读文件时也基于 ctx.cwd 解析相对路径：

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

Claude Code 中有 `runWithCwdOverride` 这类机制，本质就是避免全局 cwd 污染。

## 33.17 给子 Agent 注入 worktree 提示

如果子 Agent 是 fork path，可能继承了父 Agent 的上下文。父上下文中的路径指向主工作区，而子 Agent 实际运行在 worktree 中。这时必须提醒它路径变了。

可以注入一条消息：

```txt
你正在一个隔离 worktree 中运行。
原始工作区路径：/repo/my-app
当前 worktree 路径：/repo/my-app/.agent/worktrees/agent-a1b2c3d4
如果历史消息中出现原始路径，请转换到当前 worktree。
请重新读取你要修改的文件，因为 worktree 内容可能和历史上下文不同。
```

这个提示能减少一种常见错误：子 Agent 根据父上下文里的旧文件内容直接编辑，而没有在 worktree 中重新读取最新文件。

## 33.18 检测 worktree 是否有修改

子 Agent 结束后，系统要决定清理还是保留 worktree。

最简单的检测：

```bash
git status --porcelain
```

如果有输出，就说明有修改。

教学版：

```python
from dataclasses import dataclass
from pathlib import Path
import subprocess

@dataclass
class AgentWorktree:
    path: Path
    branch: str

def create_agent_worktree(repo: Path, branch: str, target: Path) -> AgentWorktree:
    subprocess.run(["git", "worktree", "add", str(target), branch], cwd=repo, check=True)
    return AgentWorktree(path=target, branch=branch)

def remove_agent_worktree(repo: Path, worktree: AgentWorktree) -> None:
    subprocess.run(["git", "worktree", "remove", str(worktree.path)], cwd=repo, check=True)
```

源码中还会基于创建时的 head commit 做更细的判断。因为有些情况下工作树干净，但分支 HEAD 已经变化，例如子 Agent 提交了 commit。

更完整：

```python
from dataclasses import dataclass
from pathlib import Path
import subprocess

@dataclass
class AgentWorktree:
    path: Path
    branch: str

def create_agent_worktree(repo: Path, branch: str, target: Path) -> AgentWorktree:
    subprocess.run(["git", "worktree", "add", str(target), branch], cwd=repo, check=True)
    return AgentWorktree(path=target, branch=branch)

def remove_agent_worktree(repo: Path, worktree: AgentWorktree) -> None:
    subprocess.run(["git", "worktree", "remove", str(worktree.path)], cwd=repo, check=True)
```

这样可以检测：

- 未提交修改。
- 暂存修改。
- 新提交。

## 33.19 无修改自动清理

如果子 Agent 没有产生修改，worktree 应该自动删除。否则临时目录会越来越多。

```python
from dataclasses import dataclass
from pathlib import Path
import subprocess

@dataclass
class AgentWorktree:
    path: Path
    branch: str

def create_agent_worktree(repo: Path, branch: str, target: Path) -> AgentWorktree:
    subprocess.run(["git", "worktree", "add", str(target), branch], cwd=repo, check=True)
    return AgentWorktree(path=target, branch=branch)

def remove_agent_worktree(repo: Path, worktree: AgentWorktree) -> None:
    subprocess.run(["git", "worktree", "remove", str(worktree.path)], cwd=repo, check=True)
```

注意要从主仓库根目录执行，而不是从 worktree 目录执行。因为你正在删除 worktree，cwd 不能在被删除目录里。

清理函数应该尽量幂等。也就是说，多调用一次不应该造成严重错误。因为真实系统里 finally、错误处理、后台任务结束回调可能重复触发。

## 33.20 有修改时保留 worktree

如果子 Agent 修改了文件，就不能自动删除。否则用户会丢失工作成果。

应该返回：

```python
from dataclasses import dataclass
from typing import Any, Protocol

@dataclass
class WorktreeResult:
    worktreePath: str
    worktreeBranch: str
```

给父 Agent 的结果可以包含：

```txt
子 Agent 在隔离 worktree 中产生了修改：
worktreePath: /repo/my-app/.agent/worktrees/agent-a1b2c3d4
worktreeBranch: worktree-agent-a1b2c3d4

请检查该 worktree 的 diff 后决定是否合并。
```

Claude Code 的 AgentTool 也是类似：如果 worktree 没变化，删除；如果有变化，保留并把路径和分支放进结果。

## 33.21 如何查看子 Agent 的 diff

有了 worktreePath 后，可以运行：

```bash
git -C /path/to/worktree diff
git -C /path/to/worktree status --short
```

教学版可以提供工具函数：

```python
from dataclasses import dataclass
from pathlib import Path
import subprocess

@dataclass
class AgentWorktree:
    path: Path
    branch: str

def create_agent_worktree(repo: Path, branch: str, target: Path) -> AgentWorktree:
    subprocess.run(["git", "worktree", "add", str(target), branch], cwd=repo, check=True)
    return AgentWorktree(path=target, branch=branch)

def remove_agent_worktree(repo: Path, worktree: AgentWorktree) -> None:
    subprocess.run(["git", "worktree", "remove", str(worktree.path)], cwd=repo, check=True)
```

但返回给父 Agent 时要注意 diff 大小。可以先返回 stat 和文件列表，让父 Agent 选择性读取。

```python
from dataclasses import dataclass
from pathlib import Path
import subprocess

@dataclass
class AgentWorktree:
    path: Path
    branch: str

def create_agent_worktree(repo: Path, branch: str, target: Path) -> AgentWorktree:
    subprocess.run(["git", "worktree", "add", str(target), branch], cwd=repo, check=True)
    return AgentWorktree(path=target, branch=branch)

def remove_agent_worktree(repo: Path, worktree: AgentWorktree) -> None:
    subprocess.run(["git", "worktree", "remove", str(worktree.path)], cwd=repo, check=True)
```

## 33.22 如何合并 worktree 修改

合并方式有很多，取决于你是否让子 Agent 提交 commit。

方式一：直接复制 patch。

```bash
git -C /path/to/worktree diff > /tmp/agent.patch
git apply /tmp/agent.patch
```

优点是简单。缺点是如果 worktree 中有新提交或复杂状态，处理起来麻烦。

方式二：让子 Agent commit，然后 cherry-pick。

```bash
git -C /path/to/worktree add .
git -C /path/to/worktree commit -m "agent fix auth flow"
git cherry-pick worktree-agent-a1b2c3d4
```

优点是 Git 原生处理冲突。缺点是要求子 Agent 能提交，且 commit message、作者信息要规范。

方式三：父 Agent 读取 diff，然后在主工作区重新应用。

这更安全，但成本高。父 Agent 可以审查子 Agent 的 diff，再用 Edit 工具在主工作区做同样修改。

对新手系统，建议先实现方式一：

```python
from dataclasses import dataclass
from pathlib import Path
import subprocess

@dataclass
class AgentWorktree:
    path: Path
    branch: str

def create_agent_worktree(repo: Path, branch: str, target: Path) -> AgentWorktree:
    subprocess.run(["git", "worktree", "add", str(target), branch], cwd=repo, check=True)
    return AgentWorktree(path=target, branch=branch)

def remove_agent_worktree(repo: Path, worktree: AgentWorktree) -> None:
    subprocess.run(["git", "worktree", "remove", str(worktree.path)], cwd=repo, check=True)
```

Python 的 `execFile` 对 stdin 支持不如 spawn 直观，实际实现可以用 `spawn`。这里重点是流程。

## 33.23 合并前必须做冲突预检查

不要直接把子 Agent 的 diff 应用到主工作区。先检查主工作区状态。

```python
from typing import Any

def example(context: dict[str, Any]) -> dict[str, Any]:
    return {"ok": True, "context": context}
```

也可以允许主工作区有修改，但要做更精细的文件重叠检查。

```python
from typing import Any

def example(context: dict[str, Any]) -> dict[str, Any]:
    return {"ok": True, "context": context}
```

如果子 Agent 修改的文件和主工作区修改的文件重叠，就要求用户确认。

## 33.24 多个子 Agent 并行时如何合并

假设有三个子 Agent：

- A 修改 `auth/token.py`
- B 修改 `auth/session.py`
- C 修改 `auth/token.py` 和 `auth/session.py`

合并顺序会影响结果。你需要一个策略。

最简单策略：按完成顺序尝试 apply patch，失败就保留冲突让用户处理。

更好的策略：先按文件集合分组。

```python
from typing import Any

def example(context: dict[str, Any]) -> dict[str, Any]:
    return {"ok": True, "context": context}
```

非冲突组可以更安全地合并。冲突组需要人工或更强 Agent 做仲裁。

## 33.25 父 Agent 如何比较多个方案

worktree isolation 还有一个很大的好处：它允许多个子 Agent 提出不同方案。

例如父 Agent 可以启动两个修复 Agent：

```txt
Agent A：请用最小改动修复这个测试。
Agent B：请从架构角度重构相关逻辑，但控制范围。
```

两个 Agent 分别在不同 worktree 中修改。父 Agent 最后比较：

- 修改文件数量。
- diff 大小。
- 测试结果。
- 风险。
- 可读性。
- 与用户需求匹配程度。

结果可能是：

```txt
Agent A 的方案更小，只改 2 个文件，相关测试通过。
Agent B 的方案更完整，但改动 8 个文件，引入额外风险。
我建议采用 Agent A 的方案。
```

没有 worktree isolation，这种“并行试验”很难做。

## 33.26 测试 worktree isolation

worktree 功能必须测试，因为它涉及文件系统和 Git 状态。

测试一：创建 worktree。

```python
from dataclasses import dataclass
from pathlib import Path
import subprocess

@dataclass
class AgentWorktree:
    path: Path
    branch: str

def create_agent_worktree(repo: Path, branch: str, target: Path) -> AgentWorktree:
    subprocess.run(["git", "worktree", "add", str(target), branch], cwd=repo, check=True)
    return AgentWorktree(path=target, branch=branch)

def remove_agent_worktree(repo: Path, worktree: AgentWorktree) -> None:
    subprocess.run(["git", "worktree", "remove", str(worktree.path)], cwd=repo, check=True)
```

测试二：slug 路径逃逸被拒绝。

```python
from dataclasses import dataclass
from pathlib import Path
import subprocess

@dataclass
class AgentWorktree:
    path: Path
    branch: str

def create_agent_worktree(repo: Path, branch: str, target: Path) -> AgentWorktree:
    subprocess.run(["git", "worktree", "add", str(target), branch], cwd=repo, check=True)
    return AgentWorktree(path=target, branch=branch)

def remove_agent_worktree(repo: Path, worktree: AgentWorktree) -> None:
    subprocess.run(["git", "worktree", "remove", str(worktree.path)], cwd=repo, check=True)
```

测试三：无修改自动清理。

```python
from dataclasses import dataclass
from pathlib import Path
import subprocess

@dataclass
class AgentWorktree:
    path: Path
    branch: str

def create_agent_worktree(repo: Path, branch: str, target: Path) -> AgentWorktree:
    subprocess.run(["git", "worktree", "add", str(target), branch], cwd=repo, check=True)
    return AgentWorktree(path=target, branch=branch)

def remove_agent_worktree(repo: Path, worktree: AgentWorktree) -> None:
    subprocess.run(["git", "worktree", "remove", str(worktree.path)], cwd=repo, check=True)
```

测试四：有修改时保留。

```python
from dataclasses import dataclass
from pathlib import Path
import subprocess

@dataclass
class AgentWorktree:
    path: Path
    branch: str

def create_agent_worktree(repo: Path, branch: str, target: Path) -> AgentWorktree:
    subprocess.run(["git", "worktree", "add", str(target), branch], cwd=repo, check=True)
    return AgentWorktree(path=target, branch=branch)

def remove_agent_worktree(repo: Path, worktree: AgentWorktree) -> None:
    subprocess.run(["git", "worktree", "remove", str(worktree.path)], cwd=repo, check=True)
```

测试五：子 Agent 的 cwd 是 worktree。

```python
from dataclasses import dataclass
from pathlib import Path
import subprocess

@dataclass
class AgentWorktree:
    path: Path
    branch: str

def create_agent_worktree(repo: Path, branch: str, target: Path) -> AgentWorktree:
    subprocess.run(["git", "worktree", "add", str(target), branch], cwd=repo, check=True)
    return AgentWorktree(path=target, branch=branch)

def remove_agent_worktree(repo: Path, worktree: AgentWorktree) -> None:
    subprocess.run(["git", "worktree", "remove", str(worktree.path)], cwd=repo, check=True)
```

这类测试一开始麻烦，但能避免非常多真实项目事故。

## 33.27 常见错误一：直接 process.chdir

全局 `process.chdir` 在单任务脚本里还可以，在并发 Agent 系统里很危险。

错误模式：

```python
process.chdir(worktreePath)
await runAgent()
process.chdir(originalCwd)
```

如果另一个异步任务在中间运行，它会继承错误 cwd。

正确模式：

```python
from dataclasses import dataclass
from pathlib import Path
import subprocess

@dataclass
class AgentWorktree:
    path: Path
    branch: str

def create_agent_worktree(repo: Path, branch: str, target: Path) -> AgentWorktree:
    subprocess.run(["git", "worktree", "add", str(target), branch], cwd=repo, check=True)
    return AgentWorktree(path=target, branch=branch)

def remove_agent_worktree(repo: Path, worktree: AgentWorktree) -> None:
    subprocess.run(["git", "worktree", "remove", str(worktree.path)], cwd=repo, check=True)
```

或者更明确地把 cwd 放进 context，让所有工具用 context.cwd。

## 33.28 常见错误二：无修改也保留所有 worktree

如果每个子 Agent 都留下 worktree，项目目录很快会堆满：

```txt
.agent/worktrees/agent-1111
.agent/worktrees/agent-2222
.agent/worktrees/agent-3333
...
```

无修改 worktree 应该自动删除。有修改 worktree 才保留。

还可以做定期清理，比如删除 30 天前无活动的临时 worktree。但清理必须谨慎，只清理符合固定命名模式的临时 worktree，不要误删用户命名的 worktree。

## 33.29 常见错误三：忽略 gitignored 配置文件

如果新 worktree 缺 `.env.local`，测试可能失败。Agent 可能误以为代码有问题，然后做错误修改。

解决方法不是盲目复制所有 gitignored 文件，因为里面可能有巨大缓存或敏感文件。更好的方式是 `.worktreeinclude`，让项目显式声明哪些 gitignored 文件应该复制。

## 33.30 常见错误四：合并时不检查主工作区

如果主工作区已经有用户修改，直接 apply 子 Agent patch 可能覆盖用户工作。

合并前至少检查：

```bash
git status --porcelain
```

如果不干净，就让用户或父 Agent先审查。不要为了自动化牺牲用户工作。

## 33.31 常见错误五：把 worktree 当安全沙箱

worktree 只隔离文件修改。它不是安全沙箱。

子 Agent 在 worktree 中运行 Bash，仍然可能：

- 访问网络。
- 读取用户目录。
- 删除 worktree 外的文件。
- 使用系统命令。
- 消耗 CPU 和磁盘。

所以 worktree isolation 必须和权限系统、命令沙箱、文件访问限制一起使用。不要以为“在 worktree 中运行”就安全了。

## 33.32 一个完整的教学版 worktree 管理器

下面是一个合并版示例：

```python
from dataclasses import dataclass
from pathlib import Path
import subprocess

@dataclass
class AgentWorktree:
    path: Path
    branch: str

def create_agent_worktree(repo: Path, branch: str, target: Path) -> AgentWorktree:
    subprocess.run(["git", "worktree", "add", str(target), branch], cwd=repo, check=True)
    return AgentWorktree(path=target, branch=branch)

def remove_agent_worktree(repo: Path, worktree: AgentWorktree) -> None:
    subprocess.run(["git", "worktree", "remove", str(worktree.path)], cwd=repo, check=True)
```

这个函数表达了核心策略：

- 不启用隔离时，直接用主 cwd。
- 启用隔离时，创建 worktree。
- 在 worktree cwd 中运行子 Agent。
- 成功后无修改清理。
- 有修改保留。
- 失败时如果无修改也清理。

生产系统还要把 worktree 信息写入 metadata，方便 resume。

## 33.33 本章小结

这一章我们讲了 worktree isolation。它解决的是多 Agent 修改代码时的文件系统隔离问题。

核心原则如下：

1. 共享工作区不适合并行写代码。
2. git worktree 可以为每个子 Agent 提供独立工作目录。
3. Agent worktree 不应该改变全局 cwd，而应该通过 cwd override 传给子 Agent。
4. worktree slug 必须校验，防止路径逃逸。
5. 创建 worktree 后要处理本地设置、gitignored 文件、hooks 和大目录。
6. 子 Agent 结束后要检测修改。
7. 无修改自动清理，有修改保留路径和分支。
8. 合并前必须检查主工作区和文件冲突。
9. worktree 不是安全沙箱，仍然需要权限系统和命令沙箱。

到这里，我们已经有了多 Agent 修改代码的基本工程底座：上下文隔离负责消息和状态，worktree isolation 负责文件系统。下一章会继续讲多 Agent 任务编排、取消、恢复与冲突处理，也就是当多个 Agent 真正并行跑起来后，父系统如何调度它们、观察它们、停止它们和整合它们。
