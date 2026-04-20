# Harness design for long-running application development

> Source: https://www.anthropic.com/engineering/harness-design-long-running-apps
> Published: March 24, 2026 | Anthropic Engineering
> Author: Prithvi Rajasekaran (Anthropic Labs)

---

## Overview Mind Map

```
Harness for Long-Running App Builds
├── Why a Harness
│   ├── Multi-hour builds push models past context limits
│   ├── Single-agent runs hit "context anxiety" and wrap up early
│   └── Self-grading is systematically inflated
├── Agent Roles
│   ├── Initializer / Planner — expands prompt → detailed spec
│   ├── Generator — builds features iteratively
│   └── Evaluator — tests + feedback loop (separate from generator)
├── Two Persistent Failure Modes
│   ├── Context anxiety (premature wrap-up)
│   └── Inflated self-assessment (→ separate evaluator)
├── Frontend Grading Criteria
│   ├── Design quality
│   ├── Originality
│   ├── Craft
│   └── Functionality
├── Case Studies (numbers)
│   ├── Retro game maker: 20 min $9 vs 6 hr $200
│   └── DAW: 3h 50m total / $124.70
├── Architecture Evolution (Opus 4.5 → 4.6)
│   ├── Removed sprint decomposition
│   ├── Kept planner + single-pass evaluator
│   └── Generator ran 2+ hours coherently
└── Meta-Principle
    └── Harness complexity should decrease over time
```

---

## 1. Why You Need a Harness for Multi-Hour Builds

Two failure modes keep showing up when a single agent tries to build a real application end-to-end:

### Failure Mode 1: Context Anxiety

As Claude's context window fills, the model starts wrapping up work prematurely — declaring "done" before the actual task is complete. **Context resets beat context compaction** for this problem: a fresh context with a carefully curated handoff outperforms an aggressively compacted single thread.

### Failure Mode 2: Inflated Self-Assessment

> "Agents reliably skew positive when grading their own work."

A generator agent asked "is this done?" will say yes too often. Separating generation and evaluation into **different agents with different prompts** dramatically improves quality signals.

---

## 2. The Three Agent Roles

```
┌──────────────┐    spec    ┌──────────────┐  artifacts  ┌───────────────┐
│  Initializer │ ─────────▶ │   Generator  │ ──────────▶ │   Evaluator   │
│   / Planner  │            │  (builds)    │             │ (tests, CoT)  │
└──────────────┘            └──────────────┘             └───────┬───────┘
                                   ▲                             │ feedback
                                   └─────────────────────────────┘
```

| Agent | Responsibility |
|-------|----------------|
| **Initializer / Planner** | Expands a short user prompt into a detailed product spec. Frames the whole build. |
| **Generator** | Builds features iteratively against the spec. Operates in focused sprints (originally). |
| **Evaluator** | Tests the artifact against the grading criteria and returns concrete, actionable feedback. |

Context resets happen **between** these roles — fresh context each time, curated handoff.

---

## 3. Grading Criteria for Frontend Quality

For UI-rich builds, the author codified four criteria:

| Criterion | Meaning |
|-----------|---------|
| **Design quality** | Coherent aesthetic; consistent visual language |
| **Originality** | Avoids generic / templated patterns |
| **Craft** | Typography, spacing, color harmony |
| **Functionality** | Actual usability, not just visuals |

Critically, **these criteria show up in the generator's prompt too**, not just the evaluator's. Phrases like "museum quality" measurably shifted the generator toward more considered aesthetic choices **before** any evaluator feedback.

This is a key lesson: **prompt language shapes output** — the evaluator doesn't carry all the quality signal; the generator's prompt does heavy lifting.

---

## 4. Case Studies: Hard Numbers

### Retro Game Maker

| Configuration | Time | Cost | Outcome |
|---------------|------|------|---------|
| Solo agent | 20 min | $9 | Baseline |
| Full harness | 6 hr | $200 | Noticeably superior functionality |

The harness is **20× more expensive** and produced a substantially better result. Whether the tradeoff is worth it depends on what you're building. For a production-quality demo artifact: yes. For a quick prototype: no.

### Digital Audio Workstation (DAW)

| Phase | Time | Cost |
|-------|------|------|
| Planner | 4.7 min | $0.46 |
| Build phases (combined) | ~3 hr | $113.85 |
| QA phases (combined) | ~25 min | $10.39 |
| **Total** | **3 hr 50 min** | **$124.70** |

Note the **cost distribution**: the planner is cheap (under a dollar), the evaluator is modest (~$10), and the generator eats ~90% of the budget. That's the right shape — the expensive agent should be the one doing the expensive work.

---

## 5. Architecture Evolution: Opus 4.5 → Opus 4.6

The original system (Opus 4.5 era):
- Three agents (planner, generator, evaluator).
- **Sprint-based decomposition** — generator ran in focused mini-sessions with context resets between sprints.
- Explicit context-reset machinery.

The simplified system (Opus 4.6 era):
- **Sprint construct removed** — the newer model could run coherently for 2+ hours without internal decomposition.
- Planner and single-pass evaluator retained.
- **Performance maintained** while token overhead dropped.

### The Meta-Principle

> Components of your harness encode assumptions about model limitations. Those assumptions become outdated as models improve.

So:
1. **Harness complexity should trend down over time**, not up.
2. **Re-evaluate harness components periodically** — what was load-bearing six months ago may now be dead weight.
3. **Beware the sunk-cost instinct** — a complex harness feels sophisticated, but if the model no longer needs it, the harness is now noise.

---

## 6. When Does the Evaluator Pay Its Rent?

Not always. The evaluator earns its place **when the task exceeds the model's baseline capability**. As models improve:

- Tasks that once needed an evaluator no longer do.
- The evaluator becomes a cost center without a quality dividend.

Practical rule: **ablate the evaluator** periodically. Run the same task without it. If quality holds, remove it.

---

## 7. Prompt Engineering Shapes Outputs Before Feedback Arrives

The author observed: design criteria language embedded in the generator's system prompt steered behavior **before** the evaluator ever ran. Phrases like "museum quality" measurably influenced the generator's aesthetic convergence.

Implication: **the evaluator is not the only — or even the primary — quality mechanism**. A strong generator prompt does most of the work; the evaluator catches the residual.

---

## 8. Design Principles Extracted

1. **Separate generation from evaluation.** Self-grading is biased; a fresh agent with a different prompt gives honest signal.
2. **Reset, don't compact.** Fresh context with a curated handoff beats ever-growing compressed threads.
3. **Prompt language shapes output.** Put quality criteria in the generator's prompt, not just the evaluator's.
4. **Let cost follow work.** Planner cheap, evaluator modest, generator expensive — in that order.
5. **Delete scaffolding as models improve.** Harness components encode limitations; limitations expire.

---

## 9. Connections to Other Articles

- **[Scaling Managed Agents](../managed-agents/)** — same meta-principle: harness components encode assumptions that age. Managed Agents generalizes this to the platform layer.
- **[Effective Harnesses for Long-Running Agents](../effective-harnesses-for-long-running-agents/)** — earlier harness design (init agent + coding agent). This article is the newer, multi-role generalization.
- **[Building Effective Agents](../building-effective-agents/)** — the orchestrator / evaluator-optimizer workflow patterns are exactly what this harness instantiates.
- **[Multi-Agent Research System](../multi-agent-research-system/)** — same design logic applied to research, not builds.
- **[Effective Context Engineering for AI Agents](../effective-context-engineering-for-ai-agents/)** — context anxiety is a concrete manifestation of context rot; reset-over-compact is an engineering-level fix.

---

## Key Takeaways

```
1. CONTEXT ANXIETY IS REAL
   → Models wrap up prematurely as context fills.
   → Reset > compact for long builds.

2. SELF-GRADING INFLATES
   → Generator and evaluator must be separate agents.

3. THE GENERATOR'S PROMPT DOES HEAVY LIFTING
   → Quality criteria in the generator, not just the evaluator.

4. COST SHAPE: PLANNER ≪ EVALUATOR ≪ GENERATOR
   → Spend where the work is.

5. HARNESS COMPLEXITY SHOULD TREND DOWN
   → Opus 4.5 → 4.6: sprint construct removed, 2+ hr coherence achieved.

6. ABLATE PERIODICALLY
   → If you can remove a component without losing quality, remove it.

7. 20× COST CAN BE WORTH IT
   → Retro game maker: $9 vs $200 — harness won on quality.

8. BUT NOT ALWAYS
   → The evaluator only pays its rent when tasks exceed baseline capability.
```
