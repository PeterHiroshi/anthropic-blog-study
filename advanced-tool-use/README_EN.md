# Advanced Tool Use on the Claude Developer Platform

> Source: https://www.anthropic.com/engineering/advanced-tool-use
> Published: November 24, 2025 | Anthropic Engineering

---

## Overview Mind Map

```
Advanced Tool Use (Beta)
├── Problem: Scaling tool use in production
│   ├── Too many tools = context bloat
│   ├── Too many results = intermediate data overload
│   └── Complex parameters = invocation errors
│
├── Feature 1: Tool Search Tool
│   ├── Problem: 55K tokens just for tool definitions
│   ├── Solution: Lazy-load tools on demand
│   └── Result: 85% context reduction, 54% accuracy boost
│
├── Feature 2: Programmatic Tool Calling
│   ├── Problem: 20 calls × huge results = 50KB context
│   ├── Solution: Claude writes code to orchestrate tools
│   └── Result: 37% fewer tokens, 5-10% accuracy boost
│
└── Feature 3: Tool Use Examples
    ├── Problem: JSON schema can't express usage conventions
    ├── Solution: Concrete examples alongside tool definitions
    └── Result: 72% → 90% accuracy on complex params
```

---

## Enable These Features

All three features are in beta. Activate with:

```python
client.messages.create(
    model="claude-sonnet-4-5-20250929",
    betas=["advanced-tool-use-2025-11-20"],
    # ... rest of your request
)
```

---

## Feature 1: Tool Search Tool

### The Problem: Context Explosion

Imagine you connect 5 MCP servers to your agent:

```
GitHub server      ≈ 12,000 tokens of tool definitions
Slack server       ≈  8,000 tokens
Sentry server      ≈  9,000 tokens
Grafana server     ≈ 14,000 tokens
Splunk server      ≈ 12,000 tokens
─────────────────────────────────
TOTAL              ≈ 55,000 tokens
... before a single word of conversation!
```

This isn't just wasteful — it actively **hurts performance**. Claude has to process 55K tokens of tool noise before doing anything useful.

### The Solution: Lazy Loading

```
Traditional approach:                 Tool Search approach:

ALL tool definitions               Only Tool Search Tool
loaded upfront                     loaded upfront (~500 tokens)
(55,000 tokens)                           ↓
     ↓                         User asks a question
User asks question                        ↓
     ↓                         Claude: "I need git tools"
Claude processes all 55K tokens           ↓
     ↓                         Tool Search: fetch git tools (~3K tokens)
Answers (maybe correctly)                 ↓
                               Total: ~8,700 tokens used
```

### How to Configure It

```python
# Mark tools for lazy loading
tool_definition = {
    "name": "create_github_issue",
    "description": "Create a new GitHub issue",
    "defer_loading": True,   # ← this is the key flag
    "input_schema": { ... }
}

# Claude starts with only the search tool
# and discovers other tools as needed
```

### Search Backend Options

```
Option A: Regex-based search
  → Simple, fast, no setup
  → Best for: small tool libraries

Option B: BM25-based search
  → Better relevance ranking
  → Best for: medium tool libraries

Option C: Custom embedding-based search
  → Best semantic matching
  → Best for: large tool libraries with semantic naming
```

### Performance Results

| Model | Before | After | Improvement |
|-------|--------|-------|-------------|
| Claude Opus 4 | 49% | 74% | +51% |
| Claude Opus 4.5 | 79.5% | 88.1% | +11% |

### When to Use Tool Search

```
Use Tool Search when:
  ✓ Tool definitions exceed 10,000 tokens
  ✓ You have 10+ tools available
  ✓ Tool selection accuracy is a problem
  ✓ Multiple MCP servers are connected

Skip Tool Search when:
  ✗ You have 3-5 simple tools
  ✗ All tools are always relevant
  ✗ Context budget is not a concern
```

---

## Feature 2: Programmatic Tool Calling

### The Problem: Intermediate Data Overload

Consider a budget compliance check: "For each of my 20 team members, get their Q3 travel expenses."

**Traditional approach:**
```
Claude → call get_expenses(person_1) → 100 expense lines → into context
Claude → call get_expenses(person_2) → 100 expense lines → into context
... × 20 people ...
Claude → context now contains 2,000 expense line items (50KB+)
Claude → synthesize answer from this massive context

Problems:
  - 2,000 items Claude never needed individually
  - 20 separate inference round-trips
  - 50KB+ irrelevant intermediate data in context
  - High error rate, slow, expensive
```

**Programmatic approach:**
```
Claude writes Python code:
  ──────────────────────────────────────────
  import asyncio

  async def check_compliance():
      tasks = [get_expenses(person) for person in team]
      results = await asyncio.gather(*tasks)   # parallel!
      violations = [r for r in results if r.amount > LIMIT]
      return f"{len(violations)} violations found"
  ──────────────────────────────────────────

Code executes in sandboxed environment
Only the final summary enters Claude's context (~1KB)

Benefits:
  - 2,000 items never touch Claude's context
  - Parallel execution (1 round-trip vs. 20)
  - Only meaningful result returned
  - Faster, cheaper, more accurate
```

### How to Configure It

```python
# Mark tools as callable from code execution
tool_definition = {
    "name": "get_expenses",
    "description": "Get expense records for a team member",
    "allowed_callers": ["code_execution_20250825"],  # ← key flag
    "input_schema": { ... }
}
```

### What Claude Generates

```python
# Claude writes async Python like this:
import asyncio

async def analyze_team_budgets(team_members: list):
    # Fetch all data in parallel
    expense_tasks = [
        get_expenses(member_id=m.id, quarter="Q3")
        for m in team_members
    ]
    all_expenses = await asyncio.gather(*expense_tasks)

    # Filter and aggregate (never enters Claude context)
    violations = [
        {"person": m.name, "amount": e.total}
        for m, e in zip(team_members, all_expenses)
        if e.total > TRAVEL_LIMIT
    ]

    # Only this summary enters Claude's context
    return f"Found {len(violations)} budget violations: {violations}"
```

### Performance Results

| Metric | Before | After |
|--------|--------|-------|
| Token usage | 43,588 | 27,297 (-37%) |
| Knowledge retrieval accuracy | 25.6% | 28.5% |
| GIA benchmark | 46.5% | 51.2% |

### When to Use Programmatic Tool Calling

```
Use when:
  ✓ Processing large datasets needing aggregation
  ✓ Multi-step workflows with 3+ dependent calls
  ✓ Filtering/transforming results before reasoning
  ✓ Parallel operations across many items
  ✓ Intermediate data should NOT influence reasoning

Skip when:
  ✗ Simple 1-2 tool calls
  ✗ You want Claude to reason about intermediate results
  ✗ Tools have significant side effects (proceed carefully)
```

---

## Feature 3: Tool Use Examples

### The Problem: Schema Is Not Enough

JSON Schema tells Claude *what structure* a tool accepts, but not *how* to use it well:

```json
{
  "name": "create_support_ticket",
  "input_schema": {
    "type": "object",
    "properties": {
      "title": {"type": "string"},
      "reporter_email": {"type": "string"},
      "priority": {"enum": ["low", "medium", "high", "critical"]},
      "escalation_contacts": {
        "type": "array",
        "items": {"type": "string"}
      },
      "metadata": {"type": "object"}
    }
  }
}
```

Schema can't tell Claude:
- When to include `reporter_email` vs. leave it out
- When does `critical` priority apply?
- What goes in `metadata`? When is it needed?
- What's the relationship between `priority` and `escalation_contacts`?

### The Solution: Examples

```python
tool_with_examples = {
    "name": "create_support_ticket",
    "description": "Create a customer support ticket",
    "examples": [
        {
            "description": "Minimal ticket - just a title",
            "input": {
                "title": "Login page not loading"
            }
        },
        {
            "description": "Standard ticket with reporter info",
            "input": {
                "title": "Payment processing fails for EU customers",
                "reporter_email": "sarah@company.com",
                "priority": "high"
            }
        },
        {
            "description": "Critical issue with full escalation",
            "input": {
                "title": "Production database unreachable",
                "reporter_email": "oncall@company.com",
                "priority": "critical",
                "escalation_contacts": ["cto@company.com", "devops@company.com"],
                "metadata": {"incident_id": "INC-2024-001", "affected_services": ["api", "web"]}
            }
        }
    ],
    "input_schema": { ... }
}
```

### Performance Results

```
Without examples: 72% accuracy on complex parameter handling
With examples:    90% accuracy on complex parameter handling
Improvement:      +18 percentage points
```

### Best Practices for Examples

```
Rules:
  1. Include 1-5 examples per tool (more = diminishing returns)
  2. Show the progression: minimal → partial → full
  3. Focus on ambiguous areas only (don't state the obvious)
  4. Use realistic data (not placeholder "foo" values)

When examples help most:
  ✓ Complex nested structures
  ✓ Context-dependent optional parameters
  ✓ Domain-specific conventions (date formats, ID patterns)
  ✓ When 2+ tools have similar schemas (disambiguation)

When to skip:
  ✗ Simple tools with 1-2 obvious parameters
  ✗ Self-explanatory parameter names
```

---

## The Three Features Together

These features target different bottlenecks:

```
┌────────────────────────────────────────────────────────────────┐
│                    Your Production Agent                       │
│                                                                │
│  Problem: Context bloat      → Solution: Tool Search Tool      │
│  Problem: Data overload      → Solution: Programmatic Calling  │
│  Problem: Wrong invocations  → Solution: Tool Use Examples     │
│                                                                │
│  Use together for maximum efficiency!                          │
└────────────────────────────────────────────────────────────────┘
```

### Decision Guide

```
When adding a new tool:
  → Has complex optional params? Add examples.
  → Has similar sibling tools? Add examples + namespace carefully.
  → Rarely needed? Mark defer_loading: True.
  → Returns huge data? Allow programmatic calling.

When overall performance is poor:
  → Token budget issue? → Tool Search Tool
  → Reasoning over data issue? → Programmatic Tool Calling
  → Wrong parameters? → Tool Use Examples
```

---

## Compatibility Notes

```
Tool Search Tool:
  ✓ Compatible with prompt caching
    (deferred tools excluded from initial cached prompt,
     only loaded post-search — cache still valid)

Programmatic Tool Calling:
  ✓ Tools execute in isolated sandbox
  ✓ async/await patterns supported
  ! Ensure tools are idempotent when possible

Tool Use Examples:
  ✓ Works alongside existing tool descriptions
  ✓ Additive — won't break existing implementations
```
