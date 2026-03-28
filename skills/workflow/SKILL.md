---
name: team-workflow-orchestration
description: Use when starting feature work or project tasks - orchestrates brainstorming, planning, implementation, and review with agent teams
user-invocable: true
category: workflow
---

# Team-Based Development Workflow

## Overview

**Orchestrator pattern for team-based feature development: INIT → WORKTREE → BRAINSTORM → PLAN → EXECUTE → REVIEW → COMPLETE.**

You manage the entire development lifecycle from idea to shipped code. You spawn specialized agents for each phase, read their outputs, route decisions based on criteria, and pause for user approval at design and review gates. This workflow scales from single-feature work to multi-day projects.

**Core principle:** Orchestrate teams, not code. All communication through state files. User approves design and review; orchestrator routes everything else.

---

## When to Use

**Use when:**
- Starting new feature development (bounded or exploratory)
- Complex tasks requiring brainstorming + planning + implementation
- You want your team (agents) to follow a structured, repeatable process
- You're ready to think through design before coding

**Do NOT use when:**
- Quick one-file fixes (use code-review-orchestration directly)
- You already have a complete design and just need execution (use code-review-orchestration)
- Exploratory spike with no clear endpoint (too early for formal workflow)

---

## Prerequisites

**REQUIRED BACKGROUND:** 
- `managing-ephemeral-state` - state structure and append-only patterns
- `brainstorming` - interactive design exploration (user-facing)
- `code-review-orchestration` - review loop with fix iterations

This workflow orchestrates all three. Understand each before using this skill.

---

## Workflow Diagram

```
INIT 
  ↓
WORKTREE (create git isolation)
  ↓
BRAINSTORM (interactive design) → [USER APPROVES DESIGN?]
  ↓                                ↑
PLAN (detailed implementation)      │ No → EXIT (design rejected)
  ↓
EXECUTE (code changes by agents)
  ↓
REVIEW (code-review-orchestration loop)
  ↓
[ALL ISSUES RESOLVED & TESTS PASS?]
  ├─ Yes → COMMIT & COMPLETE
  └─ No → [UNRESOLVABLE ISSUES?]
            ├─ Yes → EXIT (needs redesign)
            └─ No → Re-review (already in REVIEW loop)

User approval gates: After BRAINSTORM, After REVIEW (before commit)
```

---

## Execution Model

### You (Orchestrator)

**ONLY:**
- ✅ Spawn subagents via `task()` (always background for long tasks)
- ✅ Read state files across all phases
- ✅ Write phase-specific prompt files
- ✅ Manage routing logic and loop counters
- ✅ Write final status to `.hoid/state/workflow/status.md`
- ✅ Pause for user approval at design and review gates

**NEVER:**
- ❌ Write code files
- ❌ Run git commands directly (agents do this)
- ❌ Make design decisions (that's brainstorming phase)
- ❌ Force progression without user approval at gates

### Subagents (spawned by you)

- **brainstormer**: Interactive design exploration, writes to `.hoid/state/workflow/brainstorm/design.md`
- **planner**: Detailed implementation plan, writes to `.hoid/state/workflow/plan/implementation.md`
- **executor**: Code changes based on plan (agent type decided by plan), writes status to `.hoid/state/workflow/execute/status.md`
- **reviewer**: Uses code-review-orchestration, outputs go to `.hoid/state/workflow/review/`
- **committer**: Final commit and push

---

## Phase 1: INIT

**Goal:** Establish context, scope, and state.

**Actions:**

1. Create state directories:
   ```bash
   mkdir -p .hoid/state/workflow/{brainstorm,plan,execute,review}
   ```

2. Read inherited wisdom:
   - `.hoid/state/notes.md` (if exists)
   - `.hoid/state/decisions.md` (if exists)

3. Capture initial context:
   - What's the task/feature being requested?
   - Any constraints or dependencies?
   - Who's involved (user only, or distributed team)?

4. Write `.hoid/state/workflow/status.md`:
   ```markdown
   # Workflow Status
   
   Status: INIT
   Timestamp: [ISO 8601]
   Task: [description]
   Scope: [single feature / multi-day project / unclear yet]
   ```

5. Proceed to WORKTREE

---

## Phase 2: WORKTREE

**Goal:** Create isolated git workspace for entire workflow.

**Actions:**

1. Spawn worktree-agent (background):
   ```
   task(category="quick", load_skills=["git-master"],
        run_in_background=true,
        prompt="Create a git worktree for this feature work.
        
        Use 'git worktree add' to create isolated workspace.
        Name it descriptively (e.g., 'feature/user-auth').
        
        Write worktree path to .hoid/state/workflow/worktree-path.txt")
   ```

2. Wait for completion, read `.hoid/state/workflow/worktree-path.txt`

3. Update `.hoid/state/workflow/status.md`:
   ```markdown
   Status: WORKTREE_READY
   Worktree: [path]
   ```

4. Proceed to BRAINSTORM

---

## Phase 3: BRAINSTORM

**Goal:** Interactive design exploration with user.

**Actions:**

1. Spawn brainstormer (foreground, interactive):
   ```
   task(category="unspecified-high", load_skills=[],
        run_in_background=false,
        prompt="Use the brainstorming skill to explore this feature.
        
        Task: [from INIT]
        Inherited wisdom: [from notes.md, decisions.md]
        
        Conduct interactive brainstorming:
        - Ask clarifying questions one at a time
        - Explore 2-3 approaches
        - Present validated design
        
        Write final design to .hoid/state/workflow/brainstorm/design.md
        
        Include sections:
        - Purpose & Success Criteria
        - Architecture & Components
        - Data Flow & State
        - Error Handling
        - Testing Strategy
        - Implementation Constraints")
   ```

2. After brainstormer completes, read `.hoid/state/workflow/brainstorm/design.md`

3. **USER APPROVAL GATE:**
   - Present design to user
   - Ask: "Does this look right?"
   - If NO → EXIT with DESIGN_REJECTED
   - If YES → Proceed to PLAN

4. Append to `.hoid/state/notes.md`:
   ```
   ## [Timestamp] Workflow: Brainstorm Complete
   Design: [brief summary]
   User approved: Yes
   ```

---

## Phase 4: PLAN

**Goal:** Detailed implementation plan with agent assignment.

**Actions:**

1. Write `.hoid/state/workflow/plan/plan-prompt.md`:
   ```markdown
   # Implementation Plan Instructions
   
   ## Design Context
   Read .hoid/state/workflow/brainstorm/design.md
   
   Create detailed implementation plan covering:
   
   ### 1. Implementation Strategy
   - Phased approach (what gets built in what order?)
   - Dependencies (what must be done first?)
   - Risk areas (what could go wrong?)
   
   ### 2. Agent Assignment
   For EXECUTE phase, specify which agents/skills to use:
   - Frontend changes: [agent/skill]
   - Backend changes: [agent/skill]
   - Tests: [agent/skill]
   - Documentation: [agent/skill]
   
   ### 3. Success Criteria
   - Tests pass: [specific test suites]
   - Code quality: [linting, coverage targets]
   - Performance: [if applicable]
   - Documentation: [updated, comprehensive]
   
   ### 4. Rollback Plan
   If EXECUTE fails, what's the recovery strategy?
   
   Write output to .hoid/state/workflow/plan/implementation.md
   ```

2. Spawn planner (background):
   ```
   task(category="deep", load_skills=["git-master"],
        run_in_background=true,
        prompt="[Full plan-prompt.md content]")
   ```

3. Wait for completion, read `.hoid/state/workflow/plan/implementation.md`

4. Update `.hoid/state/workflow/status.md`:
   ```markdown
   Status: PLAN_COMPLETE
   Agents assigned: [list]
   Success criteria: [list]
   ```

5. Proceed to EXECUTE

---

## Phase 5: EXECUTE

**Goal:** Implement code changes according to plan.

**Actions:**

1. Read `.hoid/state/workflow/plan/implementation.md` to get agent assignments

2. Write `.hoid/state/workflow/execute/execute-prompt.md`:
   ```markdown
   # Execute Instructions
   
   ## Plan Context
   Read .hoid/state/workflow/plan/implementation.md
   
   Implement the following:
   [List specific tasks from plan]
   
   ## Constraints
   - Use worktree at [path from worktree-path.txt]
   - Follow coding patterns in codebase
   - Write comprehensive tests
   - Update documentation
   
   ## Completion
   Write status to .hoid/state/workflow/execute/status.md:
   - What was implemented
   - Any issues encountered
   - Tests passing (Y/N)
   ```

3. Spawn executor agents (background, per plan):
   ```
   For each agent assignment in plan:
     task(category="[agent-appropriate-category]", 
          load_skills=["git-master"],
          run_in_background=true,
          prompt="[execute-prompt.md for this agent's portion]")
   ```

4. Wait for all executors to complete

5. Read `.hoid/state/workflow/execute/status.md`

6. Check for any failures:
   - **Execution errors** → Option to retry (re-dispatch executor) or EXIT
   - **Tests failing** → Retry executor with error details
   - **Success** → Proceed to REVIEW

7. Update `.hoid/state/workflow/status.md`:
   ```markdown
   Status: EXECUTE_COMPLETE
   Implementation: [summary]
   Tests: [passing/failing]
   Ready for review: [Y/N]
   ```

8. Proceed to REVIEW

---

## Phase 6: REVIEW

**Goal:** Code review using code-review-orchestration skill.

**Actions:**

1. Write `.hoid/state/workflow/review/review-prompt.md`:
   ```markdown
   # Code Review Instructions
   
   Use the code-review-orchestration skill to review changes.
   
   ## Context
   - Design: .hoid/state/workflow/brainstorm/design.md
   - Plan: .hoid/state/workflow/plan/implementation.md
   - Implementation: .hoid/state/workflow/execute/status.md
   
   ## Review Focus
   - Does implementation match design?
   - Are tests comprehensive?
   - Code quality meets project standards?
   - Documentation updated?
   
   ## Output
   Write findings to .hoid/state/workflow/review/review-findings.md
   (code-review-orchestration will create this)
   
   After review loop completes and all CRITICAL/HIGH issues resolved,
   write summary to .hoid/state/workflow/review/summary.md
   ```

2. Spawn reviewer (background, using code-review-orchestration):
   ```
   task(category="deep", load_skills=["git-master"],
        run_in_background=true,
        prompt="[Full review-prompt.md content]
        
        Execute the code-review-orchestration workflow:
        REVIEW → ROUTE → FIX (loop) → TEST → COMMIT
        
        All state files go in .hoid/state/workflow/review/
        (Override the usual .hoid/state/code-review/ path)")
   ```

3. Wait for reviewer to complete

4. Read `.hoid/state/workflow/review/review-findings.md`

5. **USER APPROVAL GATE:**
   - Present review summary to user
   - Show: Issues found, fixes applied, tests passing
   - Ask: "Ready to commit and ship?"
   - If NO → Option to re-open review or EXIT
   - If YES → Proceed to COMPLETE

6. Append to `.hoid/state/notes.md`:
   ```
   ## [Timestamp] Workflow: Review Complete
   Issues found: CRITICAL [N], HIGH [N], MEDIUM [N], LOW [N]
   All resolved: Yes
   User approved commit: Yes
   ```

---

## Phase 7: COMPLETE

**Goal:** Signal completion and cleanup.

**Actions:**

1. Write final `.hoid/state/workflow/status.md`:
   ```markdown
   # Workflow Status: COMPLETE
   
   Timestamp: [ISO 8601]
   Task: [from INIT]
   
   ## Timeline
   - Brainstorm: [time]
   - Plan: [time]
   - Execute: [time iterations]
   - Review: [time iterations]
   
   ## Summary
   - Issues found and fixed: [count by severity]
   - Tests: ALL PASSING
   - Code reviewed: ✓
   - User approved: ✓
   - Committed: ✓
   - PR/Branch: [URL or ref]
   
   ## Next Steps
   Monitor CI/CD. If failures appear, exit with NEEDS_REVIEW.
   ```

2. Optional worktree cleanup:
   ```
   task(category="quick", load_skills=["git-master"],
        run_in_background=true,
        prompt="Worktree cleanup is optional.
        User can keep worktree for further work or delete.
        For now, just log the worktree path for user reference.")
   ```

3. Append to `.hoid/state/notes.md`:
   ```
   ## [Timestamp] Workflow: COMPLETE
   Feature: [from INIT]
   PR: [URL]
   Status: Ready for deployment
   ```

---

## Exit States

### DESIGN_REJECTED
```markdown
Status: DESIGN_REJECTED
Timestamp: [ISO 8601]
Reason: User did not approve design
Design: [path to rejected design.md]

Next steps: User can re-brainstorm or pivot task
```

### EXECUTE_FAILED
```markdown
Status: EXECUTE_FAILED
Timestamp: [ISO 8601]
Reason: Implementation errors prevented completion
Errors: [from execute/status.md]

Next steps: User can retry, modify plan, or re-brainstorm
```

### REVIEW_NEEDS_REDESIGN
```markdown
Status: REVIEW_NEEDS_REDESIGN
Timestamp: [ISO 8601]
Reason: Code review found unresolvable issues after 3 fix iterations
Issues: [CRITICAL/HIGH from review findings]

Next steps: Return to BRAINSTORM with review findings as context
```

### COMPLETE
```markdown
Status: COMPLETE
Timestamp: [ISO 8601]
PR: [URL]
Tests: PASSING
User approved: Yes
Ready for: Deployment
```

---

## State Management

**Structure:**
```
.hoid/state/workflow/
  status.md                    # Overall workflow status
  notes.md                     # Session notes (shared with managing-ephemeral-state)
  decisions.md                 # Design decisions (shared with managing-ephemeral-state)
  worktree-path.txt            # Git worktree location
  
  brainstorm/
    design.md                  # Final validated design
  
  plan/
    implementation.md          # Detailed implementation plan with agent assignments
  
  execute/
    status.md                  # Execution completion report
  
  review/
    review-findings.md         # Code review findings (from code-review-orchestration)
    review-summary.md          # User-facing review summary
```

**Append-only files:**
- `.hoid/state/notes.md` - Each phase appends completion milestone
- `.hoid/state/decisions.md` - Design decisions logged here

**Mutable files:**
- `status.md` - Updated at each phase transition
- `brainstorm/design.md` - Validated once, then immutable
- `plan/implementation.md` - Finalized in PLAN, then immutable
- `execute/status.md` - Finalized after EXECUTE
- `review/review-findings.md` - Built by code-review-orchestration

---

## Integration Points

### With managing-ephemeral-state
- Reads `.hoid/state/notes.md` and `.hoid/state/decisions.md` at INIT
- Appends completion milestones to both after each phase

### With brainstorming
- BRAINSTORM phase spawns brainstormer agent using brainstorming skill
- Outputs go to `.hoid/state/workflow/brainstorm/design.md`

### With code-review-orchestration
- REVIEW phase spawns reviewer using code-review-orchestration skill
- Code-review outputs go to `.hoid/state/workflow/review/`
- Review loop handles MEDIUM/HIGH issues automatically

---

## User Approval Gates

### Gate 1: After BRAINSTORM
**What user sees:**
```
## Design Review

[Full design from .hoid/state/workflow/brainstorm/design.md]

Does this look right? (Yes/No)
```

**If YES:** Proceed to PLAN
**If NO:** EXIT with DESIGN_REJECTED

### Gate 2: After REVIEW
**What user sees:**
```
## Review Complete

Issues found: CRITICAL [N], HIGH [N], MEDIUM [N], LOW [N]
All issues resolved: Yes
Tests passing: Yes

Ready to commit and ship? (Yes/No/Re-review)
```

**If YES:** Proceed to COMPLETE
**If NO:** EXIT with REVIEW_REJECTED (optional cleanup)
**If Re-review:** Loop back to REVIEW phase

---

## Common Mistakes

| Mistake | Reality | Fix |
|---------|---------|-----|
| **Skipping BRAINSTORM** | Design flaws found late | Always brainstorm first |
| **Not reading PLAN before EXECUTE** | Agents don't know constraints | Read plan, pass to executor |
| **Ignoring user approval gates** | Shipping unvetted designs | Always pause at gates |
| **Looping infinitely on EXECUTE** | No exit criteria | Max 3 retries, then EXIT |
| **Reviewing before tests pass** | Broken code reviewed | Ensure EXECUTE tests pass first |
| **Committing without approval** | Shipping broken work | USER gate is mandatory |

---

## Quick Reference

| Phase | Purpose | Duration | Spawns | User Input |
|-------|---------|----------|--------|-----------|
| INIT | Scope work | <5 min | — | Task description |
| WORKTREE | Git isolation | 1-2 min | worktree-agent | — |
| BRAINSTORM | Design exploration | 15-45 min | brainstormer | Approval gate |
| PLAN | Implementation detail | 10-30 min | planner | — |
| EXECUTE | Code changes | 30 min - 4 hrs | executor(s) | — |
| REVIEW | Code review loop | 15-120 min | reviewer | Approval gate |
| COMPLETE | Finalize & ship | <5 min | committer | — |

---

## When to Use This vs. Code-Review-Orchestration

| Scenario | Use This Skill | Use Code-Review Instead |
|----------|---|---|
| New feature from scratch | ✓ | ✗ |
| Design already complete | ✗ | ✓ |
| Bug fix with clear scope | ✗ | ✓ |
| Exploratory work | ✓ | ✗ |
| Code review only | ✗ | ✓ |
| Full feature lifecycle | ✓ | ✗ |
