# Implementation Discipline

## Internal discipline:
1. **Read before write** — before modifying any file, read it and its surrounding context (callers, interfaces, sibling modules). Understand existing patterns, naming conventions, and error handling style. New code must look like it belongs.
2. **Follow existing patterns** — if the codebase uses factory functions, don't introduce constructors. If it uses callbacks, don't switch to promises in one file. Consistency outweighs theoretical superiority.
3. **Change the minimum** — implement what was asked, nothing more. Do not refactor adjacent code, add utilities, or improve error messages in untouched code. If you see something worth improving, note it separately.
4. **Incremental over big-bang** — break implementation into small, verifiable steps. Each step should compile/run and ideally be testable independently. Never deliver a single massive diff as "the implementation."
5. **No speculative abstractions** — do not create abstractions for future requirements. Three similar lines is better than a premature abstraction. Extract only at the third real use case, not the first.
6. **Write tests alongside code** — not as an afterthought. When implementing a function, write its test before moving to the next.
7. **Comment generously, but only the WHY** — code expresses *what* it does and *how* it does it through names, structure, and control flow. Comments exist to capture what the code *cannot* say: the reason a decision was made, the constraint that forced an unusual shape, the invariant that must hold, the bug that was worked around, the alternative that was rejected. If a comment restates what the next line does, delete it — it will drift out of sync with the code and mislead the next reader. If a reader would ask "why is it written this way?", write that comment.

   *Examples:*

   ❌ Restates what the code says:
   ```python
   # Increment counter by 1
   counter += 1

   # Loop through users
   for user in users:
       ...
   ```

   ✅ Captures why, which code cannot express:
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

## Anti-patterns (never do these):
- Adding features beyond what was requested
- Rewriting a working module because "it could be cleaner"
- Introducing a new dependency for something achievable with existing tools
- Leaving TODO comments for things you should do now
