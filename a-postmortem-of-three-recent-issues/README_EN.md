# A Postmortem of Three Recent Issues

> Source: https://www.anthropic.com/engineering/a-postmortem-of-three-recent-issues
> Published: September 2025 | Anthropic Engineering

---

## Overview Mind Map

```
Postmortem: Three Infrastructure Bugs (Aug-Sep 2025)
├── Bug 1: Context Window Routing Error (Aug 5 – Sep 18)
│   ├── Cause: Requests routed to 1M-token-configured servers
│   ├── Scope: 0.8% → 16% of Sonnet 4 requests
│   └── Fix: Corrected routing, full rollout Sep 18
│
├── Bug 2: Output Corruption (Aug 25 – Sep 2)
│   ├── Cause: TPU server misconfiguration → wrong token probabilities
│   ├── Symptoms: Thai/Chinese chars, syntax errors in English
│   ├── Scope: Opus + Sonnet models
│   └── Fix: Rolled back change, added detection tests
│
├── Bug 3: Approximate Top-k XLA:TPU Miscompilation (Aug 25 – ?)
│   ├── Cause: Code deployment triggered latent compiler bug
│   ├── Scope: Haiku 3.5 confirmed; Sonnet 4, Opus 3 potentially
│   └── Fix: Switched from approximate to exact top-k
│
├── Why Detection Was Hard
│   ├── Bugs overlapped in time (same symptom sources)
│   ├── Privacy protections limit engineer access to user data
│   └── Inconsistent symptoms across platforms
│
└── Improvements Committed
    ├── More sensitive evaluations
    ├── Continuous production monitoring
    └── Better debugging infrastructure
```

---

## Overview

Between August 5 and mid-September 2025, three distinct infrastructure bugs degraded Claude's response quality. These bugs overlapped in time, making diagnosis challenging. Anthropic published this postmortem to be transparent about what happened and what improvements are being made.

**Anthropic's stated principle:**
> "We never reduce model quality due to demand, time of day, or server load."

---

## Bug 1: Context Window Routing Error

### Timeline
- **August 5:** Bug introduced
- **August 29:** Load balancing change escalates from 0.8% to 16% of requests
- **September 4:** Fix identified and deployed
- **September 18:** Full rollout completed

### What Happened

```
Normal routing:
User request → Server configured for standard context window → Normal response

Buggy routing:
User request → Server configured for 1M token context window → Degraded response

Why the wrong configuration causes problems:
Standard model         vs.   1M-token model configuration
├── Optimized for         ├── Different attention patterns
  standard sequences         optimized for very long sequences
├── Normal performance    └── Suboptimal for typical requests
└── Expected behavior         → Confused, lower quality responses
```

### The Sticky Nature of the Bug

```
Load balancing behavior:
├── Once a user's request went to an affected server
└── Subsequent requests from that user stayed on that server
    → "Sticky" sessions

Effect:
├── Random 0.8% of Sonnet 4 users affected initially
└── Once affected → persistently poor experience for that user
    → User thinks model degraded permanently
    → Disproportionate impact on user perception
```

### Scale of Impact

```
Aug 5 - Aug 28:   ~0.8% of Sonnet 4 requests affected
Aug 29 - Sep 4:   Up to 16% of Sonnet 4 requests affected
                  (after load balancing change amplified bug)
Sep 4 onwards:    Fix deployed; impact decreasing
Sep 18:           Full rollout; 0% affected
```

---

## Bug 2: Output Corruption

### Timeline
- **August 25:** Misconfiguration deployed
- **September 2:** Identified and rolled back
- **Post-fix:** Detection tests added

### What Happened

```
Cause: TPU (Tensor Processing Unit) server misconfiguration

Normal token probability assignment:
Token "the" → probability 0.23
Token "a"   → probability 0.18
Token " "   → probability 0.15
(reasonable distribution → normal text generation)

Buggy token probability assignment:
Token "the" → probability 0.001  ← wrong!
Token "ก"   → probability 0.15   ← Thai character with inflated probability
Token " "   → probability 0.05
(distorted distribution → unexpected characters appear)
```

### Symptoms Users Observed

```
English query input → Response with unexpected characters:
├── Thai characters inserted mid-sentence: "The result ก is..."
├── Chinese characters: "We recommend 的 the following..."
├── Syntax errors in code: def func(x)ะ: [Thai char instead of colon]
└── Garbled text at random points in otherwise normal responses
```

### Affected Models

```
Confirmed affected:
├── Claude Opus (all versions on affected TPUs)
└── Claude Sonnet (all versions on affected TPUs)

Not affected:
└── Haiku (different hardware configuration)
```

### Resolution

1. Rolled back the misconfiguration on September 2
2. Added automated detection tests to catch similar issues before deployment
3. Detection tests check for unexpected character distributions in outputs

---

## Bug 3: Approximate Top-k XLA:TPU Miscompilation

### Timeline
- **August 25:** Code deployment triggers latent compiler bug
- **Discovery:** Found during investigation of Bug 2 symptoms
- **Fix:** Switched to exact top-k implementation

### What Happened (Technical Deep Dive)

```
Background: Token selection in language models

After computing token probabilities, models select next tokens using "top-k":
1. Identify the k tokens with highest probability
2. Sample from those k tokens

Approximate top-k: Faster, uses clever distributed sorting
Exact top-k: Slower, mathematically guaranteed correct results
```

**The bug:**

```
XLA (Accelerated Linear Algebra) compiler for TPU
└── Compiled "approximate top-k" operation

Latent compiler bug triggered by new code:
└── Complex precision arithmetic in distributed sorting
    across multiple TPU chips
    → Miscompilation: output of approximate top-k was wrong
    → Wrong tokens selected at subtle but systematic rate

Effect: Token selection biased in subtle ways
→ Responses "drift" in unexpected directions
→ Quality degrades in hard-to-characterize ways
```

### Affected Models

```
Confirmed affected:
└── Haiku 3.5

Potentially affected (under investigation):
├── Sonnet 4
└── Opus 3
```

### The Challenge of "Approximate"

```
Why approximate top-k exists:
├── Exact top-k requires all TPU chips to agree on rankings
├── Very slow across thousands of chips
└── Approximate top-k is 10-100× faster

Why the bug was hard to find:
├── Approximation errors are small (1-2% token drift)
├── Not obvious in any single output
├── Only visible as statistical degradation over many outputs
└── Symptoms overlap with other bugs (both bugs affected same period)
```

### Resolution

```
Decision: Switch from approximate to exact top-k
Trade-off:
├── Cost: Slightly slower inference (higher latency or cost)
└── Benefit: Mathematically guaranteed correct token selection

Rationale: Quality reliability > marginal speed improvement
```

---

## Why Detection Was Hard

### The Overlap Problem

```
Timeline:
Aug  5:  Bug 1 begins (context window routing)
         ↓
Aug 25:  Bugs 2 and 3 begin simultaneously
         ↓
Aug 25 - Sep 2: All THREE bugs active simultaneously

Diagnostic challenge:
├── User reports: "Claude seems degraded"
├── Possible cause: Bug 1? Bug 2? Bug 3? All three?
└── Each bug's symptoms overlap with others
    → Can't isolate root cause from symptoms alone
    → Requires deep infrastructure investigation
```

### Privacy Protections vs. Debugging

```
Tension:
├── Privacy protection: Engineers cannot access user conversations
│   → Prevents debugging directly from user reports
└── Debugging need: See actual problematic outputs to diagnose
    → Must rely on aggregate statistics and reported examples

Effect:
├── Bugs with inconsistent symptoms are hard to detect
├── Users must explicitly report with examples
└── Privacy-preserving monitoring needed (aggregate, not individual)
```

### Monitoring Gaps Revealed

```
Before these bugs, monitoring detected:
├── Hard failures (crashes, timeouts, errors)
└── Statistical outliers in aggregate quality metrics

These bugs revealed gaps:
├── Routing misconfiguration not caught by existing checks
├── TPU probability distortion not caught by existing evaluations
└── Compiler miscompilation not caught by standard testing
```

---

## Improvements Committed

### 1. More Sensitive Evaluations

```
Before: Evaluations measured quality at production scale
        → Detected large quality drops
        → Missed subtle systematic degradation

After: New evaluations specifically designed to detect:
       ├── Character distribution anomalies (Bug 2 type)
       ├── Token selection quality metrics (Bug 3 type)
       └── Response consistency across server types (Bug 1 type)
```

### 2. Continuous Production Monitoring

```
Before: Periodic batch evaluations
        → Hours between degradation and detection

After: Continuous monitoring
       ├── Real-time quality signals
       ├── Automatic alerts on quality drops
       └── Faster escalation path for user reports
```

### 3. Better Debugging Infrastructure

```
Before: Debugging required reading user conversations (privacy issue)
        → Engineers couldn't directly investigate

After: Privacy-preserving debugging tools
       ├── Aggregate quality metrics by server/configuration
       ├── Canary testing infrastructure
       └── Easier isolation of specific hardware configurations
```

---

## Lessons Learned

| Lesson | Details |
|--------|---------|
| Overlapping bugs multiply difficulty | Three simultaneous bugs created diagnostic noise |
| Approximate ≠ Good enough | Approximations in critical paths can have subtle but real quality effects |
| Privacy ↔ Debugging tension | Privacy protections must be balanced with ability to diagnose real issues |
| Sticky routing amplifies bugs | Small routing bugs can become large user experience bugs |
| Compiler bugs can be latent | Code changes can trigger pre-existing compiler bugs |

---

## Quick Reference

**The three bugs in brief:**

| Bug | Cause | Peak Impact | Fixed |
|-----|-------|-------------|-------|
| Context Window Routing | Wrong server config | 16% of Sonnet 4 | Sep 18 |
| Output Corruption | TPU misconfiguration | All Opus/Sonnet | Sep 2 |
| Top-k Miscompilation | XLA compiler bug | Haiku 3.5 + others | After discovery |

**Key commitment:**
> "We never reduce model quality due to demand, time of day, or server load."

**Three improvements:** More sensitive evals + continuous monitoring + better privacy-preserving debugging tools.
