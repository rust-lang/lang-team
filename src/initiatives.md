# Initiatives

A lang team **initiative** is some active effort with a clear goal or deliverable.
Typically initiatives are changes to the language, but they could also be documentation, specifications, or something internal to the lang team.

## [Active initiatives][pb]

Active initiatives are initiatives that have been assigned a lang-team [liaison] and which are actively underway. A complete list can be found in [this GitHub project board][pb]. Note that this list doesn't represent all unstable features; older features in particular were added without active initiatives.

Each initiative on the project board is linked to a tracking issue and has a status:

* The [owner] and [liaison] are assigned to the issue.
* If the initiative has a dedicated repository, the issue is created on that repository (some initiatives don't require their own repos; they are found on rust-lang/rust or rust-lang/lang-team).
* The [stage] of the initiative:
    * [Experimental] -- Drafting RFC; implementation work may begin on nightly as well
    * [Development] -- Approved RFC; implementation is in progress on nightly
    * [Feature complete] -- Implementation is complete on nightly and ready for widespread testing
    * [Stabilized] -- Implementation is complete and available on stable
        * To be stabilized, there must be a pending PR adding the feature to the Rust reference, but this PR may not yet have landed.
        * Other forms of integration, such as rustfmt, often take place after stabilization as well.

## Proposed initiatives

You can see the [currently proposed initiatives] on Github. We review this list during [triage meetings](./meetings.md) and decide whether to assign a liaison or close the proposal. If initiatives haven't received a liaison after 6 weeks of activity, we take that as a sign that there is no enthusiasm to pursue this and close the issue, but you are welcome to re-open the issue if you believe someone would be willing to liaison.

[currently proposed initiatives]: https://github.com/rust-lang/lang-team/issues?q=is%3Aissue+is%3Aopen+label%3Amajor-change

## How does one propose a new initiative?

It's easy! You just open a short issue describing your idea. Read more in the [process](./initiatives/process.md) page!

[pb]: https://github.com/orgs/rust-lang/projects/16/
[proposal]: ./initiatives/process/stages/proposal.md
[experimental]: ./initiatives/process/stages/experimental.md
[development]: ./initiatives/process/stages/development.md
[feature complete]: ./initiatives/process/stages/feature_complete.md
[stabilized]: ./initiatives/process/stages/stabilized.md
[Stage]: ./initiaives/process/stages.md
[Owner]: ./initiaives/roles/owner.md
[Liaison]: ./initiaives/roles/liaison.md
