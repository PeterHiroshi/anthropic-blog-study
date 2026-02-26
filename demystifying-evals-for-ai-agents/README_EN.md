# Demystifying Evals for AI Agents

> Source: https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents
> Published: 2025 | Anthropic Engineering

---

## Overview Mind Map

```
Demystifying Evals for AI Agents
├── Core Definitions
│   ├── Eval — a test for an AI system
│   ├── Task — input specification
│   ├── Trial — one execution of a task
│   ├── Grader — scores the output
│   ├── Transcript — full execution record
│   ├── Outcome — pass/fail/score
│   ├── Eval Harness — infrastructure running evals
│   └── Eval Suite — collection of evals
│
├── Why Evals Matter
│   ├── Force explicit success criteria
│   ├── Prevent regressions
│   └── Replace reactive debugging
│
├── Three Grader Types
│   ├── Code-based — fast, objective, brittle
│   ├── Model-based — flexible, needs calibration
│   └── Human — gold standard, doesn't scale
│
├── Agent-Specific Approaches
│   ├── Coding agents — unit tests + LLM rubrics
│   ├── Conversational agents — state + interaction quality
│   ├── Research agents — groundedness + coverage
│   └── Computer use agents — sandboxed + state verification
│
└── Implementation Roadmap
    ├── Start with 20-50 real tasks
    ├── Write unambiguous specs
    ├── Build balanced problem sets
    └── Read transcripts regularly
```

---

## 1. Core Definitions

The article establishes precise terminology to cut through confusion:

```
The Eval Ecosystem

┌─────────────────────────────────────────────────────┐
│                   EVAL SUITE                        │
│  (collection of related evals for one capability)   │
│                                                     │
│  ┌───────────────────────────────────────────────┐  │
│  │              EVAL HARNESS                     │  │
│  │  (infrastructure: runs evals, collects data)  │  │
│  │                                               │  │
│  │   TASK ──▶ AGENT HARNESS ──▶ TRIAL            │  │
│  │   (what   (runs the agent,  (one execution    │  │
│  │    to do) provides tools)   of the task)      │  │
│  │                    │                          │  │
│  │                    ▼                          │  │
│  │               TRANSCRIPT                      │  │
│  │         (full record of execution)            │  │
│  │                    │                          │  │
│  │                    ▼                          │  │
│  │               GRADER ──▶ OUTCOME              │  │
│  │         (scores output)  (pass/fail/score)    │  │
│  └───────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
```

**Key distinctions:**

| Term | What it is | Example |
|------|-----------|---------|
| **Task** | Input specification for one test case | "Write a function to sort a list" |
| **Trial** | One execution of a task | Agent runs and writes `bubble_sort()` |
| **Transcript** | Full record of one trial | All tool calls, responses, reasoning |
| **Outcome** | Result grading of a trial | Pass (correct) or Fail (incorrect) |
| **Eval** | A test: task + grading logic | The task + "unit tests must pass" |
| **Eval Suite** | Collection of evals | All 200 coding eval tasks |

---

## 2. Why Evals Matter

### The cost of not having evals:

```
Without Evals:                    With Evals:

Detect issues: Production only    Detect issues: Before deployment

Fix cycle:                        Fix cycle:
  Bug found in production          Unit test fails in CI
  → User complains                 → Developer sees immediately
  → Investigation                  → Fix and re-run
  → Fix might break other things   → Confident no regressions
  → Another bug emerges

Team behavior:                    Team behavior:
  Reactive firefighting            Proactive improvement
  Fear of changes                  Confident iteration
```

**Quote from article:** "Teams without evals face reactive debugging—catching issues only in production, where fixing one failure creates others."

### Benefits at each lifecycle stage:

```
Early Stage:
└── Forces you to explicitly define "what does success look like?"
    → Resolves vague requirements before they cause problems

Growing Stage:
└── Catch regressions when adding features
    → New feature works; old feature breaks → eval catches it

Mature Stage:
└── Maintain quality bar as model updates and changes happen
    → Model update ships; evals confirm no quality regression
```

---

## 3. Three Grader Types

### Grader Type 1: Code-Based

**What it is:** Automated code that checks outputs — string matching, binary tests, static analysis.

```python
# Example: Code-based grader for a sort function
def grade_sort_function(agent_output: str) -> bool:
    # Run unit tests against the generated code
    test_cases = [
        ([3, 1, 2], [1, 2, 3]),
        ([], []),
        ([1], [1]),
        ([-1, -5, 2], [-5, -1, 2])
    ]

    exec_namespace = {}
    exec(agent_output, exec_namespace)
    sort_fn = exec_namespace.get('sort_list')

    for input_val, expected in test_cases:
        if sort_fn(input_val) != expected:
            return False
    return True
```

| Aspect | Detail |
|--------|--------|
| Speed | Very fast (milliseconds) |
| Cost | Minimal (compute only) |
| Objectivity | Fully objective |
| Limitation | Brittle — fails on valid variations |
| Best for | Code correctness, structured output |

**Brittleness example:**
```python
# Agent output A: def sort_list(arr): return sorted(arr)   → PASS
# Agent output B: def sort_list(a):   return sorted(a)     → FAIL (different param name!)
# Both are correct! Code grader fails on valid variation.
```

### Grader Type 2: Model-Based (LLM-as-Judge)

**What it is:** Use another LLM to evaluate outputs against a rubric.

```python
GRADER_PROMPT = """
You are evaluating an AI agent's response to a coding task.

Task given to agent:
{task}

Agent's response:
{agent_response}

Rubric:
1. Correctness (0-10): Does the code solve the stated problem?
2. Code quality (0-10): Is it readable, efficient, well-structured?
3. Edge cases (0-10): Does it handle edge cases appropriately?
4. Documentation (0-10): Is it adequately documented?

Respond with JSON: {"correctness": N, "quality": N, "edge_cases": N,
                    "documentation": N, "reasoning": "..."}
"""

score = grader_model.evaluate(GRADER_PROMPT.format(
    task=task_description,
    agent_response=agent_output
))
```

| Aspect | Detail |
|--------|--------|
| Flexibility | High — handles subjective dimensions |
| Cost | Moderate (LLM API calls) |
| Speed | Seconds per evaluation |
| Limitation | Requires calibration against human judgment |
| Best for | Quality assessment, subjective tasks |

**Important:** Must calibrate against human judgments before trusting at scale.

### Grader Type 3: Human Graders

**What it is:** Human experts evaluate agent outputs directly.

| Aspect | Detail |
|--------|--------|
| Quality | Gold standard |
| Scalability | Poor — doesn't scale beyond ~hundreds per day |
| Cost | High |
| Best for | Calibrating other graders, edge cases, high-stakes decisions |

**Recommended use:** Use humans to calibrate code-based and model-based graders, not as primary graders at scale.

---

## 4. Agent-Specific Evaluation Approaches

### Coding Agents

```
Two-layer evaluation:

Layer 1: Code-based grading
└── Unit tests verify correctness of output
    → Pass/fail based on test results

Layer 2: LLM rubric assessment
└── Evaluate code quality (structure, patterns, documentation)
    → Score on quality dimensions

Combined: A coding agent eval suite might look like:
├── 100 tasks with unit tests (objective correctness)
├── 50 tasks with LLM quality rubrics (subjective quality)
└── 20 tasks requiring human review (edge cases, complex tradeoffs)
```

### Conversational Agents

```
Two-layer evaluation:

Layer 1: State verification
└── After conversation, check that the intended state was reached
    → "Was the flight actually booked? Is the database updated?"

Layer 2: Interaction quality metrics
└── Evaluate the conversation itself
    → Was it natural? Efficient? Did it gather necessary info?
    → Checked via LLM rubric or human review

Important: Both matter! A booking agent that books correctly but
asks 10 unnecessary questions fails the interaction quality eval.
```

### Research Agents

```
Two-layer evaluation:

Layer 1: Groundedness check
└── Are claims supported by the sources cited?
    → Automated: extract claim → check against source
    → LLM-based: "Does [source] support [claim]?"

Layer 2: Coverage verification
└── Did the agent find all major relevant information?
    → Compare against reference answer
    → Check key facts are present

Challenge: Research has no single correct answer.
→ Rubric must measure quality dimensions, not binary correctness.
```

### Computer Use Agents

```
Special requirements:

Requirement 1: Sandboxed execution environment
└── Agent must interact with real UI safely
    → Isolated VMs or containers
    → Reset state between test runs

Requirement 2: State verification
└── Did the task actually complete?
    → Check file system, database, UI state
    → Not just "did agent say it completed"

Example eval setup:
├── Spin up fresh VM with target application
├── Run agent task in VM
├── Verify post-task state (file created, form submitted, etc.)
└── Destroy VM → reset for next trial
```

---

## 5. Implementation Roadmap

The article recommends this sequence for teams building evals:

### Step 1: Start with 20-50 Real Tasks

```
Sources for real tasks:
├── Existing failures (bugs reported by users)
├── Representative user requests from logs
├── Edge cases discovered in testing
└── Business-critical workflows

Don't start with synthetic tasks — they miss real distribution!
```

### Step 2: Write Unambiguous Specifications

**Bad spec:**
```
Task: "Write a function to process user data"
→ Too vague: What does "process" mean? What data? What output?
→ Can't grade consistently: Different graders will disagree
```

**Good spec:**
```
Task: "Write a Python function process_users(users: list[dict]) → list[dict]
       Input: list of user dicts with keys: name, email, age
       Output: list of filtered dicts where age >= 18, sorted by name
       Requirements: Handle empty list, handle missing keys gracefully"
→ Clear: precise input, output, and requirements
→ Gradeable: objective criteria for pass/fail
```

### Step 3: Build Balanced Problem Sets

```
Eval Suite Balance:
├── Easy tasks (30%): Agent should pass reliably
│   → Catch regressions (if easy tasks fail, something broke)
├── Medium tasks (50%): Real capability test
│   → Shows actual capability level
└── Hard tasks (20%): Stretch goals
    → Distinguishes excellent from good

Avoid: 90% easy + 10% hard
→ High pass rate that doesn't mean much
→ Hard to distinguish capability improvements
```

### Step 4: Read Transcripts Regularly

```
Don't just look at scores!

Transcript reveals:
├── How agent reasons (good vs. bad patterns)
├── Where agent gets confused (ambiguous instructions)
├── Unexpected failure modes (surprising edge cases)
└── Whether grader is actually measuring what matters

"Eval saturation" warning: When an agent passes all solvable tasks,
you've exhausted this eval's utility. Time to add harder tasks.
```

---

## 6. Complementary Quality Methods

Evals work alongside other quality assurance practices:

```
Quality Assurance Ecosystem:

┌───────────────────────────────────────────────┐
│ Automated Evals                               │
│ (before deployment, regression testing)       │
└───────────────────────────────────────────────┘
          +
┌───────────────────────────────────────────────┐
│ Production Monitoring                         │
│ (real-world quality signals, error rates)     │
└───────────────────────────────────────────────┘
          +
┌───────────────────────────────────────────────┐
│ A/B Testing                                   │
│ (compare versions on real user queries)       │
└───────────────────────────────────────────────┘
          +
┌───────────────────────────────────────────────┐
│ User Feedback                                 │
│ (thumbs up/down, complaint tickets)           │
└───────────────────────────────────────────────┘
          +
┌───────────────────────────────────────────────┐
│ Systematic Human Studies                      │
│ (expert review of outputs at scale)           │
└───────────────────────────────────────────────┘
```

---

## Quick Reference

**Grader selection:**
```
Need fast, objective, binary?  → Code-based
Need flexible, quality-focused? → Model-based
Need gold-standard calibration? → Human
Best comprehensive eval?        → All three, layered
```

**Eval implementation checklist:**
- [ ] Start with 20-50 real tasks (not synthetic)
- [ ] Write unambiguous task specifications
- [ ] Build balanced easy/medium/hard problem sets
- [ ] Choose appropriate graders for each task type
- [ ] Read transcripts, don't just look at scores
- [ ] Watch for eval saturation, add harder tasks over time
- [ ] Complement with production monitoring + A/B testing

**Key warning:** "Eval saturation occurs when an agent passes all solvable tasks" — this means your eval suite is no longer discriminating. Expand it.
