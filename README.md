# Anthropic Engineering Blog Study

Anthropic 工程博客精读笔记库。每篇文章整理为结构化双语文档（中英文），包含思维导图、代码示例和生活化类比。

```
每篇文章目录结构：
<blog-slug>/
├── META.md        # 元数据（来源、分类、摘要）
├── README_EN.md   # 英文详细整理（思维导图 + 全文笔记）
└── README_CN.md   # 中文详细整理（思维导图 + 类比说明）
```

---

## 文章索引

### Agent 设计与架构

| 文章 | 核心主题 | 难度 |
|------|----------|------|
| [Building Effective Agents](./building-effective-agents/) | 5 种 Workflow 模式、Agent vs Workflow 区分、工具设计原则 | 中级 |
| [Building Agents with the Claude Agent SDK](./building-agents-with-the-claude-agent-sdk/) | Agent SDK 三步循环（收集→执行→验证）、Bash/文件系统/MCP 工具 | 初中级 |
| [Effective Harnesses for Long-Running Agents](./effective-harnesses-for-long-running-agents/) | 初始化 Agent + 编码 Agent 双阶段架构、跨上下文窗口进度追踪 | 中级 |
| [Multi-Agent Research System](./multi-agent-research-system/) | Opus 4 主 Agent + Sonnet 4 子 Agent、8 条提示工程原则、90.2% 性能提升 | 高级 |
| [Building a C Compiler with Parallel Claudes](./building-c-compiler/) | 16 个并行 Agent 构建 10 万行 C 编译器、锁文件协调、GCC oracle 验证 | 高级 |

### 工具设计与使用

| 文章 | 核心主题 | 难度 |
|------|----------|------|
| [Writing Tools for Agents](./writing-tools-for-agents/) | 5 大工具设计原则、三阶段开发流程（原型→评估→优化） | 中级 |
| [Advanced Tool Use](./advanced-tool-use/) | Tool Search Tool、Programmatic Tool Calling、Tool Use Examples（Beta） | 中高级 |
| [Claude Think Tool](./claude-think-tool/) | think 工具 JSON 规范、54% tau-bench 提升、vs Extended Thinking 对比 | 中级 |
| [Code Execution with MCP](./code-execution-with-mcp/) | Agent 写代码调用 MCP、98.7% token 消耗降低、渐进式信息披露 | 高级 |

### 上下文与检索

| 文章 | 核心主题 | 难度 |
|------|----------|------|
| [Effective Context Engineering for AI Agents](./effective-context-engineering-for-ai-agents/) | 上下文工程 vs 提示工程、上下文腐化问题、长期任务管理 | 中级 |
| [Contextual Retrieval](./contextual-retrieval/) | RAG 改进方案、67% 检索失败降低、上下文嵌入 + BM25 + 重排序 | 中级 |

### Agent Skills 与扩展

| 文章 | 核心主题 | 难度 |
|------|----------|------|
| [Equipping Agents with Agent Skills](./equipping-agents-for-the-real-world-with-agent-skills/) | SKILL.md 结构、自动发现机制、跨平台开放标准 | 初中级 |
| [Desktop Extensions](./desktop-extensions/) | .mcpb 一键安装格式、内置 Node.js 运行时、企业 MDM 控制 | 初中级 |

### Claude Code

| 文章 | 核心主题 | 难度 |
|------|----------|------|
| [Claude Code Best Practices](./claude-code-best-practices/) | 4 阶段工作流（探索→规划→编码→提交）、CLAUDE.md、上下文窗口管理 | 初中级 |
| [Claude Code Sandboxing](./claude-code-sandboxing/) | 84% 权限提示减少、bubblewrap/seatbelt 沙盒、Web 云沙盒 | 中级 |

### 评估与可靠性

| 文章 | 核心主题 | 难度 |
|------|----------|------|
| [Demystifying Evals for AI Agents](./demystifying-evals-for-ai-agents/) | 评估分类法、3 种评分器类型、Agent 专项评估方案 | 中级 |
| [A Postmortem of Three Recent Issues](./a-postmortem-of-three-recent-issues/) | 3 个基础设施 Bug（路由/TPU 配置/XLA 编译器）、Aug-Sep 2025 事故复盘 | 中级 |
| [Infrastructure Noise in Agentic Evals](./infrastructure-noise/) | 基础设施配置造成 6pp 分数波动、3pp 怀疑阈值、资源限制两阶段影响 | 中高级 |
| [Eval Awareness in BrowseComp](./eval-awareness-browsecomp/) | 模型自主识别评估、解密加密答案、多 Agent 3.7x 污染放大 | 高级 |
| [Designing AI-Resistant Technical Evaluations](./AI-resistant-technical-evaluations/) | AI 超越 90% 人类面试者、约束编程谜题、真实性 vs 抗 AI 性取舍 | 中级 |
| [SWE-bench Verified with Claude 3.5 Sonnet](./swe-bench-sonnet/) | 49% SWE-bench SOTA、极简 3 工具 Agent、工具描述 > 复杂脚手架 | 中级 |

---

## 知识地图

```
Anthropic Engineering Blog
│
├── Agent 设计
│   ├── 核心原则：简单优于复杂
│   ├── Workflow 模式（5种）
│   │   ├── Prompt Chaining
│   │   ├── Routing
│   │   ├── Parallelization
│   │   ├── Orchestrator-Subagents
│   │   └── Evaluator-Optimizer
│   └── 长任务架构
│       ├── Harness 框架（初始化 + 编码 Agent）
│       ├── Multi-Agent（主 Agent 协调子 Agent）
│       └── 并行 Agent 构建编译器（16 实例协作）
│
├── 工具生态
│   ├── 工具设计原则（5条）
│   ├── Think Tool（中间推理空间）
│   ├── Advanced Tool Use（Beta）
│   │   ├── Tool Search Tool（解决上下文膨胀）
│   │   ├── Programmatic Tool Calling（减少中间结果）
│   │   └── Tool Use Examples（提升参数准确率）
│   ├── MCP + 代码执行（98.7% token 节省）
│   └── Desktop Extensions（.mcpb 一键安装）
│
├── 上下文管理
│   ├── 上下文工程 vs 提示工程
│   ├── 上下文腐化问题与解决方案
│   └── Contextual Retrieval（RAG 增强）
│       ├── 上下文嵌入
│       ├── 上下文 BM25
│       └── 重排序
│
├── Claude Code
│   ├── 4 阶段工作流
│   ├── CLAUDE.md 配置
│   ├── Agent Skills 框架
│   └── 沙盒安全机制
│
└── 系统可靠性
    ├── Eval 体系（任务/试验/评分器）
    ├── 基础设施事故复盘
    ├── 基础设施噪声（6pp 评估波动）
    ├── 评估感知行为（模型识别基准测试）
    ├── 抗 AI 技术评估（面试设计挑战）
    └── SWE-bench（49% SOTA 极简 Agent）
```

---

## 关键数字速查

| 数据点 | 来源文章 |
|--------|----------|
| Think 工具带来 **54%** tau-bench 性能提升 | Claude Think Tool |
| MCP 代码执行使 token 消耗降低 **98.7%**（150K→2K） | Code Execution with MCP |
| Contextual Retrieval 检索失败降低 **67%**（5.7%→1.9%） | Contextual Retrieval |
| Multi-Agent 系统相比单 Agent 性能提升 **90.2%** | Multi-Agent Research System |
| Claude Code 沙盒减少 **84%** 权限提示 | Claude Code Sandboxing |
| 基础设施配置造成 **6 pp** 评估分数波动 | Infrastructure Noise |
| 多 Agent 评估感知污染放大 **3.7x** | Eval Awareness BrowseComp |
| 16 个并行 Agent 构建 **10 万行**编译器，成本 **$20K** | Building C Compiler |
| Claude Opus 4 超越 **90%** 人类面试者 | AI-Resistant Technical Evaluations |
| Claude 3.5 Sonnet 达到 SWE-bench **49%** SOTA | SWE-bench Sonnet |

---

## 阅读建议

**入门路径**（了解 Agent 基础）：
```
Building Effective Agents
  → Building Agents with the Claude Agent SDK
  → Claude Code Best Practices
```

**进阶路径**（构建生产级 Agent）：
```
Writing Tools for Agents
  → Effective Context Engineering for AI Agents
  → Effective Harnesses for Long-Running Agents
  → Multi-Agent Research System
```

**专项深入**（特定技术）：
```
工具优化：Advanced Tool Use → Claude Think Tool → Code Execution with MCP → Desktop Extensions
检索增强：Contextual Retrieval
评估体系：Demystifying Evals → Infrastructure Noise → Eval Awareness BrowseComp → SWE-bench Sonnet
编码能力：SWE-bench Sonnet → AI-Resistant Technical Evaluations → Building C Compiler
安全机制：Claude Code Sandboxing
```
