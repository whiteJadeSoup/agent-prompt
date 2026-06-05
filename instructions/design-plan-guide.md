# Design Plan Guide

A design plan exists so others can review your design decisions. Write one only for changes others must understand: cross-module work, new modules, architecture shifts, multi-person collaboration.

The document has two parts: **Overview** (why + what) confirmed first, then **Detail** (how) below it. The final doc reads top to bottom in one pass.

> **Apply Design Thinking Rules to every section.**

> **Diagrams follow the C4 model** (Simon Brown). Mapping:
> - Overview Architecture Diagram = Context (L1) + Container (L2) combined
> - Detail Section 1 Component diagram = Component layer (L3)
> - Detail Section 5 Implementation Flows = Dynamic at Component level
>
> Standard C4 box / arrow notation applies throughout (box: name + tech + responsibility; arrow: action + protocol).

## Writing Principles

1. Every section traces back to a Goal. If a paragraph relates to no Goal, delete it.
2. If removing a paragraph still lets the reader make correct decisions, the paragraph should not exist.
3. Record **why**, not **what**. Code is the authoritative "what".
4. **Diagrams over prose.** Use words only for what diagrams cannot show. Omitting a diagram requires an explicit reason ("single linear step, no branching").
5. **Diagram format selection** (applies to all diagrams in this guide — Overview and Detail alike):
   - Simple flow (≤7 boxes, linear / tree-shaped call chain) → **ASCII** (zero tooling, diff-friendly, LLM reads it as 2D structure directly)
   - System architecture, state machine, complex data flow (needs boundaries / colors / change markers) → **Mermaid** (GitHub renders natively since 2022; `subgraph` + `classDef` cover styling needs; source remains diff-friendly)
   - **PlantUML**: escape hatch for cases Mermaid cannot express; reaching this tier usually means the diagram should be split instead
6. Expose uncertainty honestly. Hidden risks waste reviewers' attention.

## Part 1: Overview Design

**1. Problem & Goals**
- **Problem** — the pain point and trigger. Quantify if data exists. Describe the pain, not the solution.
- **Goals** — how success is judged. Quantify when possible. For performance / capacity / SLA goals, quantification is **not optional in Overview**. For other goals, if a number is unavailable now, write "to be quantified in Detail Design" — never leave the gap implicit.
- **Non-Goals** — required output. What this work explicitly does not cover. Drop any non-goal nobody would do anyway.

**2. Solution Design** — presents **the chosen solution** (alternatives belong in Section 3).

**Diagrams are required, and they come before prose.** Prose explains only what diagrams cannot — participant roles, dependency types, error-path triggers. Restating the diagram in words is grounds for review rejection.
- **Architecture Diagram** — a single C4 diagram combining Context (L1) and Container (L2) in one view: external actors and external systems outside the boundary, internal containers inside, stay above class/function level. The diagram must answer three questions: (1) what responsibility blocks exist, (2) **what data flows where, and why**, (3) where the key design decision lands. Rules:
  - Unit of analysis is a **responsibility module or service** — never a file or function
  - **Each box must list "core design" — 3–6 what-level bullets** (responsibilities, key constraints, traceability IDs back to Goals / Risks). Write what-level details; how-level details (algorithms, thresholds, field formats, function signatures) belong in Detail Design.

    <example>`行号化输出` · `read-gate (G3)` · `staleness 检查 (B2)`</example>
    <counterexample>`cat -n 输出 (lineno + tab + content)` · `> 256KB 拒` · `stored mtime != FS mtime → 拒`</counterexample>

  - **Arrows carry data flow with both payload and purpose**: each label takes the form `<what data> / <purpose>` (e.g., `path / read-gate check`, `user profile / authz check`). A reader looking at any single arrow alone must know what crosses it and why.
  - **Numbered execution sequence (⓪①②③) is optional**: use it when the design has a natural temporal flow that helps the reader follow the main path; skip for purely static structures or multi-entry systems where one sequence cannot represent all paths
  - Annotate the critical design decision directly on the diagram (e.g., "dispatch table — replaces if-elif chain")
  - Mark added / modified / removed via color or annotation
  - Split into separate Context + Container diagrams when box count exceeds ~15, or when external ecosystem is itself the core complexity
  - **Pass/fail test**: a reader who only reads the diagram (no prose) must be able to answer "how does this design achieve the goal?" If they cannot, the diagram is incomplete.

    <counterexample>File tree listing files and their new constants/functions — that is a change inventory, not an architecture</counterexample>
    <counterexample>Boxes labeled only with names; arrows labeled only with verbs or call sites</counterexample>
    <example>Responsibility boxes with what-level core design bullets, arrows labeled `data / purpose`, and a callout on the key decision</example>

- **Flow Diagrams** — one core flow (main path) and 1–2 edge flows (critical error / boundary paths). Participants are modules / services / roles. Keep function names out of flow diagrams — those belong in Detail. Flows must show execution order (numbered steps or directed arrows), not just static dependencies.

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

**1. Module Responsibilities & Interfaces**

When the module has ≥3 sub-modules, **first show a C4 Component diagram** of its internal structure (mark added / modified / removed), then list per-component contracts. The diagram is the visual map; the contracts are the precise boundaries.

Per-component contracts: full contract per component. Existing interfaces: signature only. New or modified: full definition (parameters, return, errors, preconditions).

<format>
```
TaskStore.acquire(taskId, agentId, ttl) -> Lease
  pre:    taskId exists; no live lease on it
  returns: Lease{ id, expiresAt }
  errors: TaskNotFound, AlreadyLeased
```
</format>

**2. Data Model / Schema** — per structure: fields with types and defaults; constraints (uniqueness, nullability, cross-field rules); query patterns and indexes; migration path with rollback when changing existing schema.

<format>
```
table tasks(
  id          uuid pk
  status      enum('pending','running','done','failed') not null
  lease_id    uuid null
  updated_at  timestamptz not null
) index (status, updated_at)
invariant: status='running' ⇒ lease_id is not null
```
</format>

**3. Core Algorithms / State Machines** — write only what a competent engineer cannot reconstruct in five minutes. State machines: transition diagrams with triggers and side effects. Algorithms: pseudocode plus complexity if performance matters. Document invariants — they become test cases.

<format>
```
pending --start()--> running --finish()--> done
                       │
                       └--timeout()--> failed   (lease released)
invariant: at most one running lease per task
```
</format>

**4. Error Handling** — for each external interaction (network, IO, user input, cross-module call): failure modes, handling (retry / degrade / propagate / ignore), and what the caller observes.

<format>
```
call           failure          handling          caller sees
storage.write  transient IO     retry x3 jitter   ok | PersistError
payment.api    timeout          fail-fast         PaymentTimeout
```
</format>

**5. Implementation Flows** — a **C4 Dynamic diagram at component level**: function-level call sequences between Section 1's components, with class/function names, parameters, and every branch. Only when more detail than the Overview flow is needed; otherwise write "see Overview" and skip.

Per Writing Principles #5: ≤2 branches or concurrent participants → ASCII call tree; ≥3 → Mermaid sequence diagram.

<example>
```
TaskController.start(req)
  └─ TaskService.start(taskId, agentId)
       ├─ TaskStore.acquire(taskId, agentId, ttl=30s)
       │    └─ AlreadyLeased → 409
       └─ Worker.dispatch(taskId)
            └─ dispatch error → release lease, 500
```
</example>

**6. Performance Estimation** — required when Goals are quantitative. Estimate throughput / latency on the critical path. State input assumptions (QPS, payload size) and the bottleneck. Stop once the estimate clears the Goal.

<format>
```
target: p99 < 200ms at 1000 QPS
path:   auth(3ms) + lookup(20ms) + work(80ms) + write(15ms) ≈ 118ms
bottleneck: write under contention → connection pool ≥ 32
```
</format>

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

<format>
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
</format>

**8. Migration & Compatibility** — required when modifying existing systems. Cover migration steps in order, canary / rollback plan, compatibility while old and new coexist.

<format>
```
1. add column status_v2 (nullable)
2. dual-write v1 + v2 for 24h
3. backfill v2 from v1
4. switch readers to v2 (canary 1% → 100%)
5. drop v1 after 7-day soak
rollback: keep dual-write, revert reader switch.
```
</format>
