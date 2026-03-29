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

## Testing Requirements

- Every feature or bug fix must include unit tests
- Write e2e tests for user-facing flows when applicable
- After coding is complete, run all tests and provide a pass/fail summary report before finishing

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
