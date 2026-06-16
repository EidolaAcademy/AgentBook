---
layout: default
title: 从零搭建 AI Agent
---

# 从零搭建 AI Agent：从 Claude Code 源码到工程实战

这是一套面向新手的 AI Agent 工程教材，基于 Claude Code 源码结构进行架构拆解，并从零构建一个 mini-agent。

## 阅读目录

- [前言](chapters/00-preface.md)
- [第 1 章：AI Agent 到底是什么](chapters/chapter-01.md)
- [第 2 章：从一次模型调用到 Agent 循环](chapters/chapter-02.md)
- [第 3 章：Claude Code 源码地图](chapters/chapter-03.md)
- [第 4 章：开发环境与第一个 CLI](chapters/chapter-04.md)
- [第 5 章：消息协议与工具调用](chapters/chapter-05.md)
- [第 6 章：Tool 接口与第一个 Read 工具](chapters/chapter-06.md)
- [第 7 章：搜索工具，让 Agent 学会探索项目](chapters/chapter-07.md)
- [第 8 章：Shell 工具，最强大也最危险的能力](chapters/chapter-08.md)
- [第 9 章：权限系统，给 Agent 装上刹车](chapters/chapter-09.md)
- [第 10 章：写文件与编辑文件，让 Agent 真正改变项目](chapters/chapter-10.md)
- [第 11 章：把工具串成完整任务闭环](chapters/chapter-11.md)
- [第 12 章：接入真实模型 API](chapters/chapter-12.md)
- [第 13 章：流式响应，让 Agent 更像实时助手](chapters/chapter-13.md)
- [第 14 章：上下文预算，避免 Agent 被历史淹没](chapters/chapter-14.md)
- [第 15 章：会话持久化与恢复](chapters/chapter-15.md)
- [第 16 章：测试 Agent，不要相信“看起来能跑”](chapters/chapter-16.md)
- [第 17 章：从 mini-agent 映射回 Claude Code 源码](chapters/chapter-17.md)
- [第 18 章：Tool Schema，模型和程序之间的合同](chapters/chapter-18.md)
- [第 19 章：工具执行生命周期](chapters/chapter-19.md)
- [第 20 章：并发工具调度，什么时候可以一起跑](chapters/chapter-20.md)
- [第 21 章：工具结果预算，大输出不能直接塞进上下文](chapters/chapter-21.md)
- [第 22 章：Bash 安全，不能只看命令开头](chapters/chapter-22.md)
- [第 23 章：MCP 工具包装，把外部能力接进 Agent](chapters/chapter-23.md)
- [第 24 章：ToolSearch，工具太多时如何延迟加载 schema](chapters/chapter-24.md)
- [第 25 章：权限、安全与沙箱总览](chapters/chapter-25.md)
- [第 26 章：权限规则，allow、deny、ask 如何匹配工具和输入](chapters/chapter-26.md)
- [第 27 章：沙箱，限制命令执行的真实边界](chapters/chapter-27.md)
- [第 28 章：Hooks 与外部策略，让 Agent 可被项目和组织治理](chapters/chapter-28.md)
- [第 29 章：审计、日志与安全可解释性](chapters/chapter-29.md)
- [第 30 章：子 Agent、任务分解与多 Agent 协作总览](chapters/chapter-30.md)
- [第 31 章：深入 AgentTool，如何把一个任务交给子 Agent](chapters/chapter-31.md)
- [第 32 章：子 Agent 上下文隔离、结果汇总与成本控制](chapters/chapter-32.md)
- [第 33 章：worktree isolation 与并行实现策略](chapters/chapter-33.md)
- [第 34 章：多 Agent 任务编排、取消、恢复与冲突处理](chapters/chapter-34.md)
- [第 35 章：teammate、agent swarms 与团队式协作](chapters/chapter-35.md)
- [第 36 章：Agent memory、skills preload 与长期协作能力](chapters/chapter-36.md)
- [第 37 章：自定义 Agent、插件 Agent 与分发机制](chapters/chapter-37.md)
- [第 38 章：评估、回放、调试与生产可观测性](chapters/chapter-38.md)
- [第 39 章：从 mini-agent 到生产级 Agent 平台的完整路线图](chapters/chapter-39.md)
- [第 40 章：部署、文档站、GitHub Pages 与发布流程](chapters/chapter-40.md)
- [附录 A：AI Agent 工程术语表](chapters/appendix-a-glossary.md)
- [附录 B：Claude Code 源码阅读索引与学习路线](chapters/appendix-b-source-map.md)
- [附录 C：mini-agent 完整 Python 代码清单](chapters/appendix-c-mini-agent-complete-code.md)
