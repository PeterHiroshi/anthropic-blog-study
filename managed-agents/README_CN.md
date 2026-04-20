# 扩展托管 Agent：将"大脑"与"双手"解耦

> 来源：https://www.anthropic.com/engineering/managed-agents
> Anthropic Engineering | 作者：Lance Martin、Gabe Cemaj、Michael Cohen

---

## 整体思维导图

```
扩展 Managed Agents
├── 问题：Harness 变成了"宠物"
│   ├── 单容器 = Claude + 工具 + 沙盒 + 会话
│   ├── 容器挂 = 会话丢
│   ├── 扩容靠手养
│   └── Harness 里写死的能力假设会过时
├── 灵感：操作系统级虚拟化
│   ├── 把硬件抽象成 process/file 等概念
│   └── 抽象要够通用，能承载尚未写出的程序
├── 解耦：大脑 vs 双手
│   ├── 大脑 = Claude + Harness（无状态）
│   ├── 双手 = 沙盒 + 工具（按需）
│   ├── 接口：execute(name, input) → string
│   └── 容器变成可替换的基础设施
├── 会话作为外部状态
│   ├── 追加式事件日志（持久化）
│   ├── 会话 ≠ Claude 上下文窗口
│   ├── wake(sessionId) + getSession(id) 恢复
│   └── getEvents() 做位置切片（回溯/恢复）
├── 凭据与代码分离
│   ├── 资源绑定鉴权（初始化阶段 Git token）
│   └── OAuth 存 vault，MCP proxy 中转
├── 性能
│   ├── p50 首 token ↓ ~60%
│   └── p95 首 token ↓ >90%
└── 多大脑 × 多双手
    ├── 任务可路由到不同 Harness 和沙盒
    ├── 平台不绑定任何具体 Harness
    └── Claude 能力升级时方便拆掉脚手架
```

---

## 1. "不要养宠物"

早期设计把 Claude、Harness、工具、沙盒、会话状态**全打进一个长生命周期容器**。这个容器是典型的"宠物"（pet）：

- 容器挂了 = **整个会话丢失**；
- 升级工具要动所有活会话；
- Agent 空闲也得占着整个 Claude + 工具镜像的资源。

类比："宠物 vs 牲口"——宠物要精心照料，死一只就是悲剧；牲口是可替换的，死一头无感。

但更深的问题是：**Harness 会把"Claude 当前做不到什么"的假设固化成代码**。模型升级后，这些脚手架就变成了"给鬼盖的房子"。

> 例子：Sonnet 4.5 时期为对抗"上下文焦虑"（context anxiety）而加的防护，Opus 4.5 时已经不需要了。但防护代码留在 Harness 里，反而悄悄扭曲了模型行为。

---

## 2. 大脑与双手解耦

修复思路借鉴操作系统：**把硬件虚拟化成抽象，而且抽象要通用到能承载尚未写出来的程序**。

| 层 | 旧架构 | 新架构 |
|----|--------|--------|
| 大脑（推理） | 绑定容器生命周期 | **无状态 Harness** |
| 双手（执行） | 烧在同一容器里 | **按需沙盒 + 工具** |
| 会话（状态） | 进程内存 | **外部追加式日志** |

Harness 调用双手只走一个接口：

```
execute(name: string, input: object) → string
```

容器成为**可替换的基础设施**：工具调用来了就起一个，跑完就销毁。大脑从不直接看见容器，只接到一个字符串返回。

---

## 3. 会话 ≠ Claude 的上下文窗口

这是最关键的概念区分。**会话（Session）** 是：

- 一个**追加式事件日志**，独立于任何进程；
- 用 `getEvents()` 按位置切片；
- 在进入 Claude 上下文之前可以任意变换（压缩、缓存优化、截断）。

Claude 的上下文窗口是短暂的；会话是持久的。你可以：

- **恢复**：从最后一条事件继续；
- **回溯**：取到某个时点为止的事件，从那里重入；
- **变形**：把事件塑形成当前 Harness 需要的提示结构。

Harness 崩了也能恢复：`wake(sessionId)` 启一个新的 Harness，`getSession(id)` 把历史交给它，继续执行。

---

## 4. 凭据与生成代码分离

沙盒里跑的是 Claude 生成的代码，所以**任何用户凭据都不能流进沙盒**。两种模式：

**模式 A — 资源绑定鉴权**
- 例：Git checkout 沙盒在初始化时收到一个**作用域 token**；
- token 只在 checkout 步骤存在，从不暴露给生成代码。

**模式 B — OAuth 通过 MCP Proxy**
- 用户的长期 OAuth token 留在安全 vault 里；
- MCP proxy 代理调用——沙盒只看到"已鉴权的工具"，看不到密钥。

---

## 5. 性能：解耦的红利

旧模型：每个会话都要先付**容器启动成本**，Claude 才能吐第一个 token。

新模型：大脑立刻开始推理；双手**按需**在工具调用时才被拉起。

| 指标 | 改进 |
|------|------|
| p50 首 token 延迟 | ↓ ~60% |
| p95 首 token 延迟 | ↓ >90% |

**p95 改善最值钱**——过去那种"容器启动慢了十几秒"的长尾 case 最消耗用户信任。

---

## 6. 多大脑 × 多双手

因为大脑 ↔ 双手的接口是窄的、无状态的，平台可以自由调度：

- **多大脑**：不同任务配不同 Harness（编码 Harness / 研究 Harness / 创意写作 Harness）。
- **多双手**：不同沙盒口味（临时 vs 持久、Linux vs 浏览器、本地 vs 远端）。

架构对"具体 Harness 长什么样"**保持不表态**。Harness 可以是任务专属的、短命的；接口不把它们焊死。

这是最大的收益：Claude 能力变强时，你可以**拆掉**自己的 Harness 脚手架，而不是在它上面继续加层。

---

## 7. 深度思考：为什么这是"基础设施层"的设计智慧

### 类比：操作系统的胜利

Unix 之所以能跑 40 年，不是因为实现多精巧，而是因为 **`read()` / `write()` / `fork()` 这样的抽象足够通用**，能承载 40 年前没想过的程序（浏览器、容器、AI 训练集群）。

Managed Agents 在做同样的事：把 Agent 平台的原语收窄到 `execute / getEvents / wake / getSession`，让未来的 Harness——我们现在还想象不到的 Harness——能直接复用。

### 对中国团队的启示

国内很多 Agent 平台早期做法是把**编排、工具、模型、会话**全耦合进一个"大框架"，升级 Claude 或换模型时要改一大片代码。这篇文章给出的反向建议：

1. **状态放在运行时之外**——会话是产品，运行时是耗材；
2. **接口越窄越长命**——`execute(name, input) → string` 看起来平庸，这就是它的优点；
3. **Harness 是脚手架，不是楼**——设计时就预期它会被拆掉。

### "Harness 会过时"的商业含义

Harness 里每一条 `if (model_can't_do_X)` 都是一条**技术债**，而且这条债的到期日是**模型下一次升级**。所以 Harness 代码应该：

- 集中在可审计的地方（不要散在 100 个文件里）；
- 有自动化手段识别"哪条假设已经过时"；
- 团队定期做"Harness 体检"，果断删除老的防护逻辑。

---

## 8. 与其他文章的关联

- **[Effective Harnesses for Long-Running Agents](../effective-harnesses-for-long-running-agents/)** — Managed Agents 是对原始"单 Harness"哲学的演进，把"一个长生命 Harness"泛化到"多个无状态 Harness + 一个持久会话"。
- **[Harness Design for Long-Running Application Development](../harness-design-long-running-apps/)** — 同期研究，讨论 Harness 的高层组件（Planner/Generator/Evaluator），它们其实都运行在本文的"大脑/双手"底座之上。
- **[Building Effective Agents](../building-effective-agents/)** — "最简脚手架"原则。本文把这个原则下沉到**平台层**。
- **[Claude Code Sandboxing](../claude-code-sandboxing/)** — 沙盒即"双手"，Managed Agents 把它进一步做成了**可远程化**的资源。

---

## 核心要点速查

```
1. 不要养宠物
   → 单容器 Harness 一挂就丢会话。

2. 大脑/双手解耦
   → 无状态 Harness + 按需沙盒 = 牲口，不是宠物。

3. 会话 ≠ 上下文窗口
   → 独立的事件日志让恢复变成例行公事。

4. OS 级虚拟化思想
   → 窄抽象活得比程序长。

5. 性能跟着架构走
   → p50 首 token ↓60%，p95 ↓90%，来自懒加载沙盒。

6. 多大脑 × 多双手
   → 任务可路由到合适的 Harness 和沙盒。

7. Harness 是脚手架
   → 它编码的是"模型时代假设"，要设计成可拆的。
```
