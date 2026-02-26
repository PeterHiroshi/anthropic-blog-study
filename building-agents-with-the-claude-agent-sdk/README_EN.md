# Building Agents with the Claude Agent SDK

> Source: https://claude.com/blog/building-agents-with-the-claude-agent-sdk
> Published: September 29, 2025 | Claude Blog

---

## Overview Mind Map

```
Claude Agent SDK
├── Core Philosophy
│   └── Give Claude a computer (same tools as programmers)
├── The Agent Loop
│   ├── Gather Context
│   ├── Take Action
│   └── Verify Work
├── Gather Context Methods
│   ├── Agentic file system search
│   ├── Semantic search
│   ├── Subagents
│   └── Compaction
├── Take Action Methods
│   ├── Tools (primary building blocks)
│   ├── Bash & scripts
│   ├── Code generation
│   └── MCPs (Model Context Protocol)
├── Verify Work Methods
│   ├── Rule-based feedback
│   ├── Visual feedback
│   └── LLM as judge
└── Agent Types You Can Build
    ├── Finance agents
    ├── Personal assistant agents
    ├── Customer support agents
    └── Deep research agents
```

---

## 1. Background: From Code SDK to Agent SDK

The Claude Code SDK was originally built to make Claude a powerful coding assistant. But something interesting happened:

```
Original purpose:    Coding tool for Anthropic developers
↓
Actual usage:        Deep research, video creation, note-taking,
                     and nearly ALL major agent loops at Anthropic
```

This expansion revealed a universal truth: **Giving Claude access to a computer unlocks general-purpose agent capabilities.** The SDK was renamed to **Claude Agent SDK** to reflect this broader vision.

---

## 2. Core Design Principle: Give Claude a Computer

The analogy is simple:

```
Just as programmers need:        Claude needs:
  → Find files                     → Search file system
  → Write/edit code                → Write/edit files
  → Run code                       → Execute bash commands
  → Debug errors                   → Analyze outputs
  → Iterate until it works         → Iterative feedback loop
```

When Claude has these same capabilities, it can tackle not just coding, but:
- Read and analyze CSV files
- Search the web
- Build visualizations
- Interpret metrics
- And countless other digital tasks

---

## 3. The Agent Loop (The Heart of Everything)

Every agent built with the SDK follows this three-step loop:

```
┌──────────────────────────────────────────────────┐
│                  AGENT LOOP                      │
│                                                  │
│  ┌─────────────┐    ┌─────────────┐              │
│  │   GATHER    │    │    TAKE     │              │
│  │  CONTEXT   │───→│   ACTION   │              │
│  └─────────────┘    └─────────────┘              │
│         ↑                  ↓                     │
│         └────────────────────────────────────    │
│              ┌─────────────┐                     │
│              │   VERIFY   │                     │
│              │    WORK    │                     │
│              └─────────────┘                     │
│                                                  │
│              (REPEAT until done)                 │
└──────────────────────────────────────────────────┘
```

Let's walk through each step using an **Email Agent** as our running example.

---

## 4. Step 1: Gather Context

Before acting, an agent needs to understand the situation. Here are the tools available:

### Method 1: Agentic File System Search

```
The file system = Agent's memory / knowledge base

How it works:
  Claude encounters a large file (e.g., email logs)
       ↓
  Claude decides: use grep? use tail? use full read?
       ↓
  Loads only relevant portions into context

Email Agent Example:
  Store conversations in 'Conversations/' folder
  → Agent greps for relevant past exchanges
  → Only loads relevant threads, not entire inbox
```

### Method 2: Semantic Search

```
Semantic Search vs Agentic Search:

                Semantic Search    Agentic Search
Speed:          Faster             Slower
Accuracy:       Less               More
Maintenance:    Hard               Easier
Transparency:   Lower              Higher

Recommendation: Start with agentic search.
                Add semantic only if you need speed.
```

### Method 3: Subagents

```
Why use subagents?

1. PARALLELIZATION
   Main Agent → spawn Subagent A (task 1)
             → spawn Subagent B (task 2)  ← run simultaneously
             → spawn Subagent C (task 3)
             ← collect all results

2. CONTEXT MANAGEMENT
   Problem: Sifting through 1000 emails uses huge context
   Solution: Subagent reads emails, returns only relevant excerpts
             (Main agent never sees the full 1000 emails)

Email Agent Example:
  Spawn multiple search subagents in parallel
  Each runs different queries against email history
  Return only relevant excerpts to orchestrator
```

### Method 4: Compaction

```
Problem: Long-running agents hit context limits
Solution: Compaction automatically summarizes past messages
          when approaching the context limit

This is built on Claude Code's /compact slash command.
Your agent won't run out of context in long sessions.
```

---

## 5. Step 2: Take Action

Once context is gathered, the agent acts using these mechanisms:

### Mechanism 1: Tools (Primary Building Blocks)

```
Tools are the MOST prominent items in Claude's context window.
Claude treats them as the primary available actions.

Design implication: Be intentional about which tools you expose.
                    Each tool takes up valuable context space.

Email Agent Tools:
  fetchInbox      ← primary frequent action
  searchEmails    ← primary frequent action
  draftEmail      ← primary frequent action
  sendEmail       ← high-stakes, maybe requires confirmation
```

### Mechanism 2: Bash & Scripts

```
Bash = General-purpose flexible computer access

Use when: Tasks are too specific for predefined tools
          but Claude can figure them out dynamically

Email Agent Example:
  User has important data in PDF attachments
  → Claude writes bash to: download PDF → convert to text → grep it
  → All done dynamically, no special tool needed
```

### Mechanism 3: Code Generation

```
Why code generation is powerful for agents:
  ✓ Code is precise (no ambiguity)
  ✓ Code is composable (reuse pieces)
  ✓ Code is infinitely reusable
  ✓ Code is verifiable (does it run? does it pass tests?)

Real example from Anthropic:
  Claude.ai file creation (Excel, PowerPoint, Word docs)
  → Claude writes Python scripts to create them
  → Ensures consistent formatting + complex functionality

Email Agent Example:
  User wants rules for inbound emails
  → Claude writes code to evaluate rules on incoming events
  → Consistent, testable, modifiable
```

### Mechanism 4: MCPs (Model Context Protocol)

```
MCPs = Pre-built integrations to external services

What they handle:
  Authentication flows
  API calls
  Rate limiting
  Error handling

Available integrations (out of the box):
  Slack, GitHub, Google Drive, Asana, and growing...

Email Agent Example:
  search_slack_messages  ← understand team context
  get_asana_tasks        ← check if request is already assigned

  No custom OAuth code needed!
```

---

## 6. Step 3: Verify Work

Agents that check their own output are fundamentally more reliable.

```
Verify = Catch mistakes before they compound
       = Self-correct when drifting
       = Get better through iteration
```

### Method 1: Rule-Based Feedback

```
Best form of feedback: Define clear rules, explain which failed and why.

Code linting is the gold standard example.

Email Agent Rules:
  ✓ Email address is valid format?    (error if not)
  ✓ Sent email to this address before? (warning if not)
  ✓ Subject line under 100 chars?     (error if not)
  ✓ No sensitive data in body?        (error if detected)
```

### Method 2: Visual Feedback

```
For visual tasks: show the model what it created.

Email Agent Example:
  Claude drafts HTML email
  → Screenshot the rendered email
  → Feed screenshot back to Claude
  → Claude checks: layout OK? styling correct? readable?
  → Iterate if needed

What to check visually:
  Layout: Are elements positioned correctly?
  Styling: Do colors/fonts appear as intended?
  Content hierarchy: Proper emphasis and order?
  Responsiveness: Does it look broken?

Tools: Playwright MCP server can automate this
```

### Method 3: LLM as Judge

```
Use another LLM to evaluate output quality.

When to use: When no clear rules exist, but quality matters
Warning: Slower + more expensive
         Not robust for binary pass/fail tasks
         Better for fuzzy quality assessment

Email Agent Example:
  Subagent judge evaluates draft tone
  → Does it match user's writing style?
  → Is it appropriately formal/casual?
```

---

## 7. Testing and Improving Your Agent

```
Question to ask when agent underperforms:

If agent MISUNDERSTANDS the task:
  → It lacks key information
  → Fix: Better search APIs / richer context

If agent FAILS a task repeatedly:
  → It's a systematic error
  → Fix: Add formal rules in tool calls to catch + fix it

If agent CAN'T FIX its own errors:
  → It needs different approaches
  → Fix: Give it more creative/varied tools

If agent PERFORMANCE VARIES with new features:
  → Build a representative test set
  → Run programmatic evaluations (evals)
  → Base evals on real customer usage patterns
```

---

## 8. Agent Types You Can Build

```
FINANCE AGENT
  What: Understands your portfolio and evaluates investments
  How:  Accesses external APIs + runs calculations + stores data

PERSONAL ASSISTANT AGENT
  What: Books travel, manages calendar, schedules appointments
  How:  Connects to internal data + tracks cross-app context

CUSTOMER SUPPORT AGENT
  What: Handles high-ambiguity support tickets
  How:  Reviews user data + connects to APIs + escalates to humans

DEEP RESEARCH AGENT
  What: Comprehensive research across large document collections
  How:  Searches file systems + synthesizes from multiple sources
        + cross-references data + generates detailed reports
```

---

## Quick Reference: SDK Capabilities

| Capability | What it enables | When to use |
|------------|----------------|-------------|
| File system search | Agent reads/searches files | Always useful |
| Semantic search | Fast vector search | When speed > accuracy |
| Subagents | Parallelism + context isolation | Complex multi-part tasks |
| Compaction | Long-running agents | Extended sessions |
| Tools | Primary defined actions | Core agent operations |
| Bash | Flexible computer access | Dynamic one-off tasks |
| Code generation | Precise, reusable logic | Complex computations |
| MCP integrations | External service access | Third-party services |
| Rule-based verification | Systematic quality checks | When rules are clear |
| Visual feedback | Check rendered output | UI/document generation |
| LLM judge | Fuzzy quality evaluation | Tone, style, nuance |
