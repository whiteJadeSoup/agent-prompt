# Instruction Clarity & Compression — Design

**Date:** 2026-06-03
**Status:** Approved (brainstormed 2026-06-03), pending implementation
**Relationship:** Extends `2026-05-23-instruction-token-optimization-design.md` (which trimmed redundant *examples* conservatively). This pass applies four prompt-engineering principles repo-wide and adds cross-guide deduplication. The 2026-06-01 subagent-architecture design's *intent* is preserved — only its expression is compressed.

## Problem

`CLAUDE.md` + the 9 `instructions/*.md` files load as always-on context every session (twice in this repo: global mirror + project copy). They total ~1,192 lines / ~10,706 words / ~71 KB per copy, dominated by `code-review.md` (430 lines) and `design-review.md` (367 lines). The text is verbose, dilutes attention, and costs tokens on every turn. Four prompt-engineering levers are under-applied:

1. **Conciseness** — prose restates what tables/diagrams already show; rules are wordy.
2. **XML-tag delimiting** — examples, formats, and hard stops blur into surrounding prose; Claude parses tagged boundaries more reliably (Anthropic best practice).
3. **Positive phrasing** — many directives are framed as prohibitions ("do not…", "never…", "Anti-patterns").
4. **Example relevance/diversity** — examples pile on the same principle and read as one Java app (repeated `ModelService`/duplicate-name scenario).

## Goals

- **G1 — Apply the four principles repo-wide.** Verifiable: every file uses the tag vocabulary below; prohibition-style guidance is converted to positive directives except where `<avoid>` is justified; no two surviving examples illustrate the same rule.
- **G2 — Deduplicate the triplicated review machinery.** Verifiable: the findings schema, label taxonomy, author-response/thread-resolution, and re-review rules appear once (in `code-review.md`), and `design-review.md` references them.
- **G3 — Preserve behavioral fidelity.** Verifiable: every dropped *directive* (not example/wording) appears in an approved cut list; the diff audit finds nothing else removed.
- **Quantification:** target ~40–50% fewer lines across the set; the two review guides are the primary targets.

## Non-Goals

- **No skill-loaded modules** — rules stay always-on (carried veto from 2026-05-23).
- **No asymmetric global/project layout** — all edits symmetric and mirror-able to `~/.claude/` (carried veto).
- **No change to the *intent*** of the multi-agent review architecture (2026-06-01 design) — angles, agent rosters, model tiers, the consensus protocol all keep their meaning; only their expression is compressed.
- **No change to `docs/specs/*`** — not always-on context.

## Decisions (from brainstorming)

1. **Cut depth = B:** compress expression *and* make judgment content cuts (merge overlapping directives, drop low-value ones). Every *content* cut is flagged for approval before commit; expression-compression, tagging, and positive-rephrasing proceed without itemization.
2. **Structure = Approach 2, refined:** `code-review.md` is the canonical owner of review-shared machinery; `design-review.md` references it; `CLAUDE.md` keeps only the generic Subagent Execution umbrella it already has.

```
CLAUDE.md › Subagent Execution        generic only (already exists, tightened)
   model table · challenge-to-consensus loop · "fan out → Opus orchestrator"

code-review.md  = canonical owner:
   findings JSON schema · label taxonomy + response rules
   author-response + thread-resolution · re-review rules

design-review.md  references code-review.md for the above;
   keeps OWN: three-regrets, regret-agents + core questions, questioning
   paths, per-agent checklists, Detail Review, and discipline-specific
   overrides (blocking = "fails design goals"; conclusions = Confirm/…)
```

3. **XML tag vocabulary** (semantic, content-type boundaries — *not* every sentence):

| Tag | Wraps |
|-----|-------|
| `<example>` / `<counterexample>` | a worked example (replaces `✅` / `❌` markers) |
| `<format>` | a template emitted ~verbatim (findings JSON, PR-description template, comment template) |
| `<gate>` | a hard stop that must never be missed (e.g. "give a conclusion", force-push refusal) |
| `<avoid>` | the exception-case prohibition list, used only to counter a strong default tendency |

Markdown headers + tables remain for all other structure (token-efficient, GitHub-rendered).

4. **Positive phrasing:** action-first by default ("Implement only what was asked" not "don't add features"). Hard stops phrased as strong positive requirements that still read as stops ("Get explicit confirmation before `git push`"). Explicit prohibitions kept only inside `<avoid>`, where countering a default tendency *is* the point.

5. **Example policy:** each surviving example is the minimal illustration of *its own* rule; diversify scenarios so they don't all read as one app; keep the strong, already-diverse ones (NPE `[blocking]` comment, `OrderService` `[suggestion]`); eval each survivor against "illustrates *this* rule, non-redundant".

## Anticipated content cuts (B — approve per file during execution)

The exhaustive list is finalized per file at execution time (each surfaced before commit). Notable candidates identified now:

- **design-review.md** — replace the restated findings JSON schema with a reference to `code-review.md`; replace the re-review section's restated rules with a reference; keep its discipline-specific blocking standard and conclusions.
- **code-review.md** — fold the "Anti-patterns"/"Do not flag" rows into a single `<avoid>` block; merge the Step-2 skeptical-reading patterns that overlap with the Performance angle-table rows (state each concern once).
- **implementation.md** — convert the "Anti-patterns (never do these)" list into positive directives folded into the matching rules.
- **CLAUDE.md / both guides** — guides reference the Subagent Execution umbrella for the consensus-loop mechanics instead of restating them.

## Risks

- **Type A — cost of choosing.** Dedup couples `design-review.md` to `code-review.md` (it stops being readable in total isolation). Mitigated: `design-review.md` already references code-review; both always load alongside `CLAUDE.md`; the orchestrator already injects the full guide into sub-agents.
- **Type B — intrinsic fragility.** Aggressive compression could silently drop a load-bearing nuance. Mitigated: B's per-file approval gate, the diff audit, and the completeness walkthrough below.

## Verification

Docs — no automated tests apply (`testing.md` governs code). Two checks before each commit:

1. **Diff audit** — every removed *directive* appears in the approved cut list for that file; nothing else (no `must`/`required`/`should`/`never` sentence) is silently lost.
2. **Completeness walkthrough** — mentally re-run one recent code-review task and one design task against the trimmed files; restore anything left ungrounded.

## Implementation order

1. `CLAUDE.md` — tighten the umbrella + answering principles; establishes the tag vocabulary and positive style for the rest.
2. Small files — `design-thinking`, `implementation`, `refactoring`, `bug-fixing`, `testing`, `git-conventions`.
3. `code-review.md` — canonical review owner (largest cut).
4. `design-review.md` — reference code-review; compress unique content.
5. `design-plan-guide.md`.

Commit per logical group; surface content cuts for approval per file. User syncs `instructions/*.md` → `~/.claude/` manually afterward.

## Open Questions

None — depth (B), structure (Approach 2 refined), tag vocabulary, and phrasing policy all confirmed during brainstorming.
