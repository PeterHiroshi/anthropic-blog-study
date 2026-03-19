# 桌面扩展：Claude Desktop 一键安装 MCP 服务器

> 来源：https://www.anthropic.com/engineering/desktop-extensions
> 发布时间：2025年6月26日 | Anthropic Engineering

---

## 整体思维导图

```
桌面扩展（.mcpb）
├── 要解决的问题
│   ├── MCP 安装需要开发工具（Node.js、Python）
│   ├── 手动编辑 JSON 配置
│   ├── 依赖冲突
│   └── 无发现机制
├── 解决方案：.mcpb 格式
│   ├── 包含一切的 ZIP 归档
│   ├── manifest.json（元数据 + 配置）
│   ├── 捆绑的服务器 + 依赖
│   └── 拖放安装
├── 架构
│   ├── 内置 Node.js 运行时
│   ├── 自动更新
│   ├── OS 钥匙串存储密钥
│   └── 模板字面量支持动态值
├── 开发流程
│   ├── 1. npx @anthropic-ai/mcpb init
│   ├── 2. 定义用户配置需求
│   ├── 3. npx @anthropic-ai/mcpb pack
│   └── 4. 拖放测试
├── 高级特性
│   ├── 跨平台配置覆盖
│   ├── 功能声明（工具、提示）
│   ├── 目录访问权限
│   └── API 密钥管理
└── 企业安全
    ├── 组策略 / MDM 控制
    ├── 预装批准的扩展
    ├── 发布者黑名单
    └── 私有扩展目录
```

---

## 1. 问题：MCP 安装门槛太高

安装 MCP 服务器之前是一个只有开发者才能完成的操作：

```
此前的 MCP 安装流程：
1. 在 GitHub 上找到 MCP 服务器
2. 安装 Node.js 或 Python（如果没有的话）
3. 克隆仓库
4. 安装依赖（解决冲突）
5. 手动编辑 claude_desktop_config.json
6. 配置环境变量
7. 重启 Claude Desktop
8. 如果出错，开始调试...
```

这就像让普通用户通过编辑注册表来安装软件一样——技术上可行，但实际上把 99% 的用户挡在门外。

---

## 2. 解决方案：.mcpb 格式

Desktop Extensions 是包含所有必需内容的 ZIP 归档：

```
my-extension.mcpb（ZIP 归档）
├── manifest.json          # 必需：元数据 + 配置
├── server.js              # 服务器实现
├── node_modules/          # 捆绑的依赖
├── icon.png               # 可选：图标
└── screenshots/           # 可选：截图
```

**安装方式：** 将 .mcpb 文件拖到 Claude Desktop 设置界面。完成。

---

## 3. Manifest 结构

`manifest.json` 声明服务器配置和需求：

关键特性：
- **模板字面量**支持动态值：
  - `${__dirname}` — 安装路径
  - `${user_config.api_key}` — 用户提供的配置
- **用户配置模式** — 声明用户需要提供什么（API 密钥、路径等）
- **功能声明** — 列出扩展提供的工具和提示
- **平台覆盖** — 为 Windows/macOS/Linux 提供不同配置

---

## 4. 开发流程

```bash
# 1. 初始化 manifest
npx @anthropic-ai/mcpb init

# 2. 在 manifest.json 中定义用户配置
# （API 密钥、目录路径等）

# 3. 打包扩展
npx @anthropic-ai/mcpb pack

# 4. 测试：拖放 .mcpb 文件到 Claude Desktop 设置
```

工具链自动处理依赖打包和 ZIP 归档创建。

---

## 5. Claude Desktop 提供的关键能力

| 能力 | 收益 |
|------|------|
| 内置 Node.js 运行时 | 无需安装外部运行时 |
| 自动更新 | 扩展自动保持最新 |
| OS 钥匙串存储 | 安全存储 API 密钥和密码 |
| 目录访问权限 | 细粒度文件系统访问控制 |

---

## 6. 企业安全

组织可以通过以下方式管理 Desktop Extensions：

- **组策略 / MDM 控制** — 限制可安装的扩展
- **预装批准的扩展** — 向所有用户部署扩展
- **发布者黑名单** — 阻止来自不受信任来源的扩展
- **私有扩展目录** — 仅限内部的扩展仓库

---

## 7. 与其他文章的关联

- **[MCP 代码执行](../code-execution-with-mcp/)** — Desktop Extensions 简化了分发 MCP 服务器（如代码执行服务器）
- **[Agent SDK](../building-agents-with-the-claude-agent-sdk/)** — Agent SDK 的 MCP 工具集成通过 Desktop Extensions 对终端用户更可及
- **[工具设计原则](../writing-tools-for-agents/)** — 遵循工具设计原则的工具现在可以作为 Desktop Extensions 分发
- **[Agent Skills](../equipping-agents-for-the-real-world-with-agent-skills/)** — Desktop Extensions 和 Agent Skills 都旨在通过标准化打包扩展 Claude 的能力

---

## 8. 深度思考

### 对 MCP 生态的影响

Desktop Extensions 解决了 MCP 生态的一个关键瓶颈：**分发**。之前 MCP 有很好的协议设计，但安装过程阻碍了非技术用户的采用。.mcpb 格式就像移动应用的 .apk/.ipa 一样，为 MCP 生态打开了大众市场。

### 对比其他扩展生态

| 生态 | 安装方式 | 运行时 | 安全模型 |
|------|---------|--------|---------|
| VS Code 扩展 | 一键安装 | 内置 Node.js | 沙箱 + 权限 |
| Chrome 扩展 | 一键安装 | 浏览器 V8 | manifest + 权限 |
| **Desktop Extensions** | **拖放安装** | **内置 Node.js** | **钥匙串 + MDM** |

Desktop Extensions 借鉴了成熟的扩展生态模式，但针对 AI 助手场景做了定制（如钥匙串存储 API 密钥、目录权限控制）。

### 对国内开发者的机会

1. **构建中国特色的 MCP 服务器** — 集成微信、钉钉、飞书等国内工具
2. **企业内部工具分发** — 利用私有扩展目录部署内部 MCP 服务器
3. **开源贡献** — MCPB 规范和工具链是开源的，可以贡献跨平台改进

---

## 核心要点速查

```
1. 一键安装消除技术门槛
   → 非开发者也能使用 MCP 服务器

2. 一切打包在 ZIP 中
   → manifest.json + 服务器 + 依赖 = .mcpb

3. 模板字面量实现动态配置
   → ${__dirname} 和 ${user_config.*} 提供灵活性

4. 内置运行时消除依赖
   → Claude Desktop 提供 Node.js，无需外部安装

5. 企业控制保障安全
   → MDM、黑名单、私有目录

6. 开源规范推动生态
   → 其他 AI 应用可以采用相同格式
```
