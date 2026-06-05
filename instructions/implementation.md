# Implementation Discipline

## Internal discipline:
1. **Read before write** — before modifying any file, read it and its surrounding context (callers, interfaces, sibling modules). Understand existing patterns, naming conventions, and error handling style. New code must look like it belongs.
2. **Follow existing patterns** — if the codebase uses factory functions, use factory functions; if it uses callbacks, use callbacks in that file. Consistency outweighs theoretical superiority. Reuse existing tools and dependencies rather than introducing new ones for something already achievable.
3. **Change the minimum** — implement exactly what was asked: no more features, no refactoring adjacent code, no new utilities, no improved error messages in untouched code. If you see something worth improving, note it separately. Finish every task now rather than leaving a TODO.
4. **Incremental over big-bang** — break implementation into small, verifiable steps. Each step should compile/run and ideally be testable independently.
5. **No speculative abstractions** — create abstractions only at the third real use case, not the first. Three similar lines is better than a premature abstraction.
6. **Write tests alongside code** — not as an afterthought. When implementing a function, write its test before moving to the next.
7. **Comment generously, but only the WHY** — code expresses *what* it does and *how* through names, structure, and control flow. Comments capture what the code *cannot* say: the reason a decision was made, the constraint that forced an unusual shape, the invariant that must hold, the bug that was worked around, the alternative that was rejected. Write the comment when a reader would ask "why is it written this way?"

<counterexample>
```python
# Increment counter by 1
counter += 1
```
Restates what the code already says — will drift out of sync and mislead the next reader.
</counterexample>

<example>
```python
# Retry up to 3 times — downstream service has a known transient failure
# during its 02:00 UTC index rebuild (see incident #4821).
for attempt in range(3):
    ...

# Must run before cache warmup: warmup reads from this table and will
# populate stale entries if the migration hasn't completed.
run_migration()

# Using a list instead of a set here: order matters for audit log replay,
# and the collection is bounded at ~50 items so O(n) lookup is fine.
processed_ids: list[str] = []
```
Each comment states why — a constraint, ordering dependency, or deliberate trade-off the code itself cannot express.
</example>
