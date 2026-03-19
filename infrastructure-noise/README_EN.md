# Quantifying Infrastructure Noise in Agentic Coding Evals

> Source: https://www.anthropic.com/engineering/infrastructure-noise
> Published: February 3, 2026 | Anthropic Engineering

---

## Overview Mind Map

```
Infrastructure Noise in Agentic Evals
├── Core Finding
│   └── Infrastructure config alone causes ~6 pp score swings
├── Resource Configuration Impact
│   ├── 1x specs → 5.8% infra error rate
│   ├── 3x headroom → 2.1% error rate
│   └── Uncapped → 0.5% error rate
├── Two Phases of Impact
│   ├── 1x→3x: Stability (fewer spurious failures)
│   └── 3x→uncapped: Strategy (enables resource-heavy approaches)
├── Other Confounders
│   ├── Time-of-day API latency
│   ├── Cluster health / concurrency
│   └── Egress bandwidth / hardware specs
└── Recommendations
    ├── Eval devs: Separate guaranteed vs. kill threshold
    ├── Consumers: Skepticism below 3 pp differences
    └── Public benchmarks: Run across multiple times/days
```

---

## 1. The Core Problem

Agentic coding benchmarks run inside containers. Container runtimes enforce resources via two parameters:
- **Guaranteed allocation** — what the container is promised
- **Hard kill threshold** — when the OOM killer fires

When these are set equal (common "pinned" configuration), any transient memory spike causes a spurious failure — the agent's code was correct, but the infrastructure killed it.

**Key insight:** This is a hidden variable that changes between eval providers, making cross-leaderboard comparisons unreliable.

---

## 2. Experimental Results

### Terminal-Bench 2.0

| Configuration | Infra Error Rate | Score Impact |
|---------------|-----------------|--------------|
| 1x (strict) | 5.8% | Baseline |
| 3x headroom | 2.1% (p < 0.001) | Within noise |
| Uncapped | 0.5% | +6 pp (p < 0.01) |

### SWE-bench

- Tested 227 problems, 10 samples each
- 5x RAM vs. 1x: +1.54 pp increase
- Smaller effect because SWE-bench tasks are less resource-intensive

---

## 3. Two Distinct Phases of Impact

### Phase 1: 1x → 3x (Stability Improvement)

Infrastructure errors drop significantly, but **success rates stay within noise** (p=0.40). The agent isn't doing anything differently — it's just not being killed by spurious OOM events.

### Phase 2: 3x → Uncapped (Strategy Enablement)

With abundant resources, agents can employ resource-intensive strategies:
- Install full dependency stacks (e.g., entire Python data science ecosystem)
- Run memory-intensive test suites
- Spawn expensive subprocesses

**Critical point:** Different models default to different strategies. Some install lean dependencies; others install everything. Resource limits determine which strategies succeed, creating model-specific bias.

### Example: `bn-fit-modify` Task

Some models install the full Python data science stack for Bayesian network fitting. Under tight constraints, this fails. Under generous constraints, it succeeds. Leaner implementations exist but aren't universally adopted by all models.

---

## 4. Other Confounders

Beyond resource limits, the article identifies several additional noise sources:

- **Time-of-day variations** — API latency fluctuates with usage patterns
- **Cluster health** — Shared infrastructure degrades under load
- **Concurrency levels** — Running many eval instances simultaneously
- **Egress bandwidth** — Network-dependent operations (package downloads)
- **Hardware specifications** — CPU architecture, disk I/O differences

---

## 5. Practical Recommendations

### For Eval Developers

```
DO:   Specify guaranteed allocation AND kill threshold separately
DO:   Calibrate the gap so scores fall within statistical noise
DO:   Run evaluations at multiple times and days

DON'T: Pin resources to a single value
DON'T: Assume equal configs across providers
```

**Calibration heuristic:** A 3x ceiling over per-task specs reduced infra errors from 5.8% to 2.1% while keeping score lift within noise — a reasonable tradeoff.

### For Leaderboard Consumers

> Treat differences below 3 percentage points with skepticism unless eval configurations are documented and matched.

Standard confidence intervals (1-2 pp) stack **atop** infrastructure confounders, not within them. The real uncertainty is wider than reported.

### For Public Benchmarks

Run evaluations at multiple times and days to average out temporal noise.

---

## 6. Connection to Other Articles

- **[Demystifying Evals](../demystifying-evals-for-ai-agents/)** — This article adds a crucial infrastructure layer to the eval framework discussed there. Even a well-designed eval can produce misleading results if infrastructure isn't controlled.
- **[SWE-bench Sonnet](../swe-bench-sonnet/)** — The SWE-bench results discussed here provide additional context for interpreting the scores reported in that article.

---

## Key Takeaways

```
1. INFRASTRUCTURE IS A HIDDEN VARIABLE
   → 6 pp swings from config alone — can exceed model gaps

2. TWO PHASES: STABILITY vs. STRATEGY
   → 1x→3x removes spurious failures
   → 3x→uncapped enables different agent strategies

3. 3 PP SKEPTICISM THRESHOLD
   → Below this, differences may be infrastructure, not capability

4. SEPARATE GUARANTEED vs. KILL THRESHOLD
   → Pinning them equal maximizes spurious failures

5. CROSS-PROVIDER COMPARISONS ARE SUSPECT
   → Different providers, different configs, different scores
```
