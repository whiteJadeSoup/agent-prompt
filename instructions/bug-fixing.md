# Bug Fixing Protocol

## Internal discipline (how to work):
1. **Reproduce before analyzing** — establish exact input, steps, and environment that trigger the bug. If you cannot reproduce, state what you tried and ask — do not guess the root cause.
2. **Trace the actual execution path** — read the code line by line from input to error. Do not assume you know what it does.
3. **Narrow by evidence, not intuition** — at each step, run a check or read specific code to confirm before narrowing further.
4. **Single root cause** — do not present multiple possibilities as your conclusion. Narrow to one. If you can't, state what info you need to narrow further.
5. **Fix the cause, not the symptom** — if your fix is adding defensive code (null checks, try-catch, default values), pause. You may be masking the symptom. Defensive code is acceptable only at system boundaries.
6. **Never suggest a fix before confirming root cause.**

## Output requirements (what the user sees):

Present the result in this structure:

7. **Root cause statement**: "[location] does [X], causing [Y], because [Z]." One sentence, precise.
8. **Evidence**: the specific code, log output, or test result that confirms the root cause. Not your reasoning chain — the proof.
9. **Fix and rationale**: what you changed and why at this layer (not another).
10. **Verification**: reproduction scenario re-run showing it passes, plus regression checks on related paths.

## Language rules:
- Never write "the issue is likely/probably X" — either confirmed or explicitly state "not yet confirmed, need to check Y."
- Never present uncertainty as conclusion.
