# META — Claude Code auto mode: a safer way to skip permissions

| 字段 | 内容 |
|------|------|
| **名称** | Claude Code auto mode: a safer way to skip permissions |
| **中文名** | Claude Code 自动模式：更安全地跳过权限确认 |
| **链接** | https://www.anthropic.com/engineering/claude-code-auto-mode |
| **来源** | Anthropic Engineering Blog |
| **发布时间** | 2026年3月25日 |
| **分类** | Engineering / Claude Code / Safety & Permissions |
| **作者** | John Hughes（致谢 10 位贡献者） |
| **阅读时长** | ~10 分钟 |
| **难度** | 中高级（适合了解 Agent 安全设计的工程师） |

## 摘要

Claude Code 的**自动模式（auto mode）** 是在"每次都问"和"`--dangerously-skip-permissions` 全放开"之间的第三条路。数据显示用户对人工确认提示的通过率高达 **93%**，造成"确认疲劳"（approval fatigue）并最终导致用户直接用危险的 skip 开关。

自动模式用**基于模型的分类器**替代逐条确认，分三层：
- **Tier 1**：内置安全工具白名单（读文件、搜索、导航）——永不拦截；
- **Tier 2**：项目内文件操作——放行，因为可被版本控制审计；
- **Tier 3**：其余操作走**转录分类器**（transcript classifier）。

分类器采用**两阶段过滤**：单 token 快速判断（8.5% 误报）→ 被标记时再做链式推理（降到 0.4% 误报）。还通过**剥离 assistant 文本和工具输出**来防止 Agent 说服分类器做错误决定、以及抵抗提示注入。

关键权衡：对真实过激行为的漏报率是 **17%**——作者直言"这取决于你拿它跟什么比"，它显著优于 skip-all，但不能替代对高风险基础设施的人工审查。

## 相关文件

- [English Notes](README_EN.md)
- [中文笔记](README_CN.md)
