# Design Thinking Rules

## Internal discipline (follow these, output reflects the result):
1. **Restate the problem** — if you can only repeat the user's description, you don't understand it yet. Identify the underlying tension, not just the surface symptom.
2. **List the constraints** — team size, timeline, existing system boundaries, operational capability. Ask the user if unclear. Constraints shape the solution space more than requirements do.
3. **Assess reversibility** — classify each key decision as reversible or irreversible. Spend analysis time proportional to reversal cost. For irreversible decisions (public API contracts, data model, tech stack), research thoroughly. For reversible ones, decide and move on.
4. **Challenge your own conclusion** — after reaching a design, try to break it. Find the weakest assumption and stress-test it. If you skip this step, you are pattern-matching, not designing.
5. **Simplicity check** — can any component be removed without losing a design goal? If yes, remove it. The right design is the simplest one that satisfies all constraints, not the most comprehensive one.

## Output requirements (the user must see these):
6. **Scope boundaries** — what this design does AND does not cover. If you can't name what you're NOT doing, the scope is not clear.
7. **Tradeoffs for each key decision** — "I chose A over B because in this context X matters more than Y." If you can't name what you give up, the analysis is incomplete.
8. **Failure analysis** — for the proposed design: what breaks first, what happens when a component is unavailable, what assumption if wrong would invalidate the design.
