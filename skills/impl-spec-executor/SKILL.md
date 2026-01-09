---
name: impl-spec-executor
description: Execute implementation specs autonomously. Takes an impl-spec file, creates a topic branch, spawns implementation subagents per phase, runs reviewers at checkpoints. Surfaces to user only when issues can't be auto-resolved.
---

# Implementation Spec Executor

Execute validated implementation specs through to completion using subagents.

## Usage

```
planspec:impl-spec-executor planspec/implementations/[topic].md
```

## Prerequisites

- Implementation spec exists and has `status: ready`
- Design spec referenced in impl-spec is approved
- Working directory is clean (no uncommitted changes)

## Architecture

This skill runs in **main context** and spawns subagents for:
- **Implementation** - One subagent per phase (implements all tasks in phase)
- **Code review** - At checkpoints (always)
- **Security review** - At checkpoints (only if `Security:` line present)

```
impl-spec-executor (SKILL - main context)
    │
    ├─► Phase 1: Task(general-purpose) → implements all phase 1 tasks
    │       └─► checkpoint: spawn reviewers
    │           └─► Task(general-purpose, code-reviewer-prompt.md)
    │
    ├─► Phase 2: Task(general-purpose) → implements all phase 2 tasks
    │       └─► checkpoint: spawn reviewers (parallel if both)
    │           ├─► Task(general-purpose, code-reviewer-prompt.md)
    │           └─► Task(general-purpose, security-reviewer-prompt.md)
    ...
```

---

## Process

### Step 1: Initialize

1. **Validate input:**
   - Read impl-spec file
   - Verify `status: ready`
   - Verify linked design spec exists and read it

2. **Check git state:**
   - Verify working directory is clean
   - If uncommitted changes exist, abort with message

3. **Create topic branch:**
   - Extract topic from impl-spec filename (e.g., `user-auth` from `user-auth.md`)
   - Create and checkout branch: `git checkout -b feature/[topic]`
   - If branch exists, abort and ask user how to proceed

4. **Update impl-spec status:**
   - Change `status: ready` → `status: in-progress`
   - Commit: `chore([topic]): start implementation`

### Step 2: Execute Phases

For each phase in order:

1. **Extract phase info:**
   - Phase number and name
   - All tasks in the phase
   - Checkpoint requirements (parse `Security:` line if present)

2. **Spawn implementation subagent:**

   Use `Task` tool with `subagent_type: "general-purpose"`:

   ```
   Task(
     description: "Implement Phase [N]: [phase name]",
     prompt: """
       You are implementing Phase [N] of [topic].

       ## Context
       - Implementation spec: [path]
       - Design spec: [path]

       ## Tasks to implement:
       [Full text of all tasks in this phase]

       ## Instructions:
       1. Implement each task in order
       2. Follow the design spec requirements exactly
       3. Match existing codebase patterns
       4. Run tests after implementation
       5. Commit after significant tasks with format:
          feat([topic]): [what was done]

          Task [X.Y]: [task title]

       ## Report back:
       - Tasks completed
       - Files created/modified
       - Test results
       - Any blockers or questions
     """
   )
   ```

3. **Handle subagent response:**
   - If questions → answer and re-dispatch
   - If blockers → surface to user
   - If completed → proceed to checkpoint

4. **At checkpoint** → see Review Checkpoint below

5. **Proceed to next phase** only after checkpoint passes

### Review Checkpoint

At each `### CHECKPOINT` in the impl-spec:

1. **Parse checkpoint:**
   - Look for `Security:` line
   - If present, extract the concerns (e.g., "Auth flow, user input handling")
   - If absent, no security review needed

2. **Gather context for reviewers:**
   - Phase number and name
   - Tasks completed this phase
   - Files changed: `git diff --name-only [start-sha]..HEAD`
   - Design spec path

3. **Spawn code reviewer (always):**

   Read `./code-reviewer-prompt.md`, substitute variables:
   - `{PHASE_NUMBER}`, `{PHASE_NAME}`
   - `{IMPL_SPEC_PATH}`, `{DESIGN_SPEC_PATH}`
   - `{PHASE_TASKS}`, `{FILES_CHANGED}`

   ```
   Task(
     description: "Code review Phase [N]",
     prompt: [constructed from template]
   )
   ```

4. **Spawn security reviewer (if `Security:` line present):**

   Read `./security-reviewer-prompt.md`, substitute variables:
   - `{PHASE_NUMBER}`, `{PHASE_NAME}`
   - `{IMPL_SPEC_PATH}`, `{DESIGN_SPEC_PATH}`
   - `{SECURITY_CONCERNS}`, `{FILES_CHANGED}`

   ```
   Task(
     description: "Security review Phase [N]",
     prompt: [constructed from template]
   )
   ```

   **Run both reviewers in parallel** if security review needed.

5. **Evaluate results:**

   **All pass → proceed to next phase**

   **Issues found → resolution loop:**

   ```
   WHILE blocking issues exist:
     1. Read all issues from reviewers
     2. Fix each issue (in main context, not subagent)
     3. Commit fixes: "fix([topic]): address review feedback"
     4. Re-run same reviewers that found issues
     5. IF same issues persist after 3 attempts:
          Surface to user with:
          - What was tried
          - What's still failing
          - Request guidance
          WAIT for user response
          Apply user guidance
          Continue loop
   ```

6. **Only proceed when all reviewers pass**

### Step 3: Completion

After all phases complete:

1. **Final verification:**
   - All tests passing
   - All checkpoints passed

2. **Update impl-spec:**
   - Change `status: in-progress` → `status: completed`
   - Add completion date
   - Commit: `docs([topic]): mark implementation complete`

3. **Report summary:**
   ```
   ✅ Implementation Complete: [topic]

   Branch: feature/[topic]
   Commits: [N]
   Phases completed: [X]
   Review cycles: [Y]

   Design spec criteria met:
   - [criterion 1]: ✓
   - [criterion 2]: ✓

   Ready for: merge to main / PR creation
   ```

---

## Error Handling

### Unresolvable Review Issues

When issues can't be auto-resolved after 3 attempts:

```
⚠️ Review Issues Need Guidance

Phase: [N]
Attempts: 3

Unresolved issues:
- [Issue 1]: [what was tried, why it didn't work]
- [Issue 2]: [what was tried, why it didn't work]

Options:
1. Provide guidance on how to resolve
2. Mark as acceptable (with justification)
3. Abort implementation

Waiting for input...
```

### Implementation Subagent Failure

When subagent reports blocker:

```
⚠️ Implementation Blocked

Phase: [N]
Task: [X.Y] - [title]
Blocker: [from subagent report]

Options:
1. Provide additional context
2. Modify approach (will note deviation from spec)
3. Skip task (will note incomplete)
4. Abort implementation
```

### External Failures

Test failures, build errors, etc.:

```
⚠️ External Failure

Type: [test failure / build error / etc.]
Details: [error output]

Attempting auto-fix...
[If fix fails, surface to user]
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

- **Spec is source of truth** - Don't deviate without explicit approval
- **Atomic commits** - Each commit is a logical unit
- **Phase-level subagents** - One implementation subagent per phase (not per task)
- **Parallel reviews** - Code and security reviews run simultaneously when both needed
- **Fail fast, surface early** - Don't spin on unresolvable issues
- **Clean state** - Always leave repo in buildable, testable state

---

## Prompt Templates

Reviewer prompts are in this directory:
- `./code-reviewer-prompt.md` - Code quality and correctness review
- `./security-reviewer-prompt.md` - Security-focused review

These templates have variables that get substituted with actual values:
- `{PHASE_NUMBER}`, `{PHASE_NAME}`
- `{IMPL_SPEC_PATH}`, `{DESIGN_SPEC_PATH}`
- `{PHASE_TASKS}`, `{FILES_CHANGED}`
- `{SECURITY_CONCERNS}` (security reviewer only)

---

## Quick Reference

| Input | Impl-spec file path |
|-------|---------------------|
| Output | Completed implementation on topic branch |
| Branch | `feature/[topic]` |
| Implementation | One subagent per phase |
| Code review | At every checkpoint |
| Security review | Only when `Security:` line present |
| Escalation | After 3 failed resolution attempts |
