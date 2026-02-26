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
│       └── Multi-Agent（主 Agent 协调子 Agent）
│
├── 工具生态
│   ├── 工具设计原则（5条）
│   ├── Think Tool（中间推理空间）
│   ├── Advanced Tool Use（Beta）
│   │   ├── Tool Search Tool（解决上下文膨胀）
│   │   ├── Programmatic Tool Calling（减少中间结果）
│   │   └── Tool Use Examples（提升参数准确率）
│   └── MCP + 代码执行（98.7% token 节省）
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
    └── 基础设施事故复盘
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
工具优化：Advanced Tool Use → Claude Think Tool → Code Execution with MCP
检索增强：Contextual Retrieval
评估体系：Demystifying Evals for AI Agents
安全机制：Claude Code Sandboxing
```
