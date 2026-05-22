# Instruction Token Optimization — Design

**Date**: 2026-05-23
**Status**: Approved, pending implementation
**Scope**: Content compression of `instructions/*.md` to reduce always-on context cost

## Problem

The 9 instruction files referenced by `CLAUDE.md` load on every session as always-on context, totaling ~1230 lines per single copy. In the `agent-prompt` repo specifically, the global mirror and the project copy both load (~2460 lines), but that double-load is out of scope for this change — see Non-Goals.

Within those 1230 lines, three classes of redundancy exist:

1. **Pedagogical pile-up** — `code-review.md` contains both per-subsection PR description examples AND a full integrated example that recapitulates them.
2. **Wrong-then-right pairs** — each PR description subsection shows a one-line ❌ wrong example before the ✅ right example. The prose already states the principle; the wrong examples are one-shot emphasis, not durable reference.
3. **Cross-file repetition** — `design-review.md` declares "same rules as Code Review Part 3" then re-lists those rules anyway.

A fourth, smaller case: `implementation.md` shows two ❌ wrong-example code blocks for "comment generously, but only the WHY", where one minimal example would suffice.

## Goals

- **Reduce always-on per-session context cost** by trimming truly redundant examples in three instruction files.
- **Preserve all rule text** — every `must`, `should`, `do not` directive stays. Only worked examples that pile up on the same principle get cut.
- **Maintain sync-ability** — all edits are symmetric (apply identically to global and project copies). The user mirrors `instructions/*.md` from project to `~/.claude/instructions/` manually after this work; this design does not rely on asymmetric file layouts between the two.

**Quantification**: ~63 lines removed per single-copy load (48 in `code-review.md` cut 1 + 5 in cut 2 + 6 in `design-review.md` + 4 in `implementation.md`). In `agent-prompt` repo (double-loaded), ~126 lines per session.

## Non-Goals

- **Restructuring heavy guides into Skill-loaded modules** — explicitly rejected; the user wants rules to remain always-on so Claude applies them without first having to recognize the scenario.
- **Deduplicating the global ↔ project mirror in `agent-prompt`** — would require asymmetric file layout (e.g., project `CLAUDE.md` becoming a stub), which the user vetoed because the project files are meant to be mirror-able to global as-is.
- **Compressing "Review angles" tables, the OrderService [suggestion] example, or git-conventions commit-type table** — these are in the more aggressive "Approach Z" the user did not pick; they carry real risk of losing concrete grounding for edge cases.

## Changes

### `instructions/code-review.md` (≈ −53 lines)

**Cut 1 — integrated PR description example, lines 163–210 (~48 lines)**

The block starting with `*Example of a complete, well-written PR description:*` (line 163) and ending with the closing ` ``` ` of the Verification regression curl (line 209), plus the trailing blank line (210), recapitulates Background / Root Cause / Solution Overview / Changes / Verification examples already given individually in lines 102–161. Delete entirely.

**Cut 2 — five one-line ❌ wrong-examples in PR description subsections (~5 lines)**

Delete:
- line 104  `❌ "Added modelId to modelBasicInfo"`
- line 109  `❌ "modelId was not passed"`
- line 125  `❌ "Added modelId to `modelBasicInfo`"`
- line 143  `❌ Only: "Added modelId to modelBasicInfo"`
- line 151  `❌ "Tested, works fine"`

The ✅ examples and prose directives remain. The contrast pedagogy of "wrong vs right" is one-shot emphasis; the principle is already in the section's prose lead.

**Preserved**:
- All full ✅ examples (with their diagrams and call chains)
- "Level of detail by change type" table
- Part 2 questioning patterns and their examples
- OrderService [suggestion] example
- UserService NPE [blocking] comment example

### `instructions/design-review.md` (≈ −6 lines)

**Cut — "Author's Responsibility After Receiving Comments" section (lines 206–216)**

Currently 11 lines: a "same as Code Review" header, a 4-row label-response list, then two paragraphs on thread resolution that restate code-review.md Part 3.

Replace with the following:

```
## Author's Responsibility After Receiving Comments

Same rules as Code Review Part 3 of `code-review.md` — `[blocking]` / `[question]` /
`[suggestion]` / `[nit]` response rules and thread-resolution rules apply unchanged
to design comments.
```

### `instructions/implementation.md` (≈ −4 lines)

**Cut — second ❌ example in rule #7 "Comment generously" (lines 14–22, the ❌ code block)**

The ❌ block currently shows two trivial wrong comments (`# Increment counter by 1` plus `# Loop through users`). One minimal example suffices to convey "don't restate what code says". Replace the 9-line block with the 5-line minimal version below, keeping only the `# Increment counter by 1` example (net −4 lines):

```
   ❌ Restates what the code says:
   ```python
   # Increment counter by 1
   counter += 1
   ```
```

The longer ✅ block below (3 examples spanning retry / migration ordering / list-vs-set) is preserved — each illustrates a distinct *why-comment* pattern, not the same one repeated.

## Risks

**Type A — cost of choosing**: Cutting the integrated PR description example removes a single place where Claude (or a human reader) sees a complete PR description top-to-bottom. New readers might find the assembly-from-parts harder. Mitigation: subsection examples are themselves complete; the integrated form added no new information.

**Type B — intrinsic fragility**: If a future rule change in code-review.md (e.g., adding a new mandatory section like "Rollback Plan") would naturally extend the integrated example, that example will no longer exist to update. The 5 subsection examples must each be extended instead. Acceptable — they're the canonical source anyway.

## Verification

No automated test exists for "rules still teach correctly". Two manual checks before commit:

1. **Diff audit**: read `git diff` for each file. Confirm only ❌ example lines and the integrated example block are removed. No rule text (sentences containing `must`, `required`, `do not`, `should`, `cannot`) deleted.
2. **Rule-completeness walkthrough**: pick one recent code review task done with the current rules. Mentally re-run the PR submission checklist using only the trimmed `code-review.md`. If any step now lacks grounding, restore the relevant example.

## Implementation Order

1. Edit `instructions/code-review.md` (largest cut, easiest to verify in isolation)
2. Edit `instructions/design-review.md`
3. Edit `instructions/implementation.md`
4. Commit: `docs: trim redundant examples to reduce always-on context size`
5. User syncs project `instructions/*.md` → `~/.claude/instructions/` manually

## Open Questions

None — scope, cuts, and risks all confirmed during brainstorming.
