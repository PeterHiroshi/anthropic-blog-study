# META — How We Built Claude Code Auto Mode

| 字段 | 内容 |
|------|------|
| **名称** | How we built Claude Code auto mode: a safer way to skip permissions |
| **中文名** | 我们如何构建 Claude Code 自动模式：一种更安全的跳过权限方式 |
| **链接** | https://www.anthropic.com/engineering/claude-code-auto-mode |
| **来源** | Anthropic Engineering Blog |
| **发布时间** | 2026年3月25日 |
| **分类** | Engineering / Agent Safety |
| **作者** | John Hughes |
| **阅读时长** | ~12 分钟 |
| **难度** | 中高级（需要了解 Agent 权限模型与分类器基础） |

## 摘要

用户会批准约 **93%** 的权限提示——这说明逐条人工审批带来的主要是"审批疲劳"，而非等量的安全收益。但已有的两种替代方案都不令人满意：沙箱安全却**高维护成本**（每新增一种能力就要配置一次），`--dangerously-skip-permissions` 零维护却**毫无保护**。

自动模式（auto mode）的目标被明确表述为：**"捕捉那些与用户意图不一致的危险动作，同时让其余动作无需批准提示即可运行。"**

实现方式是**两层防御**：

- **输入层** — 服务端的提示注入探针（prompt-injection probe），在工具输出进入 Agent 上下文之前扫描它们，检测到可疑内容时向上下文追加警告；
- **输出层** — 基于 Sonnet 4.6 的**转录分类器（transcript classifier）**，采用两阶段设计：Stage 1 是不做推理的单 token 快速 yes/no 过滤，Stage 2 仅对被标记项做链式思考推理。

关键设计选择是分类器**只看用户消息与 Agent 的工具调用**，刻意剥离 Claude 自己的解释文字和工具输出——既防止 Agent 用花言巧语说服分类器，也大幅降低提示注入暴露面。

评测结果：真实流量（10K）Stage 1 假阳率 8.5%，完整流水线降至 **0.4%**；真实"过度积极"样本（52 例）Stage 1 假阴率 6.6%，流水线假阴率 **17%**；合成数据外泄（1K）Stage 1 假阴率 1.8%，流水线假阴率 **5.7%**。文章诚实地把 17% 假阴率称为"the honest number"。

## 相关文件

- [English Notes](README_EN.md)
- [中文笔记](README_CN.md)
