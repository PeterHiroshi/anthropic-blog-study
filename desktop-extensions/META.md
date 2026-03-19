# META — Desktop Extensions: One-click MCP Server Installation

| 字段 | 内容 |
|------|------|
| **名称** | Desktop Extensions: One-click MCP Server Installation for Claude Desktop |
| **中文名** | 桌面扩展：Claude Desktop 一键安装 MCP 服务器 |
| **链接** | https://www.anthropic.com/engineering/desktop-extensions |
| **来源** | Anthropic Engineering Blog |
| **发布时间** | 2025年6月26日 |
| **分类** | Engineering / Tools / MCP |
| **作者** | Anthropic Engineering Team |
| **阅读时长** | ~8 分钟 |
| **难度** | 初中级（适合了解 MCP 基础的开发者） |

## 摘要

本文介绍了 Desktop Extensions（.mcpb 文件），这是一种新的打包格式，将 MCP 服务器的安装简化为**一键操作**。此前安装 MCP 服务器需要外部开发工具（Node.js、Python）、手动编辑 JSON 配置和处理依赖冲突，现在只需将 .mcpb 文件拖放到 Claude Desktop 设置中即可。

Desktop Extensions 是包含 manifest.json、服务器实现文件和捆绑依赖的 ZIP 归档。Claude Desktop 提供内置 Node.js 运行时、自动更新和 OS 钥匙串存储等能力。文章还介绍了开发工作流、跨平台支持和企业安全策略。

## 相关文件

- [English Notes](README_EN.md)
- [中文笔记](README_CN.md)
