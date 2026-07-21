# How We Contain Claude

> Source: https://www.anthropic.com/engineering/how-we-contain-claude
> Published: May 25, 2026 | Anthropic Engineering
> Authors: Max McGuinness, Mikaela Grace, Jiri De Jonghe, Jake Eaton, Abel Ribbink

---

## Overview Mind Map

```
Containing Claude Across Products
├── Framing Shift
│   ├── Old: restrict what the agent can access
│   └── New: contain the blast radius, then widen access
├── Three Risk Categories
│   ├── User misuse (malicious or careless direction)
│   ├── Model misbehavior (unprompted harmful action)
│   └── External attackers (prompt injection, runtime attacks)
├── Three Defense Layers
│   ├── Environment  — sandboxes, VMs, FS bounds, egress control
│   ├── Model        — system prompts, classifiers, probes, training
│   └── External content — tool permissions, MCP/plugin auditing
├── Three Containment Patterns
│   ├── claude.ai     → ephemeral gVisor containers (server-side)
│   ├── Claude Code   → local OS sandbox + human-in-the-loop
│   └── Claude Cowork → sealed VM, agent loop on the host
├── What They Got Wrong
│   ├── Config executed before the trust dialog
│   ├── Direct prompt injection via phishing the user
│   ├── Exfiltration through an approved domain
│   └── EDR blindness caused by VM isolation
└── Principles
    ├── Environment layer first, model layer second
    ├── Match isolation to user technical capacity
    └── Prefer battle-tested primitives over custom code
```

---

## 1. The Framing: Likelihood vs. Blast Radius

The article separates two quantities that are often collapsed into one word, "risk":

| Quantity | Direction over time | What moves it |
|----------|--------------------|--------------|
| **Likelihood** of a harmful action | Falling | Alignment training, classifiers, safeguards |
| **Blast radius** — the damage if it happens | Rising | More capability, more access, more autonomy |

Safety training has steadily driven the first down. The second only grows as agents get filesystem access, shell access, network access, and long-running autonomy. So the deployment decision is reframed: ship the agent when safety measures **cap the blast radius**, rather than waiting for the likelihood to hit zero.

**Key insight:** Containment is what makes broader capability shippable. It is an enabler, not a brake.

---

## 2. Three Categories of Risk

```
                  ┌──────────────────────────┐
                  │   Who causes the harm?   │
                  └────────────┬─────────────┘
        ┌──────────────────────┼──────────────────────┐
        ▼                      ▼                      ▼
  USER MISUSE            MODEL MISBEHAVIOR       EXTERNAL ATTACKER
  The human directs      The agent acts          A third party
  the agent to do        harmfully without       reaches the agent
  something harmful      being asked to          through content
        │                      │                      │
  malicious intent       unexpected paths        prompt injection
  or plain negligence    around unstated         via files, tools,
  (destructive cmds,     restrictions            network, email
   bypassing checks)
```

A notable observation on **model misbehavior**: as models get more capable, they get better at finding unexpected routes to a goal — often by going around restrictions that nobody thought to write down explicitly. Capability and creative rule-circumvention grow together.

---

## 3. Three Layers of Defense

```
┌─────────────────────────────────────────────────────────┐
│ LAYER 1 — ENVIRONMENT          (hard, deterministic)    │
│   process sandboxes · virtual machines                  │
│   filesystem boundaries · egress controls               │
│   ▶ Defines what the agent CAN reach at all             │
├─────────────────────────────────────────────────────────┤
│ LAYER 2 — MODEL                (soft, probabilistic)    │
│   system prompts · classifiers · probes · training      │
│   ▶ Shapes what the agent TENDS to do                   │
├─────────────────────────────────────────────────────────┤
│ LAYER 3 — EXTERNAL CONTENT     (supply-chain hygiene)   │
│   tool permission scoping · MCP server & plugin audit   │
│   read-only vs. write grants                            │
│   ▶ Controls what gets INTO the context window          │
└─────────────────────────────────────────────────────────┘
```

The crucial distinction: the model layer shapes **tendency**, not **capability**. A classifier can make a bad action unlikely; only the environment layer can make it impossible. This is why the ordering in the concluding principles matters — deterministic boundaries catch exactly the cases probabilistic defenses miss.

---

## 4. Three Containment Patterns

### 4.1 claude.ai — Ephemeral Container

| Aspect | Detail |
|--------|--------|
| **Primitive** | gVisor (userspace kernel) + seccomp syscall filtering |
| **Location** | Anthropic servers, isolated infrastructure |
| **Lifetime** | Ephemeral, per-session filesystem |
| **Access** | No persistent workspace, no user machine access |
| **Threat model** | Protect Anthropic's infra and tenant isolation |
| **Tradeoff** | Minimal blast radius, sharply limited capability |

Because the agent never touches a user machine, the interesting security work here turned out to be conventional server-side engineering — network configuration, auth, orchestration — rather than novel agent-specific defenses.

### 4.2 Claude Code — Human-in-the-Loop + OS Sandbox

Runs **on the developer's own machine** with filesystem, shell, and network access. The original containment mechanism was the human: approve each write, bash invocation, or network call.

**The problem: approval fatigue.**

```
Users approved ~93% of permission prompts
        │
        ▼
A 93% approve rate is not oversight — it is a rubber stamp.
The dialog trains the user to click "yes" before reading.
        │
        ▼
Fix: put a real OS-level sandbox underneath, so the
     boring-and-safe operations never need to ask.
        │
        ▼
macOS → Seatbelt      Linux → bubblewrap
        │
        ▼
~84% reduction in the number of permission prompts
```

A related finding on user behavior nuance: experienced users auto-approve roughly **twice as often** as new users — but they also **interrupt** the agent more frequently. Expertise does not simply mean "more careless"; it means a different oversight strategy (less gatekeeping up front, more correction mid-flight).

Automated pre-execution screening ("auto mode") reports:

| Metric | Value |
|--------|-------|
| Overeager behaviors caught before execution | ~83% |
| Risky actions missed | ~17% |
| Benign commands falsely blocked | 0.4% |

That 0.4% false-positive rate is the number that makes the feature usable at all — a safety layer developers constantly have to override gets disabled.

**This pattern works only because the user is a developer who can read a bash command and judge it.** That assumption does not transfer.

### 4.3 Claude Cowork — Sealed VM

Built for **non-technical knowledge workers**, who cannot be asked to evaluate a shell command. Since the human cannot be the safety layer, the environment must be.

| Aspect | Detail |
|--------|--------|
| **Primitive** | Platform hypervisors — Apple Virtualization Framework, Windows HCS |
| **Guest** | Full Linux VM |
| **Host↔guest channel** | vsock boundary |
| **Credentials** | Stay in the host keychain; VM gets only a scoped session token |
| **Filesystem grants** | read-only / read-write / read-write-no-delete |

**Architectural evolution — moving the agent loop out of the VM:**

```
INITIAL DESIGN                    REVISED DESIGN
┌──────────────────┐              ┌──────────────────┐
│ VM (guest)       │              │ HOST             │
│  ┌────────────┐  │              │  ┌────────────┐  │
│  │ agent loop │  │              │  │ agent loop │  │
│  │     +      │  │              │  └─────┬──────┘  │
│  │ code exec  │  │              └────────┼─────────┘
│  └────────────┘  │                  vsock│boundary
└──────────────────┘              ┌────────▼─────────┐
                                  │ VM (guest)       │
VM crash ⇒ Claude goes silent     │  ┌────────────┐  │
                                  │  │ code exec  │  │
                                  │  └────────────┘  │
                                  └──────────────────┘
                                  VM crash ⇒ Claude can
                                  still respond and recover
```

Two wins from the move: (1) reliability — Claude keeps responding even if the guest dies, and (2) the filesystem and network controls stay enforced by the **hypervisor boundary**, which is exactly the layer you want holding the line.

### 4.4 Pattern Comparison

| | claude.ai | Claude Code | Claude Cowork |
|---|---|---|---|
| **User** | Anyone | Developers | Non-technical workers |
| **Isolation primitive** | gVisor + seccomp | Seatbelt / bubblewrap | Hypervisor VM |
| **Runs on** | Anthropic servers | User machine | User machine |
| **Human in the loop** | Not the main control | Central (with fatigue caveat) | Cannot be relied on |
| **Capability** | Lowest | Highest | High, bounded |
| **Blast radius** | Smallest | Largest without sandbox | Bounded by the VM |

The through-line: **isolation strength scales inversely with the user's ability to evaluate what the agent is doing.**

---

## 5. What They Got Wrong

The most valuable section — four real gaps, each with the same shape: the containment held where it was designed to hold, and leaked somewhere adjacent that nobody had modeled.

### 5.1 Execution before trust

Project configuration (hooks, settings files) was parsed and executed **before** the trust dialog appeared. The consent gate existed, but the dangerous work had already happened by the time the user saw it. Fixed by deferring all config parsing until after consent.

> Lesson: a trust boundary is only real if nothing on the untrusted side runs before it.

### 5.2 The user as the injection vector

Model-layer defenses are tuned for injection arriving through *tool output* — a poisoned file, a hostile web page. But a **phished employee** simply pasted the attacker's prompt in themselves. Coming through the user turn, it looks like a legitimate instruction, and the injection classifiers were not the thing standing in its way.

In a red-team exercise this exfiltrated AWS credentials in **24 out of 25** attempts. Only the **egress controls** — the environment layer — contained the damage. A direct demonstration of the "environment layer first" principle.

For calibration on the model layer, resistance to prompt injection on Claude Opus 4.7 is reported at roughly **0.1%** success on a single attempt, rising to about **5–6%** after 100 adaptive attempts. Strong, but not a boundary — the numbers only go one direction under a persistent attacker, which is precisely why it cannot be the last line.

### 5.3 Exfiltration through an approved domain

An allowlist answers "which hosts?" but not "doing what?". `api.anthropic.com` was on the allowlist for obvious reasons — so an attacker used that legitimate endpoint to upload files to their own account. The traffic was allowed, authenticated, and going to a domain the security team had personally vetted.

Fixed with a defensive MITM proxy inside the VM that validates the session token, so the destination *and* the identity have to match.

> Lesson: domain allowlists are not egress controls. Trust the host and you still have to ask which account.

### 5.4 EDR goes blind

Enterprise endpoint detection and response tooling monitors processes on the endpoint. A sealed VM is opaque to it. The isolation that protects the user's machine also **removes the enterprise's visibility** into what the agent did. Security improved and observability regressed at the same time, from the same change.

> Lesson: isolation has a monitoring cost. Ask who else was watching before you sealed the box.

---

## 6. Principles

```
1. CONTAIN AT THE ENVIRONMENT LAYER FIRST, STEER AT THE MODEL LAYER SECOND
   → Deterministic boundaries catch what probabilistic defenses miss.
   → Classifiers reduce likelihood; sandboxes cap damage.

2. MATCH ISOLATION STRENGTH TO USER TECHNICAL CAPACITY
   → A developer who reads bash needs a different design than a
     knowledge worker who cannot. One-size-fits-all under-protects
     someone.

3. BE WARY OF WHAT YOU BUILT YOURSELF
   → "The software you build yourself is often the weakest."
   → Hypervisors, seccomp, and gVisor held. The custom proxies and
     custom allowlist logic are where the failures happened.
```

Principle 3 deserves emphasis because it cuts against engineering instinct. A bespoke proxy feels *more* secure — it is tailored to your exact threat model. But it has one team's review hours behind it, while a hypervisor has a decade of adversarial attention from the entire industry.

---

## 7. Open Problems

The article closes without claiming the problem is solved:

- **Persistent memory poisoning** — an injection that survives across sessions, contaminating future runs
- **Multi-agent trust escalation** — how privilege moves when agents delegate to agents
- **Cross-platform agent identity** — no standard for who an agent *is* across systems
- **Live monitoring inside isolated environments** — the direct descendant of the EDR blindness problem

---

## 8. Connections to Other Articles

- **[Claude Code Sandboxing](../claude-code-sandboxing/)** — the deep dive on the Seatbelt/bubblewrap layer summarized here. Read that for the mechanism, this one for where the mechanism sits in the overall strategy. The 84% prompt reduction is the shared anchor point between the two.
- **[Claude Code Auto Mode](../claude-code-auto-mode/)** — the ~83% catch rate and 0.4% false-positive rate cited in this article are the operating characteristics of that feature.
- **[Building Effective Agents](../building-effective-agents/)** — the autonomy spectrum discussed there is exactly what containment is buying. More containment means you can afford to sit further toward the autonomous end.
- **[Effective Harnesses for Long-Running Agents](../effective-harnesses-for-long-running-agents/)** — long-horizon autonomy raises the blast radius most sharply, since there is the least human observation per unit of action.
- **[Code Execution with MCP](../code-execution-with-mcp/)** — that pattern gives the agent an execution environment; this article is the argument for what that environment must be bounded by.
- **[A Postmortem of Three Recent Issues](../a-postmortem-of-three-recent-issues/)** — same intellectual honesty in publishing what broke; the "what we got wrong" sections are the highest-value part of both.

---

## Key Takeaways

```
1. CONTAIN THE BLAST RADIUS, DON'T JUST LOWER THE ODDS
   → Likelihood falls with training; blast radius grows with capability
   → Ship when the worst case is capped, not when risk hits zero

2. 3 RISKS × 3 LAYERS
   → misuse / misbehavior / attackers
   → environment (hard) / model (soft) / external content (hygiene)

3. THREE PATTERNS, ONE VARIABLE: THE USER
   → claude.ai: ephemeral gVisor container
   → Claude Code: OS sandbox + developer judgment
   → Claude Cowork: sealed VM, agent loop on the host

4. 93% APPROVAL = NO OVERSIGHT
   → Human-in-the-loop degrades to rubber-stamping
   → OS sandbox cut prompts ~84% by not asking about safe things

5. THE WEAKEST LAYER IS THE ONE YOU BUILT YOURSELF
   → Hypervisors, seccomp, gVisor held
   → Custom proxies and allowlists failed

6. ALLOWLISTS ARE NOT EGRESS CONTROLS
   → Approved domain ≠ approved destination account

7. ISOLATION HAS A MONITORING COST
   → Sealing the box also blinds whoever was watching it
```
