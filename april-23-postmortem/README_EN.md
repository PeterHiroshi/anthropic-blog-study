# An Update on Recent Claude Code Quality Reports

> Source: https://www.anthropic.com/engineering/april-23-postmortem
> Published: April 23, 2026 | Anthropic Engineering

---

## Overview Mind Map

```
Claude Code Quality Postmortem (March–April 2026)
├── Issue 1: Reasoning Effort Default (Mar 4 – Apr 7)
│   ├── Cause: default effort lowered high → medium to cut latency
│   ├── Symptom: "Claude Code felt less intelligent"
│   ├── Tradeoff: slightly lower intelligence, significantly less latency
│   └── Fix: reverted Apr 7 → xhigh (Opus 4.7), high (other models)
│
├── Issue 2: Thinking Cache Bug (Mar 26 – Apr 10, v2.1.101)
│   ├── Cause: clear_thinking_20251015 + keep:1 cleared EVERY turn
│   ├── Intent: clear old reasoning ONCE for idle sessions
│   ├── Symptoms: forgetful, repetitive, odd tool choices, faster limit drain
│   ├── Masked by: orthogonal thinking-display change
│   └── Fix: v2.1.101 on Apr 10
│
├── Issue 3: Verbosity System Prompt (Apr 16 – Apr 20)
│   ├── Cause: length limits shipped in Opus 4.7 system prompt
│   ├── Text: ≤25 words between tool calls, ≤100 words final response
│   ├── Measured: ~3% quality drop for BOTH Opus 4.6 and 4.7
│   └── Fix: reverted Apr 20
│
├── Resolution
│   ├── All three resolved as of April 20 (v2.1.116)
│   └── Usage limits reset for ALL subscribers (Apr 23)
│
└── Five Commitments
    ├── More internal staff on the PUBLIC Claude Code build
    ├── Better Code Review tooling (internal + customer-facing)
    ├── Tighter system prompt controls + per-model evals
    ├── Soak periods before rollout
    └── Gradual/staged deployment for intelligence-affecting changes
```

---

## Overview

Between early March and late April 2026, users reported that Claude Code had gotten worse — more forgetful, more terse, less capable. Anthropic traced these reports to **three separate changes** affecting Claude Code, the Claude Agent SDK, and Claude Cowork.

The crucial distinction from the [September 2025 postmortem](../a-postmortem-of-three-recent-issues/): **none of these were infrastructure bugs, and no model weights changed.** These were product-side decisions and code changes — a default value, a caching optimization, and a system prompt line. All three degraded quality in ways that were individually small and collectively very visible.

> "This isn't the experience users should expect from Claude Code."

---

## Timeline

```
2026
Mar  4  ●━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓
        │ Issue 1 begins                    ┃
        │ reasoning effort: high → medium    ┃
        │                                    ┃
Mar 26  │    ●━━━━━━━━━━━━━━━━━━━━━┓         ┃
        │    │ Issue 2 begins       ┃         ┃
        │    │ thinking cache bug   ┃         ┃
        │    │                      ┃         ┃
Apr  7  ●━━━━┿━━━━━━━━━━━━━━━━━━━━━┿━━━━━━━━━┛  Issue 1 REVERTED
             │                      ┃            → xhigh / high restored
             │                      ┃
Apr 10       ●━━━━━━━━━━━━━━━━━━━━━━┛            Issue 2 FIXED (v2.1.101)
             │
Apr 16       │        ●━━━━━━━━━━┓               Issue 3 begins
             │        │           ┃               verbosity prompt w/ Opus 4.7
             │        │           ┃
Apr 20       │        ●━━━━━━━━━━━┛               Issue 3 REVERTED
             │                                    ALL CLEAR — v2.1.116
             │
Apr 23       ●                                    Postmortem published
                                                  Usage limits reset for all

        ├───── Mar 26 – Apr 7: TWO issues overlapping ─────┤
```

**Peak overlap:** March 26 → April 7, when both the lowered reasoning effort *and* the thinking cache bug were active simultaneously. This is likely when user reports were loudest.

---

## Issue 1: Reasoning Effort Default Change

### Timeline
- **March 4:** Default reasoning effort changed from `high` to `medium`
- **April 7:** Reverted after customer feedback

### What Happened

```
The problem being solved:
  High reasoning effort → long latency
  → the CLI UI appeared to freeze while waiting
  → users saw an unresponsive terminal

The change:
  default reasoning effort: high ──────► medium
  Affected: Sonnet 4.6, Opus 4.6

The measured tradeoff (internal evals):
  medium effort = slightly lower intelligence
               + significantly less latency
  → judged a good trade internally
```

### Why It Was Wrong

Anthropic's internal evals said the intelligence loss was small. Users disagreed about the *value* of the trade, not the *measurement*:

```
Anthropic's assumption:   "Users want responsiveness."
Users' actual preference: "We want intelligence. We'll wait."

→ "users began reporting that Claude Code felt less intelligent"
```

This is not an engineering bug. It is a **product-judgment error**: an internally optimal tradeoff that was wrong for the actual user population. No amount of eval coverage would have caught it, because the evals were correct.

### Resolution

```
Reverted April 7:
├── Opus 4.7        → xhigh   (highest tier)
└── All other models → high
```

---

## Issue 2: The Thinking Cache Bug

### Timeline
- **March 26:** Prompt-caching optimization deployed
- **April 10:** Fixed in v2.1.101

### The Intent

```
Observed problem:
  A session sits idle for a long time (>1 hour)
  → its prompt cache entry expires
  → the next turn must re-send the ENTIRE thinking history
  → expensive, slow

Intended optimization:
  On resuming an idle session, clear OLD reasoning blocks ONCE,
  keeping only the most recent one.

Mechanism:
  clear_thinking_20251015 API header  +  keep:1
```

### The Bug

```
INTENDED behavior:
  Turn 1 (idle resume) → clear old thinking ✓  (once)
  Turn 2               → keep thinking
  Turn 3               → keep thinking
  Turn 4               → keep thinking
  → cache warm, history intact

ACTUAL behavior:
  Turn 1 → clear thinking ✓
  Turn 2 → clear thinking ✗  ← should not have
  Turn 3 → clear thinking ✗
  Turn 4 → clear thinking ✗
  → "instead of clearing thinking history once, it cleared it
     on every turn for the rest of the session"
```

### The Two-Headed Damage

```
Head 1 — QUALITY
  Thinking blocks dropped from every subsequent request
  → Claude loses its own chain of reasoning between turns
  → Symptoms: forgetfulness
               repetition (re-deriving what it already worked out)
               unusual tool choices

Head 2 — COST / USAGE LIMITS
  Dropping thinking blocks changes the prompt prefix EVERY turn
  → cache MISS on every turn
  → full prompt re-processed at uncached rates
  → users' usage limits drained faster than normal
```

This second head is why the bug was doubly painful: users got worse output **and** paid more of their quota for it.

### Why It Was Hard to Detect

```
The masking effect:
  An orthogonal change in how thinking is DISPLAYED
  → suppressed the visible symptom in most CLI sessions
  → "we didn't catch it even when testing external builds"

So:
  ├── Internal testing: symptom hidden by display change
  ├── External build testing: symptom hidden by display change
  └── Only aggregate user reports surfaced it
```

**Lesson:** a UI change and a backend change, each fine alone, combined into a blind spot. Two orthogonal changes are not orthogonal when one hides the other's evidence.

### The Code Review Back-Test

After the fact, Anthropic ran its Code Review tool against the offending pull requests:

| Model | Given full repo context | Found the bug? |
|-------|------------------------|----------------|
| Opus 4.7 | Yes | **Yes** |
| Opus 4.6 | Yes | No |

The critical qualifier: **"when provided the code repositories necessary to gather complete context."** The bug was findable by automated review — but only with enough context and a strong enough model. This directly motivated commitment #2 below.

---

## Issue 3: The Verbosity Reduction Prompt

### Timeline
- **April 16:** Shipped alongside Opus 4.7
- **April 20:** Reverted

### What Happened

The Claude Code system prompt gained a length constraint:

> "Length limits: keep text between tool calls to ≤25 words. Keep final responses to ≤100 words unless the task requires more detail."

```
Intent:  less chatter, tighter output, faster to read
Reality: a hard word budget on the model's own working space
```

### Why a Verbosity Cap Costs Intelligence

```
Text between tool calls is NOT decoration. It is where the model:
  ├── states what it just learned
  ├── plans the next step
  ├── notes constraints it must remember
  └── reasons about which tool to use next

Cap that at 25 words ──► you have compressed the model's
                          externalized scratchpad

Cap the final answer at 100 words ──► truncated explanations,
                                       omitted caveats,
                                       dropped edge cases
```

### The Measurement

```
Internal evaluation result:
  ~3% quality drop  ── for Opus 4.6
  ~3% quality drop  ── for Opus 4.7

Note: the drop hit BOTH model versions
      → it was the prompt, not the new model
```

**Why 3% matters more than it sounds:** as [Quantifying Infrastructure Noise](../infrastructure-noise/) argues, differences below ~3 percentage points on agentic benchmarks can be indistinguishable from infrastructure noise. A change that reliably and reproducibly costs 3% across two model versions is therefore right at the edge of what evals can even see — which is precisely why it shipped.

### Resolution

Reverted April 20. This was the last of the three fixes, making **v2.1.116** the first all-clear build.

---

## Comparing the Three Issues

| | Issue 1: Reasoning Effort | Issue 2: Thinking Cache | Issue 3: Verbosity Prompt |
|---|---|---|---|
| **Type** | Product judgment | Code bug | Prompt change |
| **Introduced** | Mar 4 | Mar 26 | Apr 16 |
| **Resolved** | Apr 7 (revert) | Apr 10 (v2.1.101) | Apr 20 (revert) |
| **Duration** | ~34 days | ~15 days | ~4 days |
| **Main symptom** | Shallower answers | Forgetful, repetitive | Terse, incomplete |
| **Hit usage limits?** | No | **Yes** (cache misses) | No |
| **Was it measured?** | Yes — and accepted | No — masked by UI change | Yes — after the fact |
| **Root failure** | Wrong values in the tradeoff | Insufficient review context | Insufficient pre-ship evals |

---

## Why All Three Slipped Through

```
Issue 1 ──► The eval was RIGHT, the DECISION was wrong.
            "Slightly less intelligent, much faster" was
            measured accurately and valued incorrectly.
            → Evals cannot tell you what users want.

Issue 2 ──► The eval never RAN on the real behavior.
            A display change hid the symptom from every
            testing path, internal and external.
            → Observability failure, not a testing failure.

Issue 3 ──► The eval was NOT RUN BROADLY ENOUGH before ship.
            A ~3% drop is small enough to hide in a narrow
            eval suite, large enough for users to feel daily.
            → Coverage failure.

Common thread:
  Each change was individually defensible.
  None was individually catastrophic.
  Two of them overlapped in time (Mar 26 – Apr 7).
  → Users experienced the SUM, and reported the sum.
```

---

## The Five Commitments

### 1. More internal staff on the public build

```
Problem: Anthropic staff often run internal/dev builds
         → they don't experience what customers experience
Fix:     increase staff usage of the PUBLIC Claude Code build
         → dogfooding the real artifact, not a privileged variant
```

### 2. Better Code Review tooling

```
Evidence: Opus 4.7 + full repo context FOUND the cache bug
          Opus 4.6 did not
Fix:      improve the Code Review tool
          ├── for internal use (catch this class of bug pre-merge)
          └── for customer deployment (ship the capability out)
Key enabler: multi-repository context — the model needs enough
             of the codebase to see cross-cutting effects
```

### 3. Tighter system prompt controls

```
Problem: Issue 3 was one line of prompt text with a 3% cost
Fix:     system prompt changes now require
         ├── explicit review controls
         └── comprehensive evaluations PER MODEL
             (Issue 3 hit 4.6 and 4.7 — a single-model eval
              would still have caught it, but a narrow eval
              suite did not)
```

### 4. Soak periods

```
Before: change → ship
After:  change → SOAK (run in limited exposure for a period)
                 → observe → ship
Rationale: Issue 2 took ~15 days to surface through user
           reports. A soak period surfaces slow-burn
           regressions before full exposure.
```

### 5. Gradual rollouts for intelligence-affecting changes

```
Any change that could affect intelligence now ships staged:
  small % of traffic → monitor → widen → monitor → full

Why this specifically helps:
  ├── gives a clean A/B comparison population
  ├── bounds blast radius during the soak
  └── makes "did quality drop?" answerable with data,
      not just with user sentiment
```

---

## Remediation for Users

```
All three issues resolved:  April 20, 2026 (v2.1.116)

Compensation:
  "Today we are resetting usage limits for all subscribers."
  → applies to ALL subscribers, not only those who complained
  → directly addresses Issue 2's quota-drain side effect
```

---

## Connection to Other Articles

- **[A Postmortem of Three Recent Issues](../a-postmortem-of-three-recent-issues/)** — The direct sibling. Same shape (three overlapping issues, degraded quality, public postmortem), opposite root cause: that one was **infrastructure** (routing, TPU config, XLA miscompilation); this one is **product and process** (a default, a cache optimization, a prompt line). Read together, they show that quality regressions arrive from both below and above the model.
- **[Quantifying Infrastructure Noise in Agentic Coding Evals](../infrastructure-noise/)** — Explains why a 3% eval drop is dangerously close to the noise floor, and why "our evals didn't flag it" is a weak defense for small-but-real regressions.
- **[Claude Code Best Practices](../claude-code-best-practices/)** — All three issues touch levers that article discusses directly: reasoning effort, thinking blocks, context/cache behavior, and system prompt design.

---

## Lessons Learned

| Lesson | Detail |
|--------|--------|
| Evals measure, they don't decide | Issue 1's eval was accurate; the *tradeoff* was wrong. Users valued intelligence over latency. |
| Orthogonal changes aren't orthogonal | A thinking-*display* change hid a thinking-*caching* bug from all test paths. |
| Prompt text is production code | One line of system prompt cost ~3% quality across two model versions. |
| Caching bugs are cost bugs | Dropping blocks every turn caused cache misses, which drained user quotas faster. |
| Context is what makes review work | Opus 4.7 found the bug — but only with the full repositories available. |
| Small + small + overlapping = loud | Three modest regressions, two overlapping, produced a very visible quality complaint wave. |

---

## Quick Reference

**The three issues in brief:**

| Issue | Root Cause | Introduced | Resolved |
|-------|-----------|-----------|----------|
| Reasoning effort default | `high` → `medium` for latency | Mar 4 | Apr 7 (revert → xhigh/high) |
| Thinking cache bug | `clear_thinking_20251015` cleared every turn | Mar 26 | Apr 10 (v2.1.101) |
| Verbosity prompt | ≤25 / ≤100 word limits, ~3% quality drop | Apr 16 | Apr 20 (revert) |

**All clear:** April 20, 2026 — **v2.1.116**

**Compensation:** usage limits reset for all subscribers (April 23)

**Five commitments:** public-build dogfooding · better Code Review tooling · stricter system prompt controls with per-model evals · soak periods · gradual rollouts

**Closing line:**
> "We're immensely grateful for your feedback and for your patience."
