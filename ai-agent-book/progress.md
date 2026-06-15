# 书稿进度

目标总字数：不少于 50 万字。

当前阶段：第 6 卷生产工程、评估、回放与部署正在展开，书稿已改为每章独立 Markdown 文件。

当前章节正文规模：约 61 万字符。

已完成：

- 全书结构规划。
- 写作原则。
- 第 1 章：AI Agent 到底是什么。
- 第 2 章：从一次模型调用到 Agent 循环。
- 第 3 章：Claude Code 源码地图。
- 第 4 章：开发环境与第一个 CLI。
- 第 5 章：消息协议与工具调用。
- 第 6 章：Tool 接口与第一个 Read 工具。
- 第 7 章：搜索工具，让 Agent 学会探索项目。
- 第 8 章：Shell 工具，最强大也最危险的能力。
- 第 9 章：权限系统，给 Agent 装上刹车。
- 第 10 章：写文件与编辑文件，让 Agent 真正改变项目。
- 第 11 章：把工具串成完整任务闭环。
- 第 12 章：接入真实模型 API。
- 第 13 章：流式响应，让 Agent 更像实时助手。
- 第 14 章：上下文预算，避免 Agent 被历史淹没。
- 第 15 章：会话持久化与恢复。
- 第 16 章：测试 Agent，不要相信“看起来能跑”。
- 第 17 章：从 mini-agent 映射回 Claude Code 源码。
- 第 18 章：Tool Schema，模型和程序之间的合同。
- 第 19 章：工具执行生命周期。
- 第 20 章：并发工具调度，什么时候可以一起跑。
- 第 21 章：工具结果预算，大输出不能直接塞进上下文。
- 第 22 章：Bash 安全，不能只看命令开头。
- 第 23 章：MCP 工具包装，把外部能力接进 Agent。
- 第 24 章：ToolSearch，工具太多时如何延迟加载 schema。
- 第 25 章：权限、安全与沙箱总览。
- 第 26 章：权限规则，allow、deny、ask 如何匹配工具和输入。
- 第 27 章：沙箱，限制命令执行的真实边界。
- 第 28 章：Hooks 与外部策略，让 Agent 可被项目和组织治理。
- 第 29 章：审计、日志与安全可解释性。
- 第 30 章：子 Agent、任务分解与多 Agent 协作总览。
- 第 31 章：深入 AgentTool，如何把一个任务交给子 Agent。
- 第 32 章：子 Agent 上下文隔离、结果汇总与成本控制。
- 第 33 章：worktree isolation 与并行实现策略。
- 第 34 章：多 Agent 任务编排、取消、恢复与冲突处理。
- 第 35 章：teammate、agent swarms 与团队式协作。
- 第 36 章：Agent memory、skills preload 与长期协作能力。
- 第 37 章：自定义 Agent、插件 Agent 与分发机制。
- 第 38 章：评估、回放、调试与生产可观测性。
- 第 39 章：从 mini-agent 到生产级 Agent 平台的完整路线图。
- 第 40 章：部署、文档站、GitHub Pages 与发布流程。
- 附录 A：AI Agent 工程术语表。
- 附录 B：Claude Code 源码阅读索引与学习路线。

已完成结构调整：

- `chapters/` 中每章一个独立 Markdown 文件。
- `README.md` 和 `index.md` 作为 GitHub Pages 入口。
- `_config.yml` 作为 Jekyll/GitHub Pages 配置。
- `.github/workflows/pages.yml` 作为 GitHub Pages 部署 workflow。
- `DEPLOY.md` 记录部署到 GitHub Pages 的步骤。
- 已在 `outputs/ai-agent-book` 初始化本地 git 仓库，分支为 `main`。
- 已将全书教学代码示例从旧版脚本风格统一改为 Python 风格，保留 JSON、YAML、bash、Mermaid、目录树等必要非 Python 格式。

下一步：

1. 可继续扩展附录：mini-agent 完整代码清单、常见问题与练习答案。
2. 最终部署到 GitHub：需要目标 GitHub 仓库 URL、git user.email，或授权使用 GitHub CLI 创建仓库。

说明：

50 万字书稿需要多轮连续生成。本文件用于保持写作连续性，避免后续扩写偏离目录。
