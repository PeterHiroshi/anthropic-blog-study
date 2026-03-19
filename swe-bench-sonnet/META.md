# META — Raising the Bar on SWE-bench Verified with Claude 3.5 Sonnet

| 字段 | 内容 |
|------|------|
| **名称** | Raising the Bar on SWE-bench Verified with Claude 3.5 Sonnet |
| **中文名** | 用 Claude 3.5 Sonnet 刷新 SWE-bench Verified 纪录 |
| **链接** | https://www.anthropic.com/engineering/swe-bench-sonnet |
| **来源** | Anthropic Engineering Blog |
| **发布时间** | 2025年1月6日 |
| **分类** | Engineering / Evaluation / Coding |
| **作者** | Erik Schluntz, Simon Biggs, Dawn Drain, Eric Christiansen 等 |
| **阅读时长** | ~8 分钟 |
| **难度** | 中级（适合了解 LLM 编程能力评估的开发者） |

## 摘要

本文介绍了升级版 Claude 3.5 Sonnet 在 SWE-bench Verified 上取得的 **49% 得分**，超越此前最优的 45%。团队设计了一个极简 Agent 架构——仅包含提示词、Bash 工具和编辑工具，模型在 200K 上下文限制内迭代工作直到完成。

文章的核心设计哲学是**详细的工具描述优于复杂的脚手架**。关键设计决策包括：强制使用绝对文件路径防止错误、使用字符串替换进行文件编辑确保可靠性。文章还讨论了 token 成本高（成功运行常超 100K token）、评分环境搭建复杂、隐藏测试用例限制自我评估等挑战。

## 相关文件

- [English Notes](README_EN.md)
- [中文笔记](README_CN.md)
