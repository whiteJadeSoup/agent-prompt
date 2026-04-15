# Design Review Guide

## Overview

Design review is executed in two separate phases, matching the two-phase writing process:

- **Phase 1: Overview Review** — validates problem definition, architecture direction, and key decisions. Must complete before Detail Design is written.
- **Phase 2: Detail Review** — verifies implementation contracts, data models, error handling, and performance estimates. Runs after Overview is confirmed.

Never mix the two phases. Raising direction-level issues during Detail Review means the review came too late — schedule a separate discussion rather than blocking the PR indefinitely.

---

## Phase 1: Overview Review

### Execution Architecture (Multi-Agent)

Overview Review is executed by **3 parallel subagents + 1 orchestrator**.

| Agent | Focus | Finding Types |
|-------|-------|--------------|
| **Agent A: Problem & Goals** | Problem definition accuracy, User Story quality, AC completeness, Non-goals validity | Primarily `[blocking]` |
| **Agent B: Architecture & Decisions** | Architecture diagram clarity, key decision rationale, alternatives considered, dependency assumptions | `[blocking]` or `[question]` |
| **Agent C: Risk & Completeness** | Risk coverage, open questions, missing scenarios, assumption stress-testing | `[question]`, `[suggestion]` |

**Before dispatching subagents**, the orchestrator must inject into each subagent's prompt:
- The design document (or its identifier so the subagent can read it)
- The agent's assigned focus area
- The complete contents of this review guide
- The output format (JSON findings schema below)

Each subagent outputs structured findings:

```json
{
  "findings": [
    {
      "id": "A-001",
      "agent": "A",
      "location": "Goals & Non-Goals",
      "severity": "blocking",
      "angle": "problem definition",
      "what": "...",
      "why": "...",
      "proof": "...",
      "suggestion": "..."
    }
  ]
}
```

**Proof must be concrete**: cite the specific claim in the document and the scenario that breaks it. If you cannot construct a concrete proof, set `severity` to `question`, never `blocking`.

After all 3 subagents complete, the orchestrator deduplicates, resolves conflicts, formats findings into comments, and writes a mandatory conclusion.

---

### Reviewer Steps

**Step 1: Verify the Problem Definition**

Before reading the solution, validate that the problem is correctly framed:

- Does the Problem Statement describe a real pain point, or does it jump straight to a solution?
- Are User Stories present for all user-facing goals? Does each Story have a specific Role (not "user"), a behavior-oriented Goal (not a feature), and a "so that" clause?
- Do the Acceptance Criteria cover all three paths: happy path, edge case, and error path?
- Are Non-goals real exclusions — things stakeholders might actually request — or just things nobody would do anyway?

**If the problem definition is incomplete or solution-first, send it back before reading further.**

**Step 2: Stress-Test the Architecture**

Read the architecture diagram and core flows with a skeptic's mindset. The document is a set of claims — your job is to find what breaks them:

**Is the architecture diagram self-explanatory?**
- Can you understand the system boundaries without reading the prose?
- Are all arrows labeled with interaction type (sync / async / polling)?
- Are new/modified components clearly distinguished from existing ones?

**Do the core flows cover the critical paths?**
- Is the main path complete end-to-end?
- Are 1–2 critical error paths shown?
- What happens when a dependency is unavailable — is that path drawn?

**What assumptions are embedded in the design?**
- What does this design assume about dependency SLAs, data volume, call ordering, or consistency guarantees?
- If any of those assumptions are wrong, does the design still hold?
- *Example: A design assumes the downstream service responds within 100ms. What if it takes 2s?*
- *Example: A design assumes events arrive in order. What if they don't?*

**Step 3: Evaluate Key Decisions**

For each key decision:
- Is there a genuine tradeoff, or is only one option presented?
- Is the rationale grounded in real constraints (team size, timeline, existing system boundaries), or in habit and preference?
- Are the rejected alternatives fairly described, or strawmanned?
- If the stated rationale changed (e.g., timeline extended, team grew), would the decision still hold?

**Step 4: Assess Risks & Open Questions**

- Does the Risks section name the actual risks, or only low-probability ones that feel safe to list?
- What is the single most likely failure mode of this design? Is it in the Risks section?
- Are open questions assigned a decision timeline, or left indefinitely open?
- What would have to be true for this design to fail completely? Is that scenario addressed?

**Step 5: Write Comments**

Every comment must include:

**1. Location** — which section of the document (e.g., "Goals & Non-Goals", "Architecture Overview", "Key Decisions — Task Storage").

**2. Problem**
- **What**: the specific claim or gap in the document
- **Why**: why it matters — draw a diagram if the issue involves a flow or dependency chain. **Every `[blocking]` comment must include a diagram or a concrete scenario.** A blocking comment without evidence is considered incomplete.
- **Proof**: the specific scenario or input that breaks the claim. Must be concrete — "if X then Y" hypotheticals are not allowed. If you cannot construct concrete proof, downgrade to `[question]`.

**3. Suggestion** — a concrete direction to resolve the issue, with brief rationale.

**Comment labels:**

| Label | Meaning | Blocks confirmation |
|-------|---------|-------------------|
| `[blocking]` | Design cannot proceed without resolving this | Yes |
| `[question]` | Need author to clarify a claim or assumption | Depends on answer |
| `[suggestion]` | Design works, but a better approach exists | No |
| `[nit]` | Minor wording or clarity improvement | No |

**`[blocking]` standard**: if this issue is not resolved, the implemented system will fail to achieve the design goals. When uncertain, use `[question]`.

**Step 6: Give a Clear Conclusion**

| Conclusion | Meaning |
|------------|---------|
| **Confirm** | Overview is sound, proceed to Detail Design |
| **Request Changes** | Has blocking issues; re-review required |
| **Comment** | Has open questions; conclusion pending |

Every Overview Review must end with an explicit conclusion. The author must not begin Detail Design until the Overview is confirmed.

---

## Phase 2: Detail Review

### Execution Architecture (Single Agent)

Detail Review is executed by a **single agent** reading sequentially through the Detail Design sections. The linear structure of Detail Design (interfaces → data model → error handling → performance → testing) does not benefit from parallel decomposition.

The agent's prompt must include:
- The confirmed Overview Design (for context)
- The Detail Design sections to review
- The complete contents of this review guide
- The output format (JSON findings schema above)

---

### Reviewer Steps

**Step 1: Verify Traceability to Overview**

Before reviewing any section, confirm the Detail Design traces back to the confirmed Overview:
- Does each module in the Detail Design correspond to a component in the Architecture Overview?
- Do the Key Flows in Detail match the Core Flows in Overview (same participants, same paths)?
- If something appears in Detail but not in Overview, flag it — it may be scope creep.

**Step 2: Review Each Section**

**Module Responsibilities & Interfaces**
- Is each interface's contract fully specified (params, return type, errors, preconditions)?
- Are new/modified interfaces clearly distinguished from existing ones?
- Does any interface expose more than it needs to (over-broad contracts)?

**Data Model / Schema**
- Are constraints explicit (nullability, uniqueness, cross-field dependencies)?
- Is the query pattern documented — are indexes aligned with how the data is actually queried?
- If modifying existing schema: is the migration path safe? Can it roll back?

**Error Handling**
- For every external interaction (network, file IO, cross-module calls): is the failure mode named and handled?
- Are retries idempotent? Can repeated execution cause side effects?
- Does the caller know how to distinguish a transient failure from a permanent one?

**Performance Estimation**
- If there are quantitative goals: does the estimation prove the design meets them?
- Is the critical path identified? Is the bottleneck addressed?
- Is there over-engineering — optimizing beyond what the goal requires?

**Testing Strategy**
- Are core invariants tested?
- Is the mock boundary justified — are mocks hiding real integration risks?
- Do pass criteria trace back to design goals?

**Step 3: Write Comments**

Same format as Phase 1 (Location, Problem, Suggestion). Same label definitions.

**`[blocking]` standard in Detail Review**: if this issue is not resolved, the implementation will be incorrect, unsafe, or unable to meet the design goals. Direction-level concerns are out of scope — if you find one, note it separately and do not block the Detail Review.

**Step 4: Give a Clear Conclusion**

| Conclusion | Meaning |
|------------|---------|
| **Confirm** | Detail Design is sound, proceed to implementation |
| **Request Changes** | Has blocking issues; re-review required |
| **Comment** | Has open questions; conclusion pending |

---

## Author's Responsibility After Receiving Comments

Same rules as Code Review (Part 3):
- `[blocking]` — revise the document, then re-request review
- `[question]` — explain your reasoning; reviewer decides if a change is needed
- `[suggestion]` — reply with "adopting", "adopting in follow-up", or "not adopting because X"
- `[nit]` — reply "done" or "not changing because X"

After addressing a comment, the **author resolves the thread** via code change or rebuttal. "Acknowledged" is not sufficient. Only the author resolves threads.

Re-request review only after all `[blocking]` issues are resolved and all threads are closed.

## Re-review Responsibilities

Same rules as Code Review (Part 3.5):
- Primary goal: verify previous `[blocking]` and `[question]` findings are addressed
- New findings are allowed but must be labeled `[new finding]`
- Re-review ends with an updated conclusion
