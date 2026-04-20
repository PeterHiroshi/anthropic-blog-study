# META — Scaling Managed Agents: Decoupling the brain from the hands

| 字段 | 内容 |
|------|------|
| **名称** | Scaling Managed Agents: Decoupling the brain from the hands |
| **中文名** | 扩展托管 Agent：将"大脑"与"双手"解耦 |
| **链接** | https://www.anthropic.com/engineering/managed-agents |
| **来源** | Anthropic Engineering Blog |
| **发布时间** | 2026年（Managed Agents 平台阶段） |
| **分类** | Engineering / Agent Infrastructure / Platform Design |
| **作者** | Lance Martin, Gabe Cemaj, Michael Cohen |
| **阅读时长** | ~12 分钟 |
| **难度** | 高级（适合构建 Agent 平台或大规模 Agent 系统的工程师） |

## 摘要

本文讲述 Anthropic 在构建 **Claude Managed Agents** 平台时从"单体容器 Harness"演进到"**大脑/双手解耦**"架构的过程。初期的 Harness 把 Claude 的推理、工具调用、沙盒、会话状态全部打包进一个长时运行的容器里——**像"宠物"**（pets）：手工维护、出故障就得抢救，会话丢了就全部丢失。

新架构借鉴操作系统虚拟化思想：把 **Harness 做成无状态**，只调用 `execute(name, input) → string` 形式的容器；**会话（Session）** 变成独立的追加式事件日志（append-only log），生存期脱离任何单个进程。结果：p50 首 token 延迟下降约 **60%**，p95 下降超过 **90%**，失败的 Harness 可通过 `wake(sessionId) + getSession(id)` 恢复。

文章的核心哲学：**Harness 里写死的所有"能力假设"都会随着模型能力进化而过时**。解耦架构允许"多个大脑 × 多个双手"，以此支撑不同任务、不同模型、不同沙盒的灵活组合。

## 相关文件

- [English Notes](README_EN.md)
- [中文笔记](README_CN.md)
