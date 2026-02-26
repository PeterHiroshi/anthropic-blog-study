# Writing Effective Tools for Agents — with Agents

> Source: https://www.anthropic.com/engineering/writing-tools-for-agents
> Published: 2025 | Anthropic Engineering

---

## Overview Mind Map

```
Writing Effective Tools for Agents
├── What is a Tool? (New mental model)
│   └── Contract between deterministic system + non-deterministic agent
│
├── Development Process (3 Phases)
│   ├── Phase 1: Build a Prototype
│   ├── Phase 2: Run an Evaluation
│   └── Phase 3: Collaborate with Agents
│
└── 5 Design Principles
    ├── 1. Choose the Right Tools
    │   ├── Agent affordances ≠ human affordances
    │   └── Consolidate granular tools
    ├── 2. Namespace Tools
    │   └── Prefix/suffix strategies
    ├── 3. Return Meaningful Context
    │   ├── Semantic identifiers > cryptic IDs
    │   └── concise/detailed modes
    ├── 4. Optimize Token Efficiency
    │   └── Pagination, filtering, truncation
    └── 5. Prompt-Engineer Tool Descriptions
        └── Tool desc = system prompt for tool behavior
```

---

## 1. What Is a Tool? A New Mental Model

Traditional software engineers think of functions as contracts between **deterministic systems**:
```
function add(a: number, b: number): number {
    return a + b;  // Always returns a + b. Predictable. Deterministic.
}
```

Tools for agents are fundamentally different — they're contracts between a **deterministic system and a non-deterministic agent**:

```
Weather question: "Should I bring an umbrella?"

What might an agent do?
  Option A: Call get_weather() tool       ← expected
  Option B: Use general knowledge         ← may be outdated
  Option C: Ask "which city?"             ← needs clarification
  Option D: Hallucinate about tool usage  ← bad, but possible

You cannot predict which path the agent takes.
Your tool design must account for this uncertainty.
```

**The goal of tool design:** Increase the surface area over which agents can be effective — enabling diverse problem-solving strategies while remaining intuitive for both agents and humans.

---

## 2. Development Process: Three Phases

### Phase 1: Build a Prototype

```
Step 1: Use Claude Code for rapid development
  → Provide documentation links for dependencies
  → Claude Code can read docs via llms.txt files

Step 2: Wrap in local MCP server for testing
  → claude mcp add <name> <command> [args...]

Step 3: Test manually
  → Try real use cases
  → Note where Claude struggles or gets confused

Step 4: Gather user feedback
  → What workflows do users actually expect?
  → What terminology do they use?
```

**Pro tip:** Official docs with `llms.txt` give Claude exactly what it needs to write code against your dependency.

### Phase 2: Run an Evaluation

**Generate Evaluation Tasks:**
```
Strong eval tasks (use these):
  ✓ "Schedule a meeting and attach the project proposal document"
  ✓ "Investigate why customer #4521 was double-charged last Tuesday"
  ✓ "Prepare a retention offer based on the customer's usage history"

  Why these are good:
    - Require multiple tool calls
    - Mirror actual workflow complexity
    - Have clear success criteria

Weak eval tasks (avoid these):
  ✗ "Schedule a meeting at 3pm"
  ✗ "Search for error logs"

  Why these are bad:
    - One-dimensional, single tool call
    - Don't reflect real-world complexity
```

**Run Evaluations Programmatically:**
```python
# Simple agentic evaluation loop
def run_eval(task, tools):
    messages = [{"role": "user", "content": task}]

    while True:
        response = call_llm(messages, tools)

        if response.stop_reason == "end_turn":
            return evaluate_result(response.content)

        # Execute tool calls
        tool_results = execute_tools(response.tool_calls)
        messages.append({"role": "assistant", "content": response.content})
        messages.append({"role": "user", "content": tool_results})

# Track these metrics per eval:
metrics = {
    "accuracy": ...,      # Did it complete correctly?
    "runtime": ...,       # How long did it take?
    "tool_calls": ...,    # How many tool calls?
    "tokens": ...,        # Total token consumption?
    "error_rate": ...,    # How often did it fail?
}
```

**Analyzing Results — Real Example:**
```
Anthropic found during their web search tool launch:

  Problem discovered: Claude was appending "2025" to search queries
  Effect:             Degraded search results (too narrow)
  Root cause:         Ambiguous tool description
  Fix:                Improved description → problem resolved

Lesson: Read the reasoning transcripts carefully.
        The agent's "thoughts" reveal the real problem.
```

### Phase 3: Collaborate with Agents

```
The Anthropic meta-workflow for tool optimization:

1. Run evaluations → collect failing transcripts
2. Paste transcripts into Claude Code
3. Ask: "Why did this agent fail? How should we fix the tool?"
4. Implement Claude's suggestions
5. Run evaluations again on HELD-OUT test set
6. Repeat

Result: Even "expert" implementations improved further
        Additional gains emerged from this iterative process
```

**Important:** Use a **held-out test set** to prevent overfitting. Don't evaluate on the same tasks you used to generate fixes.

---

## 3. Principle 1: Choose the Right Tools

### Agent Affordances vs. Human Affordances

```
Human programmer (deterministic, fast):
  → list_all_contacts()      ← iterate through 10,000 contacts
  → for contact in contacts: ← fast, no context cost
      process(contact)

LLM agent (non-deterministic, context-limited):
  → list_all_contacts()      ← gets 10,000 contacts as text
  → reads each one           ← uses enormous context
  → loses track halfway      ← context overflow or confusion

Solution: Design for HOW AGENTS THINK, not how programs run.
```

### The Consolidation Strategy

Instead of exposing raw, granular API endpoints:

```
BAD (granular, agent-unfriendly):          GOOD (consolidated, agent-friendly):

list_users()                               schedule_event()
list_events()               →              (internally: checks availability
create_event(user, time)                    books the slot, notifies attendees)

read_all_logs()                            search_logs(query, context_lines=2)
                            →              (internally: filters, returns only
                                           relevant lines + surrounding context)

get_customer_by_id()                       get_customer_context(customer_id)
list_transactions()         →              (internally: fetches customer data,
list_customer_notes()                       recent transactions, notes, all at once)
```

**Design test:** Would a human, given only these tools, naturally break the task down the same way? If yes, the tool design is right.

### How Many Tools Is Too Many?

```
Signs you have too many tools:
  ✗ Agent often picks the wrong tool
  ✗ Agent chains 5+ tools for tasks that should need 1-2
  ✗ Similar tools with overlapping purposes exist
  ✗ Context filled mostly with tool descriptions

Signs you have too few tools:
  ✗ Agent frequently uses bash for everything
  ✗ Agent asks users for information it should fetch
  ✗ Agent can't complete tasks without human help
```

---

## 4. Principle 2: Namespace Tools

When an agent has access to dozens of MCP servers, tool names can collide and confuse.

### Namespacing Strategies

```
Strategy A: Namespace by service
  asana_search          jira_search
  asana_create          jira_create
  asana_update          jira_update

Strategy B: Namespace by resource
  asana_projects_search     asana_projects_create
  asana_users_search        asana_users_create
  asana_tasks_search        asana_tasks_create

Choose consistently within a server.
```

### Why It Matters

```
Without namespacing:
  Agent sees: search(), create(), update()
  Agent thinks: "which service does 'search' refer to?"
  Result: Wrong tool called, errors, retries

With namespacing:
  Agent sees: asana_search(), jira_search()
  Agent thinks: "clearly I need asana_search for Asana data"
  Result: Correct tool, first attempt

Bonus: Reduces token usage for tool descriptions
       (shared prefix compresses mentally)
```

**Note:** Anthropic's testing found that **prefix vs. suffix** namespacing showed non-trivial performance differences *varying by model*. Test both in your evaluation framework.

---

## 5. Principle 3: Return Meaningful Context

### Semantic vs. Cryptic Identifiers

```
Cryptic (avoid):             Semantic (prefer):
uuid                    →    name
256px_image_url         →    image_url
mime_type               →    file_type
user_id_hash            →    username
evt_2024_q3_001         →    "Q3 2024 planning meeting"
```

**Why it matters:**
```
Agent with cryptic ID:
  Agent: "The user_id is 7f3a9c2b-... I need to look up..."
  → Additional tool call needed
  → Higher chance of misidentifying the user

Agent with semantic ID:
  Agent: "The user is 'Sarah Chen'..."
  → No additional lookup
  → Unambiguous, no confusion
```

### Controlled Verbosity: The Response Format Pattern

```python
# Give agents control over output verbosity
def search_slack(query: str, response_format: str = "concise"):
    results = fetch_from_slack(query)

    if response_format == "concise":
        return [{"text": r.text, "channel": r.channel} for r in results]
        # ~200 tokens for 5 results

    elif response_format == "detailed":
        return [{
            "text": r.text,
            "channel": r.channel,
            "thread_ts": r.thread_ts,    # needed for replies
            "user_id": r.user_id,        # needed for DMs
            "permalink": r.permalink,    # needed for links
            "reactions": r.reactions,    # context signal
            "file_attachments": r.files  # important attachments
        } for r in results]
        # ~600 tokens for 5 results (3x more)
```

**Rule of thumb:** Detailed responses use ~3x more tokens. Design the default around the most common use case.

---

## 6. Principle 4: Optimize Token Efficiency

### The Default Constraints

```
Claude Code: 25,000 token max on tool responses (default)
→ Design tools to stay well under this
→ Use pagination/filtering as defaults, not options
```

### Practical Patterns

```python
# Pattern 1: Pagination
def get_records(offset=0, limit=50):  # Default limit, not "get all"
    ...

# Pattern 2: Range Selection
def get_logs(start_time, end_time, level="error"):  # Filtered by default
    ...

# Pattern 3: Truncation with guidance
def search_documents(query, max_chars=5000):
    results = search(query)
    if total_chars(results) > max_chars:
        truncated = truncate(results, max_chars)
        return {
            "results": truncated,
            "truncated": True,
            "hint": "Results truncated. Use more specific queries to narrow results."
            # ← Guide the agent toward better strategy
        }
```

### Error Messages as Guidance

```
BAD error message:
  {"error": "INVALID_PARAMS", "code": 400}

  → Claude gets no useful information
  → May retry with same wrong parameters

GOOD error message:
  {
    "error": "Invalid date range",
    "provided": {"start": "2024-13-01", "end": "2024-12-31"},
    "issue": "Month value 13 is invalid",
    "correct_format": "YYYY-MM-DD",
    "example": {"start": "2024-01-01", "end": "2024-12-31"}
  }

  → Claude understands exactly what went wrong
  → Fixes it on the next attempt
```

---

## 7. Principle 5: Prompt-Engineer Tool Descriptions

Tool descriptions are loaded directly into the agent's context window. They are, in effect, **system prompts for tool behavior**.

### What to Include in Descriptions

```
Template for a high-quality tool description:

{tool_name}: {what it does, for what purpose}

Use when: {specific triggering conditions}
Do NOT use when: {anti-patterns, common mistakes}

Parameters:
  - {param1}: {description}. Format: {format}. Example: {example}
  - {param2}: {description}. Only include when: {condition}

Important notes:
  - {specialized query format if any}
  - {domain-specific terminology}
  - {relationship to other tools}
  - {known limitations}
```

### Real Impact: SWE-Bench Example

```
Anthropic's experience optimizing their coding agent:

Before description refinement:
  → Claude Sonnet 3.5: lower SWE-bench Verified score
  → Frequent path errors
  → Wrong tool selections

After precise tool description refinements:
  → SWE-bench Verified: state-of-the-art performance
  → Dramatically reduced errors
  → Improved completion rates

Lesson: Tool description quality = agent performance quality
```

### Don't Forget MCP Annotations

```
MCP tools should include annotations for:
  open-world access    ← tool can access internet/external data
  destructive changes  ← tool modifies/deletes data

Example MCP tool annotation:
{
  "name": "delete_files",
  "annotations": {
    "destructive": true,
    "requires_confirmation": true
  }
}
```

---

## Summary: The Tool Quality Checklist

```
DESIGN PHASE
  □ Does each tool match how an agent would naturally divide the task?
  □ Are granular tools consolidated into higher-level ones?
  □ Does each tool have a clear, single purpose?

NAMING PHASE
  □ Are tools namespaced consistently?
  □ Can you tell which service/resource each tool belongs to?
  □ Are parameter names unambiguous? (user_id not user)

RETURN VALUES PHASE
  □ Do return values use semantic identifiers?
  □ Is there a concise/detailed mode for verbosity control?
  □ Are responses paginated/filtered by default?

ERROR HANDLING PHASE
  □ Do error messages explain what went wrong specifically?
  □ Do error messages provide correct format examples?
  □ Do truncation messages guide the agent toward better strategies?

DOCUMENTATION PHASE
  □ Does the description explain when to use AND when NOT to use?
  □ Are specialized query formats documented?
  □ Are relationships between tools explained?
  □ Have you tested descriptions with your evaluation suite?
```
