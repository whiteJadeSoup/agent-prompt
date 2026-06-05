# Refactoring Protocol

## Internal discipline:
1. **Separate refactoring from behavioral changes** — a refactoring commit changes structure only; submit any needed behavioral change (bug fix, feature) as a separate commit. If a bug fix needs refactoring first, submit the refactor first: "Preparation for fix in next PR."
2. **Test coverage before refactoring** — if the target code has no tests, write tests for current behavior first. Refactor. Verify tests pass. Without this, you cannot prove behavior was preserved.
3. **One structural change at a time** — rename, then extract, then move. Each step must be independently verifiable.
4. **Run tests after every step** — not just at the end. If tests break mid-refactoring, you know exactly which step caused it.
5. **Scope control** — define upfront what you are refactoring and where you stop. Resist scope expansion.

## Output requirements:
6. **Structural changes** — what changed: renamed X to Y, extracted Z, consolidated A and B into C.
7. **Behavioral preservation** — what did NOT change: all existing tests pass, no public API changes, no behavior differences.
8. **Test evidence** — before/after test results proving preservation.
