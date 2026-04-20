# META — Harness design for long-running application development

| 字段 | 内容 |
|------|------|
| **名称** | Harness design for long-running application development |
| **中文名** | 为长时间应用开发设计 Harness |
| **链接** | https://www.anthropic.com/engineering/harness-design-long-running-apps |
| **来源** | Anthropic Engineering Blog |
| **发布时间** | 2026年3月24日 |
| **分类** | Engineering / Agent Design / Long-Running Builds |
| **作者** | Prithvi Rajasekaran（Anthropic Labs） |
| **阅读时长** | ~12 分钟 |
| **难度** | 中高级（适合构建多小时 Agent 编码流水线的工程师） |

## 摘要

本文记录作者在 **Anthropic Labs** 用 Claude 构建**多小时应用**（复古游戏机、数字音频工作站 DAW 等）过程中摸索出的 Harness 设计规律。

两个失败模式反复出现：**(1) 模型在上下文逐渐填满时出现"上下文焦虑"（context anxiety），会提前收尾——重置上下文比压缩效果更好；(2) Agent 自评会系统性地乐观打分**（"skew positive"），必须把 Generator 和 Evaluator **拆成两个 Agent**。

具体数字对比惊人：
- **复古游戏机构建器**：单 Agent 20 分钟/$9 vs 全 Harness 6 小时/$200——**贵 20 倍，但成果显著更好**。
- **DAW 项目**：总计 3 小时 50 分/$124.70；Planner 4.7 分/$0.46，构建 3 小时/$113.85，QA 25 分/$10.39。

最关键的观察是："**Harness 的复杂度应该随模型能力提升而递减**"。从 Opus 4.5 到 Opus 4.6，作者**拆掉了 Sprint 结构**，Generator 能连续工作 2 小时以上——Harness 组件编码的是模型能力假设，这些假设会过时。

## 相关文件

- [English Notes](README_EN.md)
- [中文笔记](README_CN.md)
