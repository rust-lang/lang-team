# Decision making

This page documents our decision making process.

## Our goal

We want the ability to make designs that feel fresh, bold, and innovative. We do not want Rust to feel like it has been "designed by committee", with all the interesting bits smoothed down.

We also want designs that meet Rust users' needs and live up to Rust's ethos of reliable, performant, accessible code.

These two goals can be in tension. The former pushes us to empower individuals. The latter pushes us to validate designs broadly. We use this decision making process to guide us in balancing those tensions.

## Design axioms

Our decision making axioms are rules that we follow to help us achieve our goal. We try to satisfy them all, but if they come into tension, we prefer items that appear first in the list.

* **No new rationale**. We make decisions only after the rationale has been presented publicly and all relevant stakeholders have had a chance to present counterarguments.
* **Not afraid to do the right thing**. At the end of the day, we have to do what we feel is *right*. Sometimes this means breaking with tradition and precedent set by other languages. Sometimes it means taking a socially uncomfortable stance (but always with respect).
* **Find common ground**. When there is disagreement, look for solutions that address everyone's concerns. Break up designs into smaller pieces if needed. But be sure that each piece solves an end-to-end problem on its own.
* **Trust each other**. Lang team members are expected to have demonstrated sharp instincts, humility, and the ability to hear and understand others. Sometimes you have to put your doubts aside and trust the others on the team.
* **Treasure dissent**. When someone raises a concern, we take it as an opportunity to improve the design, not an obstacle to be overcome. We invite people to elaborate and make sure we understand what's motivating them before we decide how to respond.

## Two kinds of decisions

We divide decisions into two categories:

* **[Champion decisions](./champion-decisions.md)** are the preferred default. They are used for [starting experiments](./how_to/experiment.md) and other decisions where we want to move quickly. A single lang-team member or advisor can champion a decision; others can raise concerns, but cannot block. A champion decision does *not* represent team consensus—it represents only the champion's point of view. Any team member can request FCP escalation at any point—during the initial triage meeting or later—which converts it to an FCP decision. Once a champion decision has been escalated and FCP'd, it becomes durable in the same way as any other FCP decision.

* **[FCP decisions](./fcp-decisions.md)** represent a significant commitment from the team. They are used for [stabilization](./how_to/stabilize.md), RFC approval, and other cases where we are making a promise to our users or taking a position we don't want to reverse lightly. FCP decisions require broader team sign-off and follow a formal process for resolving concerns.

**When to use which:** Prefer champion decisions when possible—they have lower overhead and enable faster iteration. Use FCP decisions when the decision is significant enough that reversing it should require another FCP. The expectation is that an FCP'd decision cannot be reversed without some change in circumstances: new experience, new information, or a reasoned change of mind.

## Team size

The lang team operates as a "two-pizza team" of 4-8 members. This keeps the team small enough for high-bandwidth communication and trust, while large enough for diverse perspectives.
