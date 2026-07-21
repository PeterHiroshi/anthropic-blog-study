# Scaling Managed Agents: Decoupling the Brain from the Hands

> Source: https://www.anthropic.com/engineering/managed-agents
> Published: April 8, 2026 | Anthropic Engineering
> Authors: Lance Martin, Gabe Cemaj, Michael Cohen
> Docs: https://platform.claude.com/docs/en/managed-agents/overview

---

## Overview Mind Map

```
Scaling Managed Agents
├── The Problem
│   └── Harnesses encode assumptions that go stale as models improve
│       └── Ex: Sonnet 4.5 "context anxiety" → resets
│              Opus 4.5 behavior gone → resets = dead weight
├── The OS Analogy
│   ├── Virtualize hardware into abstractions (process, file)
│   ├── General enough for "programs as yet unthought of"
│   └── read() agnostic to 1970s disk pack vs modern SSD
├── Three Virtualized Components
│   ├── session — append-only event log
│   ├── harness — loop calling Claude, routing tool calls
│   └── sandbox — execution environment
├── Don't Adopt a Pet
│   ├── All-in-one container = pets-vs-cattle problem
│   ├── Container failure lost the session
│   ├── Debugging impossible (only window = WebSocket stream)
│   └── VPC connection required network peering
├── Decouple Brain from Hands
│   ├── Harness leaves container
│   ├── execute(name, input) -> string
│   ├── provision({resources})
│   ├── Container becomes cattle; failures = tool-call errors
│   └── Harness becomes cattle: wake / getSession / emitEvent
├── Security Boundary
│   ├── Coupled: creds + untrusted code, same container
│   ├── Pattern A: auth bundled with resource (git remote)
│   └── Pattern B: vault outside sandbox (MCP OAuth proxy)
├── Session ≠ Context Window
│   ├── Compaction / memory tool / trimming = irreversible
│   ├── Session lives outside the context window, durable
│   ├── getEvents() → positional slices (resume/rewind/reread)
│   └── Harness transforms fetched events
└── Many Brains, Many Hands
    ├── No VPC peering; containers only when needed
    ├── p50 TTFT ↓ ~60%, p95 TTFT ↓ >90%
    ├── Any hand: container, phone, Pokémon emulator
    └── Brains can pass hands to one another
```

---

## 1. The Core Problem: Harnesses Encode Expiring Assumptions

A harness is the loop that calls Claude and routes its tool calls. In building it, engineers inevitably encode assumptions about **what the model cannot do** — and those assumptions have a shelf life.

**The canonical example from the article:**

| Model | Behavior | Harness response | Result |
|-------|----------|------------------|--------|
| Sonnet 4.5 | "Context anxiety" — degraded as context filled | Harness performs context resets | Necessary workaround |
| Opus 4.5 | Behavior gone | Resets still firing | **Dead weight** |

The workaround outlived the problem. Every such patch is a liability the next model release converts into overhead.

**Key insight:** you cannot build a durable agent platform on assumptions about the current model. You need abstractions that survive model turnover.

---

## 2. The Operating System Analogy

The article's central framing device: operating systems solved this exact problem decades ago.

```
Hardware          →  Virtualized abstraction  →  Property
─────────────────────────────────────────────────────────────
Disk pack (1970s) →  file                     →  read() works
SSD (modern)      →  file                     →  read() works
CPU / memory      →  process                  →  scheduler works
```

The OS virtualizes hardware into abstractions like **process** and **file** — abstractions general enough to serve **"programs as yet unthought of."** `read()` is agnostic to whether the bytes come from a 1970s disk pack or a modern SSD. The abstraction outlasted every implementation beneath it.

Managed Agents applies the same move to agents by virtualizing three components:

| Component | Definition | OS counterpart (conceptually) |
|-----------|-----------|-------------------------------|
| **session** | Append-only event log of the agent's activity | The durable file / journal |
| **harness** | The loop calling Claude and routing tool calls | The process |
| **sandbox** | The execution environment | The virtualized machine |

Be opinionated about **interfaces**, not **implementations**.

---

## 3. Don't Adopt a Pet

The first architecture put everything — harness, sandbox, session state — **inside one container**. This is the classic **pets vs. cattle** failure mode: infrastructure that must be individually named, hand-tended, and nursed back to health.

### What went wrong

```
┌─────────────────────────────────────────┐
│           ONE CONTAINER                 │
│  ┌──────────┐  ┌──────────┐            │
│  │ harness  │  │ sandbox  │            │
│  └──────────┘  └──────────┘            │
│  ┌──────────┐  ┌──────────┐            │
│  │ session  │  │ user data│            │
│  │  state   │  │ + creds  │            │
│  └──────────┘  └──────────┘            │
└─────────────────────────────────────────┘
        │
        └── WebSocket event stream (the ONLY window in)
```

| Failure | Consequence |
|---------|-------------|
| **Container dies** | The session dies with it — work is lost, unrecoverable |
| **Container hangs** | Debugging impossible: the only window in was the WebSocket event stream |
| **User data co-located** | Even that window couldn't be freely inspected — user data lived in the same container |
| **Customer VPC** | Connecting required network peering, because the harness assumed co-location with resources |

The container had become a pet. Worse, it was a pet holding both the credentials and the untrusted code.

---

## 4. Decouple the Brain from the Hands

The fix: **the harness leaves the container.**

### 4.1 The sandbox becomes a tool

Instead of running *inside* the execution environment, the harness calls it the way it calls **any other tool**:

```
execute(name, input) -> string
provision({resources})
```

That's the whole interface for a "hand." Consequences:

- The container becomes **cattle** — anonymous, replaceable, disposable.
- Container failures stop being infrastructure incidents and **come back as ordinary tool-call errors**, which the model already knows how to handle and retry around.
- Resources are requested explicitly via `provision({resources})` rather than baked into a fixed image.

### 4.2 The harness becomes cattle too

Once session state lives outside the harness, the harness itself is disposable:

```
wake(sessionId)            → reboot a harness onto an existing session
getSession(id)             → retrieve session history
emitEvent(id, event)       → append new activity to the log
```

A harness that dies is replaced by another that calls `wake(sessionId)`, reads history via `getSession(id)`, and continues appending with `emitEvent(id, event)`. Nothing about the agent's progress was ever in the harness's memory.

```
BEFORE                          AFTER
──────                          ─────
┌───────────────┐               ┌─────────┐   execute()   ┌─────────┐
│ harness       │               │ harness │ ────────────▶ │ sandbox │
│ + sandbox     │  ← pet        │ (brain) │ ◀──────────── │ (hands) │
│ + session     │               └────┬────┘   result/err  └─────────┘
└───────────────┘                    │
   ↓ dies = all lost          wake / getSession / emitEvent
                                     │
                                ┌────▼────┐
                                │ session │  ← durable, outside both
                                └─────────┘
```

---

## 5. The Security Boundary

The coupled design had a specific, serious flaw: **untrusted generated code ran in the same container as the credentials.**

```
COUPLED (unsafe)
  prompt injection → read env vars → obtain tokens
                  → spawn unrestricted sessions
```

An injection attack could read the environment and spawn unrestricted sessions. Decoupling makes this structural rather than a matter of vigilance: **tokens are never reachable from the sandbox.**

Two patterns achieve this:

### Pattern A — Auth bundled with the resource

The credential is *consumed at provisioning time* and wired into the resource, never handed to the agent.

> A Git access token clones the repo at sandbox init and is wired into the local git remote. `push` and `pull` then work **without the agent ever handling the token.**

### Pattern B — Auth held in a vault outside the sandbox

The credential lives outside and is exchanged on demand by a component the agent cannot reach.

> MCP OAuth tokens sit in a vault. Claude calls MCP through a **dedicated proxy** that exchanges a session token for the real credentials. **The harness never sees the credentials.**

| Pattern | Credential location | Agent sees | Example |
|---------|--------------------|-----------|---------|
| A: bundled with resource | Baked into resource config at init | Nothing | Git remote |
| B: vault + proxy | External vault | Session token only | MCP OAuth |

**The principle:** don't ask the agent to be careful with secrets. Arrange things so the secrets were never within reach.

---

## 6. The Session Is Not Claude's Context Window

This is the article's most conceptually load-bearing distinction.

Long-horizon work exceeds the context window, so harnesses do things to it — and **every one of those things is an irreversible decision**:

| Technique | What it discards | Why it's risky |
|-----------|-----------------|----------------|
| Compaction | Detail, replaced by summary | Summary can't be un-summarized |
| Memory tool | Whatever wasn't written down | Selection is a guess |
| Context trimming | Older turns | It's hard to know which tokens a future turn will need |

The problem is not that these techniques are bad. It's that **they are lossy and permanent**, and you're making the call before knowing what will matter.

### The fix: session as a context object outside the context window

```
        ┌──────────────── SESSION (durable store) ────────────────┐
        │  e0  e1  e2  e3  e4  e5  e6  e7  e8  e9  e10 ...        │
        └──────────────────────────────────────────────────────────┘
             │            │                    │
        getEvents()  getEvents()          getEvents()
          rewind       reread               resume
             │            │                    │
             ▼            ▼                    ▼
        ┌──────────────────────────────────────────────┐
        │ harness: transform → Claude's context window │
        └──────────────────────────────────────────────┘
```

`getEvents()` allows **positional slices** of the log, which enables three distinct moves:

- **Resume** — pick up where it stopped reading
- **Rewind** — go back to before a moment
- **Reread** — re-examine the log before taking an action

The harness may **transform** fetched events before they reach Claude — for prompt cache hit rate, or for any context engineering strategy it wants.

### Separation of concerns

| Layer | Guarantees | Free to do |
|-------|-----------|-----------|
| **Session** | Durability + interrogability | (nothing opinionated) |
| **Harness** | — | Arbitrary context management |

The session promises the bytes are still there and still inspectable. What you *show the model* is entirely the harness's business — and can change with every model release without ever losing data.

---

## 7. Many Brains, Many Hands

Decoupling paid off in ways beyond reliability.

### 7.1 Performance

Because the harness no longer lived with the sandbox, it **no longer required VPC peering**, and containers could be **provisioned only when actually needed** rather than up front for every session.

| Metric | Improvement |
|--------|-------------|
| **p50 TTFT** | dropped ~60% |
| **p95 TTFT** | dropped **over 90%** |

**TTFT** = time between accepting work and the first response token — *the latency users most feel*. The p95 number is the striking one: the worst-case waits, which are what make a product feel broken, largely disappeared.

### 7.2 Many hands

Because a hand is nothing more than `execute(name, input) -> string`, the harness **doesn't know or care what's on the other end**:

```
                  ┌─────────┐
        ┌─────────│ harness │─────────┐
        │         └────┬────┘         │
   execute()      execute()      execute()
        │              │              │
        ▼              ▼              ▼
  ┌──────────┐   ┌─────────┐   ┌──────────────┐
  │container │   │  phone  │   │   Pokémon    │
  │          │   │         │   │   emulator   │
  └──────────┘   └─────────┘   └──────────────┘
```

The article names exactly these: a container, a phone, a Pokémon emulator. Same interface, arbitrary implementation — the OS analogy paying off literally.

### 7.3 Many brains

Since brains are stateless and hands are addressable tools, **brains can pass hands to one another.** Multi-agent topologies become a composition question rather than an infrastructure question.

---

## 8. Conclusion: A Meta-Harness

Managed Agents positions itself as a **meta-harness**: **opinionated about interfaces, not implementations.**

It enforces how you manipulate state (`getSession`, `emitEvent`, `getEvents`) and how you request computation (`execute`, `provision`), while remaining entirely unopinionated about *what loop you write on top*. That's precisely the OS bargain — the abstraction is fixed so the things built on it are free to change.

The bet: model behavior will keep shifting, so the layer that survives is the one that never claimed to know what the model needed.

---

## 9. Connection to Other Articles

- **[Effective Harnesses for Long-Running Agents](../effective-harnesses-for-long-running-agents/)** — That article is about *what to put in a harness* (initializer + coding agent, progress tracking, JSON feature lists). This one is about *what a harness must never own*. Read together: the harness logic there should be the disposable, model-specific layer described here.
- **[Harness Design for Long-Running Apps](../harness-design-long-running-apps/)** — Direct companion piece on harness design tradeoffs; this article supplies the infrastructure substrate those designs run on.
- **[Effective Context Engineering for AI Agents](../effective-context-engineering-for-ai-agents/)** — That article covers compaction, memory tools, and context rot as *techniques*. Section 6 here reframes them as *irreversible decisions*, and offers durable sessions as the safety net that makes aggressive context engineering survivable.
- **[Building Effective Agents](../building-effective-agents/)** — The "simple over complex" principle appears here as an architectural argument: complexity that encodes model limitations is complexity with an expiry date.

---

## Key Takeaways

```
1. HARNESS ASSUMPTIONS EXPIRE
   → Sonnet 4.5 context resets became dead weight on Opus 4.5

2. BE OPINIONATED ABOUT INTERFACES, NOT IMPLEMENTATIONS
   → read() outlived the disk pack; execute() should outlive the container

3. NO PETS
   → Sandbox and harness are both cattle
   → Container failure = a tool-call error, not an incident

4. SECRETS SHOULD BE OUT OF REACH, NOT HANDLED CAREFULLY
   → Bundle auth with the resource, or vault it behind a proxy

5. SESSION ≠ CONTEXT WINDOW
   → Session = durable + interrogable; harness = arbitrary transformation
   → getEvents() gives resume / rewind / reread

6. DECOUPLING IS ALSO A PERFORMANCE STORY
   → p50 TTFT ↓ ~60%, p95 TTFT ↓ >90%

7. A HAND IS JUST execute(name, input) -> string
   → Container, phone, or Pokémon emulator — the brain can't tell
```
