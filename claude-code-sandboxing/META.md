# META — Making Claude Code More Secure and Autonomous (Sandboxing)

| 字段 | 内容 |
|------|------|
| **名称** | Making Claude Code More Secure and Autonomous |
| **中文名** | 让 Claude Code 更安全、更自主：沙盒隔离 |
| **链接** | https://www.anthropic.com/engineering/claude-code-sandboxing |
| **来源** | Anthropic Engineering Blog |
| **发布时间** | 2025 |
| **分类** | Security, Sandboxing, Claude Code, Prompt Injection |
| **作者** | Anthropic Engineering Team |
| **阅读时长** | ~8 分钟 |
| **难度** | Intermediate — 适合关注 AI 安全的开发者 |

## 摘要

本文介绍了 Claude Code 新增的两项沙盒安全功能，通过文件系统隔离和网络隔离两种操作系统级边界，在减少 84% 权限提示的同时提升安全性。两个新功能分别是：沙盒化 Bash 工具（使用 Linux bubblewrap 和 macOS seatbelt）和 Web 端 Claude Code（在隔离云沙盒中运行）。

## 相关文件

- [README_EN.md](./README_EN.md) — 英文详细整理
- [README_CN.md](./README_CN.md) — 中文详细整理
