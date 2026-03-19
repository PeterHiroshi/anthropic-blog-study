# Designing AI-Resistant Technical Evaluations

> Source: https://www.anthropic.com/engineering/AI-resistant-technical-evaluations
> Published: January 21, 2026 | Anthropic Engineering

---

## Overview Mind Map

```
AI-Resistant Technical Evaluations
├── The Problem
│   ├── Take-home assessments trivialized by AI models
│   ├── Opus 4: outperformed ~90% of human applicants
│   └── Opus 4.5: matched top humans in 2 hours
├── Original Assessment
│   ├── TPU-like accelerator optimization in Python
│   ├── Features: scratchpad memory, VLIW, SIMD, multicore
│   ├── Task: parallel tree traversal optimization
│   └── 4-hour window (later reduced to 2)
├── The AI Escalation
│   ├── Opus 4: beat 90% of humans
│   ├── Opus 4.5: matched top performers
│   └── With harness: even better (1363 cycles best)
├── Failed Attempts
│   └── Data transposition: Claude solved via creative reformulation
├── Successful Approach
│   ├── Constraint-based puzzles (Zachtronics-inspired)
│   ├── Minimal instruction sets
│   ├── No visualization tools
│   └── Requires building own debugging approaches
└── Key Insight
    └── "Realism may require sacrifice"
```

---

## 1. The Problem Statement

> Evaluating technical candidates becomes harder as AI capabilities improve. A take-home that distinguishes well between human skill levels today may be trivially solved by models tomorrow.

Anthropic's performance engineering team created a take-home assessment in November 2023 that worked well for evaluating candidates. Over time, Claude models caught up to and surpassed human performance on this assessment.

---

## 2. The Original Assessment

**Domain:** Performance engineering — optimizing code for a simulated TPU-like accelerator.

**Machine features:**
- Manually managed scratchpad memory
- VLIW (Very Long Instruction Word) — multiple parallel execution units
- SIMD (Single Instruction, Multiple Data) — vector operations
- Multicore parallelism

**Task:** Parallel tree traversal optimization using Python, with access to Perfetto traces for instruction-level performance analysis.

**Design rationale:** Deliberately avoided deep learning domain expertise requirements — focused on general performance optimization skills.

---

## 3. The AI Escalation

### Performance Progression (measured in clock cycles, lower is better)

| Performer | Cycles | Context |
|-----------|--------|---------|
| Claude Opus 4 | 2164 | Many hours of test-time compute |
| Claude Opus 4.5 | 1790 | Casual 2-hour session |
| Claude Opus 4.5 | 1579 | 2 hours with harness |
| Claude Sonnet 4.5 | 1548 | Extensive test-time compute |
| Claude Opus 4.5 | 1487 | 11.5 hours with harness |
| Claude Opus 4.5 | 1363 | Improved harness, many hours |

**Key milestones:**
- **Opus 4 (May 2025):** Outperformed ~90% of human applicants within the 4-hour window
- **Opus 4.5:** Matched top human performers within 2 hours by identifying bottlenecks, implementing micro-optimizations, and discovering memory bandwidth workarounds

---

## 4. Attempted Solutions

### Attempt 1: Data Transposition Problem

**Design:** Based on efficient 2D TPU register transposition with bank conflict avoidance.

**Result:** Claude Opus 4.5 solved it through **creative reformulation** — instead of transposing the data, it transposed the computation. This is a fundamentally different approach that the assessment designers didn't anticipate.

**Lesson:** AI models can find lateral solutions that bypass the intended difficulty.

### Attempt 2: Constraint-Based Puzzles (Successful)

**Design:** Inspired by Zachtronics games (TIS-100, Shenzhen I/O):
- Minimal instruction sets
- Optimize for instruction count
- No visualization tools provided
- Candidates must build their own debugging approaches

**Result:** Successfully resisted Claude's capabilities while testing genuine optimization thinking. Early results show promising correlation with candidate quality.

---

## 5. The Core Insight

> "The original worked because it resembled real work. The replacement works because it simulates novel work."

There's a tension between:
- **Realism** — Assessments that mirror actual job tasks
- **AI resistance** — Assessments that can't be trivially solved by models

The new constraint-based approach sacrifices some realism for AI resistance. Human experts retain advantages on sufficiently novel tasks where established patterns don't apply.

---

## 6. The Open Challenge

Anthropic released the original take-home publicly, inviting solutions beating 1487 cycles. Contact: performance-recruiting@anthropic.com with code and resume.

---

## 7. Connections to Other Articles

- **[Eval Awareness BrowseComp](../eval-awareness-browsecomp/)** — Both articles deal with AI models circumventing evaluations, but from different angles: one about models recognizing benchmarks, the other about models surpassing human-targeted assessments.
- **[Demystifying Evals](../demystifying-evals-for-ai-agents/)** — The challenge of designing evaluations that remain meaningful applies to both AI benchmarks and human hiring assessments.
- **[SWE-bench Sonnet](../swe-bench-sonnet/)** — The SWE-bench results demonstrate the same trajectory of rapidly improving coding capabilities.

---

## Key Takeaways

```
1. AI CAPABILITIES ESCALATE FASTER THAN EXPECTED
   → Assessments designed in 2023 were trivialized by 2025

2. CREATIVE REFORMULATION DEFEATS INTENDED DIFFICULTY
   → Models find lateral solutions humans don't anticipate

3. CONSTRAINT-BASED PUZZLES ARE MORE AI-RESISTANT
   → Novel, minimal instruction sets resist pattern matching

4. REALISM vs. AI RESISTANCE IS A TRADEOFF
   → Real-work assessments are easier for AI; novel puzzles less realistic

5. HUMAN ADVANTAGE ON NOVEL TASKS PERSISTS
   → Sufficiently constrained/novel problems still favor human ingenuity

6. CONTINUOUS ADAPTATION REQUIRED
   → Assessment design is now an ongoing engineering challenge, not a one-time effort
```
