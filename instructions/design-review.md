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

Overview Review is executed by **3 parallel sub-agents + 1 orchestrator**. Each sub-agent owns one source of regret and runs concurrently; the orchestrator interrogates their findings and synthesizes the review. Model assignment follows the **Subagent Execution** umbrella in `CLAUDE.md`.

| Agent | Regret Source | Model | Core Question |
|-------|--------------|-------|---------------|
| **Agent A: Problem** | Wrong problem | Sonnet | Does this plan solve the right problem for the right people? |
| **Agent B: Direction** | Wrong direction | Sonnet | Can this approach actually achieve the stated goals? |
| **Agent C: Reality** | Unrecognized reality | Sonnet | What hidden assumptions or constraints could invalidate this plan? |

**Before dispatching sub-agents**, the orchestrator injects into each prompt: the design document identifier (sub-agents read the document themselves), the agent's assigned focus area and core question, the complete contents of this review guide, and the findings schema.

> Findings use the JSON schema in **code-review.md › Sub-agent Output Format**, with `agent` set to the regret-source (A/B/C), `angle` to the regret type (e.g. "wrong problem"), and `location` to the document section.

**Proof must be concrete**: cite the specific claim in the document and the scenario that breaks it. Set `severity` to `question` whenever you cannot construct concrete proof — never set it to `blocking` without proof.

### Orchestrator

The orchestrator runs on **Opus**.

> Orchestration follows the challenge-to-consensus loop in **CLAUDE.md › Subagent Execution**.

Once the findings reach consensus, the orchestrator:
- **Deduplicates**: same location + same reason → keep highest severity, merge rationale
- **Resolves conflicts**: Agent A's judgment takes priority for problem-level issues, Agent B's for direction-level
- **Formats**: convert findings into the comment format below
- **Concludes**: write a mandatory conclusion

### Execution Flow

```
Design Doc
  ├─→ Agent A (wrong problem)         ─ Sonnet ┐
  ├─→ Agent B (wrong direction)       ─ Sonnet ├─ parallel
  └─→ Agent C (unrecognized reality)  ─ Sonnet ┘
                  │ findings JSON
                  ▼
        Orchestrator ─ Opus
          challenge ↔ re-engage (≤3)
          dedup · resolve · format · conclude
                  ▼
            Final Review
```

---

## How to Question

Two questioning paths apply across all sub-agents:

**Path 1 — Direction challenge** (is this claim the right answer?)
- Trace back to the Problem Statement — does this claim map to a real goal?
- Simplicity check: is there a simpler approach that achieves the same goals? If yes, why wasn't it chosen?

**Path 2 — Premise challenge** (does this claim hold in reality?)
- Identify the claim's hidden premises (dependency SLAs, data volume, call ordering, consistency guarantees, team capability)
- Construct a concrete scenario where the premise fails and describe the impact

<example>
Premise challenge: "The design assumes the downstream service responds within 100ms. What happens under load when it takes 2s — does the architecture degrade gracefully or cascade-fail?"
</example>

<example>
Direction challenge: "The design uses an async queue to decouple producer and consumer. But the Goal requires strong consistency — does async delivery actually satisfy that requirement?"
</example>

---

## Agent A: Wrong Problem

**Core question**: Does this plan solve the right problem for the right people?

**Problem Statement**
- Is this a real pain point, or a solution disguised as a problem? ("Users need a search box" is a solution, not a problem.)
- Is there quantitative data supporting the pain point, or is it assumed?

**Goals & Non-Goals**
- Is each Goal verifiable — quantified, or explicitly marked "to be quantified in Detail Design"?
- Performance / capacity / SLA Goals: are they quantified in Overview? Deferral is not allowed for these.
- Does each Goal trace back to a pain point in Problem Statement? Could all Goals be met while leaving the original pain unresolved?
- Are Non-Goals real exclusions (things stakeholders might actually request), not "things nobody would do anyway"?

---

## Agent B: Wrong Direction

**Core question**: Can this approach actually achieve the stated goals?

**Architecture covers each Goal** (static structure)
- Walk through each Goal — does the architecture have a component responsible for it?
- If a Goal has no corresponding architectural component, the design has a gap.

**Flows carry each Goal** (dynamic execution)
- Does the core flow complete end-to-end and deliver each Goal?
- Are 1–2 edge flows present for the critical error / boundary paths?
- Are flows at Overview granularity (modules / services / roles, **no function names**)?

**Research & Comparison**
- Was web search actually performed? Are alternatives concrete (industry implementations), or hand-waved?
- Are rejected alternatives fairly described, or strawmanned to make the chosen option look better?
- Does the chosen solution actually align with each Goal — including any quantified targets?
- If a stated constraint changed (e.g., timeline extended, team doubled), would the decision still hold?

**Simplicity**
- Could a simpler approach achieve the same Goals? If yes, why wasn't it chosen — is the added complexity justified by a real constraint?

---

## Agent C: Unrecognized Reality

**Core question**: What hidden assumptions or constraints could invalidate this plan?

**Hidden assumptions**
- List every implicit assumption in the design: about dependency SLAs, data volume, call ordering, failure modes, consistency guarantees, team capability, deployment environment.
- For each assumption: is it explicitly stated in the document? If not, it's a hidden risk.

**Premise failures**
- For each assumption, construct a concrete scenario where it fails.
- Describe the impact: does the system degrade gracefully, or does it fail completely?

**Risks (both types required)**
- **Type A — cost of choosing**: are the trade-offs from picking this option over the alternatives stated honestly, or only soft costs?
- **Type B — intrinsic fragility**: does the document name failure modes that exist regardless of alternatives — single points of failure, key assumptions, scale cliffs?
- What is the single most likely failure mode of this design? Is it covered by either Type A or Type B?

**Open questions** (optional section)
- If the section exists: are open questions assigned a decision timeline and owner? An open question with no deadline is a hidden dependency.
- If the section is absent: did web search genuinely close all questions, or did the author skip the section to avoid surfacing uncertainty?

---

## Writing Comments

**1. Location** — which section of the document (e.g., "Problem Statement", "Solution Design — Architecture", "Research & Comparison — Task Storage").

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

<gate>
Every review must end with an explicit conclusion. The author must not begin Detail Design until the Overview is confirmed.
</gate>

---

## Author's Responsibility After Receiving Comments

Same rules as **code-review.md › Part 3: Author's Responsibility After Receiving Comments** — the `[blocking]` / `[question]` / `[suggestion]` / `[nit]` response rules and thread-resolution rules apply unchanged to design comments.

---

## Re-review Responsibilities

Same as **code-review.md › Part 3.5: Re-review Responsibilities** — verify each prior `[blocking]`/`[question]` is addressed; label any new problem `[new finding]` in a fresh comment. Re-review ends with an updated conclusion (Confirm or Request Changes).

---

## Phase 2: Detail Design Review

### Core Question

**"If we implement this Detail Design, can it fulfill every promise made in the Overview?"**

Two failure modes:
- **Missing coverage** — a Goal or Flow from Overview has no corresponding implementation in Detail
- **Broken contract** — the spec exists but has gaps that will cause incorrect behavior at the boundary

### Common Principles

1. **Skip sections that don't exist — but ask why.** Absent section + feature that needs it → `[blocking]`
2. **Every finding traces back to Overview.** Blocking standard: if unresolved, a Goal or Flow from Overview cannot be satisfied.
3. **Find failure scenarios, not confirmations.** Find the input, state, or condition that breaks the spec.
4. **Forward compatibility is the default.** Any breaking change must explicitly state what breaks, who is affected, and the resolution steps. Breaking without explanation → `[blocking]`

### Execution Architecture

Detail Review is executed by **1 orchestrator + parallel section sub-agents**. Section 0 (Coverage Check) runs first and its coverage map is injected into every section sub-agent; Sections 1–8 then run as parallel sub-agents. Model assignment follows the **Subagent Execution** umbrella in `CLAUDE.md`.

```
Overview Design + Detail Design + Guide
       │
       ▼
   Section 0 (Coverage Check) ─ produces coverage map
       │   (map injected into every section sub-agent below)
       ▼
  ├─→ Section 1 (Module Responsibilities)  ─ Sonnet ┐
  ├─→ Section 2 (Data Model)               ─ Sonnet ├─ parallel
  └─→ …Sections 3–8…                       ─ Sonnet ┘
                  │ findings JSON
                  ▼
        Orchestrator ─ Opus
          challenge ↔ re-engage (≤3)
          dedup · format · conclude
                  ▼
            Final Review
```

> Orchestration follows the challenge-to-consensus loop in **CLAUDE.md › Subagent Execution**.

### Execution Protocol

1. **Run Section 0 first**: build the coverage map (every Goal and Flow from Overview → which Detail section addresses it). Any unmapped item is a `[blocking]` finding immediately. Inject the coverage map into every section sub-agent's prompt.
2. **Dispatch Sections 1–8 in parallel**: each section sub-agent receives the confirmed Overview Design, the Detail Design, this guide, the coverage map, and its section's questions (below). Skip a section only when the Detail Design genuinely has no content for it — and ask why the absence is acceptable; if a feature requires that section, the absence itself is `[blocking]`.
3. **Challenge to consensus**: follow the loop in **CLAUDE.md › Subagent Execution** — stress-test each finding; the section sub-agent re-engages and defends / revises / withdraws; then synthesize.
4. **Synthesize**: deduplicate same-location findings across sections, format into comments, write a mandatory conclusion.

> Findings use the JSON schema in **code-review.md › Sub-agent Output Format**, with `agent` set to the section identifier (e.g. `"agent": "Section 3"`), `angle` to the detail concern, and `location` to the document section.

---

### Section 0: Coverage Check (always required)

Map every Goal and Flow from Overview to a section in Detail. Any unmapped item → `[blocking]`.

Inject the coverage map into every section sub-agent's prompt — every section consults it.

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
- Does each flow from Overview's Solution Design have a corresponding execution path here?

---

### Section 4: Error Handling

List all external interaction points. For each: is the failure mode named and handled?

- Can the caller distinguish transient from permanent failures?
- Are retries idempotent?
- Does error path behavior match the error-path test cases in Testing Strategy?

---

### Section 5: Implementation Flows

Compare against the flows in Overview's Solution Design.

- Every Overview flow has a function-level counterpart, when granularity differs?
- All branches covered, not just the main path?
- Parameters consistent with Section 1 interfaces?
- For Goals deferred as "to be quantified in Detail Design": are the numeric criteria now present and carried by the flow?

---

### Section 6: Performance Estimation

One question: **is the estimate credible?**

- Input assumptions (QPS, data size, latency) stated explicitly and sourced?
- Bottleneck on the critical path identified?
- Conclusion aligns with quantitative Goals?

---

### Section 7: Testing Strategy

- Every Goal, core module, and core flow has happy + edge + error test cases?
- Each test case specifies concrete input, observable expected behavior, and a single test level (unit / integration / e2e)?
- Mock boundaries justified, with rationale per boundary? Does mocking hide a real integration risk (schema constraints, concurrency, external failure modes)?
- Pass criteria trace back to Goals — quantitative Goals produce comparable numbers, behavioral Goals produce assertions on observable state?

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

<gate>
Every Detail Review must end with an explicit conclusion.
</gate>
