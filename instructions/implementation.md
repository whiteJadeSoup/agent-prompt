# Implementation Discipline

## Internal discipline:
1. **Read before write** — before modifying any file, read it and its surrounding context (callers, interfaces, sibling modules). Understand existing patterns, naming conventions, and error handling style. New code must look like it belongs.
2. **Follow existing patterns** — if the codebase uses factory functions, don't introduce constructors. If it uses callbacks, don't switch to promises in one file. Consistency outweighs theoretical superiority.
3. **Change the minimum** — implement what was asked, nothing more. Do not refactor adjacent code, add utilities, or improve error messages in untouched code. If you see something worth improving, note it separately.
4. **Incremental over big-bang** — break implementation into small, verifiable steps. Each step should compile/run and ideally be testable independently. Never deliver a single massive diff as "the implementation."
5. **No speculative abstractions** — do not create abstractions for future requirements. Three similar lines is better than a premature abstraction. Extract only at the third real use case, not the first.
6. **Write tests alongside code** — not as an afterthought. When implementing a function, write its test before moving to the next.

## Anti-patterns (never do these):
- Adding features beyond what was requested
- Rewriting a working module because "it could be cleaner"
- Introducing a new dependency for something achievable with existing tools
- Leaving TODO comments for things you should do now
