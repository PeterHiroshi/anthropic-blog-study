# Claude Code auto mode: a safer way to skip permissions

> Source: https://www.anthropic.com/engineering/claude-code-auto-mode
> Published: March 25, 2026 | Anthropic Engineering
> Author: John Hughes

---

## Overview Mind Map

```
Claude Code Auto Mode
├── The Problem Auto Mode Solves
│   ├── Users accept manual prompts 93% of the time
│   ├── "Approval fatigue" leads to --dangerously-skip-permissions
│   └── Skip-all flag is the unsafe escape valve
├── Threat Model (4 risks)
│   ├── Overeager behavior
│   ├── Honest mistakes
│   ├── Prompt injection
│   └── Model misalignment
├── Three-Tier Decision Structure
│   ├── Tier 1: Built-in allowlist (safe tools, never block)
│   ├── Tier 2: In-project file ops (audit via VCS)
│   └── Tier 3: Transcript classifier (everything else)
├── Two-Layer Defense
│   ├── Input: server-side prompt-injection probe on tool outputs
│   └── Output: transcript classifier before each action
├── Classifier: Two-Stage
│   ├── Stage 1: fast single-token filter (8.5% FPR)
│   └── Stage 2: chain-of-thought only when flagged (0.4% FPR)
├── Key Design Choices
│   ├── Strip assistant text (prevent self-persuasion)
│   ├── Strip tool results (prompt-injection defense)
│   ├── Deny-and-continue (stop after 3 in-a-row or 20 total)
│   └── Multi-agent: classifier runs at delegation + return
├── Block Rule Categories (20+ defaults)
│   ├── Destroy/exfiltrate
│   ├── Degrade security
│   ├── Cross trust boundaries
│   └── Bypass review / affect others
└── Caveats
    ├── 17% FNR on real overeager actions
    └── Not a substitute for human review on high-stakes infra
```

---

## 1. The Problem Auto Mode Solves

Claude Code's default UX asks the user to approve each risky action. But:

- Users click "yes" **93%** of the time.
- That fatigue pushes them to `--dangerously-skip-permissions`, which has **zero** checks.

Auto mode is the **middle ground**: most routine actions go through without a prompt, while a model-based classifier catches the dangerous ones before they run.

---

## 2. The Threat Model

Auto mode is designed against four distinct risks, not one:

| Risk | Example |
|------|---------|
| **Overeager behavior** | Claude over-refactoring when you asked for a small fix |
| **Honest mistakes** | `rm -rf /` from a mistyped path variable |
| **Prompt injection** | Malicious content in a fetched webpage steering Claude |
| **Model misalignment** | Rare, but real; defense in depth matters |

Framing matters: a classifier that only catches prompt injection would miss the most common failure (overeager behavior). Auto mode targets all four.

---

## 3. Three-Tier Decision Structure

```
┌────────────────────────────────────────────────┐
│ Tier 1: Built-in Allowlist                     │
│   file reads, searches, navigation             │
│   → Always allowed, no classification          │
├────────────────────────────────────────────────┤
│ Tier 2: In-Project File Operations             │
│   edits under the project tree                 │
│   → Allowed; audit trail lives in VCS          │
├────────────────────────────────────────────────┤
│ Tier 3: Everything Else                        │
│   shell commands, network, cross-project IO    │
│   → Transcript classifier gates each call      │
└────────────────────────────────────────────────┘
```

Design intent: route **most** actions through Tier 1 or 2 (cheap, fast, unambiguous), concentrate model inference on the tail that actually matters.

Critically, Tier 1 **refuses to grant blanket code-execution permission** — shell/eval is never unconditionally trusted.

---

## 4. Two-Layer Defense

**Input layer (before content reaches Claude):**
- Server-side prompt-injection probe screens tool outputs (web pages, file contents, etc.).
- The goal: block known attack patterns before they corrupt Claude's context.

**Output layer (before actions leave Claude):**
- Transcript classifier evaluates each proposed tool call.
- Decisions are deny-and-continue — Claude recovers and tries safer alternatives.

This is defense in depth: even if injection slips past the input layer, the output classifier can still refuse the dangerous action.

---

## 5. The Classifier: Two Stages

Single-stage classification forces a bad cost/accuracy tradeoff. Two stages give you aggressive filtering cheaply, and careful reasoning only when warranted.

| Stage | What | Cost | Result |
|-------|------|------|--------|
| Stage 1 | Single-token filter ("safe" vs "flag") | Cheap (cacheable) | 8.5% false positive |
| Stage 2 | Chain-of-thought reasoning on flagged only | Expensive | Down to **0.4%** false positive |

### Performance Numbers

| Dataset | FPR | FNR |
|---------|-----|-----|
| Real traffic (n=10,000) | 8.5% → 0.4% | — |
| Real overeager actions (n=52) | — | 6.6% → **17%** |
| Synthetic exfiltration (n=1,000) | — | 1.8% → **5.7%** |

The 17% FNR on real overeager actions is the uncomfortable number. It's not "the classifier missed the risk" — it's "the classifier correctly saw the risk, but incorrectly judged that user consent covered it." That's a judgment call that a human would sometimes get wrong too.

---

## 6. Key Design Decisions

### Strip assistant messages from the classifier input

Why: **to prevent the agent from talking the classifier into a bad call**. If Claude's rationale were visible, a persuasive explanation could bias the judgment.

### Strip tool output from the classifier input

Why: **prompt-injection defense**. Tool outputs can contain adversarial content trying to influence whatever reads them. The classifier doesn't need the raw output to judge real-world impact — it just needs the proposed action.

### Deny-and-continue, not deny-and-halt

When blocked, Claude is told "this action was denied" and tries something safer. This preserves flow. Safety rails: stop after **3 consecutive** denials or **20 total** denials.

### Multi-agent coverage

- The classifier runs at **subagent delegation** (catches unauthorized task expansion).
- A **return-path classifier** warns when a subagent might have been compromised mid-run before its results propagate up.

---

## 7. Block Rule Categories (20+ defaults)

| Category | Examples |
|----------|----------|
| **Destroy / exfiltrate** | `git push --force` to main, mass deletion, external data transmission |
| **Degrade security** | Disabling logging, installing persistence, weakening auth |
| **Cross trust boundaries** | Running external code, scanning credential stores |
| **Bypass review / affect others** | Direct main pushes, production deploys, modifying shared resources |

Users can tune:
- **Trust boundaries** (which domains/buckets/services are "inside").
- **Allow exceptions** (carve-outs from block rules).
- **Additional infrastructure specs**.

Starter command: `claude auto-mode defaults` dumps the full baseline config.

---

## 8. Honest Caveats

The article is refreshingly candid:

> "Whether 17% is acceptable depends on what you're comparing against."

- Compared to `--dangerously-skip-permissions`: auto mode is dramatically safer.
- Compared to careful human review on high-stakes infrastructure: it's not a replacement.

This is the right framing. Auto mode is a **tool**, not a **policy**. The policy — "which repos can Claude touch unattended?" — is still up to the user.

---

## 9. Connections to Other Articles

- **[Claude Code Sandboxing](../claude-code-sandboxing/)** — the other half of the safety story; sandbox isolates consequences, auto mode classifies intent.
- **[Claude Code Best Practices](../claude-code-best-practices/)** — the workflow that auto mode is meant to accelerate without surrendering safety.
- **[Scaling Managed Agents](../managed-agents/)** — the brain/hands decoupling makes sandbox-level isolation cheaper, which complements auto mode's intent classification.
- **[Effective Harnesses for Long-Running Agents](../effective-harnesses-for-long-running-agents/)** — long-running harnesses especially benefit, since human confirmation on every step would destroy throughput.

---

## Key Takeaways

```
1. 93% OF MANUAL PROMPTS ARE "YES"
   → Approval fatigue → dangerous skip-all flag.

2. THIRD WAY: MODEL-BASED CLASSIFIER
   → Not all-yes, not all-ask; evaluate intent per action.

3. THREE TIERS
   → Built-in safe / in-project files / classifier-gated.

4. TWO STAGES
   → Cheap single-token filter → CoT reasoning only when flagged.
   → 8.5% FPR → 0.4% FPR.

5. STRIP CLAUDE'S VOICE FROM THE CLASSIFIER
   → Prevent self-persuasion; prevent prompt injection.

6. DENY-AND-CONTINUE
   → Don't halt; nudge to safer path; stop after 3/20.

7. 17% FNR ON OVEREAGER ACTIONS
   → Not a replacement for human review of high-stakes infra.

8. MULTI-AGENT: CLASSIFY AT DELEGATION + RETURN
   → Catch task creep and mid-run compromise.
```
