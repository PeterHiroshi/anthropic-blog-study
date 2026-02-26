# 用 Agent Skills 让 Agent 胜任真实世界任务

> 来源：https://claude.com/blog/equipping-agents-for-the-real-world-with-agent-skills
> 发布时间：2025年10月16日 | Claude Blog

---

## 整体思维导图

```
Agent Skills（Agent 技能）
├── 是什么
│   ├── 含有 SKILL.md 的目录
│   └── 像新员工入职手册
│
├── Claude 如何使用技能
│   ├── 自动调用（匹配描述时）
│   └── 手动调用（/skill-name）
│
├── 渐进式信息披露（核心设计原则）
│   ├── 第一层：元数据（始终加载，极少 token）
│   ├── 第二层：SKILL.md 完整内容（相关时加载）
│   └── 第三层：关联文件和脚本（按需加载）
│
├── 技能类型
│   ├── 参考型技能（知识/规范）
│   └── 任务型技能（分步工作流）
│
├── 高级功能
│   ├── 参数传递（$ARGUMENTS, $1, $2）
│   ├── 动态上下文注入（!`命令`）
│   └── 子 Agent 执行（context: fork）
│
└── 开放标准
    └── 跨平台支持：Cursor、GitHub、VS Code 等
```

---

## 1. Agent Skill 是什么？

把它想象成**给新员工的入职手册**：

```
员工入职手册包含：
  → 公司规范和标准
  → 常见任务的步骤流程
  → 模板和示例
  → 脚本和自动化工具

Agent Skill 是同一回事：
  → Claude 完成此类任务所需的领域知识
  → 如何处理这类任务的指南
  → 模板、示例、可执行脚本
  → 只在相关时才加载
```

**具体定义：** Skill 是一个包含必需的 `SKILL.md` 文件的**目录**：

```
my-skill/
├── SKILL.md           ← 必需：主要指令
├── template.md        ← 可选：输出模板
├── examples/
│   └── sample.md      ← 可选：示例输出
└── scripts/
    └── validate.sh    ← 可选：可执行脚本
```

---

## 2. SKILL.md 文件结构

每个技能从这里开始。它有两个部分：

```
───── SKILL.md ──────────────────────────────────
---                          ← YAML 前置元数据开始
name: my-skill
description: 这个技能做什么以及什么时候使用
disable-model-invocation: false
allowed-tools: Read, Grep
---                          ← YAML 前置元数据结束

# 你的技能指令

用普通 Markdown 写你的指令。
Claude 在技能被调用时读取这些内容。

包含：
- 要做什么
- 如何处理
- 不做什么
- 好输出的示例
──────────────────────────────────────────────────
```

---

## 3. 渐进式信息披露：核心设计原则

技能设计注重**上下文效率**。采用三层加载策略：

```
第一层：元数据（始终在上下文中）
  ┌────────────────────────────────────────┐
  │ name: "explain-code"                   │ ← ~50 token
  │ description: "解释代码..."             │   （始终加载）
  └────────────────────────────────────────┘

第二层：完整 SKILL.MD（相关时加载）
  ┌────────────────────────────────────────┐
  │ # 解释代码时，始终：                   │
  │ 1. 从类比开始                          │ ← ~500 token
  │ 2. 画一个图表                          │   （任务匹配时
  │ 3. 逐步讲解                            │    才加载）
  │ 4. 指出常见误区                        │
  └────────────────────────────────────────┘

第三层：关联文件（按需加载）
  ┌────────────────────────────────────────┐
  │ examples/react-hooks-explanation.md   │ ← ~2000 token
  │ examples/async-await-explanation.md   │   （只有需要时
  │ references/advanced-patterns.md       │    才加载）
  └────────────────────────────────────────┘
```

**为什么重要：** 大型参考文档永远不会淹没上下文。一个技能可以包含完整的 API 规范（5万 token），只在需要时才加载。

---

## 4. 调用模式

### 自动调用（Claude 决定）

```
Claude 读取描述字段，判断：
「这个技能与用户的请求相关吗？」

用户："这个认证代码是怎么工作的？"
Claude：检查技能..."explain-code" 的描述说：
        "用于解释代码如何工作时"
        → 匹配！自动加载并应用该技能。
```

### 手动调用（用户决定）

```
用户输入：/explain-code src/auth/login.ts
           ↑ 技能名称  ↑ 参数

Claude：加载 explain-code 技能，应用于 src/auth/login.ts
```

### 控制哪种模式可用

```yaml
# 默认：两种模式都有效
---
name: my-skill
description: 这个技能做什么
---

# 仅用户手动调用：防止 Claude 自动应用
---
name: deploy-prod
description: 部署到生产环境
disable-model-invocation: true   ← Claude 不会自动应用
---

# 仅 Agent 调用：对用户隐藏
---
name: internal-helper
user-invocable: false             ← 不出现在 / 菜单中
---
```

---

## 5. 完整前置元数据参考

```yaml
---
# 显示名称（可选——省略时使用目录名）
name: my-skill

# 功能描述（强烈建议）
description: 用类比和 ASCII 图表解释代码。
             当解释代码工作原理时使用。

# 自动补全时显示的提示（可选）
argument-hint: "[文件名] [详细程度]"

# 防止 Claude 自动加载（可选）
disable-model-invocation: true

# 对用户隐藏 / 菜单（可选）
user-invocable: false

# 无需权限提示就能使用的工具（可选）
allowed-tools: Read, Grep, Bash(git *)

# 覆盖使用的模型（可选）
model: claude-opus-4-6

# 在隔离子 Agent 上下文中运行（可选）
context: fork

# 使用哪种子 Agent 类型（可选，需要 context: fork）
agent: Explore    # 或：Plan, general-purpose, 或自定义 Agent 名称

# 技能范围的钩子（可选）
hooks:
  PostToolUse:
    - matcher: "Write"
      hooks:
        - type: command
          command: "git add -A"
---
```

---

## 6. 技能存放位置与优先级

```
优先级顺序（高优先级覆盖同名技能）：
  企业 > 个人 > 项目 > 插件

存放位置：

  企业：    管理设置（所有组织用户）

  个人：    ~/.claude/skills/<技能名>/SKILL.md
            （所有项目，全局有效）

  项目：    .claude/skills/<技能名>/SKILL.md
            （仅此项目，适合团队共享）

  插件：    <插件>/skills/<技能名>/SKILL.md
            （使用 插件名:技能名 命名空间）

注意：.claude/commands/review.md 和
      .claude/skills/review/SKILL.md
      是等效的——都创建 /review 命令
```

---

## 7. 向技能传递参数

### 基本参数

```yaml
---
name: fix-issue
description: 通过 issue 编号修复 GitHub issue
disable-model-invocation: true
---

按照我们的编码规范修复 GitHub issue $ARGUMENTS。

步骤：
1. 读取 issue：gh issue view $ARGUMENTS
2. 理解需求
3. 实现修复
4. 编写测试
5. 创建提交
```

使用方式：`/fix-issue 123` → Claude 看到"按照我们的编码规范修复 GitHub issue 123..."

### 位置参数

```yaml
---
name: migrate-component
description: 在框架之间迁移 UI 组件
---

将 $ARGUMENTS[0] 组件从 $ARGUMENTS[1] 迁移到 $ARGUMENTS[2]。
保留所有现有行为和测试覆盖率。
```

使用方式：`/migrate-component SearchBar React Vue`
- `$ARGUMENTS[0]` = SearchBar
- `$ARGUMENTS[1]` = React
- `$ARGUMENTS[2]` = Vue

### 可用变量

| 变量 | 含义 |
|------|------|
| `$ARGUMENTS` | 所有参数的字符串 |
| `$ARGUMENTS[N]` | 按索引获取特定参数（从0开始）|
| `$N` | `$ARGUMENTS[N]` 的简写 |
| `${CLAUDE_SESSION_ID}` | 当前会话 ID |

---

## 8. 高级功能：动态上下文注入

`` !`命令` `` 语法在技能发送给 Claude **之前**运行 shell 命令：

```yaml
---
name: pr-summary
description: 总结一个拉取请求
context: fork
agent: Explore
allowed-tools: Bash(gh *)
---

## 拉取请求上下文（实时获取）：
PR 差异：
!`gh pr diff`

PR 评论：
!`gh pr view --comments`

变更文件：
!`gh pr diff --name-only`

## 你的任务
审查上述 PR 并提供：
1. 变更内容和原因摘要
2. 潜在问题或风险
3. 测试建议
```

**技能运行时发生了什么：**
```
1. !`gh pr diff` 立即执行
   → 输出替换占位符
2. !`gh pr view --comments` 执行
   → 输出替换占位符
3. !`gh pr diff --name-only` 执行
   → 输出替换占位符
4. Claude 收到完整的、已填充好数据的提示词
   （包含真实的 PR 数据，而非命令本身）
```

这是**预处理**——Claude 永远看不到命令，只看到结果。

**提示：** 在技能内容中任意位置包含 `"ultrathink"` 可以为该技能启用扩展思考。

---

## 9. 高级功能：子 Agent 技能

添加 `context: fork` 可以在隔离的子 Agent 中运行技能：

```yaml
---
name: deep-research
description: 深入研究一个主题并返回详细报告
context: fork
agent: Explore
---

深入研究 $ARGUMENTS：

1. 使用 Glob 和 Grep 找出所有相关文件
2. 阅读并分析代码/文档
3. 交叉引用相关组件
4. 综合发现并附上具体文件引用
5. 返回结构化报告
```

**子 Agent 技能与普通技能的区别：**

```
普通技能：
  用户对话 → [技能加载到上下文] → Claude 执行
  （Claude 可以访问完整的对话历史）

子 Agent 技能（context: fork）：
  用户对话 → [新的隔离上下文] → 子 Agent 执行
                                        ↓
                  [结果摘要后返回主对话]
  （子 Agent 无法访问对话历史——全新上下文）
```

**何时使用子 Agent 技能：**
- 需要大量文件探索的任务（否则会污染主上下文）
- 中间结果不需要可见的研究类任务
- 应该被封装的长时多步骤流程

---

## 10. 技能类型：参考型 vs. 任务型

```
参考型技能——添加知识，内联运行
  用于：规范、模式、领域知识、风格指南

  示例：
  ---
  name: api-conventions
  description: 此代码库的 API 设计模式
  ---
  编写 API 端点时：
  - 使用 RESTful 命名：GET /resources，POST /resources，PATCH /resources/:id
  - 返回一致的错误格式：{error: string, code: string, details?: object}
  - 在控制器中验证请求，而非在服务层
  - 所有日期使用 ISO 8601 格式：2024-01-15T10:30:00Z

任务型技能——分步工作流，通常直接调用
  用于：部署、发布、代码生成、结构化流程

  示例：
  ---
  name: release
  description: 创建新版本发布
  context: fork
  disable-model-invocation: true
  ---
  创建 $ARGUMENTS 版本的发布：
  1. 用未发布的变更更新 CHANGELOG.md
  2. 更新 package.json 和 package-lock.json 中的版本号
  3. 运行测试：npm test
  4. 创建 git 标签：git tag v$ARGUMENTS
  5. 推送标签：git push origin v$ARGUMENTS
  6. 创建 GitHub 发布：gh release create v$ARGUMENTS
```

---

## 11. 安全注意事项

```
⚠️  重要安全规则：

1. 只安装来自可信来源的技能
   → 技能可以执行 bash 命令
   → 技能可以运行任意脚本

2. 使用前审查附带的脚本
   → 检查 Python/Shell 脚本是否有恶意行为
   → 查看外部网络连接

3. 检查外部 API 连接
   → 什么数据被发送出去？
   → 认证是否安全处理？

4. 使用 allowed-tools 限制权限
   → 不要给技能超过它需要的访问权限
   allowed-tools: Read        ← 只读技能
   allowed-tools: Bash(git *) ← 只允许 git 命令
```

---

## 12. 开放标准

Agent Skills 不只是 Anthropic 的专有功能：

```
开放标准：https://agentskills.io
GitHub：https://github.com/agentskills/agentskills

支持此标准的平台：
  Cursor         GitHub         VS Code
  Gemini CLI     OpenCode       OpenHands
  Roo Code       Letta          Goose（Block）
  Junie          Amp            Spring AI
  Databricks     Factory        以及更多...

这意味着：
  构建一次 → 部署到任何兼容的 Agent 平台
  团队技能可以跨所有工具使用
  社区可以共享和复用技能
```

---

## 快速开始：3 分钟创建你的第一个技能

```bash
# 第一步：创建技能目录
mkdir -p ~/.claude/skills/code-explainer

# 第二步：创建 SKILL.md
cat > ~/.claude/skills/code-explainer/SKILL.md << 'EOF'
---
name: code-explainer
description: 用类比和 ASCII 图表解释代码。当有人问"这是怎么工作的"或要求解释代码时使用。
---

解释代码时，按照此结构：

1. **一句话类比**：把代码比作日常生活中的东西
2. **ASCII 图表**：用图表展示流程或结构
3. **逐步讲解**：解释每个部分做什么
4. **常见误区**：初学者通常会误解什么？

保持口语化的表达风格。假设读者没有相关背景知识。
EOF

# 第三步：测试
# 方式 A：自然提问（Claude 自动检测匹配）
# "这个 useEffect hook 是怎么工作的？"

# 方式 B：直接调用
# /code-explainer src/hooks/useAuth.ts
```

---

## 何时该构建技能（速查）

```
应该构建技能：
  ✓ 你定期重复同类任务
  ✓ 任务需要专业领域知识
  ✓ 你希望输出格式/方法保持一致
  ✓ 工作流需要与团队共享
  ✓ 你想自动化多步骤流程
  ✓ 不同项目需要不同规范

不需要构建技能：
  ✗ 任务是一次性的或非常罕见的
  ✗ 任务足够简单，无需专门指导
  ✗ Claude 默认就能处理得很好
```

---

## 形象比喻帮助记忆

```
Agent Skills = 新员工入职手册

想象公司给新员工的入职手册：
  - 第一页：目录（技能的元数据）
    → 只需看标题就知道有什么信息
  - 相关章节：按需翻到（SKILL.md 全文）
    → 只在需要时才翻到这一章
  - 附录：详细参考资料（关联文件）
    → 大多数时候不需要，需要时能找到

渐进式披露 = 高效的入职手册设计
  不是一开始就让新员工读完所有手册
  而是「你需要做 X 任务时，参考第3章」
```
