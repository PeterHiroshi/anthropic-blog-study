# The "Think" Tool: Enabling Claude to Stop and Think

> Source: https://www.anthropic.com/engineering/claude-think-tool
> Published: March 20, 2025 | Anthropic Engineering

---

## Overview Mind Map

```
The Think Tool
├── What it is
│   ├── A "scratch pad" for mid-response reasoning
│   └── No side effects — just structured thinking space
│
├── How it differs from Extended Thinking
│   ├── Extended Thinking: BEFORE response (pre-planning)
│   └── Think Tool: DURING tool call chains (mid-response)
│
├── When to use it
│   ├── ✓ Complex tool call chains
│   ├── ✓ Policy-heavy environments
│   └── ✓ Sequential decisions with downstream consequences
│
├── Performance Results
│   ├── Airline customer service: +54% (49% → 57%)
│   ├── Retail customer service: +3.7% (78.3% → 81.2%)
│   └── SWE-Bench: +1.6% (contributed to 0.623 SOTA)
│
└── Implementation
    ├── JSON spec (simple: just a "thought" string)
    ├── System prompt placement
    └── Domain-specific examples in prompt
```

---

## 1. What Is the Think Tool?

Imagine you're a consultant. You get a call, listen to the client's situation, then... you need to think before advising. You don't want to think out loud (that would be confusing for the client), and you don't want to keep them waiting while you plan silently. You want a moment to reason clearly before giving your response.

**The think tool is exactly that moment for Claude.**

```
Without think tool:

  [Tool call result arrives]
         ↓
  Claude immediately decides next action
  (may miss complex policy implications,
   may not fully analyze tool output)

With think tool:

  [Tool call result arrives]
         ↓
  Claude uses think tool:
  {
    "thought": "The customer is asking for a refund on a flight
    cancelled 48 hours ago. Per policy section 3.2, flights
    cancelled within 72 hours are eligible for full refund IF
    the cancellation was weather-related. The flight data shows
    'weather delay' as reason. So: YES, full refund applies.
    But I also need to check if they've already used any miles
    bonus from this booking..."
  }
         ↓
  Claude proceeds with clear, policy-compliant decision
```

The key: **the think tool has no side effects**. It's a space for reasoning, not an action. Nothing is executed.

---

## 2. Think Tool vs. Extended Thinking

These are two different tools for different situations:

```
┌─────────────────────────────────────────────────────────────────┐
│                         Extended Thinking                       │
│                                                                 │
│  WHEN: Before Claude begins responding                          │
│  PURPOSE: Comprehensive pre-response planning                   │
│                                                                 │
│  User query → [Extended Thinking phase] → Response with tools  │
│                                                                 │
│  Good for:                                                      │
│    - Complex math problems                                      │
│    - Strategic planning questions                               │
│    - Non-sequential tool calls                                  │
│    - Simple instruction-following                               │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                           Think Tool                            │
│                                                                 │
│  WHEN: DURING an ongoing tool call chain                        │
│  PURPOSE: Analyze NEW information between tool calls            │
│                                                                 │
│  User query → Tool call → [RESULT] → Think → Tool call → ...   │
│                                        ↑                        │
│                                 reason about result             │
│                                 before next action              │
│  Good for:                                                      │
│    - Long tool call chains (5+ calls)                           │
│    - Policy-heavy domains                                       │
│    - Sequential decisions with dependencies                     │
│    - When mistakes cascade downstream                           │
└─────────────────────────────────────────────────────────────────┘
```

**Key insight from benchmark results:** In the retail domain, the think tool (81.2%) actually *outperformed* extended thinking (77.0%). This suggests that for real-time, sequential decision-making, mid-response reasoning beats pre-response planning.

---

## 3. Technical Implementation

### The JSON Spec (Simple by Design)

```json
{
  "name": "think",
  "description": "Use this tool to think through complex situations before taking action. Call it when you need to analyze tool outputs, check policy compliance, or reason through multi-step decisions.",
  "input_schema": {
    "type": "object",
    "properties": {
      "thought": {
        "type": "string",
        "description": "Your detailed reasoning about the current situation"
      }
    },
    "required": ["thought"]
  }
}
```

That's it. One string parameter. No side effects. Always returns an empty response.

### Where to Place It (System Prompt)

```
Option A: Inside tool description (suboptimal)
  → Claude treats it as "one of many tools"
  → May not use it contextually enough

Option B: In the system prompt (recommended)
  → Claude understands it as a core reasoning mechanism
  → Gets broader context for when to apply it
  → Better integration with task instructions

Recommended approach:
  Include the think tool definition in your tool list,
  AND add usage instructions in the system prompt.
```

### System Prompt Examples That Work

The article found that adding **concrete usage examples** dramatically improved results (especially for the airline domain where they saw the 54% gain):

```
## System prompt excerpt for airline customer service

You have access to a "think" tool. Use it to reason through
complex decisions before acting.

When to use it:
  - After receiving flight data, before determining refund eligibility
  - When multiple policy rules might apply
  - When you need to verify you have all required information

Example of good think tool usage for flight cancellations:

  think: "Customer is requesting refund for flight UA123 on Dec 15.
  Flight was cancelled. Policy section 3.2 says:
  - Weather cancellations: full refund within 7 days
  - Airline cancellations: full refund or rebooking
  - Customer cancellations: depends on fare class

  The cancellation code is 'WX' (weather). So rule: full refund applies.

  But I also need to check:
  - Has the customer already been rebooked? (check rebooking_db)
  - Any travel insurance claims? (check insurance_db)
  - Miles earned that would need reversal? (check loyalty_db)

  I need to call these three tools before confirming the refund."
```

---

## 4. Performance Results Deep Dive

### τ-bench (tau-bench) Benchmark

tau-bench tests agents in customer service scenarios requiring complex policy navigation. The key metric is **pass^k**: probability that ALL k independent trials succeed. This is stricter than single-attempt accuracy — it rewards consistency.

#### Airline Domain Results

```
Task complexity: High (complex fare rules, cancellation policies,
                 rebooking logic, refund calculations)

                             pass^1 score
Without think tool:            0.370
With think tool only:          0.450
With think tool + optimized    0.570
prompt with examples:

Relative improvement: 54%

The prompt examples were crucial — they showed Claude:
  - How to enumerate applicable rules
  - How to verify information completeness
  - How to check policy compliance step-by-step
  - How to iterate over tool results for edge cases
```

#### Retail Domain Results

```
Task complexity: Medium (product returns, exchanges,
                 availability checks)

                             pass^1 score
Extended Thinking:             0.770
Baseline (no think tool):      0.783
Think tool:                    0.812

Surprising finding:
  - Think tool BEAT extended thinking by 4.2%
  - Even beat baseline without additional prompting
  - Domain complexity affects which tool is needed
```

### Why the Difference Between Domains?

```
Airline (complex policy):
  → Many interdependent rules
  → Mistakes early compound downstream
  → Agent NEEDS explicit reasoning checkpoints
  → Think tool + prompting: big gains

Retail (simpler):
  → Fewer policy interactions
  → Think tool helps even without prompting
  → Extended thinking less useful (overkill)
```

### SWE-Bench (Software Engineering)

```
Sample with think tool:    30 tasks
Sample without:           144 tasks

Improvement: +1.6% average accuracy
Statistical test: Welch's t-test
  t(38.89) = 6.71, p < .001, d = 1.47
  (Large effect size, highly significant)

Contribution: Part of Claude 3.7 Sonnet achieving
              0.623 state-of-the-art score on SWE-Bench
```

---

## 5. When to Use vs. Avoid

```
✓ USE THE THINK TOOL WHEN:

  1. Long tool call chains
     "I'll need to call 5+ tools before I can answer"
     → Think after each critical result

  2. Policy-heavy environments
     "There are 50 rules about refunds/eligibility/exceptions"
     → Think before applying any policy decision

  3. Sequential decisions with consequences
     "My decision now constrains what I can do next"
     → Think before each fork in the decision tree

  4. Analyzing complex tool outputs
     "This database result has 20 fields and I need the right ones"
     → Think to identify what matters before proceeding

✗ AVOID THE THINK TOOL WHEN:

  1. Simple, single tool calls
     "Just fetch the weather for this city"
     → No analysis needed

  2. Non-sequential/parallel tool calls
     "Fetch 3 things at once, they don't depend on each other"
     → Think doesn't add value here

  3. Simple instruction-following
     "Translate this text"
     → No complex reasoning required

  4. When cost is the primary constraint
     "We need to minimize token count above all else"
     → Think tool adds tokens for output reasoning
```

---

## 6. Getting Started: 3 Steps

```
Step 1: IDENTIFY
  Find your agent's hardest scenarios:
  - Where does it fail most often?
  - Which tasks require the most retries?
  - Where do policy errors compound?

Step 2: ADD
  Add the think tool to your tools list.
  Customize the description for your domain.
  Add 2-3 examples in the system prompt showing
  GOOD think tool usage for YOUR specific scenario.

Step 3: MONITOR & REFINE
  Watch what Claude actually writes in think calls.
  Do the thoughts reveal confusion about your policies?
  → Clarify those policies in the system prompt.
  Do the thoughts show good reasoning but wrong conclusions?
  → Your tool data may be ambiguous — improve tool returns.
```

---

## Quick Reference

| Aspect | Details |
|--------|---------|
| Tool name | `think` |
| Input | `thought: string` |
| Output | Empty (no side effects) |
| Token cost | Output tokens for reasoning text |
| When to add | Complex policy navigation + long tool chains |
| Key to success | Domain-specific examples in system prompt |
| Best result | +54% on airline τ-bench |
| Vs. Extended Thinking | Complementary — use both for max performance |
