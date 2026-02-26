# Claude 开发者平台高级工具使用功能介绍

> 来源：https://www.anthropic.com/engineering/advanced-tool-use
> 发布时间：2025年11月24日 | Anthropic Engineering

---

## 整体思维导图

```
高级工具使用（Beta）
├── 三大问题
│   ├── 工具太多 → 上下文膨胀
│   ├── 中间结果太多 → 数据过载
│   └── 参数复杂 → 调用出错
│
├── 功能一：工具搜索工具（Tool Search Tool）
│   ├── 问题：55K token 仅定义工具
│   ├── 方案：按需延迟加载工具
│   └── 效果：85% 上下文减少，准确率大幅提升
│
├── 功能二：程序化工具调用（Programmatic Tool Calling）
│   ├── 问题：20次调用 × 大量结果 = 50KB 上下文
│   ├── 方案：Claude 写代码来编排工具
│   └── 效果：37% token 减少，准确率提升
│
└── 功能三：工具使用示例（Tool Use Examples）
    ├── 问题：JSON Schema 无法表达使用惯例
    ├── 方案：在工具定义旁附上具体示例
    └── 效果：72% → 90% 复杂参数准确率
```

---

## 启用这些功能

所有三项功能目前处于 Beta 阶段，通过以下方式启用：

```python
client.messages.create(
    model="claude-sonnet-4-5-20250929",
    betas=["advanced-tool-use-2025-11-20"],
    # ... 其余请求参数
)
```

---

## 功能一：工具搜索工具

### 问题：上下文爆炸

想象你连接了 5 个 MCP 服务器：

```
GitHub 服务器      ≈ 12,000 token 的工具定义
Slack 服务器       ≈  8,000 token
Sentry 服务器      ≈  9,000 token
Grafana 服务器     ≈ 14,000 token
Splunk 服务器      ≈ 12,000 token
────────────────────────────────────
合计               ≈ 55,000 token
... 还没开始一句对话！
```

这不只是浪费，还会**主动降低性能**。Claude 在做任何有用的事情之前，要处理 55K token 的"工具噪音"。

### 解决方案：延迟加载

```
传统方式：                          工具搜索方式：

所有工具定义               只加载工具搜索工具本身
提前全部加载               （约 500 token）
（55,000 token）                  ↓
       ↓                    用户提出问题
   用户提问                         ↓
       ↓                    Claude："我需要 git 工具"
Claude 处理所有 55K token            ↓
       ↓                    工具搜索：获取 git 工具（约 3K token）
回答（也许是正确的）                  ↓
                            总计：约 8,700 token
                            节省了 85%！
```

### 配置方法

```python
# 将工具标记为延迟加载
tool_definition = {
    "name": "create_github_issue",
    "description": "创建新的 GitHub issue",
    "defer_loading": True,   # ← 关键标志
    "input_schema": { ... }
}

# Claude 初始只拿到搜索工具
# 随着需要动态发现其他工具
```

### 搜索后端选项

```
选项 A：基于正则表达式的搜索
  → 简单、快速、无需配置
  → 适合：小型工具库

选项 B：基于 BM25 的搜索
  → 更好的相关性排序
  → 适合：中型工具库

选项 C：自定义基于向量嵌入的搜索
  → 最佳语义匹配
  → 适合：大型工具库，语义命名
```

### 性能结果

| 模型 | 之前 | 之后 | 提升 |
|------|------|------|------|
| Claude Opus 4 | 49% | 74% | +51% |
| Claude Opus 4.5 | 79.5% | 88.1% | +11% |

### 何时使用工具搜索

```
应该使用当：
  ✓ 工具定义超过 10,000 token
  ✓ 有 10 个以上工具可用
  ✓ 工具选择准确性是个问题
  ✓ 连接了多个 MCP 服务器

不需要使用当：
  ✗ 只有 3-5 个简单工具
  ✗ 所有工具总是相关的
  ✗ 上下文预算不是瓶颈
```

---

## 功能二：程序化工具调用

### 问题：中间数据过载

考虑一个预算合规检查任务："检查我的20位团队成员的第三季度差旅费用。"

**传统方式：**
```
Claude → 调用 get_expenses(人员1) → 100条费用记录 → 进入上下文
Claude → 调用 get_expenses(人员2) → 100条费用记录 → 进入上下文
... × 20 人 ...
Claude → 上下文现在包含 2,000 条费用记录（50KB+）
Claude → 从这个巨大上下文中综合答案

问题：
  - 2,000 条 Claude 根本不需要逐条看的记录
  - 20 次独立推理往返
  - 50KB+ 无关中间数据进入上下文
  - 高错误率、慢、贵
```

**程序化方式：**
```
Claude 写 Python 代码：
  ──────────────────────────────────────────
  import asyncio

  async def check_compliance():
      tasks = [get_expenses(person) for person in team]
      results = await asyncio.gather(*tasks)   # 并行！
      violations = [r for r in results if r.amount > LIMIT]
      return f"发现 {len(violations)} 条违规"
  ──────────────────────────────────────────

代码在沙箱环境中执行
只有最终摘要进入 Claude 的上下文（约 1KB）

收益：
  - 2,000 条记录永远不进入 Claude 上下文
  - 并行执行（1次往返 vs. 20次）
  - 只返回有意义的结果
  - 更快、更便宜、更准确
```

### 配置方法

```python
# 标记工具可以被代码执行环境调用
tool_definition = {
    "name": "get_expenses",
    "description": "获取团队成员的费用记录",
    "allowed_callers": ["code_execution_20250825"],  # ← 关键标志
    "input_schema": { ... }
}
```

### Claude 生成的代码样式

```python
# Claude 会写这样的异步 Python 代码：
import asyncio

async def analyze_team_budgets(team_members: list):
    # 并行获取所有数据
    expense_tasks = [
        get_expenses(member_id=m.id, quarter="Q3")
        for m in team_members
    ]
    all_expenses = await asyncio.gather(*expense_tasks)

    # 过滤和聚合（永远不进入 Claude 上下文）
    violations = [
        {"人员": m.name, "金额": e.total}
        for m, e in zip(team_members, all_expenses)
        if e.total > TRAVEL_LIMIT
    ]

    # 只有这个摘要进入 Claude 的上下文
    return f"发现 {len(violations)} 条预算违规：{violations}"
```

### 性能结果

| 指标 | 之前 | 之后 |
|------|------|------|
| Token 用量 | 43,588 | 27,297（-37%）|
| 内部知识检索准确率 | 25.6% | 28.5% |
| GIA 基准测试 | 46.5% | 51.2% |

### 何时使用程序化工具调用

```
应该使用当：
  ✓ 处理需要聚合的大型数据集
  ✓ 3 步以上的多步骤依赖工作流
  ✓ 在推理之前需要过滤/转换结果
  ✓ 跨多个项目的并行操作
  ✓ 中间数据不应该影响推理

不需要使用当：
  ✗ 只有 1-2 个简单工具调用
  ✗ 你希望 Claude 对中间结果进行推理
  ✗ 工具有重大副作用（谨慎处理）
```

---

## 功能三：工具使用示例

### 问题：Schema 不够用

JSON Schema 告诉 Claude 工具接受什么**结构**，但无法表达如何**用好**它：

```json
{
  "name": "create_support_ticket",
  "input_schema": {
    "properties": {
      "title": {"type": "string"},
      "reporter_email": {"type": "string"},
      "priority": {"enum": ["low", "medium", "high", "critical"]},
      "escalation_contacts": {"type": "array"},
      "metadata": {"type": "object"}
    }
  }
}
```

Schema 无法告诉 Claude：
- 什么时候要填 `reporter_email`，什么时候省略？
- `critical` 优先级在什么情况下使用？
- `metadata` 放什么？什么时候需要？
- `priority` 和 `escalation_contacts` 有什么关联？

### 解决方案：提供具体示例

```python
tool_with_examples = {
    "name": "create_support_ticket",
    "description": "创建客户支持工单",
    "examples": [
        {
            "description": "最简工单 - 只有标题",
            "input": {
                "title": "登录页面无法加载"
            }
        },
        {
            "description": "标准工单 - 包含提交者信息",
            "input": {
                "title": "欧洲客户付款处理失败",
                "reporter_email": "sarah@company.com",
                "priority": "high"
            }
        },
        {
            "description": "紧急问题 - 全量上报",
            "input": {
                "title": "生产数据库不可访问",
                "reporter_email": "oncall@company.com",
                "priority": "critical",
                "escalation_contacts": ["cto@company.com", "devops@company.com"],
                "metadata": {
                    "incident_id": "INC-2024-001",
                    "affected_services": ["api", "web"]
                }
            }
        }
    ],
    "input_schema": { ... }
}
```

### 性能结果

```
没有示例：复杂参数处理准确率 72%
有示例：  复杂参数处理准确率 90%
提升：    +18 个百分点
```

### 示例的最佳实践

```
规则：
  1. 每个工具 1-5 个示例（更多收益递减）
  2. 展示进阶程度：最简 → 部分 → 完整
  3. 只聚焦于模糊的地方（不说显而易见的）
  4. 使用真实数据（不用 "foo"、"test" 占位符）

示例最有帮助的场景：
  ✓ 复杂嵌套结构
  ✓ 依赖上下文的可选参数
  ✓ 领域特定惯例（日期格式、ID 模式）
  ✓ 有 2 个以上相似 Schema 的工具（消歧义）

可以跳过的场景：
  ✗ 有 1-2 个显而易见参数的简单工具
  ✗ 参数名称已经自解释
```

---

## 三项功能的综合使用

这三项功能针对不同瓶颈：

```
┌────────────────────────────────────────────────────────────────┐
│                     你的生产 Agent                             │
│                                                                │
│  问题：上下文膨胀       → 方案：工具搜索工具                    │
│  问题：数据过载         → 方案：程序化工具调用                   │
│  问题：调用出错         → 方案：工具使用示例                     │
│                                                                │
│  结合使用效果最大化！                                           │
└────────────────────────────────────────────────────────────────┘
```

### 决策指南

```
添加新工具时：
  → 有复杂的可选参数？加示例。
  → 有相似的兄弟工具？加示例 + 仔细命名。
  → 很少需要？标记 defer_loading: True。
  → 返回大量数据？允许程序化调用。

整体性能不佳时：
  → Token 预算问题？    → 工具搜索工具
  → 数据推理问题？      → 程序化工具调用
  → 参数错误问题？      → 工具使用示例
```

---

## 兼容性说明

```
工具搜索工具：
  ✓ 与提示词缓存（Prompt Caching）兼容
    延迟工具不在初始缓存提示词中
    搜索后才加载，缓存仍然有效

程序化工具调用：
  ✓ 工具在隔离沙箱中执行
  ✓ 支持 async/await 模式
  ! 尽量确保工具操作是幂等的

工具使用示例：
  ✓ 与现有工具描述叠加使用
  ✓ 纯增量，不会破坏现有实现
```

---

## 形象比喻帮助记忆

```
工具搜索工具 🔍
  就像图书馆索引系统：
  你不需要先读所有书，才知道有什么书
  → 先看目录，找到需要的再去取书

程序化工具调用 💻
  就像统计员：
  不需要把所有原始数据报告给经理
  → 统计员先处理好，只汇报最终结论

工具使用示例 📖
  就像操作手册上的示例：
  规格说明告诉你"把螺钉放入孔中"
  但示例图片告诉你"应该这样拧"
```
