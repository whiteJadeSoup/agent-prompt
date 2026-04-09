# Global Claude Instructions

## Design Plan Guide

When writing a design plan, produce a single document with two parts: **Overview Design** (why + what) and **Detail Design** (how). Not every change needs a design plan — only write one when the change involves design decisions others need to understand (cross-module changes, new modules, architecture changes, multi-person collaboration).

### Part 1: Overview Design

The six sections form a decision chain — each exists because without it, the next section loses its basis for judgment:

```
Why do it? → How far? → How to split? → How does it run? → Why this split? → What's uncertain?
    │            │            │               │                  │                  │
 Problem    Goals/Non-    Architecture    Core Flows        Key Decisions     Risks & Open
 Statement  Goals         Overview                                            Questions
```

**1. Problem Statement** — Describe the current pain point and what triggered this work. Include quantitative data if available. Do NOT describe the solution here.

**2. Goals & Non-Goals**
- Goals should be verifiable ("support 100k DAU" not "improve performance").
- Non-goals are more important than goals — they prevent scope creep.
- How to identify non-goals: (a) requirements stakeholders might request but are excluded this time, (b) natural extensions of adjacent problems, (c) boundaries where the team has divergent opinions.
- Litmus test: if a non-goal is something nobody would do anyway, it carries no information — delete it.

**3. Architecture Overview** (with diagram)
- Draw the full system, highlight incremental changes (solid lines = existing, bold/dashed = new/modified, gray/strikethrough = removed).
- Group by responsibility, not by code structure.
- Only draw the layers the reader needs to understand. If the change is application-level, infrastructure is just a labeled box.
- Arrows show dependency direction; label the interaction type (sync call / async event / polling).

**4. Core Flows** (with sequence or flow diagrams)
- Participants are **modules / services / roles**, not classes or functions.
- Only main path + 1-2 critical error paths.
- Granularity rule: **no function names should appear**. Write "auth service validates request", not `authService.validate()`.

**5. Key Decisions & Alternatives** — **Before writing this section**, research industry best practices and leading implementations via web search. Reference the findings when explaining the rationale — decisions should be grounded in real-world evidence, not just local reasoning. For each decision: title states what was chosen (and what was not), body explains the rationale and key benefits, followed by "Alternatives Considered" summarizing rejected options and why. Use a comparison table only when there are ≥3 options that need multi-dimension comparison. Dimensions come from the design goals — they are not a generic checklist. If a decision has no tradeoff (only one reasonable option), it doesn't belong here.

*Narrative example (for binary decisions):*

> #### Tasks Stored as JSON Files, Not In-Memory
>
> Tasks are persisted as JSON files in a `.tasks/` directory. This has three critical benefits: (1) tasks survive process crashes, (2) multiple agents can read/write the same directory for coordination without shared memory, (3) humans can inspect and edit task files for debugging.
>
> **Alternatives Considered**: In-memory storage is simpler but loses state on crash and doesn't work across processes. A database (SQLite, Redis) provides ACID and better concurrency but adds a dependency and operational complexity. Files are the zero-dependency persistence layer that works everywhere.

*Table example (for ≥3 options):*

> | Dimension | Filesystem | In-Memory | SQLite |
> |-----------|-----------|-----------|--------|
> | Crash recovery | ✓ durable | ✗ lost | ✓ durable |
> | Multi-process | ✓ shared dir | ✗ needs IPC | ✓ shared DB |
> | Zero-dep deploy | ✓ | ✓ | ✗ needs driver |
> | Debuggability | ✓ direct edit | ✗ needs tooling | △ needs SQL client |
> | **Conclusion** | **✓** | | |

**6. Risks & Open Questions** — For each risk: description → impact scope → mitigation idea. For open questions: note the expected decision timeline.

### Part 2: Detail Design

**1. Module Responsibilities & Interfaces** — Full contract, mark incremental changes. Existing interfaces: signature only. New/modified: full definition (params, return, errors, preconditions).

**2. Data Model / Schema** — For each structure, answer: what is the structure (fields, types, optionality, defaults), what are the constraints (uniqueness, nullability, cross-field dependencies), how is it queried (indexes, common query patterns), how does it migrate (if modifying existing schema: migration path, backward compatibility, rollback).

**3. Core Algorithms / State Machines** — Only write what a competent engineer can't figure out in under 5 minutes from the requirements. State machines: transition diagrams with trigger conditions and side effects. Algorithms: pseudocode with complexity if performance-relevant. Document invariants — they are direct sources for test cases.

**4. Error Handling** — For each external interaction point (network, file IO, user input, cross-module calls), answer: how can it fail, how do we handle it (retry/degrade/propagate/ignore), how does the caller know.

**5. Key Flows** — Same flows as overview, but with class/function-level participants, full parameter details, and all branches the implementer needs to know.

**6. Performance Estimation** (required when goals have quantitative targets) — Prove the design meets the goal without over-engineering. Estimate throughput/latency on the critical path. Stop when the goal is met — don't optimize for 500k when the target is 100k.

**7. Testing Strategy** — Answer three questions: what to test (which modules/flows, prioritize core invariants), how to test (unit/integration/E2E split, where to draw mock boundaries and why), pass criteria (trace back to design goals).

**8. Migration & Compatibility** (when modifying existing systems) — Migration steps and order, canary/rollback plan, compatibility during coexistence.

### Writing Principles

1. Every section must trace back to a design goal. If it doesn't relate to any goal, delete it.
2. If removing a paragraph still lets the reader make correct decisions, that paragraph shouldn't exist.
3. Record **why**, not **what**. Code is the authoritative "what".
4. Diagrams over prose. Use text to explain what isn't obvious from the diagram.
5. Expose uncertainty honestly. Hiding risks wastes reviewers' attention.

## Code Review Guide

### Part 0: Review Architecture (Multi-Agent Execution)

Code review is executed by **3 parallel subagents + 1 orchestrator**. This reduces review time and ensures each concern gets dedicated focus.

#### Subagent Responsibilities

| Agent | Focus | Finding Types |
|-------|-------|--------------|
| **Agent A: Correctness & Safety** | Logic errors, security, concurrency, idempotency, backward compatibility, exception paths, resource management | Primarily `[blocking]` |
| **Agent B: Quality & Resilience** | Error handling, performance, observability, impact on other paths, architecture compliance, maintainability | `[blocking]` or `[question]` |
| **Agent C: Clarity & Coverage** | Readability, abstraction consistency, test coverage, dead code/redundancy | `[question]`, `[suggestion]`, `[nit]` |

#### Subagent Output Format

Each subagent must output a structured findings list. Free-text comments are not allowed at this stage.

```json
{
  "findings": [
    {
      "id": "A-001",
      "agent": "A",
      "location": "UserService.java:87",
      "severity": "blocking",
      "angle": "security",
      "what": "...",
      "why": "...",
      "proof": "...",
      "suggestion": "..."
    }
  ]
}
```

**Proof must be executable**: specify exact inputs, call sequence, or concurrent timing. If you cannot construct a concrete proof, set `severity` to `question`, never `blocking`.

#### Orchestrator Responsibilities

After all 3 subagents complete, the orchestrator:
1. **Deduplicates**: if multiple agents flag the same location for the same reason, keep the highest-severity finding and merge the rationale
2. **Resolves conflicts**: if agents disagree on severity for the same finding, use Agent A's judgment for correctness/security issues, Agent B's for quality issues
3. **Formats**: converts structured findings into the comment format defined in Step 4
4. **Concludes**: writes a mandatory conclusion (Approve / Request Changes / Comment)

#### Execution Flow

```
PR Diff
  ├─→ Agent A (correctness & safety) ─┐
  ├─→ Agent B (quality & resilience) ─┼─→ Orchestrator → Final Review
  └─→ Agent C (clarity & coverage)  ─┘
         [parallel]                        [sequential]
```

---

### Part 1: PR Submission (Author's Responsibility)

#### Size Principle

**Submit PRs frequently and keep them small. One PR does one thing.**

Judge by **intent**, not file count:

- ✅ Fix a bug / add a feature / refactor a module → one PR
- ❌ Fix a bug while refactoring, or add a feature while fixing unrelated typos → split them

If a feature requires a refactor first, submit the refactor PR before the feature PR. Note in the refactor PR description: "Preparation for upcoming X feature."

#### Before Opening a PR: Self-Review First

Go through the diff yourself:
- Any debug code, temporary comments, or console.log left behind?
- Any unrelated changes mixed in?

#### PR Description Template

```markdown
## Background
## Root Cause (required for bugs, can be merged into Background)
## Solution Overview
## Changes
## Verification
```

**Background** — Describe the scenario, not the solution.

❌ "Added modelId to modelBasicInfo"
✅ "When running `model update`, even if only field bindings are changed, the API returns a duplicate name error"

**Root Cause** — Trace to the actual cause, not just the symptom. Point to the specific code responsible, and explain how you confirmed it is the cause. **Use a diagram when the bug involves a call chain, data flow, or conditional branch** — it is often faster to read than a paragraph of prose.

❌ "modelId was not passed"

✅
> `ModelService.java:142` was missing `modelId` in the `modelBasicInfo` object it constructs. This causes the duplicate name check to always treat the current model as a stranger:
> ```
> PUT /model/123 (update bindings only)
>   └─ ModelService.buildBasicInfo()     [ModelService.java:138]
>        └─ modelBasicInfo.modelId = null   ← missing assignment
>             └─ DuplicateNameChecker.check(modelBasicInfo)
>                  └─ !Objects.equals(null, m.getId())  → always true
>                       └─ throws DuplicateNameException  ← wrong
> ```
> Confirmed by adding a log at `ModelService.java:142` to print `modelBasicInfo.getModelId()` — it printed `null` on every update request, regardless of the actual model ID.

**Solution Overview** — Explain *what* you chose to fix and *why* at that layer, not *how* (the code is the how). **Use a diagram to illustrate the solution** — show the corrected call chain, data flow, or state transition so the reviewer understands the approach before reading the diff. List rejected alternatives and why they were ruled out.

❌ "Added modelId to `modelBasicInfo`"

✅
> Populate `modelId` in `ModelService.buildBasicInfo()` so the duplicate name checker can correctly exclude the current model from comparison. Fix at the construction site rather than inside the checker — the checker should remain a pure validator that assumes valid input.
>
> ```
> PUT /model/123 (update bindings only)
>   └─ ModelService.buildBasicInfo()       [ModelService.java:138]
>        └─ modelBasicInfo { modelId: "123" }   ← assigned here
>             └─ DuplicateNameChecker.check(modelBasicInfo)
>                  └─ !Objects.equals("123", m.getId())  → false for self
>                       └─ passes correctly
> ```
>
> **Alternatives considered**: add `if (modelId == null) return false` inside `DuplicateNameChecker` — rejected because it silently swallows missing IDs instead of surfacing the upstream omission.

**Changes** — What changed AND what did not change. "What did not change" tells the reviewer where the boundary is.

❌ Only: "Added modelId to modelBasicInfo"
✅ Also: "Does not affect the create path (create uses independent logic)"

**Verification** — Two parts are required:

1. **Fix verification** — Show that the root cause scenario no longer reproduces. Paste actual commands and outputs.
2. **Regression verification** — Show that unaffected paths still work. Cover the paths most likely to be disturbed by the change.

❌ "Tested, works fine"
✅
```
# Fix verification
$ curl -X PUT /model/123 -d '{"bindings": [...]}' 
→ 200 OK (previously returned 409 duplicate name error)

# Regression: create path unaffected
$ curl -X POST /model -d '{"name": "new-model", ...}'
→ 201 Created
```

*Example of a complete, well-written PR description:*

> ## Background
> When running `model update` to change only field bindings (no name change), the API returns a 409 duplicate name error. This blocks any binding-only update on an existing model.
>
> ## Root Cause
> `ModelService.buildBasicInfo()` (`ModelService.java:138`) constructs the `modelBasicInfo` object but never assigns `modelId`. The downstream duplicate name check then compares `null` against every existing model ID, making it impossible to exclude the current model from the check:
> ```
> PUT /model/123 (update bindings only)
>   └─ ModelService.buildBasicInfo()       [ModelService.java:138]
>        └─ modelBasicInfo.modelId = null    ← missing assignment
>             └─ DuplicateNameChecker.check(modelBasicInfo)
>                  └─ !Objects.equals(null, m.getId())  → always true
>                       └─ throws DuplicateNameException  ← wrong
> ```
> Confirmed by logging `modelBasicInfo.getModelId()` at line 142 — printed `null` on every update request regardless of actual model ID.
>
> ## Solution Overview
> Populate `modelId` in `ModelService.buildBasicInfo()` so the checker can correctly exclude the current model. Fix at the construction site rather than inside the checker — the checker should remain a pure validator that assumes valid input.
> ```
> PUT /model/123 (update bindings only)
>   └─ ModelService.buildBasicInfo()       [ModelService.java:138]
>        └─ modelBasicInfo { modelId: "123" }   ← assigned here
>             └─ DuplicateNameChecker.check(modelBasicInfo)
>                  └─ !Objects.equals("123", m.getId())  → false for self
>                       └─ passes correctly
> ```
> **Alternatives considered**: add `if (modelId == null) return false` in `DuplicateNameChecker` — rejected because it silently swallows missing IDs instead of surfacing the real problem.
>
> ## Changes
> - `ModelService.java:142`: assign `modelId` when constructing `modelBasicInfo`
> - Does not affect the create path — create calls `buildCreateInfo()`, which is independent logic and untouched by this change
>
> ## Verification
> ```
> # Fix verification: binding-only update no longer returns 409
> $ curl -X PUT /model/123 -d '{"bindings": [{"fieldId": 1}]}'
> → 200 OK  (previously: 409 Duplicate name error)
>
> # Regression: name conflict detection still works
> $ curl -X PUT /model/123 -d '{"name": "existing-model-name"}'
> → 409 Duplicate name error  (expected, name genuinely conflicts)
>
> # Regression: create path unaffected
> $ curl -X POST /model -d '{"name": "new-model", "bindings": [...]}'
> → 201 Created
> ```

**Level of detail by change type:**

| Change Type | Required Sections |
|-------------|------------------|
| Typo, config tweak | One-line background is enough |
| Bug fix | Background + Root Cause + Solution Overview + Changes + Verification |
| New feature | Background + Solution Overview + Changes + Verification |
| Cross-module refactor | All sections + potential risks |

---

### Part 2: Reviewer Steps

**Step 1: Understand the Intent**

Read the PR description and build context before looking at any code: what problem does this solve, what is the root cause, where is the boundary, how was it verified.

**If the description is incomplete, send it back for the author to fill in. Do not start reviewing code.**

**Step 2: Read the Code with Questions in Mind**

- **Does the change fix the root cause?** Walk through the failure scenario mentally and confirm it can no longer reproduce.
- **Does it introduce new problems?** Don't only read the diff — read the surrounding code for context.
- **Is the scope appropriate?** Anything changed that shouldn't be? Anything missing that should be changed?

**Step 3: Assess Whether Verification Is Credible**

- Does it cover the scenario described in the root cause?
- Does it only test the happy path, missing edge cases?
- If insufficient, explicitly name the missing scenario and ask for it.

**Step 4: Write Comments**

Every comment must be structured as follows:

**1. Location** — filename + line number(s). If the problem spans multiple lines or involves an interaction between two places, cite all of them.

**2. Problem** — Structured explanation:
- **What**: which line(s) or code path cause the issue
- **Why**: the reasoning — include the execution flow if the bug is non-obvious (e.g., "A calls B with X, B passes X to C without validation, C crashes on null"). Every `[blocking]` comment **must** include a call chain or data flow diagram. A blocking comment without a diagram is considered incomplete.
- **Proof**: construct the minimal scenario that triggers the problem (e.g., specific input, concurrent timing, config flag). **Proof must be a concrete, executable minimal reproduction** (specific input values, call sequence, or concurrent timing). Hypothetical descriptions like "if X happens then Y" are not allowed. If you cannot construct a concrete proof, you must downgrade severity to `[question]`, never `[blocking]`. **Use a diagram when the trigger condition involves a sequence of steps or concurrent timing** — it is often clearer than prose.

**3. Suggestion** — A concrete fix direction, with a brief justification for why it is correct. If the suggestion involves a non-trivial change, show a sketch or pseudocode and explain why it avoids the problem. **Use a diagram to show the corrected flow when the fix changes a call chain, branching logic, or state transition.**

*Example of a well-written comment:*

> `[blocking]` **UserService.java:87, TokenValidator.java:34**
>
> **What**: `UserService.getUser()` passes the raw `userId` from the request directly to `TokenValidator.validate(userId)` (line 87) without null-checking. `TokenValidator.validate()` dereferences the parameter at line 34 without a guard.
>
> **Why**: If `userId` is absent from the request, it arrives as `null` and propagates unchecked through the call chain:
> ```
> POST /user/profile (userId=null)
>   └─ UserService.getUser(null)        [UserService.java:87]
>        └─ TokenValidator.validate(null) [TokenValidator.java:34]
>             └─ null.hashCode()  ← NullPointerException
> ```
>
> **Proof**: Send `POST /user/profile` with an empty body — no `userId` field. `TokenValidator.validate(null)` is called, throws `NullPointerException` at line 34, and the request returns 500 instead of 400.
>
> **Suggestion**: Validate `userId` at the controller boundary before it enters the service layer. Rejecting it early with a 400 keeps the service layer free of null-guard noise, and matches how other fields (e.g., `email`) are already validated in `UserController.java:52`. The fix does not need to touch `TokenValidator` — it should remain a pure validator that assumes valid input.
> ```
> POST /user/profile (userId=null)
>   └─ UserController.validate(request)  ← add null check here, return 400
>        [never reaches UserService]
>
> POST /user/profile (userId="abc")
>   └─ UserController.validate(request)  ← passes
>        └─ UserService.getUser("abc")
>             └─ TokenValidator.validate("abc")  ← safe
> ```

**Review angles and their priority:**

🔴 **Must flag** → `[blocking]`

| Angle | What to look for |
|-------|-----------------|
| **Correctness** | Logic errors, missing boundary conditions, null handling, incomplete branches |
| **Security** | Unvalidated user input, sensitive data leaked to logs or stdout |
| **Concurrency safety** | Race conditions, duplicate submission risk, incorrect async ordering |
| **Idempotency** | Is it safe to retry write operations? Can repeated execution cause side effects? |
| **Backward compatibility** | Do interface/parameter/output format changes break existing callers? |
| **Exception path integrity** | Is system state consistent after an exception? Any half-written data? |
| **Resource management** | Are files/connections properly released? Any memory leaks? |
| **Impact on other paths** | Does the change accidentally affect other functionality? Read beyond the diff. |
| **Architecture compliance** | Violates layering rules, module boundaries, or naming conventions |
| **Maintainability** | This change introduces tight coupling or low cohesion; a function/module takes on too many responsibilities |

🟡 **Should flag** → `[question]` or `[blocking]`

| Angle | What to look for |
|-------|-----------------|
| **Readability** | Inaccurate names, missing comments on complex logic, functions doing more than one thing |
| **Abstraction consistency** | High-level business logic and low-level implementation details mixed in the same function |
| **Error handling** | External calls fail without proper handling; error messages are unclear |
| **Performance** | Requests inside loops, N+1 queries, obvious performance traps |
| **Observability** | Missing logs for key operations, incorrect log levels |
| **Test coverage** | Core logic untested; bug fixes missing regression tests |
| **Dead code / redundancy** | Unreachable code, duplicated logic, unused variables |

⚪ **Do not flag**

| Angle | Reason |
|-------|--------|
| **Code formatting** | Leave it to the linter — it should not appear in review comments |
| **Personal preference** | If two approaches have no correctness difference, don't comment |
| **Pre-existing issues** | Problems in code not touched by this PR — open a separate issue instead |

**Comment labels:**

| Label | Meaning | Blocks merge |
|-------|---------|-------------|
| `[blocking]` | Must be fixed | Yes |
| `[question]` | Need author to clarify intent | Depends on answer |
| `[suggestion]` | Current solution works, but there is a better approach backed by research or scale analysis | No |
| `[nit]` | Optional improvement, author decides | No |

All comments must have a label. Before writing, ask yourself: if this is not fixed, will it cause a production issue or mislead the next person reading this code?

- Yes → `[blocking]`
- Not sure → `[question]`
- Current code works but a researched alternative is meaningfully better → `[suggestion]`
- No, but could be better → `[nit]`
- Pure personal preference → don't write it

**LLM-specific warning — prevent over-flagging**: The only criterion for `[blocking]` is: **would this cause an observable error or data problem in production if not fixed?** When uncertain, use `[question]`, not `[blocking]`. It is better to miss a blocking issue than to mislabel a question as blocking.

**Writing a `[suggestion]` comment** — The bar is higher than `[nit]`: you must research multiple options and recommend one. A suggestion without alternatives is just an opinion. Before writing a suggestion, you **must research industry practices via web search**. Suggestions based solely on internal reasoning are not allowed. Structure it as:

1. **Current behavior** — what the code does now, and under what conditions it becomes a problem (scale threshold, edge case, maintainability cliff)
2. **Options** — research ≥2 alternatives (including keeping the current approach as a baseline). For each: what it is, its key benefits, and its trade-offs. Use a comparison table when there are ≥3 options.
3. **Recommendation** — which option you suggest and why, with a diagram if it changes a flow or structure
4. **Trade-off** — what the author gives up by adopting the recommendation, so they can make an informed call

*Example:*

> `[suggestion]` **OrderService.java:203**
>
> **Current behavior**: `getOrdersByUser(userId)` loads all orders for the user into memory and filters in-application. This works today (~200 orders/user at p99), but will degrade significantly as order volume grows — at 10k orders/user the full result set is loaded on every call.
>
> **Options**:
>
> | | Current (in-memory filter) | Predicate pushdown | Cursor-based pagination |
> |---|---|---|---|
> | DB load | All rows every call | Only matching rows | Page-sized chunks |
> | Code complexity | Low | Low | Medium |
> | Handles unbounded growth | ✗ | ✓ (if result set is bounded) | ✓ |
> | Schema change needed | No | Index on `(user_id, status)` | Index on `(user_id, created_at)` |
>
> **Recommendation**: Predicate pushdown — push filter conditions into the repository query so only matching rows are transferred:
> ```
> Before:
>   DB → all orders for user → OrderService filters in memory
>
> After:
>   DB (filtered by status/date) → only matching orders → OrderService
> ```
> Simpler than pagination, sufficient if the filtered result set stays bounded (which it does for the current use case). Cursor pagination is the right next step only if a single user can accumulate unbounded matching orders.
>
> **Trade-off**: Requires a new repository method and a DB index on `(user_id, status)`. Migration is low-risk but needs a schema change. Author decides whether to address now or track as a follow-up.

**Aim for at least one `[suggestion]` per review.** Every non-trivial PR touches code that could be improved beyond correctness. Before moving to your conclusion, ask yourself: is there a scale risk, a maintainability cliff, or an industry practice the author may not be aware of? If yes, write a `[suggestion]`. If you genuinely find nothing worth suggesting, that is fine — but the absence should be a conscious decision, not an oversight.

**Step 5: Give a Clear Conclusion**

| Conclusion | Meaning |
|------------|---------|
| **Approve** | Ready to merge |
| **Request Changes** | Has blocking issues; re-review required after fixes |
| **Comment** | Has open questions; conclusion pending |

**Never finish a review without giving a conclusion.**

**Hard requirement**: Every review must end with an explicit conclusion. Outputting comments without providing an Approve / Request Changes / Comment conclusion means the review is incomplete.

---

### Part 3: Author's Responsibility After Receiving Comments

Every comment must get an explicit response — silence is not acceptable:
- `[blocking]` — Fix it, then re-request review
- `[question]` — Explain your reasoning; reviewer decides if a change is needed
- `[suggestion]` — Reply with one of: "adopting, will address in follow-up PR", "adopting now", or "not adopting because X" (X must engage with the trade-off, not just dismiss it)
- `[nit]` — Reply "done" or "not changing because X"

---

### Part 4: When Review Comes Too Late

If a **design-level problem** is found during review (wrong direction, poor module decomposition), the review came too late. Design should be aligned before writing code. When this happens, schedule a separate discussion — do not block the PR indefinitely.

---

## Testing Requirements

- Every feature or bug fix must include unit tests
- Write e2e tests for user-facing flows when applicable
- After coding is complete, run all tests and provide a pass/fail summary report before finishing

## Git Safety Rules

- `git push` 前必须告知用户并等待确认，不可直接执行
- `git push --force` / `git push -f`：**默认拒绝**，除非用户明确说"我知道风险，强制推送"

## Git Commit Convention

Follow [Conventional Commits v1.0.0-beta.4](https://www.conventionalcommits.org/en/v1.0.0-beta.4/).

### Format

```
<type>[optional scope]: <description>

[optional body]

[optional footer]
```

### Types

| Type | When to use |
|------|------------|
| `feat` | New feature |
| `fix` | Bug fix |
| `refactor` | Code change that neither fixes a bug nor adds a feature |
| `perf` | Performance improvement |
| `docs` | Documentation only |
| `test` | Adding or updating tests |
| `chore` | Build process, dependency updates, tooling |

### Rules

- Description is lowercase, imperative mood, no period at end
- Scope is optional: `feat(parser): add ability to parse arrays`
- Breaking changes: append `!` after type — `feat!: remove deprecated API`
- Body and footer are separated from description by a blank line

### Examples

```
feat: add context engineering with /compact command

fix(tools): catch TimeoutExpired in execute_command

refactor: replace file tools with bash guidance

feat!: drop support for Python 3.10

chore: update dependencies
```
