---
name: hoid:managing-ephemeral-state
description: Use when starting work in a new directory or session - establishes .hoid directory for conversation state, plans, and context that persists within a session but is not committed to git
category: workflow
---

# Managing Ephemeral State

## Overview

**State management for Hoid teams: Session-local storage that persists context between messages but is never committed.**

Your work produces *two types of artifacts*: **permanent** (code, tests, docs shipped to git) and **ephemeral** (session notes, failed approaches, context, plans). This skill establishes a `.hoid/` directory structure for ephemeral artifacts, keeping them separate from production code while enabling rich context preservation during active work.

**Core principle:** Ephemeral state lives in `.hoid/`, is added to `.gitignore`, and is discarded after each session unless explicitly promoted to permanent context.

---

## When to Use

**Create .hoid state at START of:**
- New working session in a directory
- Team collaboration involving subagents
- Complex task requiring context preservation
- Plan execution with checkpoints

**Do NOT create .hoid if:**
- One-shot, single-file edits
- No subagents involved
- No multi-step planning needed

---

## Directory Structure

```
.hoid/
  state/          # Session-local context (ephemeral)
    notes.md      # What we learned (append only)
    plans.md      # Current work plan (mutable)
    decisions.md  # Key architectural choices (append only)

  # Later: context to save permanently
  # context/
  #   [domain]/
  #     architecture.md    # Permanent reference
```

**In .gitignore:**
```
.hoid/
```

---

## Workflow

### 1. Session Start (ONCE per directory)

```bash
mkdir -p .hoid/state
echo ".hoid/" >> .gitignore
```

Create initial files:
- `notes.md` - empty header
- `plans.md` - current work plan
- `decisions.md` - key choices made

**Why separate files:**
- Notes and decisions are append-only (preserve history)
- Plans are mutable (update as you go)
- Easy for agents to READ (context) vs WRITE (append only)

### 2. During Session (EVERY delegation)

**Before delegating to subagent:**
1. Read `.hoid/state/notes.md` and `.hoid/state/decisions.md` for inherited wisdom
2. Include in prompt as "Inherited Wisdom" section
3. Instruct subagent: "Append findings to `.hoid/state/notes.md`, never overwrite"

**After subagent completes:**
1. Verify work
2. Update `.hoid/state/plans.md` with progress (mark completed items)

### 3. Session End (OPTIONAL)

**Promote to permanent:** If session produced reusable context (patterns, conventions, architecture), move to `.hoid/context/`:

```
# Before (ephemeral, session-local)
.hoid/state/notes.md
  - Convention: All components use [pattern]
  - Gotcha: [issue and workaround]

# After (permanent, near code)
.hoid/context/architecture.md
  ## Component Patterns
  All components use [pattern] because [reason]

  ## Known Issues
  [issue]: [workaround]
```

Then `.gitignore` changes:
```
.hoid/state/    # Still ephemeral
# .hoid/context/ removed - now tracked
```

---

## File Purposes

### notes.md (Append Only)

Captures learning during active work. Subagents add entries; orchestrator reads before delegation.

```markdown
# Session Notes

## [Timestamp] Task: task-name
[What we learned]

## [Timestamp] Blocker: blocker-name
[Issue encountered, attempted solution, outcome]
```

**Rules:**
- Append only (never overwrite)
- Include timestamps
- Include enough detail for next agent to understand context

### plans.md (Mutable)

Current work breakdown. Update as tasks complete.

```markdown
# Work Plan

- [x] Task 1: Done
- [ ] Task 2: In progress
- [ ] Task 3: Pending
```

### decisions.md (Append Only)

Architectural and strategic choices. Why did we do this?

```markdown
# Decisions

## [Timestamp] Decision: Use pattern X
Reasoning: [why this choice over alternatives]
Trade-offs: [what we gave up]
```

---

## Skill Overrides

**Skills can override state management:**

In your skill's frontmatter or "When to Use" section:

```markdown
## State Management Override

This skill uses .hoid/state/ differently:
- Plans stored in [custom path]
- Append to [custom file] instead of notes.md
- Discard state after [condition]

Reason: [why standard doesn't fit]
```

**Examples:**
- `test-driven-development`: May create per-test state files
- `systematic-debugging`: May preserve test runs in state/
- `using-git-worktrees`: May have separate state per worktree

---

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| **Overwriting notes.md** | Append only. Always. |
| **Storing notes in multiple places** | Single source of truth: `.hoid/state/notes.md` |
| **Committing .hoid to git** | Add to `.gitignore` immediately |
| **Forgetting to read notes before delegating** | Read BEFORE every prompt, include as context |
| **Never promoting to permanent context** | At end of session, decide: keep or discard |
| **State grows unbounded** | Archive old sessions, move learnings to permanent context |

---

## Real-World Example

### Session Start
```bash
mkdir -p .hoid/state
cat >> .gitignore << EOF
.hoid/
EOF

# Create files
touch .hoid/state/{notes,plans,decisions}.md
```

### During Work
**Orchestrator (before delegating):**
```
Inherited Wisdom from .hoid/state/:
- Convention: All components use React hooks, no class components
- Gotcha: Redux store hydration happens AFTER mount, not during
- Status: Tests for ComponentA passing, ComponentB has flaky timing issues
```

**Subagent (after completing work):**
```
Append to .hoid/state/notes.md:

## 2025-03-28 10:45 Task: Fix ComponentB flaky tests
Found: Race condition between state update and test assertion
Solution: Added waitFor() with explicit condition check
Result: Tests now pass consistently (10 runs, 0 failures)
Pattern: Similar issue likely in ComponentC, may need same fix
```

**Orchestrator (verification):**
- Read notes
- Update plans.md
- Next subagent sees pattern about ComponentC before delegating

### Session End
**Promote to permanent:**
```bash
mkdir -p .hoid/context/testing
cat > .hoid/context/testing/timing-patterns.md << EOF
# React Component Testing Patterns

## Race Condition: State Update + Assertion
Common issue: Component updates state, test asserts before state propagates

Solution: waitFor with explicit condition, not just async delay

Example:
\`\`\`
waitFor(() => expect(screen.getByText('Success')).toBeInTheDocument())
\`\`\`

When encountered: ComponentB (2025-03-28), may exist in ComponentC

Status: Template created, apply to other components
EOF

# Update .gitignore
cat >> .gitignore << EOF

# Permanent context (tracked)
!.hoid/context/
EOF

git add .hoid/context/
git add .gitignore
```

---

## When State Becomes Technical Debt

If `.hoid/state/` grows over multiple sessions:

**Archive pattern:**
```bash
# Move old session to archive
mkdir -p .hoid/archive/session-2025-03-28
mv .hoid/state/* .hoid/archive/session-2025-03-28/

# Promote learnings to context
# (extract useful patterns, discard noise)

# Start fresh
touch .hoid/state/{notes,plans,decisions}.md
```

---

## Integration with Other Skills

**Skills that interact with state:**

- **executing-plans:** Reads from plans.md, updates as tasks complete
- **subagent-driven-development:** Reads notes.md for inherited wisdom
- **systematic-debugging:** May add debug findings to notes.md
- **writing-skills:** Creates skills/ directory as *output*, not state (tracked)
- **test-driven-development:** May create per-test state files (follow its override rules)

Each skill can declare how it uses state management. This skill provides the foundation.

---

## Quick Reference

| When | What | Where | Rule |
|------|------|-------|------|
| Session start | Create directories | `.hoid/state/` | Once per directory |
| Before delegate | Read context | `notes.md`, `decisions.md` | Always |
| Subagent work | Append findings | `notes.md` | Append only |
| Task complete | Update progress | `plans.md` | Mutable |
| Session end | Decide | Keep or discard | Optional |
| Reuse learnings | Promote | `.hoid/context/` | Append to permanent |
