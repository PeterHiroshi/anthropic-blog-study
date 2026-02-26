# Effective Harnesses for Long-Running Agents

> Source: https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents
> Published: 2025 | Anthropic Engineering

---

## Overview Mind Map

```
Effective Harnesses for Long-Running Agents
├── Core Problem
│   └── Agents lose memory between context windows
│       → Complex multi-day tasks become impossible
│
├── Two-Part Solution Architecture
│   ├── Initializer Agent (session 1 only)
│   │   ├── Creates init.sh (environment setup)
│   │   ├── Creates claude-progress.txt (state tracking)
│   │   └── Establishes git repository with baseline
│   │
│   └── Coding Agent (subsequent sessions)
│       ├── Reads progress files + git logs
│       ├── Works on ONE feature per session
│       ├── Commits with descriptive messages
│       └── Updates progress documentation
│
├── Key Environment Strategies
│   ├── Feature list in JSON (200+ features, all "failing")
│   ├── Incremental progress (one feature per session)
│   └── Browser automation testing (Puppeteer MCP)
│
└── Session Startup Pattern
    ├── Verify working directory
    ├── Review progress/git logs
    ├── Read feature list
    ├── Run basic functionality tests
    └── Implement single feature
```

---

## 1. The Core Problem

**What it is:** AI agents operate within a single context window. When the window fills (or the session ends), all in-memory state is lost. For complex multi-day projects, this creates an enormous coordination challenge.

```
Multi-Day Project Without a Harness:

Session 1: Agent works on feature A
           [context window fills]
           [session ends]

Session 2: Agent starts fresh
           "I don't know what was done in session 1"
           Agent might redo work, break things, or declare victory prematurely
```

**Why agents fail without structure:**

| Failure Mode | What Happens |
|--------------|-------------|
| Premature victory | Agent thinks it's done when it's not |
| Undocumented code | Changes made without explaining what or why |
| Lost context | Can't run the application because env setup was forgotten |
| Incomplete features | Marks feature done without rigorous testing |

---

## 2. The Two-Part Solution

The harness pattern separates concerns into two distinct roles:

```
┌─────────────────────────────────────────────────────────────┐
│                    INITIALIZER AGENT                        │
│                    (Session 1 only)                         │
│                                                             │
│  Creates the "infrastructure" the coding agent needs:       │
│  1. init.sh       → reproducible environment setup          │
│  2. claude-progress.txt → shared memory across sessions     │
│  3. Initial git commit → clean baseline state               │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼ hands off to
┌─────────────────────────────────────────────────────────────┐
│                    CODING AGENT                             │
│                    (Sessions 2, 3, 4, ...)                  │
│                                                             │
│  Each session:                                              │
│  1. Run init.sh  → restore exact same environment           │
│  2. Read progress + git log → understand current state      │
│  3. Read feature list → pick next failing feature           │
│  4. Implement ONE feature                                   │
│  5. Test rigorously                                         │
│  6. git commit with descriptive message                     │
│  7. Update progress file                                    │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. Initializer Agent: What It Creates

### init.sh — Reproducible Environment

The initializer agent writes a shell script that restores the exact development environment from scratch. This ensures the coding agent can always get a running application, regardless of state.

```bash
# Example init.sh structure
#!/bin/bash

# Install dependencies
npm install

# Set up environment variables
export DATABASE_URL="postgresql://localhost/myapp"
export API_KEY="dev-key"

# Start required services
docker compose up -d

# Run database migrations
npm run db:migrate

# Verify application starts
npm run dev &
sleep 3
curl -f http://localhost:3000/health || exit 1
echo "Environment ready!"
```

### claude-progress.txt — Shared Memory

This file is the "memory" that persists across context windows:

```
# Project: MyWebApp
# Last Updated: Session 4

## Completed Features
- [x] User registration (Session 2)
- [x] Email verification (Session 2)
- [x] User login with JWT (Session 3)
- [x] Password reset flow (Session 4)

## In Progress
- [ ] User profile editing

## Pending
- [ ] Dashboard view
- [ ] Admin panel
- [ ] Export to CSV

## Key Architecture Decisions
- Using PostgreSQL for main DB (not SQLite)
- JWT stored in httpOnly cookies
- Email via SendGrid, not SMTP

## Known Issues
- Profile image upload needs S3 bucket config in production
```

### Initial Git Repository

The initializer creates the first commit with a clean, documented baseline. All subsequent work is layered on top, creating a clear audit trail.

---

## 4. Feature List: The Anti-Victory Mechanism

**The problem:** Agents tend to declare victory when things "look okay." Without explicit failure states, they stop working too soon.

**The solution:** A JSON feature list where every feature starts as "failing."

```json
{
  "features": [
    {
      "id": "user-registration",
      "description": "User can sign up with email/password",
      "status": "failing",
      "test": "POST /api/users returns 201 with valid data"
    },
    {
      "id": "email-verification",
      "description": "New users receive verification email",
      "status": "failing",
      "test": "Email sent within 60s of registration"
    }
  ]
}
```

**Critical rule quoted from article:**
> "It is unacceptable to remove or edit tests because this could lead to missing or buggy functionality."

The agent CANNOT mark a feature as complete without passing its associated test. This prevents the "looks done" failure mode.

---

## 5. Session Startup Pattern

Every coding agent session follows this exact sequence:

```
Session Start
     │
     ▼
1. Verify working directory
   └── Am I in the right project folder?
     │
     ▼
2. Review progress + git logs
   └── What was completed? What was the last commit?
     │
     ▼
3. Read feature list
   └── Which features are still "failing"?
     │
     ▼
4. Run basic functionality tests
   └── Is the application in a working state?
     │
     ▼
5. Pick ONE failing feature
   └── Choose next feature to implement
     │
     ▼
6. Implement + test rigorously
   └── Don't mark done until tests pass!
     │
     ▼
7. git commit + update progress file
   └── Descriptive commit message. Clear progress update.
```

**Why "ONE feature per session"?**
- Keeps each session focused and bounded
- Each commit represents a clean, shippable state
- Prevents "half-finished" features that break the codebase
- Makes progress measurable and verifiable

---

## 6. Testing: Browser Automation

The article specifically highlights **Puppeteer MCP** (browser automation) as a key testing tool:

```
Traditional Unit Testing:
Agent writes tests → tests pass → "Done!"
Problem: Tests may not reflect real user behavior

Browser Automation Testing:
Agent launches browser → navigates like a user →
verifies UI behavior → catches visual/UX bugs

"Testing as actual users would test"
→ Dramatically improved bug detection
```

**Why this matters for agents:**
- Agents can verify their own work end-to-end
- Catches integration bugs that unit tests miss
- Provides concrete pass/fail signal without human review

---

## 7. Common Failure Modes and Solutions

| Failure Mode | Without Harness | With Harness |
|--------------|-----------------|--------------|
| Premature victory | Agent declares done with incomplete features | Feature list with "failing" states prevents this |
| Undocumented code | Changes made without explanations | Every commit requires descriptive message |
| Env setup lost | Agent can't remember how to run app | `init.sh` always restores known-good state |
| Marks features incomplete | Tests never run, bugs shipped | Must pass tests before marking complete |

---

## 8. Future Directions

The article raises open questions:

- **Single general agent vs. specialized agents?** Does one agent that does everything outperform a team of specialists (test agent, QA agent, code agent)?
- **Domain applicability:** Pattern optimized for web development. How does it apply to scientific research or financial modeling?
- **Agent specialization:** Future work may explore dedicated testing agents, QA agents, and their coordination patterns.

---

## Quick Reference

**The two-file system:**
```
init.sh               → "How do I run this project?"
claude-progress.txt   → "What has been done? What's next?"
```

**The core session loop:**
```
run init.sh → read progress → pick ONE feature → implement → test → commit → update progress
```

**Anti-failure mechanisms:**
1. Feature list: all start as "failing", can't be removed
2. One feature per session: bounded, verifiable progress
3. Commit + document: every change is traceable
4. Browser testing: verifies real behavior, not just code
