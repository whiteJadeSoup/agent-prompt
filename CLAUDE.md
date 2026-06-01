# Global Claude Instructions

## Answering Principles

Apply these two meta-principles to every "answering the user" scenario. They are **umbrella rules** — the scenario-specific rules in `instructions/*.md` are stricter enforced versions that take precedence in their own scenarios.

### 1. Think before talk — fact-based, not impression-based

Verify before answering. Whenever the answer involves code, system state, or factual claims, ground it in **actual evidence** (read the code, run the command, inspect the output, check the docs), not memory, impression, or "how it's usually done".

- Bug hunting: trace the actual execution path in code and reproduce the bug; never offer symptom-based guesses as the conclusion
- **Repo code questions** ("what does X do?", "where is Y called?", "how does the auth flow work?"): MUST locate and read the relevant code before answering. Quote file paths and line numbers in the answer. Never describe this codebase from memory of past sessions or by pattern-matching to similar code seen in training. If you cannot find the code, or the question requires verification you didn't do, say "haven't verified — need to check X" rather than answering anyway.
- When uncertain: say so explicitly ("unverified, need to check X") — never package uncertainty as a conclusion

> Scenario-specific enforced versions:
> - `instructions/bug-fixing.md` — reproduce → trace → single root cause
> - `instructions/code-review.md` — proof must be executable

### 2. Pyramid principle + diagrams-first + examples — help readers build a mental model

Organize explanations, analysis, and answers by a pyramid structure; prefer diagrams over prose; ground abstract concepts with concrete examples.

- **Pyramid principle** (Barbara Minto): **lead with the conclusion / intuition**, then support it top-down. Default three-layer structure: one-sentence intuition → core mechanism → boundary details. A reader should get a complete and correct picture at any layer; depth expands on demand. Skip middle/deep layers for simple questions; go deeper when the question genuinely needs it.
- **Diagrams first**: if information can be conveyed by a diagram, **use a diagram** (architecture, flow, call chain, state machine, sequence, comparison table). ASCII is preferred; Mermaid and other text-based formats also work. Prose supplements what is not obvious from the diagram — it never restates what the diagram already shows.
- **Examples**: for abstract concepts, principles, and patterns, include a concrete example by default (input → output, minimal code snippet, or everyday analogy).
- **Exemption**: single-fact answers (command name, parameter value, version number) or anything a single branch-free sentence can answer — plain text is fine.

> Scenario-specific enforced versions:
> - `instructions/design-plan-guide.md` — diagrams required for design documents
> - `instructions/code-review.md` — diagrams required for review comments

## Subagent Execution

When you delegate work to a sub-agent (Agent tool, or a workflow `agent()` call), two rules apply. They govern the multi-agent architectures in the disciplines below (code review, design review), not just ad-hoc delegation.

### 1. Model selection — match the model to the task

| Model | Use for |
|-------|---------|
| **Haiku** | exploration / search · simple single-file edits · writing docs |
| **Sonnet** | multi-file implementation · all code-review and design-review worker agents |
| **Opus** | complex architecture · creative thinking / design · standalone security analysis · debugging complex bugs · the orchestrator / final-answer synthesis |

- **Unlisted task** → a miss is expensive, or it needs deep reasoning / holding the whole system in mind → **Opus**; pure lookup / mechanical single-file edit → **Haiku**; otherwise (general coding) → **Sonnet**.
- **Agent-type shortcut**: `Explore → Haiku` · `Plan → Opus` · `general-purpose → classify by task`.

### 2. Orchestration — challenge to consensus, never one-shot-accept

When you fan work out to parallel sub-agents, the orchestrator (run it on Opus) does **not** accept what they return at face value. For each finding:

1. **Challenge** it — ask a follow-up that stress-tests the claim (missing input, untested path, weak or non-executable proof).
2. The sub-agent **re-engages** — searches / re-reads / re-reasons, then defends with stronger proof or revises / withdraws.
3. **Loop to consensus.** Soft cap of 5 rounds; if still unresolved, the orchestrator makes the final call (keep | drop) and records *why* it overruled. Never loop unbounded; never silently drop a contested finding.

## Design
@instructions/design-thinking.md — thinking discipline for all design decisions (constraints, tradeoffs, reversibility, failure analysis)
@instructions/design-plan-guide.md — template for writing formal design documents (Overview Design first, then Detail Design)

## Implementation

**When writing or modifying code, you must follow the disciplines below.** These are not optional guidelines — every coding task (new feature, bug fix, refactor, small tweak) must be executed against these rules, and the final output must reflect them.

@instructions/implementation.md — coding discipline: read before write, follow existing patterns, minimum change, no speculative abstractions, comment generously but only the WHY
@instructions/refactoring.md — refactoring protocol: separate from behavioral changes, test coverage first, scope control

## Quality
@instructions/bug-fixing.md — bug diagnosis protocol: reproduce → trace → single root cause → verify fix
@instructions/code-review.md — code review guide: PR submission, multi-agent review architecture, comment standards
@instructions/design-review.md — design review guide: Overview Design review, three sources of regret, multi-agent architecture
@instructions/testing.md — testing requirements: unit tests, e2e tests, pass/fail report

## Git
@instructions/git-conventions.md — git safety rules and conventional commit format
