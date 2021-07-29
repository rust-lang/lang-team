This section documents the work-in-progress Rust language team decision
process. This process, and the `rustbot` tooling to support it, does not yet
have a finished implementation. This document serves to explain the intended
process, for the purposes of ongoing implementation.

## Prioritized principles of Rust team consensus decision-making

These are in order of priority. They're intended to be general enough that they
could apply to any Rust governance team, not just the language team.

- **Treasure dissent.** When someone raises a concern, that's a chance to
  improve the design, and to discover and explore underlying values. Dissent
  should be an amicable, cooperative process.
- **Understand and cooperatively resolve concerns.** We cannot resolve a
  concern without first understanding it, including the underlying values
  motivating it. We should demonstrate that understanding by documenting the
  concern. We should consider the tradeoffs and the impacts on users, through
  the Rust design principles. We should seek out and favor satisfying solutions
  (those that satisfy everyone's values) over
  [satisficing](https://en.wikipedia.org/wiki/Satisficing) solutions (those
  that are just good enough for people to accept them as a compromise among
  conflicting values, without actually being happy with the outcome).
- **Don't force an irreversible decision.** We should make as many decisions
  reversible as possible. Even when a decision seems irreversible (e.g.,
  stabilization), we should strive to find a subset that still allows
  addressing the use case, or a subset that will support evaluation and
  data-gathering to enable making the decision in the future. If that's not
  possible, consider the null alternative; not making a change should always be
  the easier path, and the burden of proof to override a concern on an
  irreversible decision should be high.
- **Value expertise.** When cooperatively resolving a concern, or when
  considering overriding a concern, carefully weigh the advice and
  recommendations of experts. This includes team advisors, domain experts, and
  the owners or members of relevant initiatives.
- **Recording reasoning helps ensure good, consistent decisions over time.**
  Even if we decide not to sustain an objection, we should always record the
  objection and the reasons for our decision as a "dissent", as well as any
  unresolved questions for evaluation later in the process. The team member who
  raised the objection has the perogative to author that dissent and frame the
  unresolved questions (within reason).
- **Consensus doesn't mean unanimity.** Consensus means everyone is heard and
  understood, and all concerns are addressed (even those not treated as
  blocking), and the team finds the outcome reasonable. Consensus does not mean
  everyone agrees completely.

## Consensus decision-making process

First, see [some examples of the decision-making process in
action](./decision_process/examples.md). Then, read the [decision process
reference](./decision_process/reference.md) for the full process and the
`rustbot` tooling to support it.
