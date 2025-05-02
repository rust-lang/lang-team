# Decision making

This page documents our decision making process.

## Our goal

We want the ability to make designs that feel fresh, bold, and innotative. We do not want Rust to feel like it has been "designed by committee", with all the interesting bits smoothed down.

We also want designs that meet Rust user's needs and live up to Rust's ethos of reliabile, performant, accessible code.

These two goals can be in tension. The former pushes us to empower individuals. The latter pushes us to validate designs broadly. We use this decision making process to guide us in balancing those tensions.

## Design axioms

Our decision making process axioms are rules that we follow to help us achieve our goal. We try to satisfy them all, but if they come into tension, we prefer items that appear first in the list.

* **No new rationale**. We make decisions only after the rationale have been presented publicly and all relevant stakeholders have had a chance to present counterarguments. In particular, we never make [1-way door decisions](./consensus.md) in sync meetings. Instead, we present meeting consensus and leverage rfcbot.
- **Not afraid to do the right thing**. At the end of the day, we have to do what we feel is *right*. Sometimes this means breaking with the tradition and precedent set by other languages. Sometimes it means taking a socially uncomfortable stance (but always with respect).
- **Find common ground**. It's good to break up designs into small pieces and proceed step by step. But be sure that each piece solves an end-to-end problem on its own.
- **Trust each other**. Lang team members are expected to have demonstrated sharp instincts, humility, and the ability to hear and understand others. Sometimes you have to put your doubts aside and trust the others on the team. 
- **Treasure dissent**. When someone raises a concern, we take it as an opportunity to improve the design, not an obstacle to be overcome. We invite people to elaborate and make sure we understand what's motivating them before we decide how to respond.

## Processes

We divide decisions into two categories:

* [Reversible decisions (aka, 2-way door)](./2-way-door.md), used for [starting experiments](./how_to/experiment.md) and other decisions that don't make a promise to our end users. Reversible decisions follow a "champion" rule, which means that just one person on the team has to be in favor for the decision to go through; others on the team present concerns as a way to help make sure the champion is aware of them.
* [Consensus decisions (aka, 1-way door)](./consensus.md), used for [stabilization decisions](./how_to/stabilize.md) and for RFC approvals. Consensus decisions require everyone on the team to sign off. The decision cannot be finalized if there is a blocking concern backed by 2 or more team members.
