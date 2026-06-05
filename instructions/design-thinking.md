# Design Thinking Rules

## Internal discipline (follow these, output reflects the result):
1. **Restate the problem** — identify the underlying tension, not just the surface symptom. Understanding is demonstrated by restating in your own words, not repeating the user's description.
2. **List the constraints** — team size, timeline, existing system boundaries, operational capability. Ask the user if unclear. Constraints shape the solution space more than requirements do.
3. **Assess reversibility** — classify each key decision as reversible or irreversible. Spend analysis time proportional to reversal cost. Research thoroughly for irreversible decisions (public API contracts, data model, tech stack); decide and move on for reversible ones.
4. **Challenge your own conclusion** — after reaching a design, try to break it. Find the weakest assumption and stress-test it. Skipping this step means pattern-matching, not designing.
5. **Simplicity check** — remove any component whose removal loses no design goal. The right design is the simplest one that satisfies all constraints.

## Output requirements (the user must see these):
6. **Scope boundaries** — state what this design does AND does not cover. Scope is unclear until you can name what you are NOT doing.
7. **Tradeoffs for each key decision** — "I chose A over B because in this context X matters more than Y." Analysis is incomplete until you can name what you give up.
8. **Failure analysis** — for the proposed design: what breaks first, what happens when a component is unavailable, what assumption if wrong would invalidate the design.
