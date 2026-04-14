# Design Plan Guide

A design plan is a single document with two parts written in sequence: **Overview Design** (why + what) first, **Detail Design** (how) after the overview is confirmed. Not every change needs a design plan — only write one when the change involves design decisions others need to understand (cross-module changes, new modules, architecture changes, multi-person collaboration).

**Workflow:**
1. Write Part 1 (Overview Design) and get it confirmed before proceeding.
2. Write Part 2 (Detail Design) in the same document, below the confirmed overview. The final document is self-contained — Overview Design at the top provides the full context, so readers understand the complete design from this single document.

> **Apply Design Thinking Rules when working through each section.**

## Part 1: Overview Design

The six sections form a decision chain — each exists because without it, the next section loses its basis for judgment:

```
Why do it? → How far? → How to split? → How does it run? → Why this split? → What's uncertain?
    │            │            │               │                  │                  │
 Problem    Goals/Non-    Architecture    Core Flows        Key Decisions     Risks & Open
 Statement  Goals         Overview                                            Questions
```

**1. Problem Statement** — Describe the current pain point and what triggered this work. Include quantitative data if available. Do NOT describe the solution here.

**2. Goals & Non-Goals**

Goals must be verifiable. How to express them depends on the change type:

**For user-facing features** — write each goal as a User Story:
```
As a <specific role>,
I want to <behavior, not a feature>,
so that <the value or outcome>.
```
Each User Story must include **Acceptance Criteria** covering three paths:
- **Happy path**: the core scenario works correctly
- **Edge case**: boundary conditions (empty, max-length, concurrent)
- **Error path**: how the system responds on failure

Use Given/When/Then format for AC — each criterion must be independently testable.

Self-check before moving on:
1. If you swap the Role for a different user type, does the Story still hold? If yes, the Role is too generic.
2. Remove "so that" — does the team still know why this feature exists? If not, rewrite it.
3. Can each AC be written as a standalone test case? If not, the AC is too vague.

*Example:*
> As a data analyst,
> I want to search metrics by keyword from the CLI,
> so that I can quickly locate a metric without knowing its exact name.
>
> Acceptance Criteria:
> - Given keyword "gmv", When running `metric search gmv`, Then stdout returns a list containing "gmv" (JSON format)
> - Given no matches, Then stdout returns `[]`, exit code 0
> - Given network timeout, Then stderr outputs error message, exit code 1

**For technical changes** (refactoring, performance, infrastructure) — write verifiable technical targets directly. User Story format is not required and should not be forced.

*Example:* "Reduce p99 API latency from 800ms to under 200ms under 1000 concurrent requests"

Non-goals are more important than goals — they prevent scope creep.
- How to identify non-goals: (a) requirements stakeholders might request but are excluded this time, (b) natural extensions of adjacent problems, (c) boundaries where the team has divergent opinions.
- Litmus test: if a non-goal is something nobody would do anyway, it carries no information — delete it.

**3. Architecture Overview** (diagram required — ASCII art, Mermaid, or any text-based format)
- **Draw an architecture diagram (ASCII art, Mermaid, or any text-based format). This section is not complete without one** — prose description alone is not acceptable.
- Draw the full system, highlight incremental changes (solid lines = existing, bold/dashed = new/modified, gray/strikethrough = removed).
- Group by responsibility, not by code structure.
- Only draw the layers the reader needs to understand. If the change is application-level, infrastructure is just a labeled box.
- Arrows show dependency direction; label the interaction type (sync call / async event / polling).

**4. Core Flows** (sequence or flow diagram required — ASCII art, Mermaid, or any text-based format)
- **Draw a sequence or flow diagram for each core flow (ASCII art, Mermaid, or any text-based format). This section is not complete without one** — skip only if the entire flow is a single linear step with no branching, error path, or concurrency.
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

## Part 2: Detail Design

> **Prerequisite:** Part 1 (Overview Design) must be confirmed before writing this part. The confirmed overview stays at the top of this document as context — each section below should trace back to its architecture, flows, decisions, and goals.

**1. Module Responsibilities & Interfaces** — Full contract, mark incremental changes. Existing interfaces: signature only. New/modified: full definition (params, return, errors, preconditions).

**2. Data Model / Schema** — For each structure, answer: what is the structure (fields, types, optionality, defaults), what are the constraints (uniqueness, nullability, cross-field dependencies), how is it queried (indexes, common query patterns), how does it migrate (if modifying existing schema: migration path, backward compatibility, rollback).

**3. Core Algorithms / State Machines** — Only write what a competent engineer can't figure out in under 5 minutes from the requirements. State machines: transition diagrams with trigger conditions and side effects. Algorithms: pseudocode with complexity if performance-relevant. Document invariants — they are direct sources for test cases.

**4. Error Handling** — For each external interaction point (network, file IO, user input, cross-module calls), answer: how can it fail, how do we handle it (retry/degrade/propagate/ignore), how does the caller know.

**5. Key Flows** — Same flows as overview, but with class/function-level participants, full parameter details, and all branches the implementer needs to know.

**6. Performance Estimation** (required when goals have quantitative targets) — Prove the design meets the goal without over-engineering. Estimate throughput/latency on the critical path. Stop when the goal is met — don't optimize for 500k when the target is 100k.

**7. Testing Strategy** — Answer three questions: what to test (which modules/flows, prioritize core invariants), how to test (unit/integration/E2E split, where to draw mock boundaries and why), pass criteria (trace back to design goals).

**8. Migration & Compatibility** (when modifying existing systems) — Migration steps and order, canary/rollback plan, compatibility during coexistence.

## Writing Principles

1. Every section must trace back to a design goal. If it doesn't relate to any goal, delete it.
2. If removing a paragraph still lets the reader make correct decisions, that paragraph shouldn't exist.
3. Record **why**, not **what**. Code is the authoritative "what".
4. **Diagrams over prose — default to diagrams (ASCII art, Mermaid, or any text-based format).** Use text only to explain what isn't obvious from the diagram. Omitting a diagram requires an explicit reason (e.g., "single linear step, no branching").
5. Expose uncertainty honestly. Hiding risks wastes reviewers' attention.
