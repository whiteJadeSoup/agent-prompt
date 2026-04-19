# Global Claude Instructions

## Answering Principles

Apply these two meta-principles to every "answering the user" scenario. They are **umbrella rules** — the scenario-specific rules in `instructions/*.md` are stricter enforced versions that take precedence in their own scenarios.

### 1. Think before talk — fact-based, not impression-based

Verify before answering. Whenever the answer involves code, system state, or factual claims, ground it in **actual evidence** (read the code, run the command, inspect the output, check the docs), not memory, impression, or "how it's usually done".

- Bug hunting: trace the actual execution path in code and reproduce the bug; never offer symptom-based guesses as the conclusion
- Explaining mechanics: describe what **this** code actually does, not "how people generally do it"
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

## Design
@instructions/design-thinking.md — thinking discipline for all design decisions (constraints, tradeoffs, reversibility, failure analysis)
@instructions/design-plan-guide.md — template for writing formal design documents (Overview Design first, then Detail Design)

## Implementation
@instructions/implementation.md — coding discipline: read before write, follow existing patterns, no speculative abstractions
@instructions/refactoring.md — refactoring protocol: separate from behavioral changes, test coverage first, scope control

## Quality
@instructions/bug-fixing.md — bug diagnosis protocol: reproduce → trace → single root cause → verify fix
@instructions/code-review.md — code review guide: PR submission, multi-agent review architecture, comment standards
@instructions/design-review.md — design review guide: Overview Design review, three sources of regret, multi-agent architecture
@instructions/testing.md — testing requirements: unit tests, e2e tests, pass/fail report

## Git
@instructions/git-conventions.md — git safety rules and conventional commit format
