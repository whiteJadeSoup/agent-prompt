# Design Review Guide — Overview Design

## Why Review Exists

Review's purpose is to **discover the highest-cost mistakes before committing to implementation.**

The single question every reviewer must answer: **"If we build this, will we regret it?"**

Regret has three sources, in priority order:

1. **Wrong problem** — the solution is built, but the user's pain point remains
2. **Wrong direction** — problem is correct, but the chosen approach cannot achieve the goals
3. **Unrecognized reality** — direction is correct, but a hidden constraint or assumption will invalidate the plan mid-implementation

If source 1 is confirmed, stop reviewing — the plan needs to restart. Evaluating architecture and risks on a wrong problem wastes everyone's time.

"More elegant" or "better practice" are `[suggestion]`s, not `[blocking]`. A reviewer who finds a better approach should share it — but it does not block confirmation unless the current approach will cause regret.

---

## Execution Architecture (Multi-Agent)

Overview Review is executed by **3 parallel subagents + 1 orchestrator**. Each agent owns one source of regret.

| Agent | Regret Source | Core Question |
|-------|--------------|---------------|
| **Agent A: Problem** | Wrong problem | Does this plan solve the right problem for the right people? |
| **Agent B: Direction** | Wrong direction | Can this approach actually achieve the stated goals? |
| **Agent C: Reality** | Unrecognized reality | What hidden assumptions or constraints could invalidate this plan? |

**Before dispatching subagents**, the orchestrator injects into each prompt:
- The design document identifier — subagents read the document themselves
- The agent's assigned focus area and core question
- The complete contents of this review guide
- The JSON output schema below

Each subagent outputs structured findings:

```json
{
  "findings": [
    {
      "id": "A-001",
      "agent": "A",
      "location": "Goals & Non-Goals",
      "severity": "blocking",
      "angle": "wrong problem",
      "what": "...",
      "why": "...",
      "proof": "...",
      "suggestion": "..."
    }
  ]
}
```

**Proof must be concrete**: cite the specific claim in the document and the scenario that breaks it. If you cannot construct concrete proof, set `severity` to `question`, never `blocking`.

After all 3 subagents complete, the orchestrator:
1. **Deduplicates**: same location + same reason → keep highest severity, merge rationale
2. **Resolves conflicts**: Agent A's judgment takes priority for problem-level issues, Agent B's for direction-level
3. **Formats**: converts findings into the comment format below
4. **Concludes**: writes a mandatory conclusion

---

## How to Question

Two parallel questioning paths apply across all agents:

**Path 1 — Direction challenge** (is this claim the right answer?)
- Trace back to the Problem Statement — does this claim map to a real goal?
- Simplicity check: is there a simpler approach that achieves the same goals? If yes, why wasn't it chosen?

**Path 2 — Premise challenge** (does this claim hold in reality?)
- Identify the claim's hidden premises (dependency SLAs, data volume, call ordering, consistency guarantees, team capability)
- Construct a concrete scenario where the premise fails and describe the impact

*Example of premise challenge: "The design assumes the downstream service responds within 100ms. What happens under load when it takes 2s — does the architecture degrade gracefully or cascade-fail?"*

*Example of direction challenge: "The design uses an async queue to decouple producer and consumer. But the AC requires strong consistency — does async delivery actually satisfy that requirement?"*

---

## Agent A: Wrong Problem

**Core question**: Does this plan solve the right problem for the right people?

**Problem Statement**
- Is this a real pain point, or a solution disguised as a problem? ("Users need a search box" is a solution, not a problem.)
- Is there quantitative data supporting the pain point, or is it assumed?

**User Stories & Goals**
- Is the Role specific (not "user")? A generic role means the target user hasn't been identified.
- Is the Goal a behavior, not a feature? ("find a metric by keyword" not "a search box")
- Is "so that" present and meaningful? Remove it — does the team still know why this matters?
- Do the Goals actually address the stated Problem? Could you achieve all Goals and still leave the pain point unresolved?

**Acceptance Criteria**
- Do they cover all three paths: happy path, edge case, and error path?
- Can each AC be written as a standalone test case? If not, it's too vague to drive implementation or testing.

**Non-goals**
- Are they real exclusions — things stakeholders might actually request — or just things nobody would do anyway?
- Is there a missing non-goal that would prevent scope creep?

**Role completeness**
- Are all roles that interact with this feature represented (users, admins, downstream systems, ops)?

---

## Agent B: Wrong Direction

**Core question**: Can this approach actually achieve the stated goals?

**Architecture ↔ Goals**
- Walk through each Goal — does the architecture have a clear mechanism to achieve it?
- If a Goal has no corresponding architectural component, the design has a gap.
- Draw the connection: Goal → component → flow. If you can't draw it, it doesn't exist.

**Core Flows**
- Does the main flow complete end-to-end and deliver the goal?
- What scenario is not covered by any flow? (missing flows are missing design)
- Do the flows show what happens on the critical error paths?

**Key Decisions**
- Is there a genuine tradeoff, or only one option presented?
- Is the rationale grounded in real constraints (team size, timeline, existing system boundaries), or in habit and preference?
- Are rejected alternatives fairly described, or strawmanned to make the chosen option look better?
- If the stated constraint changed (e.g., timeline extended, team doubled), would the decision still hold?

**Simplicity**
- Could a simpler approach achieve the same goals? If yes, why wasn't it chosen — is the added complexity justified by a real constraint?

---

## Agent C: Unrecognized Reality

**Core question**: What hidden assumptions or constraints could invalidate this plan?

**Hidden assumptions**
- List every implicit assumption in the design: about dependency SLAs, data volume, call ordering, failure modes, consistency guarantees, team capability, deployment environment.
- For each assumption: is it explicitly stated in the document? If not, it's a hidden risk.

**Premise failures**
- For each assumption, construct a concrete scenario where it fails.
- Describe the impact: does the system degrade gracefully, or does it fail completely?

**Risks**
- Does the Risks section name the actual risks, or only safe-to-list ones?
- What is the single most likely failure mode of this design? Is it in the Risks section?
- What would have to be true for this design to fail completely? Is that scenario addressed?

**Open questions**
- Are open questions assigned a decision timeline?
- An open question with no deadline is a hidden dependency — it will block implementation at the worst moment.

---

## Writing Comments

**1. Location** — which section of the document (e.g., "Problem Statement", "Architecture Overview", "Key Decisions — Task Storage").

**2. Problem**
- **What**: the specific claim or gap in the document
- **Why**: why it matters. Every `[blocking]` comment must include a diagram or a concrete scenario showing the failure. A blocking comment without evidence is incomplete.
- **Proof**: the specific scenario that breaks the claim. Must be concrete and executable — "if X then Y" hypotheticals are not allowed. If you cannot construct concrete proof, downgrade to `[question]`.

**3. Suggestion** — a concrete direction to resolve the issue. If you see a better approach (even if the current one isn't blocking), share it as `[suggestion]`.

---

## Comment Labels

| Label | Meaning | Blocks confirmation |
|-------|---------|-------------------|
| `[blocking]` | Unresolved → system fails to achieve design goals | Yes |
| `[question]` | Need clarification on a claim or assumption | Depends on answer |
| `[suggestion]` | Design works, but a better approach exists | No |
| `[nit]` | Minor wording or clarity improvement | No |

**`[blocking]` standard**: if this issue is not resolved, the implemented system will fail to achieve the design goals. When uncertain, use `[question]`, not `[blocking]`.

---

## Conclusion

| Conclusion | Meaning |
|------------|---------|
| **Confirm** | Overview is sound — proceed to Detail Design |
| **Request Changes** | Blocking issues present; re-review required after fixes |
| **Comment** | Open questions pending; conclusion deferred |

Every review must end with an explicit conclusion. **The author must not begin Detail Design until the Overview is confirmed.**

---

## Author's Responsibility After Receiving Comments

Same rules as Code Review (Part 3 of code-review.md):
- `[blocking]` — revise the document, then re-request review
- `[question]` — explain your reasoning; reviewer decides if a change is needed
- `[suggestion]` — reply with "adopting", "adopting in follow-up", or "not adopting because X"
- `[nit]` — reply "done" or "not changing because X"

After addressing a comment, the **author resolves the thread** via document revision or rebuttal. "Acknowledged" is not sufficient. Only the author resolves threads.

Re-request review only after all `[blocking]` issues are resolved and all threads are closed.

---

## Re-review Responsibilities

- **Primary goal**: verify that all previous `[blocking]` and `[question]` findings are correctly addressed
- **New findings are allowed**: label them `[new finding]` — open a fresh comment, do not re-open original threads
- Re-review ends with an updated conclusion (Confirm or Request Changes)

---

## Phase 2: Detail Design Review

### Core Question

**"If we implement this Detail Design, can it fulfill every promise made in the Overview?"**

Two failure modes:
- **Missing coverage** — a Goal, AC, or Core Flow from Overview has no corresponding implementation in Detail
- **Broken contract** — the spec exists but has gaps that will cause incorrect behavior at the boundary

### Common Principles

1. **Skip sections that don't exist — but ask why.** Absent section + feature that needs it → `[blocking]`
2. **Every finding traces back to Overview.** Blocking standard: if unresolved, a Goal or AC from Overview cannot be satisfied.
3. **Find failure scenarios, not confirmations.** Find the input, state, or condition that breaks the spec.
4. **Forward compatibility is the default.** Any breaking change must explicitly state what breaks, who is affected, and the resolution steps. Breaking without explanation → `[blocking]`

### Execution Architecture

```
Section 0 (Coverage Check) — single agent, sequential
         ↓  [coverage map passed to all section agents]
Sections 1–8 — parallel, one agent per section
         ↓
Orchestrator — deduplicate, format, conclude
```

**Before dispatching section agents**, the orchestrator runs Section 0 first and injects its coverage map into each section agent's prompt. Each section agent also receives: the confirmed Overview Design, the Detail Design, and the complete contents of this guide.

Each agent outputs findings in the same JSON schema as Overview Review.

---

### Section 0: Coverage Check (always required)

Map every Goal, AC, and Core Flow from Overview to a section in Detail. Any unmapped item → `[blocking]`.

Pass the coverage map to all section agents.

---

### Section 1: Module Responsibilities & Interfaces

Play the role of a caller. Using only the Detail spec, can you implement the calling side correctly?

- Preconditions complete? Any hidden assumptions the caller won't know?
- All error cases defined? Can the caller distinguish different failure types?
- Return values unambiguous? Any fields "sometimes null" without explanation?

---

### Section 2: Data Model / Schema

Construct an illegal business state that the schema permits.

- Any field combination that's schema-valid but business-invalid? (e.g., `status=completed`, `completed_at=null`)
- Any cross-field dependencies not expressed as constraints?
- Indexes aligned with actual query patterns?

---

### Section 3: Core Algorithms / State Machines

Find an input or state that makes the algorithm produce a wrong result.

- Any legal event with no defined state transition?
- Do declared invariants hold on all execution paths?
- Does each Core Flow from Overview have a corresponding execution path here?

---

### Section 4: Error Handling

List all external interaction points. For each: is the failure mode named and handled?

- Can the caller distinguish transient from permanent failures?
- Are retries idempotent?
- Does error path behavior match the AC error path?

---

### Section 5: Key Flows

Compare against Overview's Core Flows.

- Every Core Flow has a detailed counterpart?
- All branches covered, not just main path?
- Parameters consistent with Section 1 interfaces?
- Every AC scenario has a complete execution path?

---

### Section 6: Performance Estimation

One question: **is the estimate credible?**

- Input assumptions (QPS, data size, latency) stated explicitly and sourced?
- Bottleneck on the critical path identified?
- Conclusion aligns with quantitative Goals?

---

### Section 7: Testing Strategy

- If User Stories exist: every AC has a corresponding test case?
- If technical change: pass criteria trace back to quantitative Goals?
- Mock boundaries justified? Does mocking hide real integration risks?

---

### Section 8: Migration & Compatibility

Forward compatibility is the default.

- Does the change break existing callers, data readers, or rollback paths?
- If yes: are risks, affected parties, and resolution steps explicitly stated? If not → `[blocking]`

---

### Conclusion

| Conclusion | Meaning |
|------------|---------|
| **Confirm** | Detail Design is sound — proceed to implementation |
| **Request Changes** | Blocking issues present; re-review required |
| **Comment** | Open questions pending; conclusion deferred |

Every Detail Review must end with an explicit conclusion.
