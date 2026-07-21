# META — Scaling Managed Agents: Decoupling the brain from the hands

| 字段 | 内容 |
|------|------|
| **名称** | Scaling Managed Agents: Decoupling the brain from the hands |
| **中文名** | 扩展托管 Agent：解耦"大脑"与"双手" |
| **链接** | https://www.anthropic.com/engineering/managed-agents |
| **来源** | Anthropic Engineering Blog |
| **发布时间** | 2026年4月8日 |
| **分类** | Engineering / Agent Infrastructure |
| **作者** | Lance Martin, Gabe Cemaj, Michael Cohen |
| **阅读时长** | ~12 分钟 |
| **难度** | 中高级（需要了解容器化、分布式系统与 Agent harness 基础） |
| **官方文档** | https://platform.claude.com/docs/en/managed-agents/overview |

## 摘要

本文讲述 Anthropic 构建 **Managed Agents**（托管 Agent 服务）过程中的架构演进。核心命题是：**harness（脚手架）会把"模型当前做不到什么"的假设固化进代码，而模型一升级，这些假设就变成了死重（dead weight）**。文中的实例是：Sonnet 4.5 存在"上下文焦虑"（context anxiety），团队为此在 harness 里加入了上下文重置逻辑；换到 Opus 4.5 后该行为消失，重置逻辑反而成了负担。

作者借用**操作系统的思路**回应这个问题：操作系统把硬件虚拟化成 process、file 这类足够通用的抽象，通用到能服务"尚未被设想出来的程序"——`read()` 对 1970 年代的磁盘组和现代 SSD 一视同仁。Managed Agents 同样虚拟化三个组件：**session**（append-only 事件日志）、**harness**（调用 Claude 并路由工具调用的循环）、**sandbox**（执行环境）。

文章的主线是"解耦大脑与双手"：把 harness 移出容器，让它像调用任何工具一样调用沙箱（`execute(name, input) -> string`），从而把容器从需要精心照料的"宠物"变成可随时替换的"牲畜"。这一改造同时带来了安全边界（凭证永不进入沙箱）、可恢复性（`wake(sessionId)`）和性能收益（**p50 TTFT 下降约 60%，p95 下降超过 90%**）。

## 相关文件

- [English Notes](README_EN.md)
- [中文笔记](README_CN.md)
