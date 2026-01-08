---
name: code-reviewer
description: Review code at implementation checkpoints. Verifies code quality, runs tests, checks against design spec requirements. Use at phase completion checkpoints during implementation. Can be invoked standalone or as part of impl-spec execution.
---

# Code Reviewer

Review implemented code for quality, correctness, and alignment with design spec.

## Usage

At implementation checkpoints:
```
planspec:code-reviewer
```

With specific context:
```
planspec:code-reviewer --phase 2 --impl-spec planspec/implementations/feature.md
```

## When to Use

- After completing a phase of implementation tasks
- Before moving to the next phase
- When impl-spec checkpoint says "Run: `planspec:code-reviewer`"

## Process

### Step 1: Gather Context

1. **Identify what to review:**
   - If impl-spec provided: read completed phase tasks
   - Otherwise: check git diff for recent changes
   - Identify files created/modified

2. **Load design context:**
   - Find related design spec (from impl-spec or ask)
   - Extract relevant success criteria
   - Extract expected behavior (happy path, edge cases, errors)

### Step 2: Run Tests

1. **Find and run relevant tests:**
   ```
   # Detect test runner from project
   npm test / pytest / go test / etc.
   ```

2. **Report test status:**
   - ✅ All tests pass → continue review
   - ❌ Tests fail → stop, report failures, do not proceed

3. **Check test coverage relevance:**
   - Are the critical paths from design spec tested?
   - Are edge cases covered?
   - Missing tests are flagged (not blocking, but noted)

### Step 3: Code Quality Review

Review each changed file for:

#### 3.1 Correctness
- Does it implement what the design spec describes?
- Does the behavior match expected happy path?
- Are edge cases handled as specified?
- Does error handling match design spec?

#### 3.2 Code Quality
- **Clarity:** Is intent obvious? Would another dev understand it?
- **Simplicity:** Is there unnecessary complexity? Over-engineering?
- **Consistency:** Does it match existing codebase patterns?
- **Naming:** Are names descriptive and consistent with codebase conventions?

#### 3.3 Common Issues
- Unused imports/variables
- Hardcoded values that should be configurable
- Missing error handling
- Silent failures (catch without handling)
- TODO/FIXME without context
- Commented-out code

#### 3.4 Performance (if relevant)
- Obvious inefficiencies (N+1 queries, unnecessary loops)
- Missing indexes for database queries
- Unbounded operations (no limits/pagination)

### Step 4: Report Findings

Structure findings by severity:

```markdown
## Code Review: Phase [N]

### Test Results
✅ All tests passing (X tests)
- [test summary]

### Issues Found

#### 🔴 Blocking (must fix before proceeding)
- **[File:Line]** [Issue description]
  - Why: [Impact/risk]
  - Fix: [Specific suggestion]

#### 🟠 Should Fix (important but not blocking)
- **[File:Line]** [Issue description]
  - Why: [Impact/risk]
  - Fix: [Specific suggestion]

#### 🟡 Suggestions (minor improvements)
- **[File:Line]** [Suggestion]

### Missing Test Coverage
- [ ] [Scenario not tested]
- [ ] [Edge case not covered]

### Design Spec Alignment
✅ Success criteria [X]: Implemented correctly
⚠️ Success criteria [Y]: Partially implemented - [what's missing]
```

### Step 5: Resolution Loop

1. **If blocking issues exist:**
   - List specific fixes needed
   - Wait for fixes to be applied
   - Re-run tests
   - Re-review fixed areas
   - Repeat until clean

2. **If no blocking issues:**
   - Note should-fix items for follow-up
   - Approve phase completion
   - Clear to proceed to next phase

---

## Review Checklist (Quick Reference)

### Always Check
- [ ] Tests pass
- [ ] Implements design spec requirements
- [ ] Error handling present and matches spec
- [ ] No obvious bugs or logic errors
- [ ] Code is readable and maintainable

### Pattern Consistency
- [ ] Follows existing code style
- [ ] Uses existing utilities/helpers where available
- [ ] Error messages are consistent with codebase
- [ ] Logging follows project conventions

### Red Flags
- [ ] No hardcoded secrets/credentials
- [ ] No SQL string concatenation
- [ ] No unvalidated user input used directly
- [ ] No unbounded resource consumption
- [ ] No silent error swallowing

---

## Output

**On success:**
```
✅ Code Review Passed

Tests: 12 passing
Issues: 0 blocking, 0 should-fix, 2 suggestions
Design alignment: All criteria met

Cleared to proceed to Phase [N+1].
```

**On failure:**
```
❌ Code Review: Issues Found

Tests: 10 passing, 2 failing
Blocking issues: 3

[Detailed issue list]

Fix these issues before proceeding.
```

---

## Principles

- **Actionable feedback:** Every issue includes why and how to fix
- **Severity clarity:** Blocking vs should-fix vs suggestion
- **Design alignment:** Always check against spec, not just code style
- **No nitpicking:** Focus on correctness and maintainability, not preferences
- **Tests first:** Failing tests = stop everything else
