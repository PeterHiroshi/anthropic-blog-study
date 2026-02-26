# Effective Context Engineering for AI Agents

> Source: https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents
> Published: September 29, 2025 | Anthropic Engineering

---

## Overview Mind Map

```
Effective Context Engineering for AI Agents
├── Core Problem: Context as Finite Resource
│   ├── "Context rot" — performance degrades as tokens increase
│   ├── Transformer n² pairwise token relationships
│   └── Training bias toward shorter sequences
│
├── Context Engineering vs Prompt Engineering
│   ├── Prompt Engineering: writing effective instructions
│   └── Context Engineering: curating optimal token set during inference
│
├── Key Techniques
│   ├── System Prompts — right altitude guidance
│   ├── Tools — self-contained, unambiguous, minimal overlap
│   ├── Examples — curate canonical, diverse few-shot examples
│   └── Runtime Retrieval — just-in-time information fetching
│
└── Long-Horizon Solutions
    ├── Compaction — summarize and re-initiate context
    ├── Structured Note-Taking — external memory files
    └── Sub-Agent Architectures — specialized focused agents
```

---

## 1. The Core Problem: Context Rot

**What it is:** As token counts grow, LLM performance degrades in a phenomenon the article calls "context rot." The root causes are architectural.

**Why it happens:**
```
Transformer Architecture
├── n² pairwise token relationships
│   └── Each new token interacts with ALL previous tokens
│       → computational cost explodes
└── Training data bias
    └── Models trained mostly on shorter sequences
        → performance decreases on longer inputs
```

**Key insight:** Context is not free. Every token in the window has a "cost" measured in degraded performance. The authors treat context as a **finite resource with diminishing marginal returns**.

```
Context Quality vs. Token Count

Performance
    ▲
    │ ████
    │      ████
    │           ████
    │                ████
    │                     ████
    └─────────────────────────────▶ Tokens in Context
              "Context Rot" Zone
```

---

## 2. Context Engineering vs. Prompt Engineering

**Prompt Engineering** = Writing effective instructions for a single turn.

**Context Engineering** = Curating and maintaining the **optimal set of tokens** during LLM inference across multiple turns.

| Dimension | Prompt Engineering | Context Engineering |
|-----------|-------------------|---------------------|
| Scope | Single prompt | Multi-turn operation |
| Focus | What to say | What tokens to include |
| Challenge | Phrasing clarity | Token budget management |
| When it matters | Any LLM use | Agents, long tasks |

As agents operate over multiple turns, context grows to include:
- System instructions
- Tool definitions
- External data
- Full message history

Managing this accumulation is the challenge context engineering addresses.

---

## 3. System Prompts: The "Right Altitude"

**What it is:** System prompts set agent behavior, but their design must avoid two failure modes.

```
System Prompt Design Space

Too Specific                        Too Vague
(Brittle)                          (Unhelpful)
    │                                   │
    │  if user asks X → do Y            │  "Be helpful"
    │  if user asks Z → do W            │  (Assumes shared context)
    │                                   │
    └────────────────┬──────────────────┘
                     │
              "Right Altitude"
           Specific enough to guide,
           flexible enough to adapt
```

**Avoid:**
- Rigid if-else logic that breaks on edge cases
- Vague platitudes that give no real direction

**Include:**
- Clear objectives and success criteria
- Boundaries on agent behavior
- Enough context for the agent to reason independently

---

## 4. Tools: Self-Contained and Unambiguous

**What it is:** Tool design directly impacts context efficiency and agent decision-making quality.

**The problem with bloated toolsets:**
```
Agent with 50 tools → "Which tool should I use here?"
                    → Confusion and errors
                    → Wasted tool calls
                    → Context bloat from retries
```

**Principles for good tool design:**

| Principle | What it means |
|-----------|---------------|
| Self-contained | Tool description has all info needed to use it |
| Unambiguous | Clear when to use this tool vs. others |
| Minimal overlap | Each tool has a distinct, non-redundant purpose |
| Right granularity | Not too broad (does everything) or too narrow (trivial) |

**Use when:** Tools are well-suited for discrete, deterministic operations with clear inputs/outputs.

**Avoid when:** Actions are too fuzzy to specify; consider giving the agent natural language instructions instead.

---

## 5. Examples: Curate, Don't Stuff

**What it is:** Few-shot examples remain valuable, but how you curate them matters enormously.

```
Bad Approach: Stuffing Every Edge Case
────────────────────────────────────────
Example 1: normal case
Example 2: edge case A
Example 3: edge case B
Example 4: edge case C
... (50 more examples)

Result: Context bloat. Key patterns buried. Performance degrades.

Good Approach: Curated Canonical Examples
────────────────────────────────────────
Example 1: most common case (demonstrates core pattern)
Example 2: second most different case (shows flexibility)
Example 3: one tricky edge case (calibrates expectations)

Result: Clean context. Core patterns clear. Good performance.
```

**Rule of thumb:** "Curate diverse, canonical examples that demonstrate expected behavior" rather than attempting to cover every edge case.

**Why diversity matters:**
- Helps the model understand the space of valid outputs
- Each example should teach something the others don't
- Redundant examples waste tokens with no benefit

---

## 6. Runtime Retrieval: Just-in-Time Information

**What it is:** Instead of pre-loading all potentially relevant data, agents maintain lightweight identifiers and fetch information only when needed.

```
Pre-loading Approach (Inefficient)
────────────────────────────────────
System Start: Load entire document library → 50K tokens consumed
Agent: "I need doc #3"  → Doc already loaded (wasted 49K tokens)

Runtime Retrieval Approach (Efficient)
────────────────────────────────────
System Start: Load index of document IDs → ~100 tokens
Agent: "I need doc #3"  → Fetch doc #3 → ~500 tokens consumed
                           Return result → Discard doc #3
```

**Mirrors human information-seeking patterns:** People don't memorize entire libraries; they know where to look and retrieve when needed.

**Implementation:**
```
Agent maintains:
├── File paths / IDs (tiny)
├── Database keys (tiny)
├── URL references (tiny)
└── Search indexes (small)

And fetches:
├── Actual file contents (large, temporary)
├── Query results (large, temporary)
└── External data (large, temporary)
```

---

## 7. Long-Horizon Solutions

For tasks that span many turns or multiple context windows, three strategies manage context:

### 7.1 Compaction

**What it is:** Summarize the current context and re-initiate a new context window with compressed information.

```
Context Window 1:          Context Window 2:
┌──────────────────┐       ┌──────────────────┐
│ Turn 1 (full)    │       │ Summary of W1    │  ← Compressed
│ Turn 2 (full)    │ ───▶  │ Turn 15 (full)   │
│ Turn 3 (full)    │       │ Turn 16 (full)   │
│ ...              │       │ ...              │
│ Turn 14 (full)   │       └──────────────────┘
│ [WINDOW FULL]    │
└──────────────────┘
```

**Trade-off:** Some detail is lost in summarization. Best for tasks where broad patterns matter more than exact details.

### 7.2 Structured Note-Taking

**What it is:** Agents maintain external memory files they consult and update across context resets.

```
External Memory File (persistent)
├── Current task status
├── Key decisions made
├── Files modified
├── Next steps
└── Important constraints

Agent reads notes → Works → Updates notes → New context window
                    ↑              ↓
                    └──────────────┘
                    Continuity preserved!
```

**Best for:** Long-running tasks where specific details (file paths, variable names, decisions) must be remembered exactly.

### 7.3 Sub-Agent Architectures

**What it is:** Specialized agents handle focused subtasks and return distilled summaries rather than dumping raw output into the main context.

```
Main Agent Context (stays clean)
├── High-level task state
├── Key decisions
└── Summary of sub-agent results

     ↓ delegates to ↓          ↑ returns summary ↑

Sub-Agent Context (isolated, can be large)
├── Full task details
├── All research data
├── Intermediate reasoning
└── Detailed exploration
```

**Benefit:** Sub-agents can have large, messy contexts without polluting the main agent's efficient working memory.

---

## 8. Key Insight: Larger Windows Don't Solve Everything

The article makes an important counter-intuitive point:

> "Even with larger context windows, information relevance concerns will persist, making these techniques perpetually valuable."

**Why:** The problem isn't just capacity — it's quality. Even in a 1M token window, having 900K tokens of irrelevant information degrades performance. Context engineering remains relevant regardless of window size.

---

## Quick Reference

| Technique | When to Use | Benefit |
|-----------|-------------|---------|
| System Prompt "Right Altitude" | Always | Guides without brittleness |
| Minimal Tool Set | Always | Reduces decision confusion |
| Curated Examples | Few-shot tasks | Better signal-to-noise |
| Runtime Retrieval | Large knowledge bases | Reduces token load |
| Compaction | Long multi-turn tasks | Extends effective window |
| Structured Notes | Multi-session tasks | Preserves exact details |
| Sub-Agents | Complex research/exploration | Keeps main context clean |

**Core Principle:** Treat context as a finite resource. Every token must earn its place by contributing to the agent's effectiveness. Remove, compress, or externalize anything that doesn't.
