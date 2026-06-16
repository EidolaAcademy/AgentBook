# Python 代码块审计记录

本文件记录对全书 Markdown 中 Python 代码块的机械检测结果。它不是最终修复报告，而是后续逐章清理代码示例的依据。

## 2026-06-15 检测结果

检测范围：

- `outputs/ai-agent-book/chapters/*.md`

检测项：

- Python 代码块数量。
- Python AST 语法解析。
- 完全相同代码块的哈希去重。
- 第 2 章和第 5 章的重点复查。

结果：

- Python 代码块总数：`685`
- Python 语法错误：`0`
- 全书完全重复代码块分组：`20`
- 涉及重复的 Python 代码块：`601`
- 第 2 章重复代码块分组：`0`
- 第 5 章重复代码块分组：`0`

已修复：

- 第 2 章：移除重复的 `Message / TranscriptEntry` 示例，改为消息历史、最小 Agent loop、工程级 query loop 三段递进代码。
- 第 2 章：修正错误源码路径，将不存在的 `src/mini_agent/...` 和 `.js` 路径改为当前仓库真实路径。
- 第 5 章：移除多处重复的 `Message / TranscriptEntry` 示例。
- 第 5 章：将 5.7 改为工具调用配对检查函数。
- 第 5 章：将 5.8 改为 content block、Message、TranscriptEntry、辅助构造函数。
- 第 5 章：将 5.9 改为 `FakeModelClient`。
- 第 5 章：将 5.10 改为 `render_message()`。
- 第 5 章：将 5.11 改为 `extract_tool_uses()`。
- 第 5 章：将 5.12 改为 fake tool call 测试。
- 第 5 章：修正消息协议相关 Claude Code 源码路径。

仍需清理的高频重复组：

1. `3a3221dd6cdb`：`155` 次，权限决策类示例，主要集中在第 8、9、10、11、13、15 章之后。
2. `3933b8def400`：`88` 次，`resolve_inside_workspace()` / `read_text_file()` 示例。
3. `e2e178ba4406`：`83` 次，`ToolUse` / `run_agent_step()` 旧模板。
4. `db3b2768e8db`：`62` 次，`Message / TranscriptEntry` 旧模板。
5. `74e074a561af`：`35` 次，权限策略旧模板。
6. `7f5b72a2df41`：`26` 次，路径解析变体。
7. `38abae05e198`：`26` 次，子 Agent 任务状态旧模板。
8. `c62360d09f09`：`23` 次，异步命令执行旧模板。
9. `867cdb4497ba`：`20` 次，worktree/subprocess 旧模板。
10. `082a59689b08`：`18` 次，工具注册或工具搜索旧模板。

清理原则：

- 不再只做语法转换。
- 每个代码块必须服务当前小节主题。
- 可以复用公共类型，但不能把同一段代码塞进语义不同的小节。
- 章节内若需要连续构建一个 mini-agent，应保证前后代码能自然衔接。
- 源码路径必须保留真实 Claude Code 仓库路径；教学实现可以使用 Python 文件名，但不能伪造原仓库路径。
