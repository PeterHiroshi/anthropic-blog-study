# META — An Update on Recent Claude Code Quality Reports

| 字段 | 内容 |
|------|------|
| **名称** | An Update on Recent Claude Code Quality Reports |
| **中文名** | 关于近期 Claude Code 质量反馈的说明 |
| **链接** | https://www.anthropic.com/engineering/april-23-postmortem |
| **来源** | Anthropic Engineering Blog |
| **发布时间** | 2026年4月23日 |
| **分类** | Incident Report, Postmortem, Claude Code, Product Quality |
| **作者** | Anthropic Engineering Team（文章未署名个人） |
| **阅读时长** | ~8 分钟 |
| **难度** | 中级 — 适合 Claude Code 用户与关注 LLM 产品发布流程的工程师 |

## 摘要

2026 年 3 月至 4 月间，大量用户报告 Claude Code "变笨了"。Anthropic 在本文中把根因追溯到**三个各自独立的产品侧变更**——不是模型权重改变，也不是基础设施故障，而是三次"看起来无害"的配置与提示词调整：

1. **推理强度默认值下调**（3月4日）：为解决延迟过高导致的 UI 卡顿，默认 reasoning effort 从 `high` 降到 `medium`。用户明确表示宁愿慢也要聪明，4月7日回滚。
2. **Thinking 缓存 Bug**（3月26日）：一次 prompt caching 优化（`clear_thinking_20251015` 头）本应只在会话闲置后清理一次旧推理，实际却在**每一轮都清**，导致 Claude 健忘、重复、用量额度消耗加快。4月10日（v2.1.101）修复。
3. **简洁度系统提示**（4月16日）：随 Opus 4.7 发布的系统提示新增长度限制（工具调用间 ≤25 词、最终回复 ≤100 词），内部评估显示 Opus 4.6 与 4.7 均掉约 3% 智能分。4月20日回滚。

三个问题在 **4月20日（v2.1.116）** 全部解决，Anthropic 同时**为所有订阅用户重置了用量额度**作为补偿。文章末尾给出五项流程改进承诺（内部使用公开构建、强化 Code Review 工具、系统提示变更管控、soak period、灰度发布）。

与 2025 年 9 月那篇《A Postmortem of Three Recent Issues》形成鲜明对照：那次是**基础设施 Bug**，这次是**产品决策与流程失误**。

## 相关文件

- [README_EN.md](./README_EN.md) — 英文详细整理
- [README_CN.md](./README_CN.md) — 中文详细整理

## 关联阅读

- [a-postmortem-of-three-recent-issues](../a-postmortem-of-three-recent-issues/) — 姊妹篇：2025 年 8-9 月的三个基础设施 Bug
- [infrastructure-noise](../infrastructure-noise/) — 为什么"3% 掉分"需要严肃对待：评估噪声的量化
- [claude-code-best-practices](../claude-code-best-practices/) — 本文涉及的 reasoning effort、thinking、上下文管理在实践中的用法
