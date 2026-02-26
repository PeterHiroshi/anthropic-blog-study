# Best Practices for Claude Code

> Source: https://www.anthropic.com/engineering/claude-code-best-practices → https://code.claude.com/docs/en/best-practices
> Published: 2025 | Anthropic / Claude Code Docs

---

## Overview Mind Map

```
Best Practices for Claude Code
├── Core Constraint: Context Window Fills Fast
│   └── Performance degrades as context fills
│
├── Give Claude a Way to Verify Its Work
│   ├── Run tests, compare screenshots, validate outputs
│   └── Provide verification criteria in prompts
│
├── Explore → Plan → Code → Commit Workflow
│   ├── Explore: Plan Mode, read files
│   ├── Plan: detailed implementation plan
│   ├── Implement: Normal Mode + verification
│   └── Commit: descriptive message + PR
│
├── Provide Specific Context
│   ├── Scope tasks precisely
│   ├── Reference files with @
│   └── Paste screenshots/images directly
│
├── Configure Your Environment
│   ├── CLAUDE.md — persistent project context
│   ├── Permissions — allowlists and sandboxing
│   ├── Skills — domain knowledge files
│   ├── Hooks — deterministic automation
│   └── MCP servers — external tool integration
│
├── Session Management
│   ├── Esc / /rewind for checkpoints
│   ├── /clear between unrelated tasks
│   ├── Subagents for investigation
│   └── --continue and --resume flags
│
└── Scale and Automate
    ├── Headless mode: claude -p "..."
    ├── Parallel sessions
    └── Fan-out across files
```

---

## 1. The Fundamental Constraint: Context Window

**Everything in Claude Code best practices flows from one insight:**

> "Claude's context window fills up fast, and performance degrades as it fills."

```
Context Window Contents:
├── Every message in your conversation
├── Every file Claude has read
├── Every command output
└── Tool definitions

Performance Degradation Pattern:
High  │ ████████
      │          ████████
      │                   ████████
      │                            ████████
Low   └────────────────────────────────────
      Empty              Full
      Context           Context
```

**What this means practically:**
- Context is your most important resource
- Every token must earn its place
- Managing context = managing performance
- Tools like `/clear`, subagents, and compaction exist to manage context

---

## 2. Give Claude a Way to Verify Its Work

**This is the single highest-leverage practice.**

Without verification:
```
You: "Write a validateEmail function"
Claude: [writes function that looks correct]
You: [reviews it manually]
→ You become the only feedback loop
→ Every mistake requires your attention
```

With verification:
```
You: "Write a validateEmail function. Test cases:
     - user@example.com → true
     - invalid → false
     - user@.com → false
     Run the tests after implementing."
Claude: [writes function]
Claude: [runs tests]
Claude: "Test 2 failed (expected false for 'user@.com'). Fixing..."
Claude: [fixes function]
Claude: "All tests pass."
→ Claude self-corrects before you see it
```

**Verification methods by task type:**

| Task Type | Verification Method |
|-----------|---------------------|
| Code functions | Unit tests with specific assertions |
| UI changes | Screenshot + comparison with reference |
| Bug fixes | Reproduce the bug → fix → confirm fixed |
| Performance | Before/after benchmarks |
| API changes | Integration test hitting real endpoints |

**Before/After prompts:**

| Before | After |
|--------|-------|
| "implement a function that validates email addresses" | "write a validateEmail function. test cases: [examples]. run the tests after implementing" |
| "make the dashboard look better" | "[paste screenshot] implement this design. take a screenshot and compare to original. list differences and fix them" |
| "the build is failing" | "the build fails with [error]. fix it and verify the build succeeds. address root cause, don't suppress the error" |

---

## 3. The Four-Phase Workflow

### Phase 1: Explore (Plan Mode)

```
Enter Plan Mode → Claude reads files, answers questions, makes NO changes

Prompt examples:
"read /src/auth and understand how we handle sessions and login"
"also look at how we manage environment variables for secrets"
```

### Phase 2: Plan

```
Still in Plan Mode → Ask for detailed implementation plan

"I want to add Google OAuth. What files need to change?
 What's the session flow? Create a plan."

→ Press Ctrl+G to open plan in text editor for direct editing
```

### Phase 3: Implement (Normal Mode)

```
Switch to Normal Mode → Claude codes against its plan

"implement the OAuth flow from your plan. write tests for the
 callback handler, run the test suite and fix any failures."
```

### Phase 4: Commit

```
"commit with a descriptive message and open a PR"
```

**When to skip planning:**
```
Skip plan for:              Use plan for:
├── Single file fixes       ├── Multi-file changes
├── Typos                   ├── Unfamiliar code
├── Obvious bug fixes       ├── Architectural changes
└── Adding log lines        └── Uncertain approach
```

---

## 4. Provide Specific Context

**The rule:** The more precise your instructions, the fewer corrections you'll need.

### Scoping the Task

| Instead of... | Say... |
|---------------|--------|
| "add tests for foo.py" | "write a test for foo.py covering the edge case where the user is logged out. avoid mocks." |
| "fix the login bug" | "users report that login fails after session timeout. check the auth flow in src/auth/, especially token refresh. write a failing test that reproduces the issue, then fix it" |
| "add a calendar widget" | "look at HotDogWidget.php as an example. follow that pattern to implement a calendar widget letting users select month and paginate year. no new libraries." |

### Rich Content Methods

```
Reference files:     @src/auth/login.ts    (Claude reads before responding)
Paste images:        Drag/drop screenshot  (Claude analyzes visually)
Give URLs:           link to API docs      (Claude fetches)
Pipe data:           cat error.log | claude  (send stdin)
Let Claude fetch:    "run git log to understand recent changes"
```

---

## 5. CLAUDE.md: Persistent Project Context

**What it is:** A markdown file Claude reads at the start of every conversation.

```bash
/init   # Generate starter CLAUDE.md based on your project
```

**Example CLAUDE.md:**
```markdown
# Code style
- Use ES modules (import/export) syntax, not CommonJS (require)
- Destructure imports when possible (eg. import { foo } from 'bar')

# Workflow
- Be sure to typecheck when you're done making a series of code changes
- Prefer running single tests, and not the whole test suite, for performance
```

**What to include vs. exclude:**

| Include | Exclude |
|---------|---------|
| Bash commands Claude can't guess | Anything Claude infers from code |
| Code style rules differing from defaults | Standard language conventions |
| Testing instructions and test runners | Detailed API documentation (link instead) |
| Repository etiquette (branch naming, PRs) | Information that changes frequently |
| Architectural decisions specific to project | Long explanations or tutorials |
| Developer environment quirks, env vars | File-by-file codebase descriptions |
| Common gotchas or non-obvious behaviors | Self-evident practices |

**CLAUDE.md location options:**
```
~/.claude/CLAUDE.md          → All Claude sessions (global)
./CLAUDE.md                  → Project-wide, check into git
./CLAUDE.local.md            → Personal overrides, gitignored
./subdirectory/CLAUDE.md     → Loaded when Claude works in that dir
```

**Import syntax:**
```markdown
See @README.md for project overview and @package.json for available commands.

# Additional Instructions
- Git workflow: @docs/git-instructions.md
- Personal overrides: @~/.claude/my-project-instructions.md
```

---

## 6. Skills: Domain Knowledge on Demand

**What it is:** SKILL.md files in `.claude/skills/` that Claude loads when relevant.

Unlike CLAUDE.md (always loaded), skills are loaded only when needed — preserving context for other work.

```markdown
# .claude/skills/api-conventions/SKILL.md
---
name: api-conventions
description: REST API design conventions for our services
---
# API Conventions
- Use kebab-case for URL paths
- Use camelCase for JSON properties
- Always include pagination for list endpoints
- Version APIs in the URL path (/v1/, /v2/)
```

**Workflow skills (invoke with /skill-name):**
```markdown
# .claude/skills/fix-issue/SKILL.md
---
name: fix-issue
description: Fix a GitHub issue
disable-model-invocation: true
---
Analyze and fix the GitHub issue: $ARGUMENTS.

1. Use `gh issue view` to get the issue details
2. Understand the problem described in the issue
3. Search the codebase for relevant files
4. Implement the necessary changes to fix the issue
5. Write and run tests to verify the fix
6. Create a descriptive commit message
7. Push and create a PR
```

Usage: `/fix-issue 1234`

---

## 7. Hooks: Deterministic Automation

**What it is:** Shell scripts that run automatically at specific Claude workflow points.

```
Unlike CLAUDE.md instructions (advisory):
→ Claude might or might not follow them

Hooks are deterministic:
→ They WILL run. Every time. No exceptions.
```

**Example hooks:**
```json
{
  "hooks": {
    "PostToolCall": [
      {
        "matcher": {"tool": "Edit"},
        "command": "eslint --fix ${file}"
      }
    ],
    "PreToolCall": [
      {
        "matcher": {"tool": "Write", "path": "migrations/*"},
        "command": "echo 'BLOCKED: Direct writes to migrations not allowed' && exit 1"
      }
    ]
  }
}
```

Use hooks for: auto-format, lint, type-check, blocking writes to sensitive paths.

---

## 8. Session Management

### Course-Correcting

```
Esc              → Stop Claude mid-action (context preserved, redirect)
Esc + Esc        → Open rewind menu (restore state)
/rewind          → Open rewind menu
"Undo that"      → Have Claude revert changes
/clear           → Reset context for unrelated tasks
```

**When to use /clear:**
```
Use /clear when:
├── Switching to a completely unrelated task
├── After 2+ failed corrections on the same issue
│   → "Context polluted with failed approaches"
│   → Start fresh with better prompt
└── Session has grown long with irrelevant info
```

### Context Management During Long Sessions

```
Auto-compaction: Claude summarizes when approaching context limit

Manual control:
/compact                        → Compact with Claude's judgment
/compact "Focus on API changes" → Compact with your guidance
Esc+Esc → Rewind → "Summarize from here" → Compact partial session
```

**Custom compaction instructions in CLAUDE.md:**
```markdown
When compacting, always preserve:
- The full list of modified files
- Any test commands that were run
- Key architectural decisions made
```

### Subagents for Investigation

```
"Use subagents to investigate how our authentication system
 handles token refresh"

→ Subagent spins up with fresh context
→ Explores codebase, reads many files
→ Returns summary to main context
→ Main context stays clean for implementation

Result: Exploration happens without polluting your main session
```

### Resuming Sessions

```bash
claude --continue      # Resume most recent conversation
claude --resume        # Choose from recent conversations
/rename "oauth-migration"  # Name sessions for easy finding
```

---

## 9. Automation and Scaling

### Headless Mode

```bash
# One-off query
claude -p "Explain what this project does"

# Structured output for scripts
claude -p "List all API endpoints" --output-format json

# Streaming for real-time processing
claude -p "Analyze this log file" --output-format stream-json

# Pipe data in
tail -f app.log | claude -p "Slack me if you see any anomalies"
cat error.log | claude -p "what caused this error?"
```

### Parallel Sessions (Writer/Reviewer Pattern)

```
Session A (Writer)                    Session B (Reviewer)
─────────────────────────────────────────────────────────
"Implement a rate limiter"
    ↓ implements
                                      "Review @src/middleware/rateLimiter.ts
                                       for edge cases, race conditions,
                                       and consistency with patterns"
                                           ↓ returns review
"Here's feedback: [review].
 Address these issues."
```

### Fan-Out Across Many Files

```bash
# Step 1: Generate task list
claude -p "list all Python files needing migration" > files.txt

# Step 2: Process in parallel
for file in $(cat files.txt); do
  claude -p "Migrate $file from React to Vue. Return OK or FAIL." \
    --allowedTools "Edit,Bash(git commit *)" &
done
wait

# Step 3: Check results
```

---

## 10. Common Failure Patterns to Avoid

| Failure Pattern | What Happens | Fix |
|-----------------|-------------|-----|
| Kitchen sink session | One task + unrelated tangent = cluttered context | `/clear` between tasks |
| Repeated corrections | Claude still wrong after 2+ corrections | `/clear` + write better prompt |
| Over-specified CLAUDE.md | Too long → Claude ignores rules | Ruthlessly prune |
| Trust-then-verify gap | Working implementation that misses edge cases | Always provide verification |
| Infinite exploration | "investigate X" → reads hundreds of files | Scope narrowly OR use subagent |

---

## Quick Reference

**The core insight:** Context window is the primary constraint. Everything else serves to manage it.

**Top 5 practices:**
1. Give Claude tests/screenshots to self-verify
2. Explore → Plan → Code (don't jump to implementation)
3. Use `/clear` between unrelated tasks
4. Write a tight CLAUDE.md (only what Claude can't infer)
5. Use subagents for exploration that would pollute context

**Key commands:**
```
/init          → Generate CLAUDE.md
/clear         → Reset context
/rewind        → Restore previous state
/compact       → Compress context
/sandbox       → Enable OS-level isolation
--continue     → Resume last session
--resume       → Choose session to resume
claude -p      → Headless/scripted mode
```
