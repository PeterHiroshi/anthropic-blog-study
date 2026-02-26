# META — Effective Harnesses for Long-Running Agents

| 字段 | 内容 |
|------|------|
| **名称** | Effective Harnesses for Long-Running Agents |
| **中文名** | 长时运行 Agent 的有效运行框架 |
| **链接** | https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents |
| **来源** | Anthropic Engineering Blog |
| **发布时间** | 2025 |
| **分类** | AI Agents, Long-running Tasks, Software Engineering |
| **作者** | Anthropic Engineering Team |
| **阅读时长** | ~8 分钟 |
| **难度** | Intermediate — 适合构建自主编码 Agent 的工程师 |

## 摘要

本文提出了一套"运行框架"（Harness）模式，专为跨多个上下文窗口工作的 AI Agent 设计。核心方案是"初始化 Agent + 编码 Agent"的双阶段架构：初始化 Agent 建立环境和进度追踪，编码 Agent 在每次会话中读取状态、实现单个功能、提交代码并更新进度文档。

## 相关文件

- [README_EN.md](./README_EN.md) — 英文详细整理
- [README_CN.md](./README_CN.md) — 中文详细整理
