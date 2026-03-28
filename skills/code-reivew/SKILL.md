---
name: hoid:code-review-orchestration
description: Use when code is ready for review - orchestrates multi-phase review, immediate fix loop for MEDIUM/HIGH issues, testing, and commit
category: workflow
---

# Code Review Orchestration

## Overview

**Orchestrator pattern for code review: REVIEW → ROUTE → FIX (loop) → TEST → COMMIT.**

You manage the review lifecycle, not code. You spawn reviewers, read their findings, route decisions, and spawn fixers—but you never write code yourself. MEDIUM and HIGH priority issues are fixed immediately in a bounded loop (max 3 iterations). LOW/trivial issues are documented but not fixed.

**Core principle:** Orchestrate teams, not code. All communication through state files.

---

## When to Use

**Use when:**
- Code is written and ready for review
- You need to catch and fix MEDIUM/HIGH issues before commit
- You're orchestrating a team (even a team of agents)
- You want reproducible, state-driven review workflows

**Do NOT use when:**
- Code needs architectural re-design before review (use brainstorming first)
- You're in exploratory/prototyping phase (too early for formal review)
- Single file, trivial changes (overkill)

---

## Prerequisites

**REQUIRED BACKGROUND:** This skill uses the `managing-ephemeral-state` pattern. State files live in `.hoid/state/code-review/` and are ephemeral (never committed).

```
.hoid/state/code-review/
  review-findings.md       # Review results (append-only log)
  fix-prompt.md            # Instructions for fixers
  test-results.md          # Test run output
  commit-status.md         # Commit/PR status
  code-review-status.md    # Exit signal
```

---

## Workflow Diagram

```
INIT → REVIEW ──→ ROUTE ───→ (no issues) ─→ COMMIT → COMPLETE
                    ↑                           ↑
                    │  issues found             │ CI failures
                    ↓                           ↓
                   FIX ──→ TEST ────┬─→ CONTINUE_REVIEW
                    ↑                │   (re-review fixes)
                    └────────────────┘
                    (max 3 loops)

If unresolved after 3 loops or CI fails:
                    NEEDS_BRAINSTORM (exit, re-design required)
```

---

## Execution Model

### You (Orchestrator)

**ONLY:**
- ✅ Spawn subagents via `task()` (always background)
- ✅ Read `.hoid/state/code-review/*.md` files
- ✅ Write `.hoid/state/code-review/*-prompt.md` instruction files
- ✅ Manage loop counter and routing logic
- ✅ Write `.hoid/state/code-review/code-review-status.md` (exit signal)

**NEVER:**
- ❌ Write code files
- ❌ Run git commands
- ❌ Run tests
- ❌ Make implementation decisions

### Subagents (spawned by you)

- **code-reviewer**: Performs comprehensive review, writes findings to `.hoid/state/code-review/review-findings.md`
- **fixer**: Implements fixes based on `.hoid/state/code-review/fix-prompt.md`
- **tester**: Runs tests, writes results to `.hoid/state/code-review/test-results.md`
- **committer**: Commits, pushes, creates PR

---

## Phase 1: INIT

**Goal:** Establish context and scope.

**Actions:**

1. Create state directory:
   ```bash
   mkdir -p .hoid/state/code-review
   ```

2. Get changed files:
   ```bash
   git diff --name-only HEAD
   git status --short
   ```

3. Initialize loop counter = 0 (track in memory during orchestration)

4. Proceed to REVIEW

---

## Phase 2: REVIEW

**Goal:** Comprehensive code review across all changed files.

**Actions:**

1. Write `.hoid/state/code-review/review-prompt.md`:
   ```markdown
   # Code Review Instructions

   ## Scope
   Changed files:
   [list from INIT]

   Review ONLY changed files. Flag issues if the changed code introduces
   a problem in unchanged files (e.g., wrong types passed to call sites).

   ## Issue Categories
   Categorize each issue:
   - CRITICAL: Security, correctness, will cause production failure
   - HIGH: Major logic error, incorrect behavior
   - MEDIUM: Code quality, performance, maintainability issue
   - LOW: Style, minor optimization, documentation

   ## Output Format
   Write to `.hoid/state/code-review/review-findings.md`:

   ```
   # Review Findings

   ## CRITICAL Issues (N)
   - [Issue 1]: [File], [Line], [Explanation]

   ## HIGH Issues (N)
   - [Issue 1]: [File], [Line], [Explanation]

   ## MEDIUM Issues (N)
   - [Issue 1]: [File], [Line], [Explanation]

   ## LOW Issues (N)
   - [Issue 1]: [File], [Line], [Explanation]

   ## Summary
   Total: [N] CRITICAL, [N] HIGH, [N] MEDIUM, [N] LOW
   ```
   ```

2. Spawn reviewer (background):
   ```
   task(category="deep", load_skills=["git-master"],
        run_in_background=true,
        prompt="[Full review-prompt.md content]

        Complete the code review and write findings to .hoid/state/code-review/review-findings.md")
   ```

3. Wait for reviewer to complete, then read `.hoid/state/code-review/review-findings.md`

4. Proceed to ROUTE

---

## Phase 3: ROUTE

**Goal:** Decide next phase based on review findings.

**Logic:**

```
if no issues found:
    → COMMIT

elif only LOW issues:
    → COMMIT (document but don't fix)

elif CRITICAL or HIGH present AND loop < 3:
    → FIX (increment loop counter)

elif MEDIUM present AND loop < 3:
    → FIX (increment loop counter)

elif CRITICAL or HIGH present AND loop >= 3:
    → EXIT with NEEDS_BRAINSTORM
    Reason: "Unresolved CRITICAL/HIGH after 3 fix iterations"

elif MEDIUM present AND loop >= 3:
    → COMMIT (acceptable after 3 attempts)
```

---

## Phase 4: FIX

**Goal:** Fix CRITICAL, HIGH, and MEDIUM issues (in that priority order).

**Actions:**

1. Write `.hoid/state/code-review/fix-prompt.md`:
   ```markdown
   # Fix Instructions (Iteration [N])

   ## Issues to Fix
   Read `.hoid/state/code-review/review-findings.md`. Fix ALL issues of CRITICAL,
   HIGH, and MEDIUM severity in that priority order.

   Do NOT fix LOW issues (skip those).

   ## Constraints
   - Fix ONLY reported issues
   - Do NOT rewrite unrelated code
   - Do NOT introduce new functionality
   - Do NOT change public APIs unless required by fix

   ## Changed Files (for context)
   [list from INIT]

   ## Completion
   When done, append to `.hoid/state/code-review/review-findings.md`:

   ```
   # Fixes Applied (Iteration [N])
   - [Issue]: FIXED
   - [Issue]: FIXED
   ...
   ```
   ```

2. Spawn fixer (background):
   ```
   task(category="quick", load_skills=["git-master"],
        run_in_background=true,
        prompt="[Full fix-prompt.md content]

        Apply fixes and report completion in .hoid/state/code-review/review-findings.md")
   ```

3. Wait for completion, then proceed to TEST

---

## Phase 5: TEST

**Goal:** Verify fixes don't break anything.

**Actions:**

1. Write `.hoid/state/code-review/test-prompt.md`:
   ```markdown
   # Test Instructions (After Iteration [N] Fixes)

   Run full test suite if available. If not, run relevant tests for changed files.

   Report results to `.hoid/state/code-review/test-results.md`:
   - WHAT ran
   - WHAT failed (if any)
   - WHERE it failed (file:line, test name)
   - Full error output
   ```

2. Spawn tester (background):
   ```
   task(category="quick", load_skills=[],
        run_in_background=true,
        prompt="[Full test-prompt.md content]")
   ```

3. Read `.hoid/state/code-review/test-results.md`:
   - **All pass** → CONTINUE_REVIEW (go back to REVIEW to verify fixes resolved issues)
   - **Any fail** → FIX (update fix-prompt with test failures, loop back to FIX)

---

## Phase 6: CONTINUE_REVIEW

**Goal:** Verify that fixes actually resolved the reported issues.

**Actions:**

1. Update `.hoid/state/code-review/review-prompt.md`:
   ```markdown
   # Re-Review After Fixes (Iteration [N])

   [Original review instructions]

   ## Focus
   This is a focused re-review after fixes. Verify that the issues
   reported in the previous review were actually resolved:

   [List issues from review-findings.md]

   Check ONLY whether these specific issues are fixed. Do NOT run a
   full re-scan unless you find new issues.
   ```

2. Spawn reviewer again (background):
   ```
   task(category="deep", load_skills=[],
        run_in_background=true,
        prompt="[Updated review-prompt.md]\n\nWrite updated findings to .hoid/state/code-review/review-findings.md")
   ```

3. Read updated `.hoid/state/code-review/review-findings.md`

4. Return to ROUTE (check if issues are resolved or new issues found)

---

## Phase 7: COMMIT

**Goal:** Commit reviewed and fixed code.

**Actions:**

1. Write `.hoid/state/code-review/commit-prompt.md`:
   ```markdown
   # Commit Instructions

   Commit all changes reviewed and fixed in this code review cycle.

   Commit message:
   - Summarize what was fixed based on review findings
   - Include brief note on issue types addressed (e.g., "fix: resolve CRITICAL and HIGH review issues")

   Then:
   - git push [branch]
   - Create PR (if not already created)
   - Report PR URL to `.hoid/state/code-review/commit-status.md`
   ```

2. Spawn committer (background):
   ```
   task(category="quick", load_skills=["git-master"],
        run_in_background=true,
        prompt="[Full commit-prompt.md content]")
   ```

3. Read `.hoid/state/code-review/commit-status.md`:
   - **SUCCESS** → COMPLETE
   - **FAILED** → EXIT with FAILED

---

## Phase 8: COMPLETE

**Goal:** Signal completion.

**Actions:**

1. Write `.hoid/state/code-review/code-review-status.md`:
   ```markdown
   # Code Review Status

   Status: COMPLETE
   Timestamp: [ISO 8601]

   ## Summary
   - Iterations: [N]
   - Issues found: CRITICAL [N], HIGH [N], MEDIUM [N], LOW [N]
   - Issues fixed: [N]
   - Tests: PASSING
   - PR: [URL]
   ```

---

## Exit States

### COMPLETE
All issues resolved, tests passing, commit created.

### NEEDS_BRAINSTORM
```markdown
Status: NEEDS_BRAINSTORM
Reason: Unresolved CRITICAL/HIGH after 3 fix iterations
Issues: [list from review-findings.md]
Recommended: Re-design needed before further review
```

### FAILED
```markdown
Status: FAILED
Reason: [commit failed, tests crashed, etc.]
Details: [error output]
```

---

## State Management

This skill uses `.hoid/state/code-review/` (ephemeral):

| File | Purpose | Append/Overwrite |
|------|---------|------------------|
| `review-prompt.md` | Review instructions | Overwrite each REVIEW phase |
| `review-findings.md` | Review results | Append fixes applied, re-review results |
| `fix-prompt.md` | Fix instructions | Overwrite each FIX phase |
| `test-prompt.md` | Test instructions | Overwrite each TEST phase |
| `test-results.md` | Test output | Overwrite each TEST phase |
| `commit-prompt.md` | Commit instructions | Overwrite once before COMMIT |
| `commit-status.md` | Commit result | Overwrite once |
| `code-review-status.md` | Exit signal | Overwrite once at end |

---

## Integration with managing-ephemeral-state

**Before starting:**
- Read `.hoid/state/code-review/notes.md` and `.hoid/state/code-review/decisions.md` (inherited wisdom)
- Include in review-prompt.md as context

**After completing:**
- Append to `.hoid/state/code-review/notes.md`:
  ```
  ## [Timestamp] Code Review Complete
  Issues: CRITICAL [N], HIGH [N], MEDIUM [N], LOW [N]
  Iterations: [N]
  All fixed and committed to [PR URL]
  ```

---

## Common Mistakes

| Mistake | Reality | Fix |
|---------|---------|-----|
| **Writing code as orchestrator** | Not your job | Spawn agents only |
| **Forgetting loop counter** | Infinite loops | Track iterations in memory |
| **Fixing LOW issues** | Waste of time | Skip LOW, commit anyway |
| **Re-reviewing without fixes** | Wasted cycle | Only re-review after TEST passes |
| **No exit criteria** | Gets stuck | Use 3-iteration limit strictly |
| **Committing before test pass** | Broken code | Wait for TEST to PASS or NEEDS_BRAINSTORM |

---

## Quick Reference

| Phase | Purpose | Spawns | Reads |
|-------|---------|--------|-------|
| INIT | Scope work | — | git diff |
| REVIEW | Find issues | code-reviewer | review-findings.md |
| ROUTE | Decide next | — | review-findings.md counts |
| FIX | Apply fixes | fixer | fix-prompt.md |
| TEST | Verify fixes | tester | test-results.md |
| CONTINUE_REVIEW | Verify resolved | code-reviewer (focused) | review-findings.md |
| COMMIT | Save work | committer | commit-status.md |
| COMPLETE | Signal done | — | write code-review-status.md |

---

## When to Use MAX_ITERATIONS = 3

**After 3 FIX iterations with unresolved CRITICAL/HIGH:**
- Indicates a design problem, not a fix problem
- Exit with NEEDS_BRAINSTORM
- Parent workflow re-brainstorms before another review cycle

**After 3 FIX iterations with only MEDIUM remaining:**
- Acceptable to commit
- Document as "MEDIUM issues deferred, OK to ship"
