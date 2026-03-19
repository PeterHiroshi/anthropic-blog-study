# META — Quantifying Infrastructure Noise in Agentic Coding Evals

| 字段 | 内容 |
|------|------|
| **名称** | Quantifying Infrastructure Noise in Agentic Coding Evals |
| **中文名** | 量化 Agent 编程评估中的基础设施噪声 |
| **链接** | https://www.anthropic.com/engineering/infrastructure-noise |
| **来源** | Anthropic Engineering Blog |
| **发布时间** | 2026年2月3日 |
| **分类** | Engineering / Evaluation |
| **作者** | Gian Segato, Nicholas Carlini, Jeremy Hadfield, Mike Merrill, Alex Shaw |
| **阅读时长** | ~10 分钟 |
| **难度** | 中高级（需要了解容器化和基准测试基础） |

## 摘要

本文揭示了一个被广泛忽视的问题：**基础设施配置本身就能造成 Agent 编程基准测试 6 个百分点的分数波动**——有时甚至超过排行榜上顶级模型之间的性能差距。Anthropic 团队在 Terminal-Bench 2.0 和 SWE-bench 上进行了系统性实验，发现容器资源限制（RAM、CPU）的不同配置方式直接影响评估结果。

核心发现是：严格的资源限制（1x 规格）导致 5.8% 的基础设施错误率，而 3x 余量将错误率降至 2.1%。文章建议评估开发者分别指定资源保证分配和硬性上限，并建议排行榜消费者对 3 个百分点以下的差异保持怀疑态度。

## 相关文件

- [English Notes](README_EN.md)
- [中文笔记](README_CN.md)
