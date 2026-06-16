# 第 2 卷：最小 Agent 实现

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
  "path": "src/auth/login.py"
}
```

但搜索工具解决的是：你不知道路径，只知道线索。

例如：

```text
我想找所有包含 "login" 的文件。
```

或者：

```text
我想找所有 .python 文件。
```

这两类搜索不同。

第一类是路径搜索，也就是 glob：

```text
src/**/*.python
**/*auth*
**/*.test.py
```

第二类是内容搜索，也就是 grep：

```text
login
process.env.API_KEY
function parseUser
```

在真实项目里，Agent 常常先 glob，再 grep，再 read：

```text
glob_files("src/**/*.py")
  ↓
grep_files("login")
  ↓
read_file("src/auth/login.py")
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
  "include": "**/*.py"
}
```

比生成 shell 命令：

```bash
rg "login" -g "**/*.py"
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
src/mini_agent/tools/GlobTool/
src/mini_agent/tools/GrepTool/
src/mini_agent/tools/BashTool/BashTool.tsx
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

```python
from pathlib import Path

def resolve_inside_workspace(workspace: Path, user_path: str) -> Path:
    root = workspace.resolve()
    target = (root / user_path).resolve()
    if target != root and root not in target.parents:
        raise ValueError(f"路径越界: {user_path}")
    return target

def read_text_file(workspace: Path, user_path: str, limit: int = 200) -> str:
    file_path = resolve_inside_workspace(workspace, user_path)
    lines = file_path.read_text(encoding="utf-8").splitlines()
    return "\n".join(lines[:limit])
```

字段含义：

`pattern` 是 glob 模式，例如 `src/**/*.py`。

`maxResults` 是最多返回多少条，默认可以设为 100。

输出：

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

`truncated` 表示结果是否因为数量限制被截断。这个字段很重要，因为模型看到截断后，可能会换更具体的搜索模式。

### 7.7 grep_files 的输入设计

再设计 `grep_files`：

```python
from pathlib import Path

def resolve_inside_workspace(workspace: Path, user_path: str) -> Path:
    root = workspace.resolve()
    target = (root / user_path).resolve()
    if target != root and root not in target.parents:
        raise ValueError(f"路径越界: {user_path}")
    return target

def read_text_file(workspace: Path, user_path: str, limit: int = 200) -> str:
    file_path = resolve_inside_workspace(workspace, user_path)
    lines = file_path.read_text(encoding="utf-8").splitlines()
    return "\n".join(lines[:limit])
```

字段含义：

`pattern` 是要搜索的文本。

`include` 是可选的文件模式，例如 `**/*.py`。

`maxResults` 限制命中数量。

输出：

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

def get_worktree_diff(worktree: AgentWorktree) -> str:
    result = subprocess.run(["git", "diff"], cwd=worktree.path, text=True, capture_output=True, check=False)
    return result.stdout
```

再写判断函数：

```python
def shouldIgnoreDir(name: str):
    return DEFAULT_IGNORED_DIRS.has(name)
```

真实项目里，忽略规则会更复杂。你可能需要读取 `.gitignore`，还要支持用户配置、插件缓存排除、隐藏目录策略等。Claude Code 里有 `src/mini_agent/utils/git/gitignore.py`、`src/mini_agent/utils/plugins/orphanedPluginFilter.js` 等相关工具，也会优先使用 ripgrep 这类成熟搜索工具来尊重 ignore 规则。

教学项目先手写一小组默认规则，足够说明思路。

### 7.10 文件遍历函数

搜索工具需要遍历工作区。我们写一个异步递归函数。

创建 `src/mini_agent/tools/search/walkFiles.py`：

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

严格实现 glob 并不简单。`**/*.py`、`*.{ts,python}`、`!ignore`、字符范围等都有细节。

生产项目建议使用成熟库，例如：

- `fast-glob`
- `minimatch`
- `picomatch`
- `glob`

但为了教学，我们先实现一个非常简化的 glob：

```python
from dataclasses import dataclass
from typing import Any

@dataclass
class ToolUse:
    id: str
    name: str
    input: dict[str, Any]

async def run_agent_step(model: Any, messages: list[dict[str, Any]], tools: dict[str, Any]) -> dict[str, Any]:
    response = await model.complete(messages, tools)
    messages.append(response)
    return response
```

这个版本支持：

- `*`：匹配单个路径段中的任意字符。
- `**`：跨目录匹配。

例如：

```text
*.md          -> README.md
src/*.py      -> src/index.py
src/**/*.py   -> src/a/b/c.py
**/README*    -> docs/README.zh.md
```

它不支持完整 glob 语法，但足够让新手理解搜索工具的核心。

### 7.12 实现 glob_files 工具

创建 `src/mini_agent/tools/globFilestool.py`：

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

注意格式化结果中的提示：

```text
Results were truncated. Use a more specific pattern.
```

这是给模型看的，不只是给用户看的。模型看到这句话后，下一步可能会换成更具体的模式，比如从 `**/*.py` 改成 `src/**/*.py`。

这类“面向模型的错误和提示”是 Agent 工程里非常重要的写作方式。

### 7.13 grep_files 的文件读取策略

`grep_files` 需要读取很多文件。这里有几个风险：

第一，不能读取二进制文件。二进制内容转字符串可能乱码，也可能非常大。

第二，不能读取超大文件。比如日志、bundle、lockfile 都可能很大。

第三，不能让一个文件读取失败导致整个搜索失败。某个文件权限错误时，可以跳过。

先写一个简单判断：

```python
from pathlib import Path

def resolve_inside_workspace(workspace: Path, user_path: str) -> Path:
    root = workspace.resolve()
    target = (root / user_path).resolve()
    if target != root and root not in target.parents:
        raise ValueError(f"路径越界: {user_path}")
    return target

def read_text_file(workspace: Path, user_path: str, limit: int = 200) -> str:
    file_path = resolve_inside_workspace(workspace, user_path)
    lines = file_path.read_text(encoding="utf-8").splitlines()
    return "\n".join(lines[:limit])
```

这不是完美判断。没有扩展名的文本文件会被跳过，某些扩展名文件也可能是二进制。但教学项目可以先这样。

生产级 FileReadTool 会更谨慎。Claude Code 的 `FileReadTool` 会检测二进制扩展、图片扩展、PDF、Notebook，并做不同处理。

### 7.14 实现 grep_files 工具

创建 `src/mini_agent/tools/grepFilestool.py`：

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

```python

def createToolRegistry():
    registry = ToolRegistry()
    registry.register(readFileTool)
    registry.register(globFilesTool)
    registry.register(grepFilesTool)
    return registry
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

当用户说“搜索 login”时，调用 `grep_files`：

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

收到 tool_result 后，FakeModel 可以简单总结：

```python
from dataclasses import dataclass

@dataclass
class SearchResult:
    tool_name: str
    score: int
    description: str

def search_tools(query: str, tools: list[dict[str, str]], limit: int = 5) -> list[SearchResult]:
    terms = [term.lower() for term in query.split() if term.strip()]
    results: list[SearchResult] = []
    for tool in tools:
        haystack = f"{tool.get('name', '')} {tool.get('description', '')}".lower()
        score = sum(1 for term in terms if term in haystack)
        if score:
            results.append(SearchResult(tool.get("name", ""), score, tool.get("description", "")))
    return sorted(results, key=lambda item: item.score, reverse=True)[:limit]
```

真实模型会根据结果决定是否继续读取文件。FakeModel 只是帮助我们验证链路。

### 7.17 搜索结果为什么要为模型写得清楚

工具结果不是给机器看的纯数据，也不是给用户看的最终报告。它介于两者之间：模型要读它、理解它、基于它继续行动。

所以搜索结果格式非常重要。

不好的结果：

```text
["a.py","b.py","c.py"]
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
Pattern: **/*.py
Matched files: 100 (truncated)

...

Results were truncated. Use a more specific pattern.
```

这会引导模型下一步缩小搜索范围。

同理，grep 结果采用传统格式：

```text
src/auth/login.py:42: def login(...)
```

这种格式对人和模型都友好，而且模型可以直接提取路径和行号。

### 7.18 搜索工具与并发

`glob_files` 和 `grep_files` 都是只读工具，所以我们声明：

```python
from dataclasses import dataclass

@dataclass
class SearchResult:
    tool_name: str
    score: int
    description: str

def search_tools(query: str, tools: list[dict[str, str]], limit: int = 5) -> list[SearchResult]:
    terms = [term.lower() for term in query.split() if term.strip()]
    results: list[SearchResult] = []
    for tool in tools:
        haystack = f"{tool.get('name', '')} {tool.get('description', '')}".lower()
        score = sum(1 for term in terms if term in haystack)
        if score:
            results.append(SearchResult(tool.get("name", ""), score, tool.get("description", "")))
    return sorted(results, key=lambda item: item.score, reverse=True)[:limit]
```

这意味着未来如果模型一次请求多个搜索：

```text
grep_files("login")
grep_files("logout")
glob_files("src/**/*.py")
```

Agent 可以并发执行它们。

上一章我们的 AgentEngine 还是串行执行：

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
