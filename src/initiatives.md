# Initiatives

A lang team **initiative** is some active effort with a clear goal or deliverable.
Typically initiatives are changes to the language, but they could also be documentation, specifications, or something internal to the lang team.

## Active initiatives

The following is a list of **active lang-team initiatives**. Each initiative has a link to a repository
or tracking issue that goes into details on its current state, along with an [owner] and a [liaison].

Key:

* 🛑 -- Blocked (link to an issue)
* ⏳ -- Paused
* 🔬 -- Under active research
* 💻 -- Under active development
* 🚀 -- Feature complete and seeking feedback
* ✅ -- Stable

| Initiative                    |               | [Stage] | [Owner]         | [Liaison]    |
| ----------------------------- | ------------- | ------- | --------------- | ------------ |
| [Impl Trait]                  |               |         | nikomatsakis    | ?            |
| ↳ Type Alias Impl Trait       | 💻           | [▰▰▰▱▱] | oli-obk         | nikomatsakis |
| [Generic Associated Types]    | 💻           | [▰▰▰▱▱] | jackh726        | nikomatsakis |
| [let-else][#87335]            | 💻           | [▰▰▰▱▱] | fishrock        | JoshTriplett |
| [Async Foundations]           |               |         | tmandry         | nikomatsakis |
| ↳ [Async fundamentals]        | 🔬           | [▱▱▱▱▱] | tmandry         | nikomatsakis |
| ↳ [Generators]                | 🔬           | [▰▰▱▱▱] | estebank        | pnkfelix |
| Never type                    | 🔬           | [▰▰▱▱▱] | mark-simulacrum | nikomatsakis |
| [Inline assembly]             |               |         | Amanieu         | JoshTriplett |
| ↳ Core feature                | 🚀           | [▰▰▰▰▱] | Amanieu         | JoshTriplett |
| ↳ Const support               | [🛑][#76001] | [▰▰▱▱▱] |                 |              |
| [Dyn upcasting]               | 💻           | [▰▰▰▱▱] | crlf0710        | nikomatsakis |
| [Negative impls in coherence] | 🔬           | [▰▰▱▱▱] | nikomatsakis    | pnkfelix     |
| [FFI Unwind]                  |               |         | BatmanAod       | nikomatsakis |
| ↳ extern "C-unwind"           | 🚀           | [▰▰▰▰▱] | BatmanAod       | nikomatsakis |
| ↳ longjmp                     | ⏳             | [▰▰▱▱▱] | BatmanAod       | nikomatsakis |
| [Disjoint closure capture]    | ✅            | [▰▰▰▰▰] | nikomatsakis    | TBD          |
| Try and generalized `?`       |               |         |                 |              |
| ↳ `?` operator                | ✅            | [▰▰▰▰▰] |                 |              |
| ↳ [`Try` trait][#42327]       | 🚀           | [▰▰▰▰▱] | scottmcm        | ?            |
| ↳ `try` blocks                | ⏳             | [▰▰▰▱▱] | scottmcm        |              |

[▱▱▱▱▱]: ./initiatives/process/stages/proposal.md
[▰▰▱▱▱]: ./initiatives/process/stages/experimental.md
[▰▰▰▱▱]: ./initiatives/process/stages/development.md
[▰▰▰▰▱]: ./initiatives/process/stages/feature_complete.md
[▰▰▰▰▰]: ./initiatives/process/stages/stabilized.md
[Stage]: ./initiaives/process/stages.md
[Owner]: ./initiaives/roles/owner.md
[Liaison]: ./initiaives/roles/liaison.md

[#42327]: https://github.com/rust-lang/rust/issues/42327
[#76001]: https://github.com/rust-lang/rust/issues/76001
[Async Foundations]: https://rust-lang.github.io/wg-async-foundations/
[Async Fundamentals]: https://rust-lang.github.io/async-fundamentals-initiative/
[#72016]: https://github.com/rust-lang/rust/issues/72016
[#87335]: https://github.com/rust-lang/rust/issues/87335
[#34511]: https://github.com/rust-lang/rust/issues/34511
[Impl Trait]: https://github.com/rust-lang/impl-trait-initiative
[Generic Associated Types]: https://github.com/nikomatsakis/generic-associated-types-initiative/
[FFI Unwind]: https://github.com/rust-lang/project-ffi-unwind/
[Inline assembly]: https://github.com/rust-lang/project-inline-asm
[Dyn upcasting]: https://github.com/rust-lang/dyn-upcasting-coercion-initiative
[Negative impls in coherence]: https://rust-lang.github.io/negative-impls-initiative/
[Generators]: https://github.com/rust-lang/lang-team/issues/137

Note that this list doesn't represent the complete set of unstable features. We are currently in the process of transitioning into
the initiative system, so there are a number of RFCs that have been accepted (and even implemented!) which don't
have an active initiative.

## Proposed initiatives

You can see the [currently proposed initiatives] on Github.

[currently proposed initiatives]: https://github.com/rust-lang/lang-team/issues?q=is%3Aissue+is%3Aopen+label%3Amajor-change

## How does one propose a new initiative?

Read more in the [process](./initiatives/process.md) page!
