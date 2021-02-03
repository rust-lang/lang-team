# Lang Team Planning Meeting

* Meeting date: 2021-02-03
* Attending: 
    * Team: nikomatsakis, felix, josh, scott
    * Not team: simulacrum, connor horman
* [Watch the recording](https://youtu.be/r1c2tFMAYQs)

## Repeat the plan

* First meeting of every month is a planning meeting
* Begin with updates from active projects
* Schedule meetings -- issues should be filed by the meeting start

## Project updates and discussion

[Project board](https://github.com/rust-lang/lang-team/projects/2)

### async foundations

* **Liaison:** nikomatsakis
* continued work on polish
* async functions in traits ([plan](https://hackmd.io/5kCE2T6sTDijhqMx8kaikw))
    * named impl trait (`type Foo = impl Bar`)
    * generic associated types
    * and so forth
* stream trait and other traits
* [async vision document](https://hackmd.io/p6cmRZ9ZRQ-F1tlhGaN9rg) -- a work in progress
    * We should have more documents "bigger than an RFC and smaller than a 'complete' lang roadmap"

### const evaluation

* **Liaison:** nikomatsakis
* no update for this time, will sync later

### const generics

* **Liaison:** nikomatsakis
* min const generics -- in nightly right now!
* continued activity -- implementation phase for the most part

### denying trailing semicolon in expr macro bodies

* **Liaison:** pnkfelix
* Lint was implemented in [#79819](https://github.com/rust-lang/rust/pull/79819) and merged, but is allow by default
* It's a future-compatibility lint but is allow-by-default
* Next step:
    * crater run

### safe transmute

* **Liaison:** joshtriplett
* Back and forth about how to find a more minimal approach
* High-level review from lang team needed
* Next step: follow-up and make sure this picture is accurate, consider filing a design meeting proposal

### pub macro rules

* **Liaison:** rylev
* Status:
    * Niko to create tracking issue
    * [RFC #3067] opened
    * Sync with petrochenkov needed
    * Action: Niko to review 

[RFC #3067]: https://github.com/rust-lang/rfcs/pull/3067

### declarative macro repetition counts

* **Liaison:** Lokathor
* Status:
    * Inactive, let's sync up with the shepherd
    * Action item: Josh to ping person who pitched it and see what their story is

### RFC 2229

* Adopted a new migration --
    * instead of `let x = x;`, we now do `drop(&x)`
        * be careful of clippy lints!
    * `capture!(x);` macro is probably a good idea
        * lots of options like `capture!(move x);` or `capture!(&x);` or similar too...

### ffi-unwind

* **Liaison:** nikomatsakis
* Implementation PR is making progress
* Recent [blog post](https://blog.rust-lang.org/inside-rust/2021/01/26/ffi-unwind-longjmp.html) about doing longjmp work
    * Propose extended charter with a description of the general approach.

### never type stabilization

* **Liaison:** nikomatsakis
* Stalled at present

### `#[instruction_set]` attribute

* Unknown

### inline assembly

* Have gotten positive user reports, no critical issues raised
* Do we want to stabilize the mechanism all at once, or do we want to stabilize on an architecture-by-architecture basis?
    * e.g., stabilize x86/ARM, feature gate RISC-V, etc
    * Meeting consensus: start with architecture-by-architecture
* AI: Josh to reach out to Amanieu and potentially prepare stabilize report

### raw ref macros

* Stable!

## Upcoming meeting plan

### [Meeting proposals](https://github.com/rust-lang/lang-team/issues?q=is%3Aissue+is%3Aopen+label%3Ameeting-proposal):

* Improving trust in the Rust compiler [#79](https://github.com/rust-lang/lang-team/issues/79)
    * Niko to talk to Florian about agenda
* Discuss the amount of oversight that lang should have for lints [#72](https://github.com/rust-lang/lang-team/issues/72)
    * Deferred, needs a strawman proposal
* Discuss the possibility of denying `bare_trait_objects` in 2021 edition [#65](https://github.com/rust-lang/lang-team/issues/65)
    * Generalized
* Growing the team [#81](https://github.com/rust-lang/lang-team/issues/81)
    * Closed meeting, lang team members only

### Proposed meeting

* 2021 edition lints overview (2020-02-24)
    * Author: scottmcm
    * For each idiom lint:
        * summarize current state (warn by default? etc)
        * summarize concerns that have been raised
        * proposed action in 2021 Edition
            * (default: Deny by default)
        * does the warning have automatic migrations
        * crater results, if applicable
    * Decide whether to move forward and/or changes to make

### Calendar

* 2021-02-10: Growing the team [#81](https://github.com/rust-lang/lang-team/issues/81)
* 2021-02-17: Improving trust in the Rust compiler [#79](https://github.com/rust-lang/lang-team/issues/79)
* 2021-02-24: 2021 edition lints overview

### Needs minutes

*  Discuss RFC 3058, try_trait_v2 [#75](https://github.com/rust-lang/lang-team/issues/75)

## Policy

* May want to break out into a distinct meeting
* Blocking objection policy
    * Clarify that concerns/objections can block a proposal
    * Require that, after a fixed period of time (e.g. 10 days), sustaining a concern requires a "second" from another team member
    * A second does not imply agreement with the concern, only that the seconder thinks the concern should be heard out and explored further
        * In fact, a second by someone who disagrees is perfectly reasonable ("I don't agree with the objection, but I think you should explore it further and say more about it, and we should give it due consideration")
    * The person filing the concern has appropriate time to convince people. If, after that time, they cannot convince even one team member that their objection has enough merit to warrant further exploration or explanation, then 
* Notes:
    * Making it clear the mechanics of raising a concern
    * Maybe it's useful to have +0/-0 so we can sense if anyone really *wants* something
    * Maybe you should be able to raise concerns at any time, and edit it into the top comment
    * Want some way to enter dissenting opinions
    * Don't want to require seconds *immediately*, more of something that is done after discussion