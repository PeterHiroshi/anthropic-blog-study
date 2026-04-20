# Scaling Managed Agents: Decoupling the brain from the hands

> Source: https://www.anthropic.com/engineering/managed-agents
> Anthropic Engineering | Authors: Lance Martin, Gabe Cemaj, Michael Cohen

---

## Overview Mind Map

```
Scaling Managed Agents
├── The Problem: Harnesses Become "Pets"
│   ├── Single container = Claude + tools + sandbox + session
│   ├── Session lost if container fails
│   ├── Hard-to-scale hand-maintained infrastructure
│   └── Harness encodes assumptions that age badly
├── The Inspiration: OS Virtualization
│   ├── Virtualize hardware into abstractions (process, file)
│   └── Abstractions general enough for programs that don't exist yet
├── Decoupling: Brain vs Hands
│   ├── Brain = Claude + harness (stateless)
│   ├── Hands = Sandboxes + tools (provisioned on demand)
│   ├── Call pattern: execute(name, input) → string
│   └── Containers as replaceable infrastructure
├── Session as External State
│   ├── Append-only log of events (durable storage)
│   ├── Session ≠ Claude's context window
│   ├── wake(sessionId) + getSession(id) for recovery
│   └── getEvents() for positional slicing (rewind, resume)
├── Credentials Separated From Code
│   ├── Auth bundled with resources (git tokens at init)
│   └── OAuth in vault, accessed via MCP proxy
├── Performance
│   ├── p50 TTFT: ~60% reduction
│   └── p95 TTFT: >90% reduction
└── Many Brains × Many Hands
    ├── Route work across harnesses and sandboxes
    ├── Stay unopinionated about specific harness
    └── Future-proof as Claude capabilities grow
```

---

## 1. "Don't Adopt a Pet"

The legacy design ran Claude, its harness, its tools, and its session state inside one long-lived container. The metaphor: **pets vs. cattle**.

- **Pets**: hand-raised, hand-maintained; losing one is a crisis.
- **Cattle**: interchangeable, replaceable; failure is routine.

A container-as-pet meant:
- Container crash = **session lost**.
- Upgrading a tool meant touching every live session.
- Scaling required provisioning full Claude-plus-tools images, even when the agent was idle.

And — the deeper problem — **the harness encoded assumptions about what Claude couldn't do**. As models improved, those assumptions became scaffolding for ghosts.

> Example: "Context anxiety" safeguards built into the Sonnet 4.5 harness were unnecessary once Opus 4.5 shipped. But the safeguards remained, silently distorting behavior.

---

## 2. Decouple the Brain from the Hands

The fix borrows from operating systems: **virtualize hardware into abstractions general enough that future programs can use them.**

| Layer | Before | After |
|-------|--------|-------|
| Brain (reasoning) | Coupled to container lifecycle | Stateless harness |
| Hands (execution) | Baked into same container | On-demand sandboxes + tools |
| Session (state) | In-process memory | External append-only log |

The harness now invokes hands through a single minimal interface:

```
execute(name: string, input: object) → string
```

Containers become **replaceable infrastructure**, provisioned when a tool call demands them, torn down after. The brain never sees the container, only the string reply.

---

## 3. The Session Is Not Claude's Context Window

Crucial distinction. A **session** is:

- An **append-only event log** living outside any process.
- Fetched via `getEvents()`, which returns positional slices.
- Transformable on the way into Claude's context (cache-optimized, compressed, truncated).

Claude's context window is ephemeral. The session is durable. You can:

- **Resume**: pick up from the last event.
- **Rewind**: fetch events up to some point and re-enter from there.
- **Transform**: shape events into whatever prompt the current harness needs.

This is what lets a failed harness recover. `wake(sessionId)` brings a fresh harness online; `getSession(id)` gives it the full event history; execution continues.

---

## 4. Credentials: Separated from Generated Code

Because sandboxes run Claude-generated code, no user credential should ever land inside them. Two patterns:

**Pattern A — Auth Bundled With Resources**
- E.g., a git checkout sandbox receives a scoped token as part of initialization.
- Token lives for the checkout step, never surfaces to generated code.

**Pattern B — OAuth via MCP Proxy**
- Long-lived user OAuth tokens stay in a secure vault.
- An MCP proxy mediates calls — the sandbox sees only the authenticated tool, never the secret.

---

## 5. Performance: Why Decoupling Pays

Old model: every session paid container-startup cost before Claude could emit a token.

New model: brain starts inferring immediately; hands are provisioned lazily, only when a tool call demands them.

| Metric | Improvement |
|--------|-------------|
| p50 TTFT (time-to-first-token) | ~60% reduction |
| p95 TTFT | >90% reduction |

The tail improvement matters most — it's the long-container-provisioning outliers that used to burn user trust.

---

## 6. Many Brains, Many Hands

Because the brain ↔ hands interface is narrow and stateless, the platform can route:

- **Many brains**: different harnesses for different task shapes (code-editing harness vs. research harness vs. creative-writing harness).
- **Many hands**: different sandbox flavors (ephemeral vs. persistent, Linux vs. browser, local vs. remote).

The architecture stays **unopinionated about the specific harness**. Harnesses are allowed to be task-specific and short-lived; the interface does not bake them in.

This is the payoff: as Claude's capabilities expand, you can strip harness scaffolding instead of rewriting infrastructure around it.

---

## 7. Design Principles Extracted

1. **Don't adopt pets.** If a single failure costs state, you've coupled too much.
2. **State belongs outside the runtime.** The session is the product; the runtime is disposable.
3. **Narrow interfaces decouple evolution.** `execute(name, input) → string` looks too simple — that's the point.
4. **Auth must not flow through generated code.** Either bundle at provision time, or proxy.
5. **Harnesses age.** Design for eventual removal of your own scaffolding.

---

## 8. Connections to Other Articles

- **[Effective Harnesses for Long-Running Agents](../effective-harnesses-for-long-running-agents/)** — the original single-stage harness philosophy that this article evolves. Managed Agents generalizes from "one long-lived harness" to "many stateless harnesses over a durable session."
- **[Harness Design for Long-Running Application Development](../harness-design-long-running-apps/)** — concurrent research on harness components (planner, generator, evaluator) that compose on top of this brain/hands split.
- **[Building Effective Agents](../building-effective-agents/)** — minimal-scaffolding principle. Managed Agents takes it down to the platform layer.
- **[Claude Code Sandboxing](../claude-code-sandboxing/)** — sandbox-as-hands is the same abstraction; Managed Agents makes it remotable.

---

## Key Takeaways

```
1. DON'T ADOPT A PET
   → Single-container harnesses lose sessions on failure.

2. BRAIN/HANDS DECOUPLING
   → Stateless harness + on-demand sandboxes = cattle, not pets.

3. SESSION ≠ CONTEXT WINDOW
   → Append-only event log outside the runtime makes recovery routine.

4. OS-STYLE VIRTUALIZATION
   → Narrow abstractions outlive the programs that use them.

5. PERFORMANCE FOLLOWS ARCHITECTURE
   → ~60% p50 / >90% p95 TTFT drop from lazy sandbox provisioning.

6. MANY BRAINS × MANY HANDS
   → Route tasks to the right harness and the right sandbox.

7. HARNESSES ARE SCAFFOLDING
   → They encode model-era assumptions; design to tear them down.
```
