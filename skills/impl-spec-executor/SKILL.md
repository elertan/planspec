---
name: impl-spec-executor
description: Execute implementation specs autonomously. Takes an impl-spec file, creates a topic branch, spawns implementation subagents per phase, runs reviewers at checkpoints. Surfaces to user only when issues can't be auto-resolved.
---

# Implementation Spec Executor

Execute validated implementation specs through to completion using subagents and task tracking.

## Usage

```
planspec:impl-spec-executor planspec/implementations/[topic].md
```

## Prerequisites

- Implementation spec exists and has `status: ready`
- Design spec referenced in impl-spec is approved
- Working directory is clean (no uncommitted changes)

## Architecture

This skill runs in **main context** and uses:
- **Task system** - Track progress and enforce phase dependencies
- **Implementation subagents** - One per phase (implements tasks or fixes review issues)
- **Reviewer subagents** - Code review (always) and security review (when needed)

```
impl-spec-executor (SKILL - main context)
    │
    ├─► [Task Graph Created Upfront]
    │
    ├─► Initialize task
    │
    ├─► Phase 1: Implement task → Checkpoint task
    │       │                         │
    │       │                         ├─► Code Reviewer
    │       │                         └─► Security Reviewer (if needed)
    │       │                               │
    │       │                         ◄─────┘ (loop until PASS)
    │       │
    ├─► Phase 2: Implement task → Checkpoint task
    │       ...
    │
    └─► Complete task
```

---

## Task Structure

Tasks are created upfront to enforce dependencies:

```
[1] Initialize
[2] Implement Phase 1     (blockedBy: 1)
[3] Checkpoint Phase 1    (blockedBy: 2)
[4] Implement Phase 2     (blockedBy: 3)
[5] Checkpoint Phase 2    (blockedBy: 4)
...
[N] Complete              (blockedBy: last checkpoint)
```

The `blockedBy` relationships enforce:
- Can't start implementation until previous checkpoint passes
- Can't run checkpoint until implementation completes
- Can't complete until all phases pass

---

## Process

### Step 0: Create Task Graph

Before any work, create the full task graph for visibility and enforcement.

1. **Parse impl-spec** to extract:
   - Number of phases
   - Phase names
   - Which phases need security review (have `Security:` line in checkpoint)

2. **Create all tasks upfront:**

   ```
   INIT_ID = TaskCreate(
     subject: "Initialize: create branch and update status",
     description: "Create topic branch, update impl-spec status to in-progress",
     activeForm: "Initializing"
   )

   For each phase N:
     IMPL_IDS[N] = TaskCreate(
       subject: "Implement Phase N: [phase name]",
       description: "Execute all tasks in phase N via implementer subagent",
       activeForm: "Implementing Phase N"
     )

     CHECKPOINT_IDS[N] = TaskCreate(
       subject: "Checkpoint Phase N",
       description: "Run code review [+ security review], fix issues until all PASS",
       activeForm: "Reviewing Phase N"
     )

   COMPLETE_ID = TaskCreate(
     subject: "Complete: finalize and report",
     description: "Update impl-spec status to completed, output summary",
     activeForm: "Completing"
   )
   ```

3. **Set dependencies:**

   ```
   TaskUpdate(IMPL_IDS[1], addBlockedBy: [INIT_ID])
   TaskUpdate(CHECKPOINT_IDS[1], addBlockedBy: [IMPL_IDS[1]])

   For N > 1:
     TaskUpdate(IMPL_IDS[N], addBlockedBy: [CHECKPOINT_IDS[N-1]])
     TaskUpdate(CHECKPOINT_IDS[N], addBlockedBy: [IMPL_IDS[N]])

   TaskUpdate(COMPLETE_ID, addBlockedBy: [CHECKPOINT_IDS[last]])
   ```

4. **Output task graph:**

   ```
   Task graph created for [topic]:

   [1] Initialize
   [2] Implement Phase 1: [name]     (blocked by: 1)
   [3] Checkpoint Phase 1            (blocked by: 2)
   [4] Implement Phase 2: [name]     (blocked by: 3)
   [5] Checkpoint Phase 2            (blocked by: 4)
   ...
   [N] Complete                      (blocked by: [last checkpoint])

   Security reviews: Phases [list] (or "None")
   ```

### Step 1: Initialize

1. **Verify task ready:**
   - `TaskGet(INIT_ID)` → verify `blockedBy` is empty
   - If blocked → ERROR (should never happen for init)

2. **Mark in progress:**
   - `TaskUpdate(INIT_ID, status: in_progress)`

3. **Validate input:**
   - Read impl-spec file
   - Verify `status: ready`
   - Verify linked design spec exists and read it

4. **Check git state:**
   - Verify working directory is clean
   - If uncommitted changes exist, abort with message

5. **Create topic branch:**
   - Extract topic from impl-spec filename (e.g., `user-auth` from `user-auth.md`)
   - Create and checkout branch: `git checkout -b feature/[topic]`
   - If branch exists, abort and ask user how to proceed

6. **Update impl-spec status:**
   - Change `status: ready` → `status: in-progress`
   - Commit: `chore([topic]): start implementation`

7. **Mark completed:**
   - `TaskUpdate(INIT_ID, status: completed)`

### Step 2: Execute Phases

For each phase in order:

#### 2.1 Implementation

1. **Verify task ready:**
   - `TaskGet(IMPL_IDS[N])` → verify `blockedBy` is empty
   - If blocked → STOP (previous checkpoint didn't pass)

2. **Mark in progress:**
   - `TaskUpdate(IMPL_IDS[N], status: in_progress)`

3. **Record BASE_SHA** (current commit before implementation)

4. **Spawn implementation subagent:**

   Read `./implementer-prompt.md`, substitute variables:
   - `{TOPIC}` - extracted from impl-spec filename
   - `{PHASE_NUMBER}`, `{PHASE_NAME}`
   - `{IMPL_SPEC_PATH}`, `{DESIGN_SPEC_PATH}`
   - `{PHASE_TASKS}` - full text of all tasks in phase

   ```
   Task(
     subagent_type: "general-purpose",
     description: "Implement Phase [N]: [phase name]",
     prompt: [constructed from template]
   )
   ```

5. **Handle subagent response:**
   - If blocker reported → surface to user, wait for guidance
   - If execution error → surface to user (see Error Handling)
   - If completed → proceed to checkpoint

6. **Record HEAD_SHA** (current commit after implementation)

7. **Mark completed:**
   - `TaskUpdate(IMPL_IDS[N], status: completed)`

#### 2.2 Checkpoint

1. **Verify task ready:**
   - `TaskGet(CHECKPOINT_IDS[N])` → verify `blockedBy` is empty

2. **Mark in progress:**
   - `TaskUpdate(CHECKPOINT_IDS[N], status: in_progress)`

3. **Parse checkpoint requirements:**
   - Look for `Security:` line in checkpoint
   - If present, extract concerns and mark security review needed

4. **Run review loop:**

   ```
   WHILE true:
     # Run reviewers (parallel if both needed)
     code_result = spawn_code_reviewer(phase N, BASE_SHA, HEAD_SHA)

     if security_needed:
       security_result = spawn_security_reviewer(phase N, BASE_SHA, HEAD_SHA)
     else:
       security_result = N/A

     # Check verdicts
     code_pass = (code_result.verdict == "PASS" or "PASS WITH FIXES")
     security_pass = (security_result == N/A or security_result.verdict == "PASS" or "PASS WITH FIXES")

     IF code_pass AND security_pass:
       BREAK  # All passed, exit loop

     # Collect all issues
     issues = []
     if not code_pass:
       issues.extend(code_result.blocking_issues)
     if not security_pass:
       issues.extend(security_result.blocking_issues)

     # Spawn implementer to fix issues
     Task(
       subagent_type: "general-purpose",
       description: "Fix review issues for Phase [N]",
       prompt: implementer-prompt.md with:
         {PHASE_TASKS} = "Fix the following review issues:\n\n" + [formatted issues]
     )

     # Handle fix result
     if execution_error:
       surface_to_user()
       wait_for_guidance()

     # Update HEAD_SHA for next review iteration
     HEAD_SHA = current commit
   ```

5. **Output checkpoint summary:**

   ```
   ═══════════════════════════════════════════════════════════
   CHECKPOINT: Phase [N] Complete
   ═══════════════════════════════════════════════════════════
   Code Review:     [PASS/PASS WITH FIXES] - [1-line summary]
   Security Review: [PASS/PASS WITH FIXES/N/A] - [1-line summary]
   ───────────────────────────────────────────────────────────
   Verdict: PROCEED TO PHASE [N+1]
   ═══════════════════════════════════════════════════════════
   ```

6. **Mark completed:**
   - `TaskUpdate(CHECKPOINT_IDS[N], status: completed)`

7. **Proceed to next phase** (next implement task is now unblocked)

### Step 3: Completion

1. **Verify task ready:**
   - `TaskGet(COMPLETE_ID)` → verify `blockedBy` is empty

2. **Mark in progress:**
   - `TaskUpdate(COMPLETE_ID, status: in_progress)`

3. **Final verification:**
   - All tests passing
   - All checkpoints passed (verify via TaskList)

4. **Update impl-spec:**
   - Change `status: in-progress` → `status: completed`
   - Add completion date
   - Commit: `docs([topic]): mark implementation complete`

5. **Report summary:**

   ```
   Implementation Complete: [topic]

   Branch: feature/[topic]
   Commits: [N]
   Phases completed: [X]

   Design spec criteria met:
   - [criterion 1]: Y
   - [criterion 2]: Y

   Ready for: merge to main / PR creation
   ```

6. **Mark completed:**
   - `TaskUpdate(COMPLETE_ID, status: completed)`

---

## Spawning Reviewers

### Code Reviewer

Read `./code-reviewer-prompt.md`, substitute variables:
- `{PHASE_NUMBER}`, `{PHASE_NAME}`
- `{IMPL_SPEC_PATH}`, `{DESIGN_SPEC_PATH}`
- `{PHASE_TASKS}`
- `{BASE_SHA}`, `{HEAD_SHA}`

```
Task(
  subagent_type: "general-purpose",
  description: "Code review Phase [N]",
  prompt: [constructed from template]
)
```

### Security Reviewer

Read `./security-reviewer-prompt.md`, substitute variables:
- `{PHASE_NUMBER}`, `{PHASE_NAME}`
- `{IMPL_SPEC_PATH}`, `{DESIGN_SPEC_PATH}`
- `{SECURITY_CONCERNS}` - from checkpoint's `Security:` line
- `{BASE_SHA}`, `{HEAD_SHA}`

```
Task(
  subagent_type: "general-purpose",
  description: "Security review Phase [N]",
  prompt: [constructed from template]
)
```

**Run both reviewers in parallel** when security review is needed.

---

## Error Handling

Execution errors (not review issues) are surfaced to user immediately:

### Execution Errors

When a subagent or command fails (file write fails, command fails, etc.):

```
Execution Error

Phase: [N]
Task: [what was being attempted]
Error: [error message]

Options:
1. Provide guidance on how to resolve
2. Retry the operation
3. Abort implementation

Waiting for input...
```

### Implementation Blocker

When implementer subagent reports a blocker:

```
Implementation Blocked

Phase: [N]
Task: [X.Y] - [title]
Blocker: [from subagent report]

Options:
1. Provide additional context
2. Modify approach (will note deviation from spec)
3. Skip task (will note incomplete)
4. Abort implementation

Waiting for input...
```

---

## Checkpoint Format Reference

Checkpoint with code review only (default):
```markdown
### CHECKPOINT

Gate: Tests pass, issues resolved before Phase 2.
```

Checkpoint with security review:
```markdown
### CHECKPOINT

Security: Auth flow, user input handling, database queries

Gate: Tests pass, issues resolved before Phase 3.
```

---

## Principles

- **Task dependencies enforce order** - Can't skip checkpoints, `blockedBy` prevents it
- **Reviews loop until pass** - No limit on fix iterations for review issues
- **Execution errors surface immediately** - User guides resolution
- **Spec is source of truth** - Don't deviate without explicit approval
- **Atomic commits** - Each commit is a logical unit
- **Phase-level subagents** - One implementation subagent per phase (not per task)
- **Parallel reviews** - Code and security reviews run simultaneously when both needed
- **Clean state** - Always leave repo in buildable, testable state

---

## Prompt Templates

All subagent prompts are in this directory:
- `./implementer-prompt.md` - Phase implementation (or review issue fixing)
- `./code-reviewer-prompt.md` - Code quality and correctness review
- `./security-reviewer-prompt.md` - Security-focused review

Template variables:

| Variable | Used by | Description |
|----------|---------|-------------|
| `{TOPIC}` | implementer | Feature name from impl-spec filename |
| `{PHASE_NUMBER}` | all | Current phase number |
| `{PHASE_NAME}` | all | Current phase name |
| `{IMPL_SPEC_PATH}` | all | Path to implementation spec |
| `{DESIGN_SPEC_PATH}` | all | Path to design spec |
| `{PHASE_TASKS}` | implementer, code-reviewer | Full text of phase tasks (or review issues to fix) |
| `{BASE_SHA}` | reviewers | Git commit before phase |
| `{HEAD_SHA}` | reviewers | Git commit after phase |
| `{SECURITY_CONCERNS}` | security-reviewer | Why security review triggered |

---

## Quick Reference

| Input | Impl-spec file path |
|-------|---------------------|
| Output | Completed implementation on topic branch |
| Branch | `feature/[topic]` |
| Tasks per phase | 2 (Implement + Checkpoint) |
| Code review | Always, at every checkpoint |
| Security review | Only when `Security:` line present |
| Review fix loop | Unlimited iterations until PASS |
| Escalation | On execution errors (not review issues) |
