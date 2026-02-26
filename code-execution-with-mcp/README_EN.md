# Code Execution with MCP: Building More Efficient Agents

> Source: https://www.anthropic.com/engineering/code-execution-with-mcp
> Published: 2025 | Anthropic Engineering

---

## Overview Mind Map

```
Code Execution with MCP
├── Core Problem: Two Inefficiencies
│   ├── Tool Definition Overhead — upfront context bloat
│   └── Intermediate Result Bloat — data passing through model
│
├── Solution: Agents Write Code to Use MCP
│   ├── On-demand tool loading via filesystem
│   ├── In-environment data filtering
│   └── 150K → 2K tokens (98.7% reduction)
│
├── Key Benefits
│   ├── Progressive Disclosure — load only what's needed
│   ├── Context Efficiency — filter before returning to model
│   ├── Control Flow — loops, conditions, error handling
│   ├── Privacy Preservation — PII stays in environment
│   └── State Persistence — builds reusable skills
│
├── Implementation Architecture
│   ├── TypeScript file-tree approach
│   ├── Each MCP server = directory
│   └── Each tool = individual file
│
└── Important Caveats
    ├── Requires sandboxing
    ├── Resource limits needed
    └── Monitoring infrastructure required
```

---

## 1. The Core Problem

### Problem 1: Tool Definition Overhead

**What it is:** Traditional MCP implementation loads all tool definitions upfront, consuming context tokens before any work begins.

```
Traditional MCP Setup:
┌─────────────────────────────────────────────────────┐
│ Context Window                                       │
│                                                     │
│ [Tool 1 definition: 500 tokens]                     │
│ [Tool 2 definition: 500 tokens]                     │
│ [Tool 3 definition: 500 tokens]                     │
│ ... (200 tools = 100,000 tokens just for definitions)│
│                                                     │
│ [Actual user request: 50 tokens]                    │
│ [Only needs Tool 37 and Tool 142]                   │
└─────────────────────────────────────────────────────┘

99% of tool definitions consumed context but never used!
```

### Problem 2: Intermediate Result Bloat

**What it is:** Data must pass through the model multiple times, especially problematic with large datasets.

```
Traditional approach for "summarize last 3 meetings":

1. Call meeting_retrieval_tool("all") → returns 200 meeting transcripts
2. [200 transcripts, ~50,000 tokens, all passed through model]
3. Model reads all → selects 3 relevant
4. Total: 50,050 tokens for a 3-meeting summary

With code execution approach:
1. Agent writes code:
   transcripts = get_all_meetings()
   recent_3 = sort_by_date(transcripts)[-3:]
   return recent_3

2. Code runs in environment → filters to 3 meetings
3. Only 3 meeting transcripts returned to model (~1,500 tokens)
Total: ~1,550 tokens — 97% reduction!
```

---

## 2. The Solution: Agents Write Code to Use MCP

**Core idea:** Instead of exposing MCP tools as direct callable functions, agents write **code** that calls MCP servers as a library.

```
Before (Direct Tool Calling):
User: "Get meeting from yesterday and summarize"
Agent: [needs to see all tool definitions first]
Agent: call meeting_tool(date="yesterday")
→ Full transcript returned to model (50K tokens)

After (Code Execution):
User: "Get meeting from yesterday and summarize"
Agent: [reads tool filesystem as needed]
Agent: writes code:
  import mcp_calendar
  meeting = mcp_calendar.get_meeting(date="yesterday")
  # Filter to just the key points before returning
  key_points = meeting.action_items + meeting.decisions
  return key_points
→ Only key points returned to model (~500 tokens)
```

**The paradigm shift:**
```
Old mental model: Agent as function caller
└── "I have these 200 functions, which should I call?"

New mental model: Agent as programmer
└── "I can write code that uses these libraries as needed,
     processing data in the environment before returning results"
```

---

## 3. Key Benefits

### 3.1 Progressive Disclosure

**What it is:** Tools are discovered and loaded on-demand, not upfront.

```
File-tree structure:
mcp_tools/
├── calendar/
│   ├── get_meeting.ts      (loads only when needed)
│   ├── create_event.ts
│   └── list_participants.ts
├── email/
│   ├── send.ts
│   └── search.ts
└── database/
    ├── query.ts
    └── insert.ts

Agent explores filesystem:
"I need calendar tools" → reads mcp_tools/calendar/ → loads only those files
vs.
"I need all tools" → never done! Too expensive.
```

**Reduction in context consumption:**
- Only loads definitions for tools actually needed
- Discovers capabilities by exploring filesystem structure
- Mirrors how human developers use documentation

### 3.2 Context Efficiency — Demonstrated Example

The article's core example shows a dramatic token reduction:

```
Scenario: Retrieve meeting transcript for AI/ML analysis

Traditional approach:
├── Load all tool definitions:     ~5,000 tokens
├── Retrieve full transcript:     ~50,000 tokens
├── API response metadata:        ~95,000 tokens
└── Total:                       ~150,000 tokens

Code execution approach:
├── Load relevant tool file:          ~500 tokens
├── Code to filter transcript:        ~200 tokens
├── Filtered result returned:       ~1,300 tokens
└── Total:                          ~2,000 tokens

Savings: 148,000 tokens = 98.7% reduction
```

### 3.3 Control Flow

**What it is:** Native code patterns replace chained tool calls.

```
Old way (chained tool calls):
call search_tool("meeting") → get IDs
call get_meeting_tool(id_1) → transcript 1
call get_meeting_tool(id_2) → transcript 2
call get_meeting_tool(id_3) → transcript 3
call summarize_tool(transcripts) → summary
= 5 round trips through model

New way (code execution):
meeting_ids = search("meeting")
transcripts = [get_meeting(id) for id in meeting_ids[:3]]
summary = "\n".join([extract_key_points(t) for t in transcripts])
= 1 code execution, no round trips
```

### 3.4 Privacy Preservation

**What it is:** Sensitive data stays in the execution environment and never reaches the model.

```
Traditional: User data → Tool → Model → Response
            [PII exposed to model]

Code Execution: User data → Code runs in environment
                → Automatically tokenize PII → Model sees tokens
                → Response (no PII in model context)
```

Example:
```python
# Code running in environment (model never sees this data directly)
users = get_all_user_records()
anonymized = [{
    "user_id": hash(u["email"]),  # email never sent to model
    "age_group": bucket_age(u["age"]),  # exact age never sent
    "purchase_count": u["purchases"]  # safe to share
} for u in users]
return anonymized  # only anonymized data reaches model
```

### 3.5 State Persistence

**What it is:** Agents can build and reuse skills over time within the execution environment.

```
Session 1: Agent writes utility function get_recent_meetings()
           → Saved in execution environment

Session 2: Agent discovers saved function
           → Reuses it without rewriting
           → Builds more complex functions on top

Over time: Library of reusable tools grows
           → Agent becomes progressively more capable
```

---

## 4. Implementation Architecture

The article describes a TypeScript file-tree approach:

```
Project Structure:
mcp_servers/
├── calendar_server/
│   ├── get_meeting.ts
│   ├── create_event.ts
│   └── index.ts          (server entry point)
│
├── email_server/
│   ├── send_email.ts
│   └── search_emails.ts
│
└── database_server/
    ├── query.ts
    └── upsert.ts

Each tool file contains:
- Function signature (clear input/output types)
- JSDoc description (what it does, when to use)
- Implementation
- Error handling
```

**Agent discovery process:**
```
1. Agent receives task: "Summarize my emails from this week"
2. Agent: list mcp_servers/ → sees email_server/
3. Agent: read email_server/ → sees search_emails.ts, send_email.ts
4. Agent: read search_emails.ts → understands function signature
5. Agent: writes code using search_emails
6. Code executes → returns filtered results
```

---

## 5. Important Caveats

The article is explicit about requirements that must exist before using code execution:

```
Prerequisites (non-negotiable):
├── Sandboxing
│   └── Code runs in isolated environment
│       (cannot access unauthorized resources)
│
├── Resource limits
│   └── CPU, memory, time limits on code execution
│       (prevents runaway processes)
│
└── Monitoring infrastructure
    └── Log all code executions
        Alert on suspicious patterns
        Audit trail for compliance
```

**Tradeoff:**
```
Efficiency gains:          98.7% token reduction
Infrastructure cost:       Sandboxing + monitoring required

Worth it when:   High-throughput agents, large datasets, privacy requirements
Not worth it:    Simple agents with few tools, low-volume use cases
```

---

## 6. Conceptual Positioning

The article positions this as "applying established software engineering patterns to agent development":

```
Software Engineering Principle → Agent Application

Abstraction / encapsulation  → Tools as libraries, not magic functions
Lazy loading                 → Progressive tool discovery via filesystem
Separation of concerns       → Data processing in environment, reasoning in model
Code reuse                   → Persistent skill library across sessions
```

---

## Quick Reference

**The core trade:**
```
Give up:  Direct, simple tool calls
Gain:     98.7% token reduction, privacy, state, control flow
```

**When to use code execution with MCP:**
- Large datasets that need filtering before reaching the model
- Many tools where most won't be used in any given task
- Privacy-sensitive data (PII, confidential)
- Complex multi-step workflows
- Long-running agents that benefit from accumulated skills

**When to skip it:**
- Few tools (< 10) that are all relevant most of the time
- Small data volumes where bloat isn't a concern
- Simple single-turn tasks without complex data processing
- Teams without sandboxing infrastructure
