# META — How We Contain Claude

| 字段 | 内容 |
|------|------|
| **名称** | How We Contain Claude |
| **中文名** | 我们如何"围住"Claude：跨产品的 Agent 收容体系 |
| **链接** | https://www.anthropic.com/engineering/how-we-contain-claude |
| **来源** | Anthropic Engineering Blog |
| **发布时间** | 2026年5月25日 |
| **分类** | Engineering / Security & Agent Safety |
| **作者** | Max McGuinness, Mikaela Grace, Jiri De Jonghe, Jake Eaton, Abel Ribbink |
| **阅读时长** | ~15 分钟 |
| **难度** | 中高级（需要了解沙箱、虚拟化、提示注入等安全基础） |

## 摘要

本文系统阐述了 Anthropic 在三条产品线（claude.ai、Claude Code、Claude Cowork）上如何**收容 Agent 的"爆炸半径"（blast radius）**。核心思路的转变是：与其不断限制 Agent 能做什么，不如构建收容系统，让 Agent 在能力更强、自主性更高的同时，把**最坏情况的损害封顶**。

文章提出了一个三乘三的框架：**三类风险**（用户误用、模型失当行为、外部攻击者）× **三层防御**（环境层、模型层、外部内容层），并展示了三种落地形态——claude.ai 的**临时 gVisor 容器**、Claude Code 的**本地沙箱 + 人在环路审批**、Claude Cowork 的**完整虚拟机**。

关键数据包括：Claude Code 用户对权限提示的批准率高达约 **93%**（审批疲劳使人工监督形同虚设），引入 OS 级沙箱（macOS Seatbelt / Linux bubblewrap）后提示量下降约 **84%**；auto mode 可在执行前拦截约 **83%** 的过激行为，同时仅误拦 **0.4%** 的良性命令。

最重要的教训被总结为一句话：**"你自己写的那一层往往是最弱的一层"**——经过实战检验的原语（hypervisor、seccomp、gVisor）始终守住了边界，而自研的代理和白名单实现则相继失守。

## 相关文件

- [English Notes](README_EN.md)
- [中文笔记](README_CN.md)
