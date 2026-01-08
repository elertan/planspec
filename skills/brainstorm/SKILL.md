---
name: brainstorm
description: Collaborative design and specification workflow for turning ideas into detailed, implementation-ready designs. Use for feasibility checks ("is X possible?", "can we do Y?"), design exploration, and full specification. Triggers on "let's build", "I want to add", "how should we", "is it possible", "what would it take", "design", "architect", "spec out", or any non-trivial feature request. Skip ONLY for trivial single-location changes (typo, rename, color change).
---

# Brainstorm → Design → Specification

Transform ideas into validated, detailed design specifications through structured dialogue.

## When to Use

**USE this skill when:**
- User asks "is X possible?" or "can we do Y?" → Start at Phase 0
- User wants to build/add something new → Start at Phase 0 or 1
- User asks "how should we..." or "what's the best way to..."
- Any feature request that touches multiple files/components
- User explicitly mentions: design, architect, approach, brainstorm, spec
- You're uncertain about the right approach

**SKIP this skill when:**
- Single-location trivial change (typo fix, color change, rename)
- User explicitly says "just do it" or "skip brainstorming"
- Pure bug fix with obvious cause and solution

## Principles (Apply Throughout)

- **Extended thinking always** - Use extended thinking at every step. This is implicit in all instructions below—don't skip it. Think before acting, trace implications, question assumptions.
- **One question at a time** - Never overwhelm with multiple questions
- **Multiple choice preferred** - Easier than open-ended when feasible. Always accept free-form text responses—user can type anything and we infer intent. Multiple choice guides, not restricts.
- **YAGNI ruthlessly** - Cut unnecessary complexity from all designs
- **Incremental validation** - Present in chunks, confirm each before continuing
- **Be flexible** - Revisit earlier decisions when new information emerges
- **Verify, don't assume** - Discover actual constraints before designing. If you haven't inspected it, you don't know it. Assumptions must be flagged as uncertain.
- **Precision over impressiveness** - State exactly what an advantage does and doesn't protect against. Vague benefits mislead decisions. "Future-proof against X" must specify what X is and acknowledge what still breaks.
- **Assume failure** - For every operation or decision, ask: "What if this fails/goes wrong?" Don't design only the happy path. If something can fail, the design must address: (1) how we detect it, (2) whether it's recoverable, (3) what we do about it.

---

## Backtracking

Later phases may reveal issues that invalidate earlier decisions. This is normal—don't force-fit.

**When to backtrack:**
- Phase 3 discovers constraints that break Phase 2 success criteria
- Phase 4 design reveals the chosen approach won't work
- New information contradicts earlier assumptions

**How to backtrack:**
1. Surface what changed: "While designing X, I found Y, which means Z won't work"
2. Return to the earliest affected phase
3. Re-validate from that point forward (don't skip phases)
4. Acknowledge sunk cost but don't let it drive decisions

**Don't backtrack for:**
- Minor refinements (just update in place)
- User changing their mind on preferences (adapt current phase)

---

## Exit: Don't Build

Sometimes the right answer is "don't build this." This can emerge at any phase.

**Signs to consider not building:**
- The problem isn't worth solving (low impact, rare occurrence)
- Cost vastly exceeds value (complexity, maintenance burden, risk)
- A simpler non-code solution exists (process change, existing tool, manual workaround)
- All approaches have blocking issues that can't be mitigated
- The problem is a symptom; the root cause needs different treatment

**How to exit:**
1. Surface the finding clearly: "Based on X, I recommend not building this because Y"
2. Present the evidence (not just opinion)
3. Offer alternatives if they exist (simpler solution, deferral, different framing)
4. Let user decide—they may have context you don't

**Don't confuse "don't build" with:**
- "This is hard" (hard isn't a reason to skip)
- "I don't know how" (investigate further)
- "The user didn't ask for this exactly" (clarify scope instead)

---

## Phase 0: Feasibility Check

**Goal:** Quick assessment before committing to full design. Answer: "Is this viable? What would it take?"

**When to use Phase 0:**
- User asks "is X possible?" or "can we do Y?"
- User wants a gut check before investing time
- Exploratory questions: "what would it take to..."
- You sense the request might have obvious blockers

**When to skip Phase 0:**
- User explicitly wants full design ("let's design X", "spec out Y")
- You already know it's feasible from context
- User says "I know it's possible, let's figure out how"

### Steps

1. **Understand the ask** — What are they really asking? What would "yes, feasible" mean?

2. **Quick discovery** — Spend minimal time checking for obvious blockers:
   - Does the required data/API/capability exist?
   - Are there clear technical impossibilities?
   - Are there obvious permission/access issues?
   - Does this conflict with existing architecture in a fundamental way?

   **Time-box this.** 5-10 minutes of investigation, not exhaustive research.

3. **Surface findings** — Present one of:

   **Feasible:**
   ```
   "Yes, this looks feasible. [1-2 sentence reasoning]

   Key considerations:
   - [consideration 1]
   - [consideration 2]

   Want me to do a full design?"
   ```

   **Blocked:**
   ```
   "This has a blocking issue: [specific blocker]

   [Why it's blocking, not just hard]

   Alternatives to consider:
   - [alternative 1]
   - [alternative 2]"
   ```

   **Needs investigation:**
   ```
   "I can't determine feasibility yet. Need to investigate:
   - [specific unknown 1]
   - [specific unknown 2]

   Want me to dig deeper, or do you have this context?"
   ```

4. **Gate decision** — Based on user response:
   - "Yes, full design" → Proceed to Phase 1
   - "No, just wanted to know" → Done
   - "Investigate more" → Do targeted discovery, return to step 3
   - User provides context → Re-evaluate, return to step 3

### Exit Criteria

- User has feasibility answer, OR
- User has decided to proceed to full design, OR
- User has decided not to proceed

---

## Phase 1: Understand the Problem

**Goal:** Know WHY before exploring HOW. Understand actual constraints, not assumed ones.

### Steps

1. Read user's request to identify:
   - What they're explicitly asking for
   - What underlying problem they might be solving
   - What assumptions they (and you) are making

2. **[Ask]** Confirm scope before deep discovery:
   - If scope is ambiguous (e.g., "add auth" could mean basic token check or full OAuth2+MFA), ask ONE clarifying question
   - Present 2-3 scope options: "Are you thinking [minimal], [moderate], or [comprehensive]?"
   - This prevents wasted discovery on the wrong scope
   - Skip if scope is already clear from context

3. **Discover** the problem space:

   **Depth guidance:** Focus discovery on what's needed to assess viability. Don't exhaustively map everything—go deep only where uncertainty blocks decisions. You can always revisit later.

   a) **Internal context:**
      - Read relevant code files
      - Trace data flow and dependencies
      - Identify existing patterns and conventions
      - Check git history if recent changes are relevant

   b) **External context** (when solution touches external systems):
      - Identify external touchpoints: APIs, services, third-party tools
      - Inspect directly: docs, endpoints, auth flows
      - Document facts that affect viability: auth requirements, data structures, rate limits

   c) **Constraint mapping:**
      - **REQUIRE:** What does the solution need? (data fields, permissions, capabilities, access)
      - **AVAILABLE:** What exists? (verified through discovery, not assumed)
      - **GAPS:** What's missing or blocking? (surface immediately)
      - **UNCERTAIN:** What couldn't be verified? (must be resolved, not deferred)

4. If gaps/blockers found, surface to user immediately—may require pivoting

5. **Resolve uncertainties** — For each UNCERTAIN item, ask user ONE at a time:
   - Can they provide the answer?
   - Should we investigate further? (propose how)
   - Do they want to explicitly skip/defer? (acknowledge risk)

   **Do not proceed with unresolved uncertainties.** Either resolve them or get explicit user acknowledgment to defer.

6. Ask ONE additional clarifying question if needed (prefer multiple choice)

### Exit Criteria

Proceed to Phase 2 when:
- You can articulate the core problem and why it matters
- You've verified enough constraints to know an approach is viable
- Blockers have been surfaced and resolved (or user accepts the risk)
- All uncertainties are resolved OR user explicitly deferred them (with acknowledged risk)

---

## Phase 2: Define Success

**Goal:** Establish what "done" looks like—functional requirements, quality attributes, and boundaries.

### Steps

1. Draft success criteria based on Phase 1 understanding—trace what the user actually needs vs. what they asked for

2. Identify **non-functional requirements** (as relevant):
   - Performance targets (latency, throughput, resource limits)
   - Availability/reliability expectations
   - Security requirements
   - Scalability needs (current vs. projected load)
   - Compatibility constraints (browsers, devices, integrations)

3. Define **anti-goals** (what we're explicitly NOT building):
   - Features that seem adjacent but are out of scope
   - Quality attributes we're intentionally not optimizing for
   - This prevents scope creep and clarifies tradeoff decisions later

4. Surface **constraints**:
   - Tech stack restrictions (must use X, cannot use Y)
   - Timeline pressures (if any)
   - Team capacity/expertise limitations
   - Budget or resource limits

5. Present to user for validation:
   ```
   "Based on our discussion, success means:

   **Must have:**
   - [criterion 1]
   - [criterion 2]

   **Quality attributes:**
   - [e.g., Response time < 200ms for 95th percentile]

   **Not building:**
   - [anti-goal 1]

   **Constraints:**
   - [constraint 1]

   Does this capture what you need?"
   ```

6. Refine until user confirms—question whether refinements actually improve or just add complexity

### Exit Criteria

User has confirmed:
- Success criteria (what we're building)
- Non-functional requirements (how well it must work)
- Anti-goals (what we're NOT building)
- Constraints acknowledged

---

## Phase 3: Explore Approaches

**Goal:** Deeply understand what each approach entails in your specific context, then present informed options.

### Steps

1. Brainstorm 2-4 candidate approaches based on Phase 1 understanding

2. **Compatibility Analysis** — For each candidate, assess fit with your context.

   **Scale depth to risk:** Low-risk approaches need quick sanity checks. High-risk approaches (new tech, external dependencies, architectural changes) need thorough analysis.

   a) **Characterize** — What's different from current state? Pick relevant lenses:

   | Lens | Key questions |
   |------|---------------|
   | Performance | Latency? Throughput? At scale? |
   | Reliability | Failure modes? Recovery? |
   | Dependencies | What must exist first? |
   | Expertise | Can the team build and maintain this? |
   | Operational | Deploy? Monitor? Debug? Secure? |
   | Integration | Clean fit or adapters needed? |

   b) **Touchpoints** — What interacts? (code, infra, people, processes)

   c) **Find friction** — Where do approach characteristics conflict with touchpoint requirements? Only deep-dive on potential mismatches.

   d) **Classify findings**:
   - **Blocker:** non-viable
   - **Constraint:** works only if X / only for Y
   - **Mitigation needed:** requires additional work
   - **Acceptable tradeoff:** mismatch exists but tolerable because Z

3. **Prune and refine**:
   - Drop approaches where blockers make them non-viable
   - Scope approaches where constraints limit applicability
   - Note required mitigations as part of the approach
   - If all approaches have blockers, surface to user—may need to revisit problem framing

4. **Evaluate remaining approaches** against:
   - Success criteria from Phase 2
   - Implementation complexity (informed by actual dependencies/expertise needs)
   - Maintainability (informed by actual ownership/change patterns)
   - Existing codebase patterns

5. **Present approaches** with traced-through specifics:

   ```
   "I see two viable approaches:

   **Option A: [name]** (recommended)
   [What it is - 1-3 lines]

   ✓ [Specific advantage with concrete detail]
   ✗ [Specific tradeoff with concrete impact]

   Requires: [dependencies, mitigations, or constraints from analysis]
   Works for: [scope, if limited]

   **Option B: [name]**
   ...

   I recommend A because [reasoning referencing specific touchpoints and requirements from analysis]. Thoughts?"
   ```

   **Critical:** Tradeoffs must be specific and traced, not generic.
   - ✗ "might be slower" → ✓ "adds ~500ms, exceeds 200ms SLA on `/api/search`"
   - ✗ "more complex" → ✓ "requires Redis, which we don't run—adds operational overhead"
   - ✗ "could break" → ✓ "breaks on HTML selector changes—estimated quarterly per their changelog"

6. Discuss until approach is selected—if user raises new considerations, trace them through the analysis

### Exit Criteria

- User has confirmed the approach to pursue
- Constraints and required mitigations are acknowledged
- Scope limitations (if any) are understood

---

## Phase 4: Present Design

**Goal:** Validate design incrementally, ensuring enough detail for implementation spec.

### Steps

1. Present design ONE section at a time (~200-300 words each)
2. After each section, ask: "Does this make sense? Any changes?"
3. Cover these sections (skip if not applicable):

   | Section | What to validate | Skip if |
   |---------|------------------|---------|
   | **Architecture Overview** | Components, interactions, integration points | Never skip |
   | **Data Model** | New/modified structures, state management | No new data structures |
   | **Interfaces** | APIs, contracts, events | No new APIs or contracts |
   | **Behavior** | Happy path, edge cases, error handling, state transitions | Never skip |
   | **Testing Requirements** | Critical paths, edge cases, integration points | Never skip |

4. For each section, ensure you've captured enough detail that an implementer could:
   - Know what files/components to create or modify
   - Understand the data structures involved
   - Know the expected behavior for all cases (not just happy path)
   - Know what to test

5. Revise any section based on feedback—trace how changes ripple to other sections

### Exit Criteria

All applicable sections presented and validated with implementation-ready detail.

---

## Phase 5: Write Specification

**Goal:** Capture validated design in durable format.

### Steps

1. Create design spec at `planspec/designs/[topic-slug].md`—verify all validated decisions are captured:

```markdown
---
date: YYYY-MM-DD
status: approved
impl-spec: planspec:impl-spec
---

# [Design Title]

> **Next step:** `planspec:impl-spec planspec/designs/[topic].md`

## Problem

[Detailed problem statement - why this matters]

## Success Criteria

[From Phase 2 - what "done" looks like]

**Must have:**
- [criterion]

**Quality attributes:**
- [non-functional requirements]

**Not building:**
- [anti-goals]

## Approach

[Chosen approach and rationale]

### Alternatives Considered

| Alternative | Why Not |
|-------------|---------|
| [Option B] | [Specific reason - cost, complexity, mismatch with constraints] |
| [Option C] | [Specific reason] |

## Design

### Architecture Overview

[How components connect. Include diagram if complex.]

- **Components:** [List new/modified components and their responsibilities]
- **Interactions:** [How they communicate - sync/async, data flow direction]
- **Integration points:** [Where this connects to existing system]

### Data Model

[Skip if no new data structures]

- **New structures:** [Schemas, types, models being introduced]
- **Modified structures:** [Changes to existing data]
- **State management:** [Where state lives, how it's accessed]

### Interfaces

[Skip if no new APIs/contracts]

- **External APIs:** [Endpoints, request/response shapes]
- **Internal interfaces:** [Function signatures, contracts between components]
- **Events/hooks:** [If event-driven, what events are emitted/consumed]

### Behavior

- **Happy path:** [Primary flow from start to end]
- **Edge cases:** [Boundary conditions and how they're handled]
- **Error handling:** [What can fail, how each failure is handled, user-facing messages]
- **State transitions:** [If stateful, what states exist and what triggers transitions]

### Testing Requirements

[What needs to be tested - not how, but what]

- **Critical paths:** [Flows that must work]
- **Edge cases to cover:** [Specific scenarios]
- **Integration points:** [What external interactions need verification]

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| [What could go wrong] | Low/Med/High | [Consequence] | [How we address it] |

## Dependencies

- **Blocking:** [Must exist before we can build]
- **Non-blocking:** [Nice to have, can work around]

## Open Questions

[ONLY items user explicitly deferred. Should be empty in most specs.
If present, note why deferred and what risk was acknowledged.]
```

2. Use clear, concise writing (no fluff)—question every sentence for necessity

3. **Open Questions should be rare** — If this section has items, each must be:
   - Explicitly deferred by user during brainstorming (not forgotten/unasked)
   - Genuinely unknowable until implementation begins
   - Accompanied by the acknowledged risk

   Many open questions indicate incomplete Phase 1. Consider whether to revisit.

4. Commit with message: `docs: add [topic] design spec`

5. Ask user: **"Ready to create the implementation plan?"**

6. If yes, invoke `planspec:impl-spec [path-to-design-spec]` to generate the implementation plan

---

## Quick Reference

| Phase | Goal | Exit When |
|-------|------|-----------|
| 0. Feasibility | Quick viability check | User has answer, or decides to proceed/not proceed |
| 1. Understand | Know the WHY + verify constraints | Approach viable, blockers surfaced, uncertainties resolved |
| 2. Success | Define done + boundaries | User confirms criteria, NFRs, anti-goals, constraints |
| 3. Explore | Assess approaches, present options | User selects approach, constraints acknowledged |
| 4. Design | Validate implementation-ready detail | All sections confirmed with enough detail to implement |
| 5. Spec | Document | File committed, ready for implementation spec |
