# Equipping Agents for the Real World with Agent Skills

> Source: https://claude.com/blog/equipping-agents-for-the-real-world-with-agent-skills
> Published: October 16, 2025 | Claude Blog

---

## Overview Mind Map

```
Agent Skills
├── What is a Skill?
│   ├── A directory with SKILL.md
│   └── Contains instructions, scripts, resources
│
├── How Claude Uses Skills
│   ├── Automatic invocation (matches description)
│   └── Manual invocation (/skill-name)
│
├── Progressive Disclosure Architecture
│   ├── Level 1: metadata (always loaded, minimal)
│   ├── Level 2: SKILL.md content (loaded when relevant)
│   └── Level 3: linked files/scripts (loaded on demand)
│
├── Skill Types
│   ├── Reference skills (knowledge/conventions)
│   └── Task skills (step-by-step workflows)
│
├── Advanced Patterns
│   ├── Arguments ($ARGUMENTS, $1, $2)
│   ├── Dynamic context injection (!`command`)
│   └── Subagent execution (context: fork)
│
└── Open Standard
    └── Works across Cursor, GitHub, VS Code, etc.
```

---

## 1. What Is an Agent Skill?

Think of it like **onboarding a new employee**:

```
New hire onboarding guide contains:
  → Company conventions and standards
  → Step-by-step process for common tasks
  → Templates and examples
  → Scripts and automation tools

Agent Skill is the same thing:
  → Domain knowledge Claude needs for this task type
  → Instructions for how to approach the task
  → Templates, examples, scripts
  → Loaded only when relevant
```

**Concrete definition:** A Skill is a **directory** with a required `SKILL.md` file:

```
my-skill/
├── SKILL.md           ← Required: main instructions
├── template.md        ← Optional: template for output
├── examples/
│   └── sample.md      ← Optional: example outputs
└── scripts/
    └── validate.sh    ← Optional: executable scripts
```

---

## 2. The SKILL.md File

Every skill starts here. It has two parts:

```
───── SKILL.md ─────────────────────────────────
---                          ← YAML frontmatter start
name: my-skill
description: What this skill does and when to use it
disable-model-invocation: false
allowed-tools: Read, Grep
---                          ← YAML frontmatter end

# Your Skill Instructions

Write your instructions here in plain markdown.
Claude reads this when the skill is invoked.

Include:
- What to do
- How to approach it
- What NOT to do
- Examples of good output
─────────────────────────────────────────────────
```

---

## 3. Progressive Disclosure: The Key Design Principle

Skills are designed to be **context-efficient**. They use a 3-level loading strategy:

```
Level 1: METADATA (always in context)
  ┌────────────────────────────────────────┐
  │ name: "explain-code"                   │ ← ~50 tokens
  │ description: "Explains code with..."   │   (always loaded)
  └────────────────────────────────────────┘

Level 2: FULL SKILL.MD (loaded when relevant)
  ┌────────────────────────────────────────┐
  │ # When explaining code, always:        │
  │ 1. Start with an analogy               │ ← ~500 tokens
  │ 2. Draw a diagram                      │   (loaded only
  │ 3. Walk through step by step           │    when task matches)
  │ 4. Highlight gotchas                   │
  └────────────────────────────────────────┘

Level 3: REFERENCED FILES (loaded on demand)
  ┌────────────────────────────────────────┐
  │ examples/react-hooks-explanation.md   │ ← ~2000 tokens
  │ examples/async-await-explanation.md   │   (loaded only if
  │ references/advanced-patterns.md       │    explicitly needed)
  └────────────────────────────────────────┘
```

**Why this matters:** Large reference docs never overwhelm context. A skill can contain a full API spec (50K tokens) that only loads when needed.

---

## 4. Invocation Modes

### Automatic Invocation (Claude decides)

```
Claude reads the description field and asks:
"Is this skill relevant to what the user is asking?"

User: "How does this authentication code work?"
Claude: Checks skills... "explain-code" description says:
        "Use when explaining how code works"
        → Match! Load and apply the skill automatically.
```

### Manual Invocation (User decides)

```
User types: /explain-code src/auth/login.ts
             ↑ skill name  ↑ argument

Claude: Load explain-code skill, apply to src/auth/login.ts
```

### Control Which Mode Is Allowed

```yaml
# Default: Both modes work
---
name: my-skill
description: What this skill does
---

# User-only: Prevents Claude from auto-applying
---
name: deploy-prod
description: Deploy to production
disable-model-invocation: true   ← Claude won't auto-apply
---

# Agent-only: Hides from /menu
---
name: internal-helper
user-invocable: false             ← Doesn't appear in / menu
---
```

---

## 5. Complete Frontmatter Reference

```yaml
---
# Display name (optional - uses directory name if omitted)
name: my-skill

# What it does (strongly recommended)
description: Explains code with visual diagrams and analogies.
             Use when explaining how code works.

# Hint shown during autocomplete (optional)
argument-hint: "[filename] [detail-level]"

# Prevent Claude from auto-loading (optional)
disable-model-invocation: true

# Hide from / menu (optional)
user-invocable: false

# Tools usable without permission prompts (optional)
allowed-tools: Read, Grep, Bash(git *)

# Model override (optional)
model: claude-opus-4-6

# Run in isolated subagent context (optional)
context: fork

# Which subagent type to use (optional, requires context: fork)
agent: Explore    # or: Plan, general-purpose, or custom agent name

# Skill-scoped hooks (optional)
hooks:
  PostToolUse:
    - matcher: "Write"
      hooks:
        - type: command
          command: "git add -A"
---
```

---

## 6. Skill Locations and Priority

```
Priority order (higher = wins when same name):
  Enterprise  >  Personal  >  Project  >  Plugin

Where skills live:

  Enterprise:   Managed settings (all org users)

  Personal:     ~/.claude/skills/<skill-name>/SKILL.md
                (all your projects, everywhere)

  Project:      .claude/skills/<skill-name>/SKILL.md
                (this project only, great for team sharing)

  Plugin:       <plugin>/skills/<skill-name>/SKILL.md
                (uses plugin-name:skill-name namespace)

Note: .claude/commands/review.md and
      .claude/skills/review/SKILL.md
      are equivalent — both create /review
```

---

## 7. Passing Arguments to Skills

### Basic Arguments

```yaml
---
name: fix-issue
description: Fix a GitHub issue by number
disable-model-invocation: true
---

Fix GitHub issue $ARGUMENTS following our coding standards.

Steps:
1. Read the issue: gh issue view $ARGUMENTS
2. Understand requirements
3. Implement the fix
4. Write tests
5. Create a commit
```

Usage: `/fix-issue 123` → Claude sees "Fix GitHub issue 123..."

### Positional Arguments

```yaml
---
name: migrate-component
description: Migrate a UI component between frameworks
---

Migrate the $ARGUMENTS[0] component from $ARGUMENTS[1] to $ARGUMENTS[2].
Preserve all existing behavior and test coverage.
```

Usage: `/migrate-component SearchBar React Vue`
- `$ARGUMENTS[0]` = SearchBar
- `$ARGUMENTS[1]` = React
- `$ARGUMENTS[2]` = Vue

### Available Variables

| Variable | Meaning |
|----------|---------|
| `$ARGUMENTS` | All arguments as a string |
| `$ARGUMENTS[N]` | Specific argument by index (0-based) |
| `$N` | Shorthand for `$ARGUMENTS[N]` |
| `${CLAUDE_SESSION_ID}` | Current session ID |

---

## 8. Advanced: Dynamic Context Injection

The `` !`command` `` syntax runs shell commands **before** the skill is sent to Claude:

```yaml
---
name: pr-summary
description: Summarize a pull request
context: fork
agent: Explore
allowed-tools: Bash(gh *)
---

## Pull request context (fetched live):
PR diff:
!`gh pr diff`

PR comments:
!`gh pr view --comments`

Changed files:
!`gh pr diff --name-only`

## Your task
Review the above PR and provide:
1. Summary of what changed and why
2. Potential issues or risks
3. Testing recommendations
```

**What happens when this skill runs:**
```
1. !`gh pr diff` executes immediately
   → Output replaces the placeholder
2. !`gh pr view --comments` executes
   → Output replaces the placeholder
3. !`gh pr diff --name-only` executes
   → Output replaces the placeholder
4. Claude receives the full, pre-filled prompt
   (with actual PR data, not commands)
```

This is **preprocessing** — Claude never sees the commands, only the results.

**Tip:** Include `"ultrathink"` anywhere in skill content to enable extended thinking for that skill.

---

## 9. Advanced: Subagent Skills

Add `context: fork` to run the skill in an isolated subagent:

```yaml
---
name: deep-research
description: Research a topic thoroughly and return a detailed report
context: fork
agent: Explore
---

Research $ARGUMENTS thoroughly:

1. Find all relevant files using Glob and Grep
2. Read and analyze the code/documentation
3. Cross-reference related components
4. Synthesize findings with specific file references
5. Return a structured report
```

**How subagent skills differ from regular skills:**

```
Regular Skill:
  User conversation → [Skill loads into context] → Claude acts
  (Claude has access to full conversation history)

Subagent Skill (context: fork):
  User conversation → [New isolated context] → Subagent acts
                                                      ↓
                         [Result summarized and returned to main conversation]
  (Subagent has NO access to conversation history — fresh context)
```

**When to use subagent skills:**
- Task needs lots of file exploration (would pollute main context)
- Research tasks where intermediate results don't need to be visible
- Long multi-step processes that should be encapsulated

---

## 10. Skill Types: Reference vs. Task

```
REFERENCE SKILL — adds knowledge, runs inline
  Use for: Conventions, patterns, domain knowledge, style guides

  Example:
  ---
  name: api-conventions
  description: API design patterns for this codebase
  ---
  When writing API endpoints:
  - Use RESTful naming: GET /resources, POST /resources, PATCH /resources/:id
  - Return consistent error format: {error: string, code: string, details?: object}
  - Include request validation in controllers, not services
  - All dates in ISO 8601 format: 2024-01-15T10:30:00Z

TASK SKILL — step-by-step workflow, often invoked directly
  Use for: Deployments, releases, code generation, structured processes

  Example:
  ---
  name: release
  description: Create a new release
  context: fork
  disable-model-invocation: true
  ---
  Create a release for version $ARGUMENTS:
  1. Update CHANGELOG.md with unreleased changes
  2. Update version in package.json and package-lock.json
  3. Run tests: npm test
  4. Create git tag: git tag v$ARGUMENTS
  5. Push tag: git push origin v$ARGUMENTS
  6. Create GitHub release: gh release create v$ARGUMENTS
```

---

## 11. Security Considerations

```
⚠️  IMPORTANT SECURITY RULES:

1. ONLY install skills from trusted sources
   → Skills can execute bash commands
   → Skills can run arbitrary scripts

2. AUDIT bundled scripts before use
   → Check Python/shell scripts for malicious behavior
   → Review external network connections

3. REVIEW external API connections
   → What data is being sent out?
   → Is authentication handled securely?

4. Use allowed-tools to restrict permissions
   → Don't give a skill more access than it needs
   allowed-tools: Read  ← read-only skill
   allowed-tools: Bash(git *)  ← only git commands
```

---

## 12. The Open Standard

Agent Skills aren't Anthropic-only:

```
Open standard: https://agentskills.io
GitHub: https://github.com/agentskills/agentskills

Platforms that support it:
  Cursor         GitHub         VS Code
  Gemini CLI     OpenCode       OpenHands
  Roo Code       Letta          Goose (Block)
  Junie          Amp            Spring AI
  Databricks     Factory        And more...

What this means:
  Build once → Deploy to any compatible agent platform
  Team skills work across all your tools
  Community can share and reuse skills
```

---

## Quick Start: Create Your First Skill in 3 Minutes

```bash
# Step 1: Create skill directory
mkdir -p ~/.claude/skills/code-explainer

# Step 2: Create SKILL.md
cat > ~/.claude/skills/code-explainer/SKILL.md << 'EOF'
---
name: code-explainer
description: Explains code with analogies and ASCII diagrams. Use when someone asks "how does this work?" or asks to explain code.
---

When explaining code, follow this structure:

1. **One-sentence analogy**: Compare to something from everyday life
2. **ASCII diagram**: Show the flow or structure visually
3. **Step-by-step walkthrough**: Explain what each part does
4. **Common gotcha**: What do beginners usually misunderstand?

Keep the tone conversational. Assume no prior knowledge.
EOF

# Step 3: Test it
# Option A: Ask Claude naturally
# "How does this useEffect hook work?"
# (Claude auto-detects the match)

# Option B: Invoke directly
# /code-explainer src/hooks/useAuth.ts
```

---

## Summary: When to Build a Skill

```
BUILD A SKILL WHEN:
  ✓ You repeat the same type of task regularly
  ✓ The task requires specialized domain knowledge
  ✓ You want consistent output format/approach
  ✓ The workflow needs to be shared with your team
  ✓ You want to automate a multi-step process
  ✓ Different projects need different conventions

DON'T BUILD A SKILL WHEN:
  ✗ The task is one-time or very rare
  ✗ The task is simple enough without guidance
  ✗ Claude already handles it well by default
```
