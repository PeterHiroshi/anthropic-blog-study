# META — Claude Think Tool

| 字段 | 内容 |
|------|------|
| **名称** | The "Think" Tool: Enabling Claude to Stop and Think in Complex Tool Use Situations |
| **中文名** | Think 工具：让 Claude 在复杂工具调用中停下来思考 |
| **链接** | https://www.anthropic.com/engineering/claude-think-tool |
| **来源** | Anthropic Engineering Blog |
| **发布时间** | 2025年3月20日 |
| **分类** | Engineering / Tool Use / Reasoning |
| **作者** | Anthropic Engineering Team |
| **阅读时长** | ~8 分钟 |
| **难度** | 中级（需了解 Claude Tool Use 和 Agentic 工作流）|

## 摘要

本文介绍了 `think` 工具——一个让 Claude 在工具调用链中间暂停推理的轻量级机制。与 Extended Thinking（扩展思考，在响应前规划）不同，think 工具在 Claude 已开始响应后提供一个**无副作用的推理空间**，特别适用于需要遵循复杂策略、分析多步工具输出的场景。在航空客服基准测试中带来了 54% 的性能提升。

## 相关文件

- [README_EN.md](./README_EN.md) — 英文详细整理
- [README_CN.md](./README_CN.md) — 中文详细整理
