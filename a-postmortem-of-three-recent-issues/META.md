# META — A Postmortem of Three Recent Issues

| 字段 | 内容 |
|------|------|
| **名称** | A Postmortem of Three Recent Issues |
| **中文名** | 近期三个问题的事后分析 |
| **链接** | https://www.anthropic.com/engineering/a-postmortem-of-three-recent-issues |
| **来源** | Anthropic Engineering Blog |
| **发布时间** | 2025-09 |
| **分类** | Incident Report, Infrastructure, Reliability, TPU |
| **作者** | Anthropic Engineering Team |
| **阅读时长** | ~10 分钟 |
| **难度** | Intermediate — 适合关注 AI 系统可靠性的工程师 |

## 摘要

本文记录了 2025 年 8-9 月间导致 Claude 响应质量下降的三个基础设施 Bug：上下文窗口路由错误、TPU 配置错误导致的输出损坏、以及 XLA/TPU 编译器 Bug 导致的近似 Top-k 误计算。文章详细分析了每个问题的根因、影响范围和修复措施，并承诺改善监控和调试基础设施。

## 相关文件

- [README_EN.md](./README_EN.md) — 英文详细整理
- [README_CN.md](./README_CN.md) — 中文详细整理
