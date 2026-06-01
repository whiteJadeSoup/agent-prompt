# Subagent Model Selection + Challenge-to-Consensus Orchestration

**Date:** 2026-06-01
**Status:** Proposed (awaiting review)
**Supersedes (partially):** `2026-05-23-instruction-token-optimization-design.md` — the part that moved code review and design review off sub-agents onto inline sequential passes. See "Relationship to the token-optimization doc" below.

---

## 1. Problem & Goals

### Problem
Two unrelated gaps in how the instruction set directs sub-agent work:

1. **No model policy.** When Claude Code delegates to a sub-agent (Agent tool / workflow `agent()`), nothing tells it which model to pick. Mechanical lookups burn Opus; correctness-critical reasoning can land on Haiku. The `model` lever exists (`haiku` / `sonnet` / `opus`) but is unused.
2. **One-shot acceptance in reviews.** The review guides synthesize whatever the workers return. A worker's finding — its proof, its severity — is taken at face value. There is no step where the orchestrator stress-tests a finding and forces the worker to defend or withdraw it.

### Goals
- **G1 — Model-by-task routing.** Every sub-agent dispatch picks its model from a task→model table, with a default rule for unlisted tasks. Verifiable: the rule names a model for each task type below, and a fallback.
- **G2 — Challenge-to-consensus orchestration.** The orchestrator never one-shot-accepts a worker finding; it challenges, the worker re-engages, they loop to consensus under a bounded protocol. Verifiable: the protocol has a termination guarantee and a defined escalation.
- **G3 — One umbrella, inherited by all disciplines.** Both rules live in `CLAUDE.md` as a top-level umbrella so every `instructions/*.md` discipline inherits them, rather than being re-stated per file.

### Non-Goals
- Not changing `bug-fixing.md`, `implementation.md`, `refactoring.md`, `testing.md`, etc. — they do not fan out to sub-agents.
- Not introducing model *version* IDs (e.g. `claude-opus-4-8`). The Agent tool exposes only the three tiers `haiku` / `sonnet` / `opus`; the rule speaks in tiers.
- Not changing the *content* of the review angles, comment format, labels, or conclusions in the review guides — only the execution architecture (who runs, on what model, with what loop).
- Not syncing `~/.claude/` in this change (offered separately; it is a lagging deployed copy).

---

## 2. Solution Design

### 2.1 Umbrella rule in `CLAUDE.md` (G3)

A new top-level section `## Subagent Execution`, peer to the two Answering Principles, with two parts.

**(a) Model selection (G1)**

```
Haiku   exploration / search · simple single-file edits · writing docs
Sonnet  multi-file implementation · all code-review workers (A,B,C)
        · all design-review workers
Opus    complex architecture · creative thinking / design
        · security analysis (standalone audit) · debugging complex bugs
        · orchestrator / final-answer synthesis

Unlisted task → default:
  miss-is-expensive OR deep reasoning OR "hold the whole system in mind" → Opus
  pure lookup / search / mechanical single-file edit                     → Haiku
  otherwise (general coding)                                             → Sonnet

Agent-type shortcut: Explore → Haiku · Plan → Opus · general-purpose → by task
Applies to: the Agent tool and workflow agent() model option.
```

**(b) Orchestration — challenge to consensus (G2)**

```
The orchestrator (Opus) never one-shot-accepts a worker's output.
For each finding:
  1. Challenge — ask a follow-up that stress-tests the claim
     (missing input, untested path, weak/none-executable proof).
  2. Worker re-engages — searches / re-reads / re-reasons, then either
     defends with stronger proof, or revises / withdraws the finding.
  3. Loop. Soft cap 5 rounds. If consensus is reached earlier, stop.
     If still unresolved after 5: the orchestrator makes the final call
     (keep | drop) and records WHY it overruled. Never loop unbounded;
     never silently drop a contested finding.
```

### 2.2 `code-review.md` — revert Part 0 to parallel sub-agents + loop

Current Part 0 ("Sequential Multi-Pass Execution") is replaced. Workers run **in parallel**, all on **Sonnet**; the orchestrator runs on **Opus** and applies the challenge loop.

```
PR diff + guide
   ├─▶ Agent A  correctness & safety (incl. security)  ─ Sonnet ┐
   ├─▶ Agent B  quality & resilience                   ─ Sonnet ├ parallel
   └─▶ Agent C  clarity & coverage                     ─ Sonnet ┘
            │ findings JSON
            ▼
   Orchestrator (OPUS):
     for each finding:  challenge ──▶ Agent re-engages (search / reason)
                            ◀── defend w/ stronger proof | revise | withdraw
                        └─ loop ≤5 rounds → consensus, else orchestrator
                           decides (keep|drop) + records why ─┘
            ▼
   dedup · resolve conflicts · format · conclude · ANSWER
```

- The "do not spawn sub-agents — cache" note is removed.
- Output schema reverts from `"pass": "A"` to `"agent": "A"` (and the responsibilities table from "Pass A/B/C" back to "Agent A/B/C").
- **Security note:** security is an *angle* inside Agent A and runs on Sonnet like the rest of the review; the Opus orchestrator's challenge loop is the safety net for it. The table's `security analysis → Opus` is for a *standalone* security audit (e.g. the `security-review` flow), not the security angle embedded in a PR review.

### 2.3 `design-review.md` — same treatment, both phases

**Overview Review** (§ "Execution Architecture", lines ~21–83): Pass A/B/C → **Agent A/B/C, parallel, Sonnet**; orchestrator **Opus** + challenge loop. Remove the cache note (line ~25).

**Detail Review** (§ "Execution Architecture", lines ~237–269): Section 0 runs first (coverage map), then Sections 1–8 as **parallel section sub-agents on Sonnet**; orchestrator **Opus** + challenge loop; the coverage map from Section 0 is injected into each section agent. Remove the cache note (line ~241).

```
Overview Review                         Detail Review
  A wrong problem        ─ Sonnet ┐       Section 0 coverage map (first)
  B wrong direction      ─ Sonnet ├par      │ → injected into all sections
  C unrecognized reality ─ Sonnet ┘         ▼
            ▼                              Sections 1–8 ─ Sonnet · parallel
  Orchestrator (OPUS) — challenge ↔ consensus loop (≤5, else decide+record)
```

---

## 3. Research & Comparison

### Why revert to parallel sub-agents (vs. keep inline passes)

| Dimension | Inline sequential passes (current) | Parallel sub-agents (proposed) |
|---|---|---|
| Prompt cache | Warm — one context reused across passes | Cold per worker; diff+guide re-processed N× |
| Wall-clock | Serial: sum of passes | Parallel: ~slowest worker |
| Model specialization | All passes on the session model | Per-worker tier (Sonnet workers, Opus orchestrator) |
| Adversarial verification | Same context, prone to confirmation | Independent worker contexts + Opus challenge loop |

The inline design optimized **input-token cost** (cache reuse). This design optimizes **wall-clock + judgment quality** (parallelism, cheap workers, Opus-only synthesis with an adversarial loop), knowingly re-accepting the cache cost. This is a deliberate re-bet, not an oversight.

### Risks

- **Type A — cost of choosing.** Re-processing the diff/guide prefix per worker costs more input tokens than the cached inline design. Accepted in exchange for parallelism, model specialization, and the consensus loop. Mitigation: workers run on Sonnet (cheaper per token), Opus is reserved for the orchestrator.
- **Type B — intrinsic fragility.** The consensus loop can fail to converge (worker keeps defending, orchestrator keeps doubting). Mitigation: **soft cap of 5 rounds, then the orchestrator decides and records why** — a hard termination guarantee with no silent drops. Second fragility: the model table can misclassify a nuanced task; mitigation is the default rule, which errs to Opus for anything correctness-critical and reserves Haiku for clearly-mechanical work.

---

## 4. Scope Boundaries

**Does:** add the `## Subagent Execution` umbrella to `CLAUDE.md`; revert + rewire Part 0 of `code-review.md`; revert + rewire both Execution Architecture blocks of `design-review.md`; add the challenge-to-consensus loop to both review guides.

**Does not:** touch other `instructions/*.md`; alter review angles / labels / comment format / conclusions; add model version IDs; rewrite the token-optimization doc (only a one-line "superseded by" pointer, if desired); sync `~/.claude/`.

---

## 5. Failure Analysis

- **What breaks first if the model table is wrong:** a misclassified task gets the wrong tier — a deep task on Haiku (quality loss) or a trivial task on Opus (cost). The default rule and the "miss-is-expensive → Opus" bias contain the damage; fully reversible (text edit).
- **What happens if the consensus loop is unavailable** (e.g. the orchestrator cannot continue a worker's context): fall back to re-dispatching a fresh verification sub-agent with the prior finding + challenge as input. The protocol is specified behaviorally, not bound to one mechanism.
- **Assumption that, if wrong, invalidates this:** that parallel sub-agents are actually available/affordable in the runtime where these guides execute. If a runtime forbids sub-agents, the inline-pass design remains the documented fallback — but it is no longer the default.

---

## 6. Relationship to the token-optimization doc

`2026-05-23-instruction-token-optimization-design.md` moved code/design review to inline passes specifically to avoid N× prefix re-processing. This design reverses that single decision for the two review guides, on the basis that model specialization + the adversarial consensus loop are worth the cache cost. The token-optimization doc's other changes (CLAUDE.md "repo code questions" rule, implementation.md trim) are untouched.
