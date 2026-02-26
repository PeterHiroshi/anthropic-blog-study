# 用 Agent 编写高效的 Agent 工具

> 来源：https://www.anthropic.com/engineering/writing-tools-for-agents
> 发布时间：2025年 | Anthropic Engineering

---

## 整体思维导图

```
为 Agent 编写工具
├── 全新心智模型
│   └── 工具 = 确定性系统 与 非确定性 Agent 之间的契约
│
├── 开发流程（三阶段）
│   ├── 第一阶段：构建原型
│   ├── 第二阶段：运行评估
│   └── 第三阶段：与 Agent 协作优化
│
└── 五大设计原则
    ├── 1. 选对工具
    │   ├── Agent 的能力与人类不同
    │   └── 把零散工具整合成高层工具
    ├── 2. 工具命名空间
    │   └── 前缀/后缀策略
    ├── 3. 返回有意义的上下文
    │   ├── 语义标识符 > 加密 ID
    │   └── 简洁/详细模式
    ├── 4. 优化 Token 效率
    │   └── 分页、过滤、截断
    └── 5. 提示词工程化工具描述
        └── 工具描述 = 工具行为的系统提示词
```

---

## 1. 全新心智模型：工具是什么？

传统程序员眼中的函数是**确定性系统之间的契约**：
```
function add(a, b) {
    return a + b;  // 永远返回 a + b。可预测。确定。
}
```

Agent 的工具是完全不同的东西——是**确定性系统与非确定性 Agent 之间的契约**：

```
问题：「我需要带伞吗？」

Agent 可能会做：
  方案 A：调用 get_weather() 工具   ← 预期行为
  方案 B：用通用知识直接回答         ← 可能过时
  方案 C：问"你在哪个城市？"        ← 需要澄清
  方案 D：编造关于工具调用的内容     ← 错误，但可能发生

你无法预测 Agent 会走哪条路。
你的工具设计必须考虑到这种不确定性。
```

**工具设计的目标：** 扩大 Agent 能够有效发挥的覆盖面——支持多样化的问题解决策略，同时对 Agent 和人类都保持直观易用。

---

## 2. 开发流程：三个阶段

### 第一阶段：构建原型

```
步骤 1：使用 Claude Code 快速开发
  → 提供依赖项的文档链接
  → Claude Code 可以通过 llms.txt 文件读取文档

步骤 2：包装成本地 MCP 服务器进行测试
  → claude mcp add <名称> <命令> [参数...]

步骤 3：手动测试
  → 尝试真实使用场景
  → 记录 Claude 在哪里卡壳或困惑

步骤 4：收集用户反馈
  → 用户实际期望的工作流是什么？
  → 他们使用什么术语？
```

**小技巧：** 提供有 `llms.txt` 的官方文档，Claude 就能精确编写针对你的依赖项的代码。

### 第二阶段：运行评估

**生成评估任务：**
```
好的评估任务（推荐使用这类）：
  ✓ "安排会议并附上项目提案文件"
  ✓ "调查为什么客户 #4521 上周二被重复收费了"
  ✓ "根据客户的使用历史准备一个留存优惠"

  为什么好：
    - 需要多次工具调用
    - 反映真实工作流的复杂度
    - 有明确的成功标准

差的评估任务（避免）：
  ✗ "下午3点安排一个会议"
  ✗ "搜索错误日志"

  为什么差：
    - 只需要一次工具调用
    - 不反映真实复杂度
```

**以程序化方式运行评估：**
```python
def run_eval(task, tools):
    messages = [{"role": "user", "content": task}]

    while True:
        response = call_llm(messages, tools)

        if response.stop_reason == "end_turn":
            return evaluate_result(response.content)

        # 执行工具调用
        tool_results = execute_tools(response.tool_calls)
        messages.append({"role": "assistant", "content": response.content})
        messages.append({"role": "user", "content": tool_results})

# 每次评估追踪这些指标：
metrics = {
    "accuracy": ...,      # 是否正确完成？
    "runtime": ...,       # 用了多长时间？
    "tool_calls": ...,    # 调用了多少次工具？
    "tokens": ...,        # 总 token 消耗？
    "error_rate": ...,    # 失败率？
}
```

**分析结果——真实案例：**
```
Anthropic 在发布 Web 搜索工具时发现：

  发现的问题：Claude 在搜索查询末尾追加 "2025"
  影响：       搜索结果变差（范围太窄）
  根本原因：   工具描述含糊
  解决方案：   改进描述 → 问题解决

教训：仔细阅读推理记录
      Agent 的"思考过程"揭示了真正的问题。
```

### 第三阶段：与 Agent 协作优化

```
Anthropic 的元工作流（用 Agent 优化 Agent）：

1. 运行评估 → 收集失败的对话记录
2. 把对话记录粘贴到 Claude Code
3. 问：「这个 Agent 为什么失败？我们应该怎么修复工具？」
4. 实施 Claude 的建议
5. 在保留测试集上重新运行评估
6. 重复

结果：即使是"专家级"实现也进一步提升了
      这个迭代过程带来了额外的收益
```

**重要：** 使用**保留测试集**防止过拟合。不要在生成修复方案的相同任务上评估。

---

## 3. 原则一：选对工具

### Agent 的能力 ≠ 人类的能力

```
人类程序员（确定性，速度快）：
  → list_all_contacts()      ← 遍历 10,000 个联系人
  → for contact in contacts: ← 快速，无上下文成本
      process(contact)

LLM Agent（非确定性，上下文有限）：
  → list_all_contacts()      ← 获取 10,000 个联系人的文本
  → 逐条阅读                  ← 消耗巨大上下文
  → 读到一半就失去线索         ← 上下文溢出或混乱

解决方案：根据 Agent 的思维方式设计，而不是程序的运行方式。
```

### 整合策略

不要暴露原始的细粒度 API，而是整合成高层工具：

```
低效（细粒度，对 Agent 不友好）:     高效（整合，对 Agent 友好）:

list_users()                         schedule_event()
list_events()              →         （内部处理：检查空闲时间
create_event(user, time)              预订时间槽，通知与会者）

read_all_logs()                       search_logs(query, context_lines=2)
                           →          （内部处理：过滤，只返回
                                       相关行 + 上下文）

get_customer_by_id()                  get_customer_context(customer_id)
list_transactions()        →          （内部处理：获取客户数据
list_customer_notes()                  最近交易，备注，一次性返回）
```

**设计测试：** 如果只给人类这些工具，他们是否会以同样的方式拆解任务？如果是，工具设计是正确的。

---

## 4. 原则二：工具命名空间

当 Agent 访问数十个 MCP 服务器时，工具名称可能冲突并引起混淆。

### 命名策略

```
策略 A：按服务命名空间
  asana_search          jira_search
  asana_create          jira_create
  asana_update          jira_update

策略 B：按资源命名空间
  asana_projects_search     asana_projects_create
  asana_users_search        asana_users_create
  asana_tasks_search        asana_tasks_create

在同一服务器内保持一致。
```

### 为什么重要

```
没有命名空间：
  Agent 看到：search()、create()、update()
  Agent 困惑："search 指的是哪个服务？"
  结果：调用错误工具，出现错误，需要重试

有命名空间：
  Agent 看到：asana_search()、jira_search()
  Agent 理解："明显需要 asana_search 来查 Asana 数据"
  结果：首次调用即正确

额外收益：减少工具描述的 token 用量
           （共同前缀在认知上可压缩）
```

**注意：** Anthropic 测试发现，**前缀 vs. 后缀**命名空间在不同模型上的性能差异不可忽视。在你自己的评估框架中测试两种方式。

---

## 5. 原则三：返回有意义的上下文

### 语义标识符 vs. 加密标识符

```
加密（避免）：              语义（推荐）：
uuid                  →    name（姓名）
256px_image_url       →    image_url
mime_type             →    file_type
user_id_hash          →    username
evt_2024_q3_001       →    "2024年Q3规划会议"
```

**为什么重要：**
```
使用加密 ID 的 Agent：
  Agent："user_id 是 7f3a9c2b-...，我需要查询..."
  → 需要额外的工具调用
  → 更容易搞混用户

使用语义 ID 的 Agent：
  Agent："用户是陈明..."
  → 无需额外查询
  → 明确无歧义
```

### 可控详细程度：响应格式模式

```python
# 让 Agent 控制输出详细程度
def search_slack(query: str, response_format: str = "concise"):
    results = fetch_from_slack(query)

    if response_format == "concise":
        return [{"text": r.text, "channel": r.channel} for r in results]
        # ~200 token（5条结果）

    elif response_format == "detailed":
        return [{
            "text": r.text,
            "channel": r.channel,
            "thread_ts": r.thread_ts,    # 回复时需要
            "user_id": r.user_id,        # 发私信时需要
            "permalink": r.permalink,    # 生成链接时需要
            "reactions": r.reactions,    # 上下文信号
        } for r in results]
        # ~600 token（5条结果，多3倍）
```

**经验法则：** 详细响应约多 3 倍 token。默认设计应围绕最常见的使用场景。

---

## 6. 原则四：优化 Token 效率

### 默认约束

```
Claude Code：工具响应默认上限 25,000 token
→ 将工具设计为远低于此限制
→ 分页/过滤作为默认设置，而非可选项
```

### 实用模式

```python
# 模式 1：分页
def get_records(offset=0, limit=50):  # 默认限制，不是"获取全部"
    ...

# 模式 2：范围选择
def get_logs(start_time, end_time, level="error"):  # 默认过滤
    ...

# 模式 3：带引导的截断
def search_documents(query, max_chars=5000):
    results = search(query)
    if total_chars(results) > max_chars:
        truncated = truncate(results, max_chars)
        return {
            "results": truncated,
            "truncated": True,
            "hint": "结果已截断。请使用更具体的查询来缩小范围。"
            # ← 引导 Agent 采用更好的策略
        }
```

### 错误消息作为指导

```
差的错误消息：
  {"error": "INVALID_PARAMS", "code": 400}

  → Claude 获得的信息为零
  → 可能用同样错误的参数重试

好的错误消息：
  {
    "error": "无效的日期范围",
    "provided": {"start": "2024-13-01"},
    "issue": "月份值 13 无效",
    "correct_format": "YYYY-MM-DD",
    "example": {"start": "2024-01-01", "end": "2024-12-31"}
  }

  → Claude 准确知道出了什么问题
  → 下次尝试时会修正
```

---

## 7. 原则五：提示词工程化工具描述

工具描述直接加载到 Agent 的上下文窗口中。它们实际上是**工具行为的系统提示词**。

### 高质量工具描述应包含什么

```
高质量工具描述模板：

{工具名称}：{做什么，为什么用途}

使用时机：{具体触发条件}
不使用时机：{反面模式，常见错误}

参数：
  - {参数1}：{描述}。格式：{格式}。示例：{示例}
  - {参数2}：{描述}。仅在以下情况包含：{条件}

重要说明：
  - {特殊查询格式（如有）}
  - {领域特定术语}
  - {与其他工具的关系}
  - {已知限制}
```

### 真实影响：SWE-Bench 案例

```
Anthropic 优化编程 Agent 工具描述的经历：

描述优化前：
  → Claude Sonnet 3.5 SWE-bench Verified 得分较低
  → 频繁出现路径错误
  → 工具选择错误

精确优化工具描述后：
  → SWE-bench Verified：达到当时最先进水平
  → 错误大幅减少
  → 完成率显著提升

教训：工具描述质量 = Agent 性能质量
```

### 别忘了 MCP 注解

```
MCP 工具应包含以下注解：
  open-world access    ← 工具可以访问互联网/外部数据
  destructive changes  ← 工具修改/删除数据

示例 MCP 工具注解：
{
  "name": "delete_files",
  "annotations": {
    "destructive": true,
    "requires_confirmation": true
  }
}
```

---

## 工具质量检查清单（一图速查）

```
设计阶段
  □ 每个工具是否符合 Agent 自然划分任务的方式？
  □ 零散工具是否整合成了高层工具？
  □ 每个工具是否有清晰的单一目的？

命名阶段
  □ 工具是否有一致的命名空间？
  □ 看工具名就能判断它属于哪个服务/资源吗？
  □ 参数名是否无歧义？（用 user_id 而不是 user）

返回值阶段
  □ 返回值是否使用语义标识符？
  □ 是否有简洁/详细模式控制详细程度？
  □ 响应是否默认分页/过滤？

错误处理阶段
  □ 错误消息是否具体说明了出错原因？
  □ 错误消息是否提供了正确格式示例？
  □ 截断消息是否引导 Agent 采用更好的策略？

文档阶段
  □ 描述是否说明了何时使用以及何时不使用？
  □ 特殊查询格式是否有文档说明？
  □ 工具之间的关系是否有解释？
  □ 是否用评估套件测试了描述的效果？
```

---

## 形象比喻帮助记忆

```
选对工具 🎯
  就像给新来的快递员规划路线：
  不要给他一份按门牌号排列的所有地址列表
  → 给他按区域分组的地图，他能更高效地规划

命名空间 🏷️
  就像超市货架标签：
  "水果 > 苹果" 比 "苹果1、苹果2" 好找多了

返回有意义的上下文 📦
  就像快递单上写"张三的iPhone"而不是"商品SKU-ABC123"
  → 收件人一眼就知道是什么

工具描述 = 提示词工程 ✍️
  就像员工手册：
  写得好的手册让新员工独立完成工作
  写得差的手册让人到处问"这个怎么办"
```
