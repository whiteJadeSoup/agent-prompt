# Instruction Clarity & Compression — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Apply four prompt-engineering principles (conciseness, XML tags, positive phrasing, relevant/diverse examples) to `CLAUDE.md` + the 9 `instructions/*.md` files, and deduplicate the triplicated review machinery — reducing always-on context ~40–50% without dropping any unapproved directive.

**Architecture:** Per-file rewrite in the spec's implementation order. `code-review.md` becomes the canonical owner of review-shared machinery (findings schema, label taxonomy, author-response, re-review); `design-review.md` references it; `CLAUDE.md` keeps only the generic Subagent Execution umbrella. Every *content* cut (merged/dropped directive) is surfaced for user approval before its commit; expression-compression, tagging, and positive-rephrasing need no approval.

**Tech Stack:** Markdown. No build, no automated tests. Verification = diff audit + completeness walkthrough (see spec). Source of truth = repo files; user mirrors to `~/.claude/` manually after.

**Spec:** `docs/superpowers/specs/2026-06-03-instruction-clarity-and-compression-design.md`

---

## Conventions applied in EVERY task (the "definition of done" for a file)

These are mechanical and need no per-file approval. Apply uniformly:

1. **Tag vocabulary** — wrap content-type boundaries (not every sentence):
   - worked examples → `<example>` (good) / `<counterexample>` (anti-pattern), replacing `✅`/`❌` markers
   - templates emitted ~verbatim → `<format>` (findings JSON, PR-description template, comment template, schema/table snippets)
   - hard stops that must never be missed → `<gate>`
   - prohibition lists kept only to counter a default tendency → `<avoid>`
2. **Positive phrasing** — convert "do not / never / Anti-patterns" to action-first directives. Hard stops become strong positive requirements (e.g. "Get explicit confirmation before `git push`"). Keep an explicit "don't" only inside `<avoid>`.
3. **Conciseness** — delete prose that restates a table/diagram; collapse multi-sentence rules to one line where meaning survives.
4. **Examples** — each surviving example illustrates exactly one rule; no two examples illustrate the same rule; diversify scenarios away from the repeated `ModelService` app.

**Diff-audit gate (the "test") for every file** — before committing, run `git diff <file>` and confirm:
- every removed sentence containing `must`/`required`/`should`/`never`/`do not` is listed in that task's approved cut list — nothing else;
- tag vocabulary is applied to all examples/formats/gates in the file;
- no prohibition survives outside `<avoid>` or a `<gate>`-phrased positive requirement.

---

## Task 1: CLAUDE.md (sets the house style)

**Files:**
- Modify: `CLAUDE.md`

- [ ] **Step 1: Read the file**

Run: read `CLAUDE.md` in full (73 lines). Confirm current sections: Answering Principles (2 meta-principles), Subagent Execution (model table + consensus), and the `@instructions/*.md` import index.

- [ ] **Step 2: Propose content cuts for approval**

Surface this cut list to the user; commit only the approved subset:
- C1: In "Think before talk", the three bullets (bug-hunting / repo-code-questions / when-uncertain) restate the lead sentence with examples. Merge to two tight bullets, keeping the repo-code-questions directive (read code, quote `file:line`, say "unverified" otherwise) verbatim in intent.
- C2: In "Pyramid principle", collapse the three sub-bullets (pyramid / diagrams-first / examples) — keep each directive, drop the parenthetical re-explanations.
- No directive removed; these are merges. (If the user wants any kept expanded, restore.)

- [ ] **Step 3: Rewrite applying conventions**

Concrete transformations:
- Positive phrasing — `"never package uncertainty as a conclusion"` → `"State uncertainty explicitly: 'unverified, need to check X.'"`
- Tag the consensus-loop steps and model table region under `## Subagent Execution` as the canonical umbrella (no `<format>` needed — it's a table). Add one line establishing it as the source other guides reference: `> The two review guides inherit this umbrella; they do not restate the loop or the model table.`
- Keep the `@instructions/*.md` import index unchanged (it is the loader, not prose).

- [ ] **Step 4: Diff-audit**

Run: `git diff CLAUDE.md` — confirm only C1/C2 merges and wording changes; model table + consensus protocol intact; import list byte-identical.

- [ ] **Step 5: Commit**

```bash
git add CLAUDE.md
git commit -m "docs(claude): tighten answering principles and subagent umbrella"
```

---

## Task 2: Small files batch (design-thinking, implementation, refactoring, bug-fixing, testing)

**Files:**
- Modify: `instructions/design-thinking.md`, `instructions/implementation.md`, `instructions/refactoring.md`, `instructions/bug-fixing.md`, `instructions/testing.md`

- [ ] **Step 1: Read all five files**

- [ ] **Step 2: Propose content cuts for approval**

- `implementation.md` — convert "Anti-patterns (never do these)" list into positive directives folded into the matching rules (rule 3 "change the minimum" absorbs "adding features beyond requested"; rule 5 "no speculative abstractions" already covers premature utilities; "leaving TODO comments" → fold into rule 3 as "finish it now, don't leave a TODO"). Net: the standalone Anti-patterns header is removed; every directive survives inside a rule.
- `bug-fixing.md` — "Language rules" prohibitions ("Never write 'the issue is likely X'") → one positive directive: "State each finding as confirmed or explicitly unverified ('not yet confirmed, need to check Y')."
- `design-thinking.md`, `refactoring.md`, `testing.md` — no content cuts; expression + phrasing only.

- [ ] **Step 3: Rewrite each applying conventions**

Concrete:
- `implementation.md` rule 7 comment examples → `<counterexample>` (the `# Increment counter by 1` block) and `<example>` (the retry/migration/list-vs-set block). Keep all three good examples (each shows a distinct why-comment pattern — already diverse).

<counterexample>
# Increment counter by 1
counter += 1
</counterexample>

<example>
# Retry up to 3× — downstream has a known transient failure during its
# 02:00 UTC index rebuild (incident #4821).
for attempt in range(3): ...
</example>

- `bug-fixing.md` output-requirements (root cause / evidence / fix / verification) → keep as numbered list, tighten wording.
- `testing.md` (5 lines) → leave near-as-is; add `<gate>` around "run all tests and provide a pass/fail summary before finishing".

- [ ] **Step 4: Diff-audit each file** (per the gate above)

- [ ] **Step 5: Commit**

```bash
git add instructions/design-thinking.md instructions/implementation.md instructions/refactoring.md instructions/bug-fixing.md instructions/testing.md
git commit -m "docs(instructions): positive phrasing and tag examples in core disciplines"
```

---

## Task 3: git-conventions.md

**Files:**
- Modify: `instructions/git-conventions.md`

- [ ] **Step 1: Read the file**

- [ ] **Step 2: Propose content cuts for approval**

- No content cuts. The commit-type table and merge-conflict policy stay (prior design vetoed compressing the git table). Expression + tagging only.

- [ ] **Step 3: Rewrite applying conventions**

- Wrap the three hard safety rules in `<gate>`: push needs confirmation; PR-create needs confirmation; force-push is refused unless the user says "我知道风险，强制推送". Phrase as positive requirements where natural, but force-push stays an explicit refusal inside `<gate>` (a true hard stop — the prohibition framing carries the weight).
- Merge-conflict policy: keep both branches (additive → proceed; destructive → confirm). Tighten wording.

- [ ] **Step 4: Diff-audit**

- [ ] **Step 5: Commit**

```bash
git add instructions/git-conventions.md
git commit -m "docs(git): gate safety rules and tighten conflict policy"
```

---

## Task 4: code-review.md (canonical review owner — largest cut)

**Files:**
- Modify: `instructions/code-review.md`

- [ ] **Step 1: Read the file in full** (430 lines)

- [ ] **Step 2: Propose content cuts for approval**

- C1: Fold the three "Do not flag" rows (formatting / personal-preference / pre-existing) into a single `<avoid>` block.
- C2: Merge the Step-2 skeptical-reading "scale/load assumption" bullets with the "Performance — hot path / IO / scale cliff / resource limits" angle-table rows — state each performance concern once, cross-reference instead of restating.
- C3: Drop the five one-line `❌` wrong-examples in the PR-description subsections (per 2026-05-23 design — prose lead already states the principle). Keep all `✅` examples.
- C4: Drop the integrated end-to-end PR-description example (lines ~163–210 in the pre-edit file) — the per-subsection examples already cover it (per 2026-05-23).
- Mark this file as canonical owner of the shared machinery (no cut — design-review will reference it).

- [ ] **Step 3: Rewrite applying conventions**

- `<format>` around: the findings JSON schema, the PR-description template, the comment template, the `[blocking]` comment example, the `[suggestion]` (OrderService) example.
- `<example>` / `<counterexample>` for the surviving PR-description examples.
- Part 0 multi-agent execution: keep the 3-agent table + execution-flow diagram; replace the restated consensus-loop prose with: `> Orchestration follows the challenge-to-consensus loop in CLAUDE.md › Subagent Execution.`
- Keep the review-angle tables (🔴/🟡) intact except C1/C2; convert any stray "never/do not" in prose to positive.
- Confirm examples are diverse: NPE `[blocking]` (Java) and OrderService `[suggestion]` (Java) — diversify one to a non-Java scenario (e.g. a concurrency/idempotency example in pseudocode or a different domain) so the two canonical examples don't both read as the same stack. Surface this example-swap in Step 2's list if it changes a directive (it does not — it's example content).

- [ ] **Step 4: Diff-audit + completeness walkthrough**

- Diff-audit per the gate.
- Walkthrough: mentally run a real PR review using only the trimmed file — confirm submission template, skeptical patterns, angles, comment format, labels, and conclusion are all still reachable. Restore anything ungrounded.

- [ ] **Step 5: Commit**

```bash
git add instructions/code-review.md
git commit -m "docs(code-review): dedup as canonical review owner, tag formats, positive phrasing"
```

---

## Task 5: design-review.md (references code-review)

**Files:**
- Modify: `instructions/design-review.md`

- [ ] **Step 1: Read the file in full** (367 lines)

- [ ] **Step 2: Propose content cuts for approval**

- C1: Replace the restated findings JSON schema with: `> Findings use the JSON schema in code-review.md › Sub-agent Output Format, with "agent" set to the regret-source (A/B/C) or "Section N".`
- C2: Replace the "Author's Responsibility After Receiving Comments" body (already a near-pointer) with a one-line reference to code-review.md Part 3.
- C3: Replace the "Re-review Responsibilities" restated rules with a reference to code-review.md Part 3.5, keeping only the design-specific conclusion set (Confirm / Request Changes / Comment).
- KEEP (discipline-specific, not cuts): three-regrets framing, the regret-agents + core questions, direction/premise questioning paths, per-agent checklists, the full Detail Review (Section 0 coverage + Sections 1–8), and the design-specific `[blocking]` standard ("system fails to achieve design goals").

- [ ] **Step 3: Rewrite applying conventions**

- `<example>` around the premise/direction-challenge examples.
- Both Execution Architecture blocks (Overview + Detail): keep the diagrams; replace restated consensus-loop prose with the same one-line reference to CLAUDE.md › Subagent Execution.
- Compress the Detail Review Section 0–8 prose (each section's question list → tightened bullets); keep every distinct check.

- [ ] **Step 4: Diff-audit + completeness walkthrough** (run a design Overview review against the trimmed file)

- [ ] **Step 5: Commit**

```bash
git add instructions/design-review.md
git commit -m "docs(design-review): reference code-review for shared machinery, compress detail review"
```

---

## Task 6: design-plan-guide.md

**Files:**
- Modify: `instructions/design-plan-guide.md`

- [ ] **Step 1: Read the file in full** (172 lines)

- [ ] **Step 2: Propose content cuts for approval**

- C1: The C4-mapping preamble and "Writing Principles" partly overlap with per-section diagram rules — merge duplicated diagram-format guidance (the "Diagram format selection" list appears conceptually twice). State the ASCII/Mermaid/PlantUML selection rule once.
- KEEP: the Overview (Problem/Goals/Non-Goals, Solution Design, Research & Comparison, Open Questions) and Detail (Sections 1–8) templates, and the architecture-diagram pass/fail rules — these are the guide's substance.

- [ ] **Step 3: Rewrite applying conventions**

- `<example>` around the `✅`/`❌` architecture-diagram bullet examples and the per-section code-block templates → `<format>`.
- Convert "❌ File tree listing…" / "❌ Boxes labeled only with names" into a `<counterexample>` block; keep the `✅` as `<example>`.
- Tighten prose that restates the diagrams-first principle.

- [ ] **Step 4: Diff-audit**

- [ ] **Step 5: Commit**

```bash
git add instructions/design-plan-guide.md
git commit -m "docs(design-plan): dedup diagram-format guidance, tag templates and examples"
```

---

## Task 7: Final verification + spec/sync wrap-up

**Files:** none modified

- [ ] **Step 1: Whole-set diff audit**

Run: `git log --oneline -7` and `wc -l CLAUDE.md instructions/*.md`. Confirm line count dropped ~40–50% vs. the baseline (1,192 lines) and every commit's cuts were approved.

- [ ] **Step 2: Cross-reference integrity check**

Confirm every `> references code-review.md …` / `> CLAUDE.md › Subagent Execution` pointer resolves to a section that actually exists with that name. Fix any dangling reference.

- [ ] **Step 3: Commit the design spec (if not already)**

```bash
git add docs/superpowers/specs/2026-06-03-instruction-clarity-and-compression-design.md docs/superpowers/plans/2026-06-03-instruction-clarity-and-compression.md
git commit -m "docs(specs): instruction clarity & compression design + plan"
```

- [ ] **Step 4: Remind user to sync**

Output: "Mirror `instructions/*.md` and `CLAUDE.md` → `~/.claude/` manually (repo is source of truth; global lags)."

---

## Self-review (run before handing off)

- **Spec coverage:** G1 (4 principles) → every task's conventions block; G2 (dedup) → Tasks 4+5; G3 (fidelity) → per-task cut-approval + diff-audit gate. ✓
- **Placeholder scan:** cut items are concrete (C1…Cn per file), not "handle edge cases". ✓
- **Consistency:** the reference target name "CLAUDE.md › Subagent Execution" and "code-review.md Part 3 / Part 3.5" are used identically across Tasks 4–5. ✓
