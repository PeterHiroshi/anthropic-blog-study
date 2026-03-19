# META — Designing AI-Resistant Technical Evaluations

| 字段 | 内容 |
|------|------|
| **名称** | Designing AI-Resistant Technical Evaluations |
| **中文名** | 设计抗 AI 技术评估 |
| **链接** | https://www.anthropic.com/engineering/AI-resistant-technical-evaluations |
| **来源** | Anthropic Engineering Blog |
| **发布时间** | 2026年1月21日 |
| **分类** | Engineering / Hiring / Evaluation |
| **作者** | Tristan Hume (Performance Optimization Team Lead) |
| **阅读时长** | ~10 分钟 |
| **难度** | 中级（适合关注 AI 辅助编程和技术面试的开发者） |

## 摘要

本文记录了 Anthropic 在维护性能工程候选人实操评估时面临的挑战：**随着 Claude 模型能力不断提升，原本能有效区分候选人水平的测试被 AI 轻松解决**。Claude Opus 4 超越了约 90% 的人类申请者，Claude Opus 4.5 在 2 小时内达到了顶级人类表现。

文章详细记录了评估设计的迭代过程：从最初的 TPU 模拟器优化任务，到尝试数据转置问题（被 Claude 通过创造性重构解决），最终转向受 Zachtronics 游戏启发的约束编程谜题。核心洞见是：**真实感可能需要牺牲——原始评估有效是因为它像真实工作，替代方案有效是因为它模拟了新颖的工作。**

## 相关文件

- [English Notes](README_EN.md)
- [中文笔记](README_CN.md)
