# How We Built Our Multi-Agent Research System

> Source: https://www.anthropic.com/engineering/multi-agent-research-system
> Published: June 13, 2025 | Anthropic Engineering

---

## Overview Mind Map

```
Multi-Agent Research System
├── Architecture
│   ├── Lead Agent (Claude Opus 4) — orchestrator
│   └── Subagents (Claude Sonnet 4) — parallel workers
│
├── Performance
│   └── Multi-agent outperforms single-agent Opus 4 by 90.2%
│
├── 8 Prompting Principles
│   ├── 1. Think like your agents
│   ├── 2. Teach orchestration
│   ├── 3. Scale effort to complexity
│   ├── 4. Critical tool design
│   ├── 5. Let agents self-improve
│   ├── 6. Start wide, narrow down
│   ├── 7. Guide thinking processes
│   └── 8. Parallel tool calling
│
├── Evaluation Methods
│   ├── Small sample testing (20 queries)
│   ├── LLM-as-judge (rubric-based)
│   └── Human evaluation
│
└── Production Challenges
    ├── Token efficiency (15× more than chat)
    ├── State management across long runs
    ├── Non-deterministic debugging
    └── Rainbow deployments
```

---

## 1. System Architecture

**What it is:** Anthropic's Research feature (for Claude.ai) uses a multi-agent system where a lead orchestrator delegates research tasks to specialized subagents working in parallel.

```
User Query
     │
     ▼
┌────────────────────────────────────────────────────────┐
│              LEAD AGENT (Claude Opus 4)                │
│                                                        │
│  1. Analyzes query                                     │
│  2. Develops research strategy                         │
│  3. Identifies parallel research threads               │
│  4. Spawns subagents for each thread                   │
│  5. Synthesizes results into final answer              │
└────────────────────────────────────────────────────────┘
     │              │              │
     ▼              ▼              ▼
┌─────────┐  ┌─────────┐  ┌─────────┐
│Subagent │  │Subagent │  │Subagent │
│(Sonnet4)│  │(Sonnet4)│  │(Sonnet4)│
│Thread A │  │Thread B │  │Thread C │
└────┬────┘  └────┬────┘  └────┬────┘
     │              │              │
     ▼              ▼              ▼
  Web Search    Workspace      Integrations
  Results       Results        Results
     │              │              │
     └──────────────┴──────────────┘
                    │
                    ▼
           Synthesized Research
              (Final Answer)
```

**Performance result:** Multi-agent system with Claude Opus 4 lead + Claude Sonnet 4 subagents **outperformed single-agent Claude Opus 4 by 90.2%** in internal evaluations.

**Why subagents use a less powerful model (Sonnet 4):**
- Cost efficiency at scale
- Subagent tasks are more focused and bounded
- Lead agent handles complex synthesis and strategy

---

## 2. The Eight Prompting Principles

### Principle 1: Think Like Your Agents

**What it means:** Build simulations to understand how your agents experience their context and where they fail.

```
Without simulation:     With simulation:
"Why does it fail?"    → Build test harness
  → Guess             → Run 100 representative queries
  → Fix randomly      → Observe failure patterns
                       → Fix root causes
```

Understand the agent's perspective: what does it see? What decisions must it make? Where does ambiguity creep in?

### Principle 2: Teach Orchestration

**What it means:** Give the lead agent explicit instructions about how to delegate, what output formats to request, and where task boundaries lie.

Without orchestration instructions:
```
Lead Agent: "Research quantum computing"
Subagent A: [researches everything]
Subagent B: [researches the same things]
→ Duplication, wasted effort
```

With orchestration instructions:
```
Lead Agent: "Research quantum computing:
  - Subagent A: hardware developments (qubits, error correction)
  - Subagent B: software/algorithms (quantum circuits)
  - Subagent C: commercial applications and timeline
  Each: return 3-paragraph summary with key claims and sources"
→ Clear division, no overlap, consistent output format
```

### Principle 3: Scale Effort to Complexity

**What it means:** Embed explicit scaling rules in the prompt — agents should use more resources for complex queries and fewer for simple ones.

```
Simple queries:   1 agent, 3-10 tool calls
Medium queries:   2-5 subagents, focused research
Complex queries:  10+ subagents, comprehensive research
```

Example scaling rule in prompt:
```
"If the query requires synthesizing 5+ distinct topics or
  covering recent developments across multiple domains,
  spawn 8-12 subagents. For focused factual questions,
  1-3 subagents is sufficient."
```

### Principle 4: Critical Tool Design

**What it means:** Agent-tool interfaces require particularly careful design — distinct purposes and clear descriptions are essential.

| Good Tool Design | Bad Tool Design |
|------------------|-----------------|
| One tool per distinct capability | Overlapping tool purposes |
| Clear description of when to use | Ambiguous use cases |
| Consistent output format | Variable, unpredictable output |
| Explicit limitations stated | Unstated failure modes |

**Why it's harder for agents than for humans:** Humans can ask "when should I use X vs Y?" Agents must infer from descriptions alone.

### Principle 5: Let Agents Self-Improve

**What it means:** Claude models can diagnose their own failures and suggest prompt improvements.

```
Process:
1. Observe failure
2. Show agent the failure + ask: "Why did you fail? How should the
   prompt be changed to prevent this?"
3. Agent suggests improvements
4. Test and validate suggestions
5. Update prompt
```

This leverages Claude's meta-cognitive abilities to improve the very prompts that guide it.

### Principle 6: Start Wide, Narrow Down

**What it means:** Initial research queries should be broad to avoid missing relevant information; subsequent queries progressively focus.

```
Phase 1 — Wide:
"Search for 'quantum computing progress 2024'"
"Search for 'quantum error correction recent'"
"Search for 'IBM Google quantum announcements'"

Phase 2 — Narrow:
"More details on Google's Willow chip error rates"
"Specific timeline for fault-tolerant quantum computing"
"IBM quantum volume roadmap 2025-2030"

Insight: Starting narrow risks missing key context.
         Starting wide then narrowing is more reliable.
```

### Principle 7: Guide Thinking Processes

**What it means:** Using extended thinking mode improves instruction-following and research efficiency.

```
Without guided thinking:
→ Agent might jump to searching immediately
→ May miss obvious sub-questions
→ May pursue redundant threads

With extended thinking guidance:
→ Agent plans research strategy before acting
→ Identifies key questions to answer
→ Avoids wasted parallel threads
```

### Principle 8: Parallel Tool Calling

**What it means:** Parallelization can cut research time by up to 90% for complex queries.

```
Sequential research (traditional):
Search A → wait → Search B → wait → Search C → wait → Synthesize
Time: 30 seconds (10s per search)

Parallel research (multi-agent):
Search A ─┐
Search B  ├──▶ wait for all ──▶ Synthesize
Search C ─┘
Time: 12 seconds (10s max + 2s overhead)
```

**90% time reduction** for complex queries with 10+ searches.

---

## 3. Evaluation Methods

### Why Standard Evals Don't Work

Multi-agent systems take varied valid paths to solutions. There's no single "correct" series of tool calls — only "correct" final answers.

```
Query: "What are the latest breakthroughs in quantum computing?"

Valid path A: Search web → summarize → done
Valid path B: Search web → search academic → check Wikipedia → synthesize
Valid path C: Use 5 subagents searching different aspects → synthesize

All three might produce equally good answers via different routes.
→ Can't grade based on tool call sequence
→ Must grade based on output quality
```

### Small Sample Testing

Start with 20 representative queries drawn from real usage patterns.

**Query distribution for research system:**
- Queries requiring synthesis across sources
- Factual lookups with clear right answers
- Open-ended research questions
- Cross-domain queries

### LLM-as-Judge Evaluation

Use another LLM to score outputs against a rubric:

```
Evaluation Rubric:
├── Factual accuracy (0-10)
│   └── Are claims correct and well-supported?
├── Citation accuracy (0-10)
│   └── Do citations actually support the claims?
├── Completeness (0-10)
│   └── Does the answer cover all major aspects?
├── Source quality (0-10)
│   └── Are sources authoritative and current?
└── Tool efficiency (0-10)
    └── Were tool calls reasonable and non-redundant?
```

### Human Evaluation

Manual testing is irreplaceable for catching:
- Edge cases automated evals miss
- Source selection biases
- Subtle factual errors that "sound right"

The team found automated evals caught ~80% of issues; human review was needed for the remaining 20%.

---

## 4. Production Challenges

### Token Efficiency

```
Token usage comparison:
Chat interaction:        ~1× baseline
Single agent task:       ~4× baseline
Multi-agent system:      ~15× baseline

For every $1 in chat costs, expect ~$15 in multi-agent costs.
→ Must optimize prompts and tool calls aggressively
→ Use cheaper models for subagents when possible
```

### Engineering Challenges

| Challenge | Details | Solution |
|-----------|---------|----------|
| State management | Agents maintain state across long-running processes | Durable execution + error handling infrastructure |
| Non-determinism | Different runs take different paths | Log all paths; evaluate outputs not paths |
| Synchronous bottlenecks | Subagents must complete before synthesis | Async patterns; don't block on slow subagents |
| Deployment coordination | Can't interrupt running agents during deploy | Rainbow deployments (old+new versions coexist) |

### Rainbow Deployments

**What it is:** When deploying updates, old and new versions of agents run simultaneously. Long-running agent sessions use the version they started with; new sessions get the new version.

```
Deploy update:
├── Old version: continues serving in-flight agents
└── New version: starts serving new requests

Gradually:
├── Old version: fewer and fewer sessions
└── New version: more and more sessions

Eventually:
└── New version: 100% of traffic
```

---

## 5. Use Cases

The article shares top use cases for the Research system:

| Use Case | % of Usage |
|----------|-----------|
| Developing software systems | 10% |
| Creating professional content | 8% |
| Business strategy development | 8% |
| Academic research support | 7% |
| Information verification | 5% |

---

## Quick Reference

**The key architecture:**
```
Lead (Opus 4) → Strategy + Orchestration + Synthesis
Subagents (Sonnet 4) → Parallel research execution
```

**The 8 principles summary:**
1. Simulate to understand agent experience
2. Teach explicit orchestration rules
3. Scale resources to query complexity
4. Design tools with clear, distinct purposes
5. Use Claude to improve its own prompts
6. Start broad, then narrow research
7. Guide thinking with extended thinking mode
8. Parallelize aggressively

**Performance numbers:**
- 90.2% improvement over single-agent Opus 4
- 90% time reduction from parallelization
- 15× more tokens than chat (budget accordingly)
