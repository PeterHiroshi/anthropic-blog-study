# Desktop Extensions: One-click MCP Server Installation

> Source: https://www.anthropic.com/engineering/desktop-extensions
> Published: June 26, 2025 | Anthropic Engineering

---

## Overview Mind Map

```
Desktop Extensions (.mcpb)
├── Problem
│   ├── MCP install required dev tools (Node.js, Python)
│   ├── Manual JSON config editing
│   ├── Dependency conflicts
│   └── No discovery mechanism
├── Solution: .mcpb Format
│   ├── ZIP archive containing everything
│   ├── manifest.json (metadata + config)
│   ├── Bundled server + dependencies
│   └── One-click install via drag-and-drop
├── Architecture
│   ├── Built-in Node.js runtime
│   ├── Automatic updates
│   ├── OS keychain for secrets
│   └── Template literals for dynamic values
├── Development Workflow
│   ├── 1. npx @anthropic-ai/mcpb init
│   ├── 2. Define user config requirements
│   ├── 3. npx @anthropic-ai/mcpb pack
│   └── 4. Test via drag-and-drop
├── Advanced Features
│   ├── Cross-platform config overrides
│   ├── Feature declarations (tools, prompts)
│   ├── Directory access permissions
│   └── API key management
└── Enterprise
    ├── Group Policy / MDM controls
    ├── Pre-install approved extensions
    ├── Blocklist publishers
    └── Private extension directories
```

---

## 1. The Problem

Installing MCP servers before Desktop Extensions was a developer-only activity:

```
Previous MCP Installation:
1. Find an MCP server on GitHub
2. Install Node.js or Python (if not already)
3. Clone the repository
4. Install dependencies (resolve conflicts)
5. Manually edit claude_desktop_config.json
6. Configure environment variables
7. Restart Claude Desktop
8. Debug if something breaks
```

This locked out non-developer users and created a fragile setup process even for developers.

---

## 2. The Solution: .mcpb Format

Desktop Extensions are ZIP archives that bundle everything needed:

```
my-extension.mcpb (ZIP archive)
├── manifest.json          # Required: metadata + config
├── server.js              # Server implementation
├── node_modules/          # Bundled dependencies
├── icon.png               # Optional
└── screenshots/           # Optional
```

**Installation:** Drag the .mcpb file into Claude Desktop settings. Done.

---

## 3. Manifest Structure

The `manifest.json` declares server configuration and requirements:

Key features:
- **Template literals** for dynamic values:
  - `${__dirname}` — installation path
  - `${user_config.api_key}` — user-provided settings
- **User configuration schema** — declare what the user needs to provide (API keys, paths)
- **Feature declarations** — list tools and prompts the extension provides
- **Platform overrides** — different configs for Windows/macOS/Linux

---

## 4. Development Workflow

```bash
# 1. Initialize manifest
npx @anthropic-ai/mcpb init

# 2. Define user configuration in manifest.json
# (API keys, directory paths, etc.)

# 3. Package the extension
npx @anthropic-ai/mcpb pack

# 4. Test: drag .mcpb file into Claude Desktop settings
```

The toolchain handles bundling dependencies and creating the ZIP archive.

---

## 5. Key Capabilities Provided by Claude Desktop

| Capability | Benefit |
|-----------|---------|
| Built-in Node.js runtime | No external runtime installation needed |
| Automatic updates | Extensions stay current without user action |
| OS keychain storage | Secure storage for API keys and secrets |
| Directory access permissions | Granular control over filesystem access |

---

## 6. Enterprise Security

Organizations can manage Desktop Extensions through:

- **Group Policy / MDM controls** — Restrict which extensions can be installed
- **Pre-install approved extensions** — Deploy extensions to all users
- **Publisher blocklists** — Block extensions from untrusted sources
- **Private extension directories** — Internal-only extension repositories

---

## 7. Open Source Ecosystem

Anthropic open-sourced:
- MCPB specification
- Toolchain (init, pack commands)
- Reference implementations
- TypeScript schemas

This enables other AI applications to adopt the same extension format.

---

## 8. Connections to Other Articles

- **[Code Execution with MCP](../code-execution-with-mcp/)** — Desktop Extensions simplify distributing MCP servers like the code execution server discussed there
- **[Building Agents with Claude Agent SDK](../building-agents-with-the-claude-agent-sdk/)** — The Agent SDK's MCP tool integration is what Desktop Extensions make accessible to end users
- **[Writing Tools for Agents](../writing-tools-for-agents/)** — Tools designed with these principles can now be distributed as Desktop Extensions
- **[Agent Skills](../equipping-agents-for-the-real-world-with-agent-skills/)** — Desktop Extensions and Agent Skills both aim to extend Claude's capabilities through standardized packaging

---

## Key Takeaways

```
1. ONE-CLICK INSTALL REMOVES TECHNICAL BARRIERS
   → Non-developers can now use MCP servers

2. BUNDLE EVERYTHING IN A ZIP
   → manifest.json + server + dependencies = .mcpb

3. TEMPLATE LITERALS FOR DYNAMIC CONFIG
   → ${__dirname} and ${user_config.*} for flexibility

4. BUILT-IN RUNTIME ELIMINATES DEPENDENCIES
   → Claude Desktop provides Node.js, no external install

5. ENTERPRISE CONTROLS FOR SECURITY
   → MDM, blocklists, private directories

6. OPEN SOURCE SPEC ENABLES ECOSYSTEM
   → Other AI apps can adopt the same format
```
