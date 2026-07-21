# Harness Design for Long-Running Application Development

> Source: https://www.anthropic.com/engineering/harness-design-long-running-apps
> Published: March 24, 2026 | Anthropic Engineering
> Author: Prithvi Rajasekaran (Anthropic Labs)

---

## Overview Mind Map

```
Harness Design for Long-Running Apps
├── Core Idea
│   └── GAN-inspired generator/evaluator SEPARATION applied to agentic coding
├── Two Failure Modes of Naive Long-Running Agents
│   ├── Context window degradation
│   │   ├── Coherence decays as context fills
│   │   ├── "Context anxiety" — premature wrap-up near perceived limits
│   │   └── Fix: context RESETS > compaction (clean slate + handoff)
│   └── Self-evaluation bias
│       ├── Agents praise their own work regardless of quality
│       ├── Worst on subjective tasks (design) w/ no binary verification
│       └── Fix: split evaluation into a dedicated agent
├── Frontend Design Loop (generator + evaluator)
│   ├── 4 grading criteria
│   │   ├── Design Quality — coherence, mood, visual identity
│   │   ├── Originality — custom decisions vs template defaults
│   │   ├── Craft — typography, spacing, color, contrast
│   │   └── Functionality — usability, task completion
│   ├── Evaluator drives Playwright on the LIVE design before scoring
│   └── Emphasis on Quality + Originality → less generic output
├── Full-Stack: Three-Agent Architecture
│   ├── Planner    — brief prompt → detailed spec
│   ├── Generator  — implements in sprints (React/Vite/FastAPI/SQLite/git)
│   ├── Evaluator  — Playwright, checks UI + APIs + DB state
│   └── "Sprint contract" negotiated BEFORE each sprint
├── Evidence
│   ├── Retro game maker, Opus 4.5: solo 20min/$9 → broken
│   └──                             harness 6hr/$200 → playable
└── The Punchline
    └── Harness complexity should DECREASE as models improve
```

---

## 1. The Starting Point: Two Ceilings

The author was chasing two goals at once — getting Claude to produce genuinely good **frontend design**, and getting it to build **complete applications autonomously**. Prompt engineering and single-agent harness tweaks both hit a ceiling.

The unlock came from an old idea: **Generative Adversarial Networks**. GANs work because the generator and the discriminator are *separate networks with separate objectives*. Port that structure to agents and you get a generator agent and an evaluator agent — which eventually grew into a three-agent architecture for multi-hour coding sessions.

---

## 2. Why Naive Long-Running Agents Fall Short

### 2.1 Context Window Degradation

Models lose coherence as the context window fills. This is not just a hard truncation problem — quality decays well before the limit.

A specific, named symptom: **"context anxiety."** Claude Sonnet 4.5 would start *wrapping up work prematurely* as it approached what it perceived as its token limit — rushing to a conclusion rather than continuing to do the work well.

| Strategy | What it does | Verdict |
|----------|--------------|---------|
| **Compaction** | Summarizes history, preserves continuity | Keeps continuity, but carries the degradation forward |
| **Context reset** | Clears the window, restarts from a structured handoff | **Better** — a genuinely clean slate |

> The distinction matters: compaction preserves *continuity*; resets restore *capacity*.

### 2.2 Self-Evaluation Bias

Ask an agent to grade its own output and it will confidently praise mediocre work. This is worst on **subjective tasks** — design especially — where there is no binary verification (no test suite that goes green or red).

The failed approach: make the generator more self-critical.
The working approach: **take evaluation out of the generator entirely.**

```
   SELF-EVALUATION                    SEPARATED EVALUATION
   ┌──────────────┐                   ┌──────────┐   ┌───────────┐
   │  Generator   │                   │Generator │ → │ Evaluator │
   │  "looks      │                   │          │ ← │  (fresh   │
   │   great!" ✓  │                   │          │   │  context) │
   └──────────────┘                   └──────────┘   └───────────┘
   No leverage                        Leverage: a critic with no
                                      ego investment in the output
```

---

## 3. Frontend Design: Making Subjective Quality Gradable

The hard part of design is that "good" isn't checkable. The solution was to make it **gradable** — define explicit criteria and give a separate agent the job of applying them.

### The Four Grading Criteria

| Criterion | What it measures |
|-----------|------------------|
| **Design Quality** | Coherence, mood, a distinct visual identity |
| **Originality** | Custom design decisions vs. falling back on template defaults |
| **Craft** | Typography, spacing, color, contrast |
| **Functionality** | Usability, does it let the user complete the task |

**Weighting insight:** the evaluator put *more* emphasis on **Design Quality** and **Originality**, because Claude already scores well on Craft and Functionality by default. Pushing on the first two is what moved output away from generic.

### The Loop

```
 ┌─────────────┐   builds HTML/CSS/JS    ┌──────────────┐
 │  GENERATOR  │ ──────────────────────► │  live page   │
 └─────────────┘                          └──────┬───────┘
        ▲                                        │
        │                                 Playwright MCP:
        │  critique feeds next iteration   navigate, click,
        │                                  screenshot, inspect
        │                                        │
        │            ┌──────────────┐            │
        └─────────── │  EVALUATOR   │ ◄──────────┘
                     │ scores on 4  │
                     │  criteria    │
                     └──────────────┘

 Built on the Claude Agent SDK + Playwright MCP.
 Runs lasted 4+ hours, 5–15 iterations per generation.
```

The evaluator does **not** grade from source code or a static screenshot — it *interacts with the live design* first, then scores.

### The Museum Example

A Dutch art museum site started as a conventional dark-themed landing page and was refined incrementally through iteration nine. Then, on **iteration ten**, the generator scrapped the approach entirely and reimagined the site as a **spatial experience** — a 3D room with a checkered floor rendered in CSS perspective, artwork hung on the walls in free-form positions, and doorway-based navigation between gallery rooms.

> This is the payoff of grading Originality: sustained pressure eventually forces a *qualitative* jump rather than another round of polish.

---

## 4. Scaling to Full-Stack: The Three-Agent Architecture

```
┌──────────────────────────────────────────────────────────────┐
│  PLANNER                                                     │
│  brief prompt ──► detailed product spec                      │
│  • emphasizes ambitious SCOPE and high-level technical        │
│    direction                                                 │
│  • deliberately AVOIDS over-specifying implementation         │
└───────────────────────────┬──────────────────────────────────┘
                            ▼
        ┌───────── sprint contract negotiated ─────────┐
        │  generator + evaluator agree on deliverables │
        │  and success criteria BEFORE implementation  │
        └───────────────────┬──────────────────────────┘
                            ▼
┌──────────────────────────┐        ┌──────────────────────────┐
│  GENERATOR               │───────►│  EVALUATOR               │
│  implements sprint       │        │  tests like a USER via   │
│  React · Vite · FastAPI  │◄───────│  Playwright              │
│  SQLite/PostgreSQL · git │  QA    │  checks UI + APIs + DB   │
│  self-evals before QA    │        │  hard thresholds         │
└──────────────────────────┘        └──────────────────────────┘
```

### Component Roles

| Agent | Job | Key design choice |
|-------|-----|-------------------|
| **Planner** | Expand a one-line brief into a full product spec | Ambitious scope + technical direction, but let implementation details emerge organically |
| **Generator** | Build features sprint by sprint | Real stack (React/Vite/FastAPI/SQLite or PostgreSQL) with git version control |
| **Evaluator** | Verify against specific criteria | Clicks through the app like a user; grades product depth, functionality, design, code quality against hard thresholds |

### The Sprint Contract

Before each sprint, generator and evaluator **negotiate a contract** — what will be delivered and what counts as success — *before* any implementation. This is the structural fix for "the generator decides after the fact that whatever it built was the goal."

---

## 5. Evidence: Solo Agent vs. Full Harness

Same prompt (a **2D retro video game maker**), same model (**Claude Opus 4.5**):

| | Solo agent | Full harness |
|---|-----------|--------------|
| **Duration** | 20 minutes | 6 hours |
| **Cost** | $9 | $200 |
| **Outcome** | **Broken core functionality** — entities appeared but didn't respond to input | **Fully playable game** |
| **Features** | — | Sprite animation systems, behavior templates, AI-assisted generation, game export |

The planner expanded the one-sentence prompt into **16 features across ten sprints**.

> The cost ratio is ~22x and the time ratio ~18x. That's the honest price of the result — this is not a free lunch, it's a deliberate trade of compute for quality on tasks where quality is the binding constraint.

---

## 6. Iterating: Removing Complexity as Models Improve

When **Claude Opus 4.6** arrived — planning more carefully, sustaining agentic tasks longer, with better code review and debugging — the author did the counterintuitive thing: **deleted part of the harness.**

```
  OPUS 4.5 HARNESS                    OPUS 4.6 HARNESS
  ┌─────────┐                         ┌─────────┐
  │ Planner │                         │ Planner │
  └────┬────┘                         └────┬────┘
       ▼                                   ▼
  ┌─────────────┐                     ┌───────────┐
  │  SPRINTS    │  ◄── REMOVED        │ Generator │  (long coherent run)
  │ decomposed  │                     └─────┬─────┘
  └────┬────────┘                           ▼
       ▼                                ┌───────────┐
  ┌───────────┐                         │ Evaluator │
  │ Evaluator │                         └───────────┘
  └───────────┘
```

The **sprint construct was removed**; planner and evaluator were kept. The evaluator shifted toward single-pass, end-of-run grading — though it retains its value for tasks at the edge of model capability.

### Result: A Browser DAW

A **Digital Audio Workstation** built with the updated harness:

| Metric | Value |
|--------|-------|
| Total duration | ~3 hours 50 minutes |
| Total cost | **$124.70** |

And critically — **the evaluator still earned its place.** It caught real gaps the generator had missed and reported as done, including missing interactive features and an audio recording implementation that was not actually there.

> Note the direction of the numbers: less harness, comparable ambition, *lower* cost (~$125 vs ~$200). Complexity was carrying weight for 4.5 that 4.6 carries itself.

---

## 7. The Central Lesson

> **Harness complexity should DECREASE as model capability increases.**

Every component of a harness **encodes an assumption about a model limitation**. Sprints encoded "this model can't sustain a long coherent build." Compaction encoded "this model can't manage its own context." Those assumptions have expiry dates.

```
        Model capability ──────────────────────────►
        │
        │  ████████████  harness complexity
        │  ████████
        │  ████
        │
        │  ░░░░  independent model capability
        │  ░░░░░░░░
        │  ░░░░░░░░░░░░
        ▼
   The boundary of independent capability moves OUTWARD.
   Harness work is REDIRECTED, not eliminated.
```

### The Operating Principles

```
1. Find the simplest solution possible, and only increase
   complexity when needed.

2. Every harness component encodes an assumption about a model
   limitation — stress-test those assumptions regularly.

3. Re-run your harness against each new model release.

4. Read traces on realistic problems — don't reason about
   limitations abstractly.

5. Remove components that are no longer load-bearing.
```

The author is explicit that this is *not* an argument that harness design goes away: the space of interesting harness combinations doesn't shrink as models improve — **it moves**.

---

## 8. Appendix: The RetroForge Planner Output

The article includes an example of planner output — the full spec generated from the one-line "2D retro game maker" prompt, named **RetroForge**. Its structure illustrates the expansion:

- **Overview** — platform positioning and target audience
- **Features**, broken into subsections such as:
  - **Project Dashboard & Management** — user stories covering create, view as visual cards, open for editing, delete with confirmation, duplicate
  - **Project Data Model** — metadata structure including resolution, tile size, palette, and content
  - Further sections covering tile-based level editing, pixel-art sprite creation, visual entity behavior systems, and playable test modes

The point of the appendix is to make concrete what "expands brief prompts into detailed specs" actually produces — and to show the planner writing *user stories and data models* rather than code.

---

## 9. Connection to Other Articles

- **[Effective Harnesses for Long-Running Agents](../effective-harnesses-for-long-running-agents/)** — the direct predecessor. That article established the initializer + coding agent pattern with progress tracking; this one adds the **evaluator** as a first-class separate agent and then shows how to *remove* scaffolding as models improve.
- **[Building a C Compiler](../building-c-compiler/)** — another long-horizon autonomous build. Useful contrast: compiler work has strong binary verification (does it compile, do the tests pass), which is exactly the property design work lacks and which motivates the separate evaluator here.
- **[Effective Context Engineering for AI Agents](../effective-context-engineering-for-ai-agents/)** — the theory behind failure mode #1. "Context rot" there is the same phenomenon as the degradation and "context anxiety" described here; this article contributes the practical finding that **resets beat compaction**.
- **[Managed Agents](../managed-agents/)** — the productized side of the same problem. Where this post hand-builds orchestration on the Agent SDK, managed agents push that infrastructure server-side.
- **[Building Effective Agents](../building-effective-agents/)** — the "evaluator-optimizer" workflow pattern named there is precisely what this article scales to multi-hour production runs. The "simplest solution first" principle is restated here as a rule about harness lifespan.

---

## Key Takeaways

```
1. SEPARATE THE EVALUATOR FROM THE GENERATOR
   → Agents cannot grade themselves; splitting the role is the leverage

2. RESETS BEAT COMPACTION
   → Compaction preserves continuity; resets restore a clean slate
   → Watch for "context anxiety": premature wrap-up near token limits

3. MAKE SUBJECTIVE QUALITY GRADABLE
   → Design Quality · Originality · Craft · Functionality
   → Weight the criteria the model is WEAK on (quality, originality)

4. EVALUATE THE RUNNING ARTIFACT, NOT THE CODE
   → Playwright: click through UI, hit APIs, check DB state

5. CONTRACT BEFORE BUILD
   → Negotiate success criteria before implementation, not after

6. THE HARNESS EARNS ITS COST OR IT DOESN'T
   → 20min/$9 broken vs 6hr/$200 playable (Opus 4.5, game maker)

7. COMPLEXITY SHOULD SHRINK AS MODELS GROW
   → Opus 4.6 → sprints removed, planner+evaluator kept
   → DAW: ~3h50m, $124.70 — evaluator still caught real gaps

8. FIND THE SIMPLEST SOLUTION POSSIBLE
   → Only increase complexity when needed
   → Harness work is redirected by better models, not eliminated
```
