# Design Plan Guide

A design plan exists so others can review your design decisions. Write one only for changes others must understand: cross-module work, new modules, architecture shifts, multi-person collaboration.

The document has two parts: **Overview** (why + what) confirmed first, then **Detail** (how) below it. The final doc reads top to bottom in one pass.

> **Apply Design Thinking Rules to every section.**

## Part 1: Overview Design

**1. Problem & Goals**
- **Problem** — the pain point and trigger. Quantify if data exists. Do not describe the solution.
- **Goals** — how success is judged. Quantify when possible. For performance / capacity / SLA goals, quantification is **not optional in Overview**. For other goals, if a number is unavailable now, write "to be quantified in Detail Design" — never leave the gap implicit.
- **Non-Goals** — required output. What this work explicitly does not cover. Drop any non-goal nobody would do anyway.

**2. Solution Design** — presents **the chosen solution** (alternatives belong in Section 3).

**Diagrams are required, and they come before prose.** Prose explains only what diagrams cannot — participant roles, dependency types, error-path triggers. Restating the diagram in words is grounds for review rejection.
- **Architecture Diagram** — modules and dependency directions; mark added / modified / removed; stay above class/function level.
- **Flow Diagrams** — one core flow (main path) and 1–2 edge flows (critical error / boundary paths). Participants are modules / services / roles. **No function names** — those belong in Detail.

**3. Research & Comparison** — **web search is mandatory before writing this section.** Decisions must be grounded in industry practice, not local reasoning.
- Alternatives considered (industry practice, leading implementations)
- How they compare (table when ≥3 options; dimensions come from the design goals)
- Why the Section 2 solution wins
- **Risks of the chosen solution** — both kinds, both required:
  - **Type A — cost of choosing**: trade-offs accepted by picking this over the others
  - **Type B — intrinsic fragility**: weaknesses of the solution itself (key assumptions, single points of failure, scale cliffs) — present even if no alternative existed
  - State impact scope and mitigation per risk.

**4. Open Questions (optional)** — keep only if research left items unresolved. For each: the question, decision timeline, decision owner. Omit the section if there are none.

## Part 2: Detail Design

> **Prerequisite**: Overview confirmed. Each section traces back to a Goal, architecture, flow, or decision in the Overview.

**1. Module Responsibilities & Interfaces** — full contract per module. Existing interfaces: signature only. New or modified: full definition (parameters, return, errors, preconditions).

```
TaskStore.acquire(taskId, agentId, ttl) -> Lease
  pre:    taskId exists; no live lease on it
  returns: Lease{ id, expiresAt }
  errors: TaskNotFound, AlreadyLeased
```

**2. Data Model / Schema** — per structure: fields with types and defaults; constraints (uniqueness, nullability, cross-field rules); query patterns and indexes; migration path with rollback when changing existing schema.

```
table tasks(
  id          uuid pk
  status      enum('pending','running','done','failed') not null
  lease_id    uuid null
  updated_at  timestamptz not null
) index (status, updated_at)
invariant: status='running' ⇒ lease_id is not null
```

**3. Core Algorithms / State Machines** — write only what a competent engineer cannot reconstruct in five minutes. State machines: transition diagrams with triggers and side effects. Algorithms: pseudocode plus complexity if performance matters. Document invariants — they become test cases.

```
pending --start()--> running --finish()--> done
                       │
                       └--timeout()--> failed   (lease released)
invariant: at most one running lease per task
```

**4. Error Handling** — for each external interaction (network, IO, user input, cross-module call): failure modes, handling (retry / degrade / propagate / ignore), and what the caller observes.

```
call           failure          handling          caller sees
storage.write  transient IO     retry x3 jitter   ok | PersistError
payment.api    timeout          fail-fast         PaymentTimeout
```

**5. Implementation Flows** — the Overview flows at function-level granularity, only when the implementer needs more detail. Show class/function names, parameters, every branch. If the Overview flow already suffices, write "see Overview" and skip.

```
TaskController.start(req)
  └─ TaskService.start(taskId, agentId)
       ├─ TaskStore.acquire(taskId, agentId, ttl=30s)
       │    └─ AlreadyLeased → 409
       └─ Worker.dispatch(taskId)
            └─ dispatch error → release lease, 500
```

**6. Performance Estimation** — required when Goals are quantitative. Estimate throughput / latency on the critical path. State input assumptions (QPS, payload size) and the bottleneck. Stop once the estimate clears the Goal.

```
target: p99 < 200ms at 1000 QPS
path:   auth(3ms) + lookup(20ms) + work(80ms) + write(15ms) ≈ 118ms
bottleneck: write under contention → connection pool ≥ 32
```

**7. Testing Strategy** — this section is the implementer's reference for writing code and tests. **Every Goal, core module, and core flow needs multiple test cases**, covering all three paths:

| Path | Meaning |
|------|---------|
| **Happy** | Smallest verification of the main scenario |
| **Edge** | Boundaries (empty, max, concurrency, zero, null) |
| **Error** | How failure is observed (exception, fallback, error return) |

Each test case specifies:
- **Input** — concrete values or preconditions (not "a user calls in")
- **Expected** — observable output / state / side effect (not "should work")
- **Level** — unit / integration / e2e (pick one)

**Mock boundaries with rationale** — list what is mocked and what is not, and why each boundary was drawn there (e.g., "DB not mocked: schema constraints are a key test point. External payment service mocked: covered by contract tests.").

**Pass criteria** trace back to the Goal. Quantitative Goal → comparable numeric output. Behavioral Goal → assertion on observable state.

```
Goal: TaskStore prevents concurrent leases on the same task

T1 happy   unit         acquire(t1, A) → Lease; expiresAt within ttl ±100ms
T2 edge    integration  acquire(t1, A) || acquire(t1, B) → exactly one Lease,
                        the other raises AlreadyLeased
T3 error   unit         acquire(unknownId) → TaskNotFound
T4 edge    integration  lease expires → next acquire succeeds; old lease invalid

Mocks: storage not mocked (uniqueness is the test); clock mocked in T4.
Pass:  all four pass; T2 repeated 100× with zero flakes.
```

**8. Migration & Compatibility** — required when modifying existing systems. Cover migration steps in order, canary / rollback plan, compatibility while old and new coexist.

```
1. add column status_v2 (nullable)
2. dual-write v1 + v2 for 24h
3. backfill v2 from v1
4. switch readers to v2 (canary 1% → 100%)
5. drop v1 after 7-day soak
rollback: keep dual-write, revert reader switch.
```

## Writing Principles

1. Every section traces back to a Goal. If a paragraph relates to no Goal, delete it.
2. If removing a paragraph still lets the reader make correct decisions, the paragraph should not exist.
3. Record **why**, not **what**. Code is the authoritative "what".
4. **Diagrams over prose.** Use words only for what diagrams cannot show. Omitting a diagram requires an explicit reason ("single linear step, no branching").
5. Expose uncertainty honestly. Hidden risks waste reviewers' attention.
