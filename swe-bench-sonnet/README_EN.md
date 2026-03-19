# Raising the Bar on SWE-bench Verified with Claude 3.5 Sonnet

> Source: https://www.anthropic.com/engineering/swe-bench-sonnet
> Published: January 6, 2025 | Anthropic Engineering

---

## Overview Mind Map

```
SWE-bench Verified with Claude 3.5 Sonnet
├── Achievement
│   └── 49% on SWE-bench Verified (prev SOTA: 45%)
├── Agent Architecture (Minimal)
│   ├── A prompt (suggested approach)
│   ├── Bash tool (execute commands)
│   ├── Edit tool (view + modify files)
│   └── 200K context limit
├── Performance Progression
│   ├── Claude 3 Opus: 22%
│   ├── Claude 3.5 Sonnet (old): 33%
│   ├── Previous SOTA: 45%
│   └── Claude 3.5 Sonnet (new): 49%
├── Design Principles
│   ├── Detailed tool descriptions > complex scaffolding
│   ├── Require absolute file paths
│   └── String-replacement for file edits
└── Challenges
    ├── High token costs (>100K per success)
    ├── Complex grading environments
    ├── Hidden test cases
    └── Limited multimodal capabilities
```

---

## 1. The Achievement

| Model | SWE-bench Verified Score |
|-------|-------------------------|
| Claude 3 Opus | 22% |
| Claude 3.5 Sonnet (original) | 33% |
| Previous SOTA (any system) | 45% |
| **Claude 3.5 Sonnet (upgraded)** | **49%** |

A 4 percentage point improvement over SOTA, achieved with a deliberately minimal agent architecture.

---

## 2. About SWE-bench

SWE-bench evaluates AI systems' ability to resolve **real GitHub issues** from open-source Python repositories. The benchmark assesses:
- Not just the model alone, but an entire **agent system** (model + scaffolding)
- Real-world software engineering tasks (bug fixes, feature additions)
- Ability to navigate large codebases, understand context, and make correct changes

---

## 3. The Minimal Agent Architecture

The agent design is deliberately simple — three components only:

```
┌─────────────────────────────────────────┐
│           Minimal Agent                  │
│                                         │
│  ┌──────────────────────────────┐       │
│  │  Prompt                      │       │
│  │  (suggested problem-solving  │       │
│  │   approach)                  │       │
│  └──────────────────────────────┘       │
│                                         │
│  ┌──────────────┐  ┌──────────────┐     │
│  │  Bash Tool   │  │  Edit Tool   │     │
│  │  (execute    │  │  (view +     │     │
│  │   commands)  │  │   modify)    │     │
│  └──────────────┘  └──────────────┘     │
│                                         │
│  Loop until done OR 200K context limit  │
└─────────────────────────────────────────┘
```

**Key insight:** The power is in the **tool descriptions**, not in complex multi-step scaffolding.

---

## 4. Design Principles

### Principle 1: Detailed Tool Descriptions Over Complex Scaffolding

The prompt and tool descriptions carry the weight. Rather than building elaborate orchestration logic, the team invested in making tool interfaces clear and comprehensive.

This aligns with the ACI (Agent-Computer Interface) design philosophy from [Building Effective Agents](../building-effective-agents/).

### Principle 2: Require Absolute File Paths

```
Bad:  edit_file("utils.py", ...)          # Which utils.py?
Good: edit_file("/repo/src/utils.py", ...)  # Unambiguous
```

This is a **poka-yoke** (error-proofing) design — absolute paths eliminate an entire class of errors.

### Principle 3: String-Replacement for File Edits

Rather than line-number-based editing (fragile) or full file rewrites (wasteful), the system uses string matching and replacement. This ensures:
- Edits target the right location
- Partial file modifications are possible
- Changes are human-readable

---

## 5. Challenges

### High Token Costs
Many successful runs exceeded **100K tokens**. At scale, this creates significant cost pressure. Context window management is critical for practical deployment.

### Complex Grading
SWE-bench grading requires:
- Setting up the exact repository state
- Installing all dependencies
- Running hidden test suites
- Environment validation

### Hidden Test Cases
Models can't see the test cases they're evaluated against, limiting their ability to self-assess. This makes the task harder but more realistic — in real software engineering, you don't always know all test cases upfront.

### Limited Multimodal
Some tasks produce visual output (charts, images) that the model can't view during the agent loop. This is a capability gap for certain problem types.

---

## 6. Connections to Other Articles

- **[Building Effective Agents](../building-effective-agents/)** — The SWE-bench agent is a textbook example of "simplest solution that meets needs." Three tools, one prompt, minimal scaffolding.
- **[Writing Tools for Agents](../writing-tools-for-agents/)** — The tool design principles (absolute paths, string replacement, detailed descriptions) directly implement the recommendations there.
- **[Claude Think Tool](../claude-think-tool/)** — The think tool could further improve SWE-bench performance by giving the agent explicit reasoning space.
- **[Infrastructure Noise](../infrastructure-noise/)** — SWE-bench scores are also affected by infrastructure configuration (1.54 pp from RAM variation alone).
- **[AI-Resistant Technical Evaluations](../AI-resistant-technical-evaluations/)** — While SWE-bench tests AI coding ability, the other article discusses what happens when that ability surpasses human candidates.

---

## Key Takeaways

```
1. MINIMAL ARCHITECTURE, MAXIMUM RESULTS
   → 3 tools + 1 prompt achieved SOTA on SWE-bench

2. TOOL DESCRIPTIONS > SCAFFOLDING
   → Invest in ACI quality, not orchestration complexity

3. ERROR-PROOFING MATTERS
   → Absolute paths and string-replacement prevent failure classes

4. TOKEN COST IS A REAL CONSTRAINT
   → >100K tokens per success; context management is critical

5. HIDDEN TESTS MAKE THE TASK REALISTIC
   → Real engineering: you don't always know all test cases

6. MODEL + AGENT = SYSTEM
   → SWE-bench evaluates the whole system, not just the model
```
