---
name: impl-spec
description: Generate implementation specs from design specs. Takes a design spec file and produces a detailed, phased task breakdown ready for autonomous execution. Use after brainstorming is complete and design spec is written. Invoke with the design spec file path as argument.
---

# Implementation Spec Writer

Transform validated design specs into actionable, phased implementation plans.

## Usage

```
planspec:impl-spec planspec/designs/[topic].md
```

## Input Requirements

The design spec must contain (as produced by `planspec:brainstorm`):
- Problem statement
- Success criteria (must have, quality attributes, anti-goals)
- Approach with alternatives considered
- Design (architecture, data model, interfaces, behavior, testing requirements)
- Risks and mitigations
- Dependencies

## Output

Creates `planspec/implementations/[topic].md` with:
- Phased task breakdown
- Review checkpoints
- Security review gates (where applicable)

---

## Process

### Step 1: Validate Input

1. Read the design spec file
2. Verify all required sections exist:
   - Problem
   - Success Criteria
   - Approach
   - Design (with sub-sections)
   - Risks
   - Dependencies

3. If sections missing, abort with specific feedback:
   ```
   "Cannot generate implementation spec. Design spec missing:
   - [missing section 1]
   - [missing section 2]

   Run planspec:brainstorm to complete the design spec first."
   ```

### Step 2: Analyze Codebase

1. Read existing codebase structure:
   - File organization patterns
   - Naming conventions
   - Existing similar implementations (for consistency)
   - Test file locations and patterns

2. Map design components to concrete file paths:
   - Where should new files go?
   - What existing files need modification?
   - Where do tests live?

3. Identify project conventions:
   - Import style
   - Error handling patterns
   - Logging patterns
   - Test framework and patterns

### Step 3: Determine Review Requirements

From the design spec, extract:

1. **Security review needed if** design spec mentions:
   - Authentication/authorization
   - User input handling
   - Database queries
   - API keys/secrets
   - File system access
   - External service calls with credentials
   - Any item in Risks related to security

2. **Mark phases for security review** based on which tasks touch these areas

### Step 4: Generate Task Breakdown

#### Task Granularity Rules

Each task should be:
- **Atomic:** One logical unit of work
- **Verifiable:** Clear acceptance criteria
- **Contextual:** Includes why, not just what
- **Bounded:** Completable in one focused session

#### Task Structure

```markdown
### Task [Phase].[Number]: [Action verb] [what]

**Context:** [Why this task exists - link to design decision]

**Files:**
- Create: `path/to/new/file.ts`
- Modify: `path/to/existing/file.ts` - [what changes]

**Requirements:**
- [Specific requirement from design spec]
- [Another requirement]

**Acceptance Criteria:**
- [ ] [Verifiable condition - can be checked]
- [ ] [Another verifiable condition]

**Dependencies:** None | Task X.Y, Task X.Z
```

#### Phase Structure

Group tasks into logical phases:

```markdown
## Phase 1: [Foundation/Core/Setup]

[Tasks that must come first - shared infrastructure, types, utilities]

### Task 1.1: ...
### Task 1.2: ...

---

## Phase 2: [Core Implementation]

[Main feature implementation]

### Task 2.1: ...
### Task 2.2: ...
### Task 2.3: Write tests for Phase 2

**Tests should cover:**
- [Critical path from design spec]
- [Edge cases from design spec's Behavior section]
- [Error cases from design spec]

### CHECKPOINT: Review

**Run:** `planspec:code-reviewer`
- Reviews code quality and patterns
- Runs tests to verify correctness
- Fix any issues before proceeding

---

## Phase 3: [Integration/Security-Sensitive]

### Task 3.1: ...
### Task 3.2: Write tests for Phase 3

### CHECKPOINT: Review + Security

**Run:** `planspec:code-reviewer`
**Run:** `planspec:security-reviewer`
- Security review for auth/input handling/etc.
- Fix any issues before proceeding
```

### Step 5: Map Design to Tasks

| Design Section | Maps To |
|----------------|---------|
| Architecture Overview | Phase 1 tasks (setup, structure) |
| Data Model | Early tasks (types, schemas, migrations) |
| Interfaces | API/contract tasks |
| Behavior - Happy Path | Core implementation tasks |
| Behavior - Edge Cases | Test requirements |
| Behavior - Error Handling | Implementation + test tasks |
| Testing Requirements | Test tasks per phase |
| Risks - Security | Security review checkpoints |
| Dependencies - Blocking | Task dependencies, Phase 1 setup |

### Step 6: Write Implementation Spec

Create file at `planspec/implementations/[topic].md`:

```markdown
---
date: YYYY-MM-DD
design-spec: ../designs/[topic].md
status: ready
executor: planspec:impl-spec-executor
---

# Implementation: [Title]

> **Execute with:** `planspec:impl-spec-executor planspec/implementations/[topic].md`

## Overview

[1-2 sentence summary of what we're implementing]

**Design spec:** [link to design spec]
**Security review required:** Yes/No (phases X, Y)

## Prerequisites

- [ ] [Dependency from design spec]
- [ ] [Another dependency]

## Phase 1: [Name]

### Task 1.1: [Title]
...

---

## Phase 2: [Name]

### Task 2.1: [Title]
...

### Task 2.N: Tests for Phase 2

**Test file:** `path/to/tests/feature.test.ts`

**Cover:**
- [ ] [Happy path scenario]
- [ ] [Edge case 1]
- [ ] [Edge case 2]
- [ ] [Error case 1]

### CHECKPOINT

| Review | Required | Focus |
|--------|----------|-------|
| Code review | Yes | Quality, patterns, correctness |
| Security review | No | - |

**Gate:** All tests pass, review issues resolved before Phase 3.

---

## Completion Checklist

- [ ] All tasks completed
- [ ] All tests passing
- [ ] All review checkpoints passed
- [ ] Design spec success criteria met:
  - [ ] [Criterion 1]
  - [ ] [Criterion 2]
```

### Step 7: Output Summary

After writing the file:

```
Implementation spec created: planspec/implementations/[topic].md

Summary:
- Phases: [N]
- Tasks: [M]
- Review checkpoints: [X]
- Security reviews required: [Y/N] (phases: [list])

Prerequisites to verify:
- [Any blocking dependencies]

Ready to execute with: planspec:impl-spec-executor planspec/implementations/[topic].md
```

---

## Principles

- **Traceability:** Every task links back to design decisions
- **No gaps:** Design spec coverage should be 100%
- **No invention:** Don't add tasks not justified by design spec
- **Concrete paths:** Use real file paths from codebase analysis
- **Meaningful tests:** Only where they verify correctness, not for coverage theater
- **Review gates:** Prevent forward progress with broken/insecure code

---

## Quick Reference

| Input | Design spec file path |
|-------|----------------------|
| Output | `planspec/implementations/[topic].md` |
| Phases | Logical groupings with review gates |
| Tasks | Mid-level granularity, verifiable |
| Tests | Per-phase, meaningful edge cases |
| Reviews | Code review every phase, security review where flagged |
