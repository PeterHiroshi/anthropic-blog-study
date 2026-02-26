# META — Code Execution with MCP: Building More Efficient Agents

| 字段 | 内容 |
|------|------|
| **名称** | Code Execution with MCP: Building More Efficient Agents |
| **中文名** | 使用 MCP 代码执行构建高效 Agent |
| **链接** | https://www.anthropic.com/engineering/code-execution-with-mcp |
| **来源** | Anthropic Engineering Blog |
| **发布时间** | 2025 |
| **分类** | MCP, Code Execution, Token Efficiency, Agent Architecture |
| **作者** | Anthropic Engineering Team |
| **阅读时长** | ~10 分钟 |
| **难度** | Advanced — 适合熟悉 MCP 协议的工程师 |

## 摘要

本文探讨如何通过代码执行环境与 MCP 结合，大幅提升 Agent 效率。核心思想是让 Agent 通过编写代码来调用 MCP 服务器，而非直接暴露所有工具定义，从而实现按需加载工具、在环境内过滤数据。典型案例中 token 消耗从 150,000 降至 2,000（节省 98.7%）。

## 相关文件

- [README_EN.md](./README_EN.md) — 英文详细整理
- [README_CN.md](./README_CN.md) — 中文详细整理
