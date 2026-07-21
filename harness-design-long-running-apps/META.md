# META — Harness Design for Long-Running Application Development

| 字段 | 内容 |
|------|------|
| **名称** | Harness Design for Long-Running Application Development |
| **中文名** | 面向长时程应用开发的 Harness 设计 |
| **链接** | https://www.anthropic.com/engineering/harness-design-long-running-apps |
| **来源** | Anthropic Engineering Blog |
| **发布时间** | 2026年3月24日 |
| **分类** | Engineering / Agent Architecture |
| **作者** | Prithvi Rajasekaran（Anthropic Labs） |
| **阅读时长** | ~15 分钟 |
| **难度** | 中高级（需要了解多 Agent 架构与上下文管理基础） |

## 摘要

本文把 **GAN（生成对抗网络）的"生成器 / 评估器分离"思想**迁移到 Agent 编程场景，构建了一套用于**多小时自主应用开发**的 harness，效果显著优于单 Agent 直跑。

作者首先指出朴素长时程 Agent 编程的两个失效模式：**（1）上下文退化**——模型随上下文填满而失去连贯性，Claude Sonnet 4.5 甚至出现"上下文焦虑"（context anxiety），在接近感知到的 token 上限时草草收尾；压缩（compaction）能保留连续性，但**上下文重置**（context reset）提供的干净白板才是更好的解法。**（2）自评偏差**——Agent 对自己的产出几乎无条件称赞，在缺少二元验证的主观任务（如设计）上尤其严重；把评估**分离**成独立 Agent 才产生杠杆。

在前端设计上，作者用**生成器–评估器循环**配合四条评分标准（Design Quality / Originality / Craft / Functionality），评估器通过 Playwright 真实操作页面后再打分。在全栈开发上扩展为 **Planner + Generator + Evaluator 三 Agent 架构**，并在每个 sprint 前让生成器与评估器协商"**sprint 契约**"。

对比实验：同一个"复古游戏制作器"提示词下，Claude Opus 4.5 单 Agent 跑 20 分钟、花费 $9，产出核心功能损坏的成品；完整 harness 跑 6 小时、花费 $200，交付了可玩的完整游戏。

最关键的结论是反直觉的：**随着模型能力提升，harness 复杂度应当下降**。Claude Opus 4.6 发布后，作者移除了 sprint 机制，仅保留 planner 与 evaluator，用它构建的 DAW 耗时约 3 小时 50 分、花费 $124.70。

## 相关文件

- [English Notes](README_EN.md)
- [中文笔记](README_CN.md)
