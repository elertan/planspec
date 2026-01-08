---
name: impl-spec-executor
description: Execute implementation specs autonomously. Takes an impl-spec file, creates a topic branch, executes all phases/tasks, runs reviewers at checkpoints, and commits after significant tasks. Surfaces to user only when issues can't be auto-resolved.
---

# Implementation Spec Executor

Execute validated implementation specs through to completion.

## Usage

```
planspec:impl-spec-executor planspec/implementations/[topic].md
```

## Prerequisites

- Implementation spec exists and has `status: ready`
- Design spec referenced in impl-spec is approved
- Working directory is clean (no uncommitted changes)

## Process

### Step 1: Initialize

1. **Validate input:**
   - Read impl-spec file
   - Verify `status: ready`
   - Verify linked design spec exists

2. **Check git state:**
   - Verify working directory is clean
   - If uncommitted changes exist, abort with message

3. **Create topic branch:**
   - Extract topic from impl-spec filename (e.g., `user-auth` from `user-auth.md`)
   - Create and checkout branch: `git checkout -b feature/[topic]`
   - If branch exists, abort and ask user how to proceed

4. **Update impl-spec status:**
   - Change `status: ready` → `status: in-progress`

### Step 2: Execute Phases

For each phase in order:

1. **Read phase tasks**
2. **Execute tasks** (see Task Execution below)
3. **At checkpoint** (see Review Checkpoint below)
4. **Proceed to next phase** only after checkpoint passes

### Task Execution

For each task in phase:

1. **Read task requirements:**
   - Context (why)
   - Files to create/modify
   - Requirements from design spec
   - Acceptance criteria

2. **Implement:**
   - Follow requirements exactly
   - Match existing codebase patterns
   - No over-engineering beyond spec

3. **Self-verify:**
   - Check acceptance criteria
   - Run relevant tests if they exist

4. **Commit when appropriate:**

   **Commit after:**
   - Each significant standalone task
   - A logical group of small related tasks (e.g., 1.1 + 1.2 + 1.3 if they form one unit)

   **Commit message format:**
   ```
   feat([topic]): [what was done]

   Task [X.Y]: [task title]
   - [key change 1]
   - [key change 2]
   ```

### Review Checkpoint

At each `### CHECKPOINT` in the impl-spec:

1. **Determine required reviews:**
   - Code review: always required
   - Security review: only if checkpoint specifies

2. **Spawn reviewers in parallel:**
   ```
   # Always:
   - planspec:code-reviewer --phase [N] --impl-spec [path]

   # Only if checkpoint specifies security review:
   - planspec:security-reviewer --phase [N] --impl-spec [path]
   ```

3. **Wait for all spawned reviewers to complete**

4. **Evaluate results:**

   **All pass → proceed to next phase**

   **Issues found → enter resolution loop:**

   ```
   WHILE issues exist:
     1. Read all issues from reviewers
     2. Attempt to fix each issue
     3. Commit fixes: "fix([topic]): address review feedback"
     4. Re-run same reviewers that were spawned (in parallel if multiple)
     5. IF same issues persist after 3 attempts:
          Surface to user with:
          - What was tried
          - What's still failing
          - Request guidance
          WAIT for user response
          Apply user guidance
          Continue loop
   ```

5. **Only proceed when all spawned reviewers pass**

### Step 3: Completion

After all phases complete:

1. **Final verification:**
   - All tasks marked complete in impl-spec
   - All tests passing
   - All checkpoints passed

2. **Update impl-spec:**
   - Change `status: in-progress` → `status: completed`
   - Add completion date

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

### Task Implementation Failure

When a task can't be implemented as specified:

```
⚠️ Task Implementation Blocked

Task: [X.Y] - [title]
Blocker: [specific issue]

The design spec may need revision, or there's missing context.

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

## Principles

- **Spec is source of truth** - Don't deviate without explicit approval
- **Atomic commits** - Each commit is a logical unit
- **Parallel reviews** - Code and security reviews run simultaneously
- **Fail fast, surface early** - Don't spin on unresolvable issues
- **Clean state** - Always leave repo in buildable, testable state

---

## Sub-Agent Protocol

This executor runs as a sub-agent and spawns its own sub-agents.

**When spawning reviewers:**
- Pass impl-spec path and current phase
- Reviewers return structured results (pass/fail + issues)
- Parse results to determine next action

**When reporting back:**
- Return completion status
- Include summary of work done
- Include any deviations from spec (with justification)
- Include branch name for next steps

---

## Quick Reference

| Input | Impl-spec file path |
|-------|---------------------|
| Output | Completed implementation on topic branch |
| Branch | `feature/[topic]` |
| Commits | After significant tasks |
| Reviews | Parallel at checkpoints |
| Escalation | After 3 failed resolution attempts |
