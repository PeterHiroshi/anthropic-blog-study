# Building Effective Agents

> Source: https://www.anthropic.com/engineering/building-effective-agents
> Published: December 2024 | Anthropic Engineering

---

## Overview Mind Map

```
Building Effective Agents
├── Core Message
│   └── Simplest solution that meets needs > Most sophisticated solution
├── Foundations
│   ├── Augmented LLM (Retrieval + Tools + Memory)
│   └── Workflow vs Agent distinction
├── 5 Workflow Patterns
│   ├── 1. Prompt Chaining
│   ├── 2. Routing
│   ├── 3. Parallelization
│   ├── 4. Orchestrator-Workers
│   └── 5. Evaluator-Optimizer
├── Autonomous Agents
│   ├── How they work
│   ├── Risks
│   └── When to use
├── Design Principles
│   ├── Maintain Simplicity
│   ├── Prioritize Transparency
│   └── Engineer ACI carefully
└── Tool Design
    ├── Provide reasoning space
    ├── Use natural formats
    ├── Minimize formatting overhead
    ├── Poka-yoke (error-proof) design
    └── Write comprehensive docs
```

---

## 1. The Core Message

Anthropic has worked with dozens of organizations building LLM-based agentic systems. Their #1 finding:

> **The most successful implementations were NOT the most complex ones. They were teams using simple, composable patterns.**

**Start simple. Escalate only when needed.**

```
Optimization Path:
Single LLM call → Add retrieval → Chain steps → Build agents
     (try first)    (if needed)   (if needed)   (last resort)
```

---

## 2. The Foundation: Augmented LLM

Every agentic system is built on this core building block:

```
┌─────────────────────────────────────────┐
│            Augmented LLM                │
│                                         │
│   ┌──────────┐   ┌──────────────────┐   │
│   │ Retrieval│   │     Memory       │   │
│   │ (search) │   │ (what to keep)   │   │
│   └──────────┘   └──────────────────┘   │
│         ↘              ↗                │
│           ┌──────────┐                  │
│           │   LLM    │                  │
│           └──────────┘                  │
│                ↕                        │
│           ┌──────────┐                  │
│           │  Tools   │                  │
│           │(run code,│                  │
│           │call APIs)│                  │
│           └──────────┘                  │
└─────────────────────────────────────────┘
```

The model **actively uses** these capabilities — it generates search queries, picks which tools to call, and decides what to remember. Use **Model Context Protocol (MCP)** to integrate third-party tools.

---

## 3. Workflows vs. Agents

| | Workflow | Agent |
|---|---|---|
| **Control** | Predefined code paths | LLM directs its own process |
| **Flexibility** | Fixed flow | Dynamic, adaptive |
| **Cost** | Lower | Higher |
| **Predictability** | Higher | Lower |
| **Best for** | Structured, known tasks | Open-ended, unknown tasks |

---

## 4. The Five Workflow Patterns

### Pattern 1: Prompt Chaining

**What it is:** Break a task into a fixed sequence. Each step processes the previous output. Add "gates" to validate intermediate results.

```
Input → [LLM Step 1] →✓Gate→ [LLM Step 2] →✓Gate→ [LLM Step 3] → Output
```

**Use when:**
- Task has clear sequential sub-steps
- Accuracy > speed
- Intermediate outputs can be validated

**Example:**
```
Draft outline → ✓ (meets length?) → Write full article → ✓ (on topic?) → Translate
```

---

### Pattern 2: Routing

**What it is:** First classify the input, then send it to the right specialized handler.

```
                    ┌→ [Billing Handler]
Input → [Classifier]┼→ [Tech Support Handler]
                    └→ [Account Handler]
```

**Use when:**
- Workload has meaningfully different input types
- Different categories need different prompts/models
- You want to save cost (simple queries → cheap model)

**Example:**
```
Customer query → classify → "billing"? → billing-expert prompt
                         → "tech"?     → tech-expert prompt + Haiku model
                         → "complex"?  → Sonnet/Opus model
```

---

### Pattern 3: Parallelization

**What it is:** Run multiple LLM calls simultaneously. Two variants:

```
Sectioning:                      Voting:
Task → [Sub-task A] → ┐         Task → [LLM Run 1] → ┐
     → [Sub-task B] → ┤ Merge        → [LLM Run 2] → ┤ Consensus
     → [Sub-task C] → ┘              → [LLM Run 3] → ┘
```

**Use when:**
- Sub-tasks are independent (sectioning)
- Need higher confidence / reduce variance (voting)
- Speed is critical

**Example:**
```
Code review (sectioning):
  Run security check ──┐
  Run style check    ──┤ → Combined report
  Run logic check    ──┘

Translation (voting):
  Model A translation ──┐
  Model B translation ──┤ → Most consistent result
  Model C translation ──┘
```

---

### Pattern 4: Orchestrator-Workers

**What it is:** A central orchestrator dynamically breaks down a task and delegates sub-tasks to workers.

```
                    ┌─ [Worker: File A edits]
[Orchestrator] ────┼─ [Worker: File B edits]  → [Orchestrator synthesizes]
  (analyzes        └─ [Worker: Tests]
   problem &
   delegates)
```

**Key difference from Parallelization:** Sub-tasks are **not predefined** — the orchestrator decides based on its analysis.

**Use when:**
- Complex, unpredictable tasks
- You don't know the solution path upfront
- Sub-tasks depend on intermediate analysis

**Example:**
```
"Fix this bug across the repo"
→ Orchestrator reads codebase structure
→ Delegates: Worker A edits module X, Worker B edits tests, Worker C updates docs
→ Orchestrator reviews combined result
```

---

### Pattern 5: Evaluator-Optimizer

**What it is:** A two-LLM feedback loop — one generates, one evaluates and provides feedback. Repeat until quality meets the bar.

```
[Generator] → output → [Evaluator] → feedback → [Generator] → output → ...
                              ↓
                    (stop when quality OK)
```

**Use when:**
- Clear evaluation criteria exist
- Iterative refinement demonstrably improves quality
- Task is hard to nail in one shot

**Example:**
```
Translation loop:
  LLM translates → Evaluator checks fluency/fidelity → "needs more natural phrasing"
  → LLM revises  → Evaluator: "approved" → Done
```

---

## 5. Autonomous Agents

Beyond workflows, fully autonomous agents take control of an open-ended loop:

```
                    ┌──────────────────────────────┐
                    │           AGENT              │
Goal ──────────────→│                              │
                    │  receive → decide → act      │
                    │     ↑                ↓       │
                    │  feedback ←── environment    │
                    │     (tool outputs, errors)   │
                    │                              │
                    │  (continues until done)      │
                    └──────────────────────────────┘
```

**Agents are best for:**
- Open-ended problems with unknown solution paths
- Tasks where success can be verified (e.g., tests pass)
- Long-running tasks with checkpoints

**Risks to manage:**
- Error compounding over many steps
- Higher cost and latency
- Unpredictable behavior

**Requirements:**
- Run in **sandboxed environments**
- Include **human oversight checkpoints**
- Have appropriate **guardrails**

---

## 6. Three Agent Design Principles

### Principle 1: Maintain Simplicity
```
Rule: Minimum viable architecture
Don't add complexity without measurable benefit
```

### Principle 2: Prioritize Transparency
```
Rule: Show the agent's work
Explicit planning steps → interpretable → debuggable → trustworthy
```

### Principle 3: Engineer ACI Carefully
```
UI (User Interface) : humans
ACI (Agent-Computer Interface) : LLM agents

Just as you invest in UI for users,
invest in ACI quality for agents.
```

---

## 7. Tool Design Guide

### The Golden Rules

| Rule | Bad Example | Good Example |
|------|-------------|--------------|
| Give reasoning room | Force immediate format | Let model think first |
| Use natural formats | Invented markup syntax | Standard markdown/JSON |
| Minimize overhead | Count-specific formats | Simple structures |
| Poka-yoke design | Allow relative paths | Require absolute paths |
| Document thoroughly | No docs | Full docs with examples |

### What "Comprehensive Tool Documentation" Looks Like

```markdown
## tool_name

What it does: [clear description]

Parameters:
- param1 (required): [description + format]
- param2 (optional): [description + when to use]

Examples:
  tool_name("good/input/here")     # normal case
  tool_name("edge/case/input")     # edge case

Does NOT do: [explicit boundaries]

Common mistakes to avoid: [list]
```

---

## 8. Real-World Applications

### Customer Support Agents
```
Why it works:
✓ Combines conversation + external data access
✓ Clear success metric: resolution rate
✓ Well-scoped domain
✓ Natural human escalation paths

Business model emerging: pay-per-resolved-ticket
```

### Coding Agents
```
Why it works:
✓ Objective verification via tests
✓ Natural feedback loop (run → fail → fix → pass)
✓ Well-structured problems
✓ Measurable quality

Note: Human review still essential for broader alignment
```

---

## 9. Framework Guidance

```
Framework = Abstraction = Hidden complexity

Recommendation:
1. Start with direct API calls (understand mechanics)
2. Only adopt frameworks when you understand what they abstract
3. Always be able to inspect what happens underneath

Approved frameworks: Claude Agent SDK, LangChain, Rivet, Vellum
```

---

## Key Takeaways (Quick Reference)

```
1. START SIMPLE
   → Best architecture = simplest that meets needs

2. KNOW THE DIFFERENCE
   → Workflow = fixed paths | Agent = dynamic decisions

3. BUILD EVALUATIONS FIRST
   → Know when to upgrade complexity

4. INVEST IN TOOL DESIGN
   → ACI quality = prompt quality in importance

5. KEEP HUMANS IN THE LOOP
   → Checkpoints for long/high-stakes tasks

6. UNDERSTAND YOUR FRAMEWORKS
   → Direct API calls first, frameworks second

7. COMPLEXITY IS A COST
   → Only pay it when the benefit is proven
```
