# Code Review Guide

## Part 0: Review Architecture (Multi-Agent Execution)

Code review is executed by **3 parallel subagents + 1 orchestrator**. This reduces review time and ensures each concern gets dedicated focus.

### Subagent Responsibilities

| Agent | Focus | Finding Types |
|-------|-------|--------------|
| **Agent A: Correctness & Safety** | Logic errors, security, concurrency, idempotency, backward compatibility, exception paths, resource management | Primarily `[blocking]` |
| **Agent B: Quality & Resilience** | Error handling, performance, observability, impact on other paths, architecture compliance, maintainability | `[blocking]` or `[question]` |
| **Agent C: Clarity & Coverage** | Readability, abstraction consistency, test coverage, dead code/redundancy | `[question]`, `[suggestion]`, `[nit]` |

### Subagent Output Format

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

### Orchestrator Responsibilities

After all 3 subagents complete, the orchestrator:
1. **Deduplicates**: if multiple agents flag the same location for the same reason, keep the highest-severity finding and merge the rationale
2. **Resolves conflicts**: if agents disagree on severity for the same finding, use Agent A's judgment for correctness/security issues, Agent B's for quality issues
3. **Formats**: converts structured findings into the comment format defined in Step 4
4. **Concludes**: writes a mandatory conclusion (Approve / Request Changes / Comment)

### Execution Flow

```
PR Diff
  ├─→ Agent A (correctness & safety) ─┐
  ├─→ Agent B (quality & resilience) ─┼─→ Orchestrator → Final Review
  └─→ Agent C (clarity & coverage)  ─┘
         [parallel]                        [sequential]
```

---

## Part 1: PR Submission (Author's Responsibility)

### Size Principle

**Submit PRs frequently and keep them small. One PR does one thing.**

Judge by **intent**, not file count:

- ✅ Fix a bug / add a feature / refactor a module → one PR
- ❌ Fix a bug while refactoring, or add a feature while fixing unrelated typos → split them

If a feature requires a refactor first, submit the refactor PR before the feature PR. Note in the refactor PR description: "Preparation for upcoming X feature."

### Before Opening a PR: Self-Review First

Go through the diff yourself:
- Any debug code, temporary comments, or console.log left behind?
- Any unrelated changes mixed in?

### PR Description Template

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

**Root Cause** — Trace to the actual cause, not just the symptom. Point to the specific code responsible, and explain how you confirmed it is the cause. **Draw a diagram (architecture, sequence, flow, call chain, or state — using ASCII art, Mermaid, or any other text-based format) unless the entire root cause fits in one sentence with no branching.** Prose alone is not acceptable for multi-step or conditional causes.

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

**Solution Overview** — Explain *what* you chose to fix and *why* at that layer, not *how* (the code is the how). **Draw a diagram (architecture, sequence, flow, call chain, or state — using ASCII art, Mermaid, or any other text-based format) showing the corrected path — required unless the fix is a single-line change with no structural effect.** The reviewer must understand the approach from the diagram before reading the diff. List rejected alternatives and why they were ruled out.

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

## Part 2: Reviewer Steps

**Step 1: Understand the Intent**

Read the PR description and build context before looking at any code: what problem does this solve, what is the root cause, where is the boundary, how was it verified.

**If the description is incomplete, send it back for the author to fill in. Do not start reviewing code.**

**Step 2: Read the Code Skeptically**

Approach the code as a skeptic, not a validator. The PR description is a hypothesis — your job is to stress-test it, not confirm it. Assume the author is competent and well-intentioned. The question is not "did they make a mistake?" but **"what situation would make this solution wrong?"**

Work through the following questioning patterns before forming any judgment:

**Does this actually fix the root cause?**
- The PR says X causes Y. Is there another path that also causes Y, which this fix doesn't touch?
- The fix is applied at layer A. Could the same bug re-enter from layer B?

*Example: A fix that adds a null-check in the service layer — but the controller can also call the downstream directly, bypassing the check entirely.*

**What if the inputs are different?**
- What if this field is null / empty / negative / max-int / a very long string?
- What if the caller sends two concurrent requests for the same resource?
- What if this runs during a partial failure — e.g., the DB write succeeds but the cache invalidation fails?

*Example: A "create if not exists" implementation that works correctly in isolation but creates duplicates under concurrent load.*

**What if the environment changes?**
- What if the dependent service is slow or unavailable?
- What if this is called at 10x the expected volume?
- What if this code executes in a different order than the author assumed (e.g., retries, async callbacks)?

*Example: A retry loop that is safe when idempotent, but causes double-charges when the underlying operation is not.*

**Does the fix hold at the boundary?**
- The fix works for the described scenario. Does it also work for the adjacent scenario the PR doesn't mention?
- Is there an off-by-one, a timezone edge, an encoding edge, a locale-specific behavior?

*Example: A date comparison that works correctly in UTC but silently breaks for users in UTC+14.*

**Are the claims in the PR description actually verified?**
- The PR says "confirmed by logging X" — is that log sufficient proof, or could it be misleading?
- The PR says "this doesn't affect path Z" — did the author verify that, or just assert it?

*Example: A PR claims the create path is unaffected, but the reviewer finds both create and update share the same helper function that was modified.*

**Output rule**: Every question you raise must be resolved before moving to Step 3.
- If reading the code answers it → answer it and move on
- If the code doesn't answer it → it becomes a `[blocking]` or `[question]` comment
- Do not silently drop questions you cannot answer

**Closing checklist** (after the skeptical pass):
- Does the change fix the root cause?
- Does it introduce new problems? Don't only read the diff — read the surrounding code for context.
- Is the scope appropriate? Anything changed that shouldn't be? Anything missing that should be changed?

**Step 3: Assess Whether Verification Is Credible**

- Does it cover the scenario described in the root cause?
- Does it only test the happy path, missing edge cases?
- If insufficient, explicitly name the missing scenario and ask for it.

**Step 4: Write Comments**

Every comment must be structured as follows:

**1. Location** — filename + line number(s). If the problem spans multiple lines or involves an interaction between two places, cite all of them.

**2. Problem** — Structured explanation:
- **What**: which line(s) or code path cause the issue
- **Why**: the reasoning — include the execution flow. **Draw a diagram (architecture, sequence, flow, call chain, or state — using ASCII art, Mermaid, or any other text-based format) by default. Skip only if the entire Why fits in one sentence with no branching or concurrency.** A `[blocking]` comment without a diagram is considered incomplete.
- **Proof**: construct the minimal scenario that triggers the problem (e.g., specific input, concurrent timing, config flag). **Proof must be a concrete, executable minimal reproduction** (specific input values, call sequence, or concurrent timing). Hypothetical descriptions like "if X happens then Y" are not allowed. If you cannot construct a concrete proof, you must downgrade severity to `[question]`, never `[blocking]`. **Draw a diagram when the trigger involves a sequence of steps or concurrent timing.**

**3. Suggestion** — A concrete fix direction, with a brief justification for why it is correct. **Draw a diagram (sequence, flow, or call chain — ASCII art, Mermaid, or any text-based format) showing the corrected flow — required unless the fix is a single-line change with no structural effect.** If the suggestion involves a non-trivial change, also show pseudocode and explain why it avoids the problem.

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
3. **Recommendation** — which option you suggest and why. **Draw a diagram (sequence, flow, or call chain — ASCII art, Mermaid, or any text-based format) showing the recommended flow — required unless the recommendation involves no structural change.**
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

## Part 3: Author's Responsibility After Receiving Comments

Every comment must get an explicit response — silence is not acceptable:
- `[blocking]` — Fix it, then re-request review
- `[question]` — Explain your reasoning; reviewer decides if a change is needed
- `[suggestion]` — Reply with one of: "adopting, will address in follow-up PR", "adopting now", or "not adopting because X" (X must engage with the trade-off, not just dismiss it)
- `[nit]` — Reply "done" or "not changing because X"

---

## Part 4: When Review Comes Too Late

If a **design-level problem** is found during review (wrong direction, poor module decomposition), the review came too late. Design should be aligned before writing code. When this happens, schedule a separate discussion — do not block the PR indefinitely.
