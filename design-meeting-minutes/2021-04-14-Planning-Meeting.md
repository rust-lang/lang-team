# Planning meeting 2021-04-14

## Attending

* Team: Niko, Josh, Felix, Scott, Taylor
* Other: simulacrum, Amanieu, Esteban
* Action item scribe: simulacrum
* Minutes scribe: nikomatsakis
* [Watch the recording](https://youtu.be/-V2zxC8YpnI)

## Design meeting candidates

* Rust language "guiding principles" [lang-team#91](https://github.com/rust-lang/lang-team/issues/91)
    * nikomatsakis has been sketching guiding principles from async vision doc and they seem like decent principles for the language overall
    * not quite ready to schedule this this month
    * pnkfelix has related ["compiler team contributors" guiding principles](https://hackmd.io/TYGRjVIbSBmxpbcfDzll-w)
* Discuss the amount of oversight that lang should have for lints [lang-team#72](https://github.com/rust-lang/lang-team/issues/72)
    * needs a proposal before we decide whether to have a meeting
    * it may be that no meeting is required
* Discuss the possibility of denying bare_trait_objects in 2021 edition [lang-team#65](https://github.com/rust-lang/lang-team/issues/65)
    * action item: Mark to close this
    * PR is https://github.com/rust-lang/rust/pull/83213#issuecomment-812120957
    * Just got its third checkbox; now in FCP merge.
* Ongoing discussion about `Drop` and `Pin<&mut self>` for 2021 edition
    * 2021 planning ruled this out because it didn't meet the "accepted RFC" deadline
    * not scheduled this round, there is ongoing discussion on Zulip
        * should move that discussion somewhere else on Zulip instead of "Edition 2021" stream
* New ABI: "wasm" [lang-team#90](https://github.com/rust-lang/lang-team/issues/90)
    * alexcrichton is working towards wasm interop
    * the wasm ABI being proposed here is the proposal that should work across runtimes
    * lang team question: do we want to have an ABI named wasm and how does it differ from C ABI?
    * may also touch on bigger questions like how wasm could integrate into rust language design
    * doc expectations:
        * Details about the ABI, but only if they raise complex language questions
            * Specific Questions: Why is this not just the C ABI for the WASM targets?
        * Understanding the roadmap and where lang design changes may be needed
* Generators planning [lang-team#92](https://github.com/rust-lang/lang-team/issues/92)
    * Estaban wants to see driven to completion, ideally this year
    * Related to async vision doc effort
    * doc expectations:
        * Sketch and present a plan for how to settle the open questions
        * Strawperson for a plan with details to help gauge where people fall

### Dates

* [ ] April 21: [lang-team#90](https://github.com/rust-lang/lang-team/issues/90)
* [ ] April 28: [lang-team#92](https://github.com/rust-lang/lang-team/issues/92)
* [ ] May 5 <-- next planning meeting
* Doc and agenda available the day before the meeting

## Updates

### Questions to consider for updates

* What are recent topics of discussion?
* Any exciting developments since the last check-in?
* Any points where the group is stuck and the lang-team could weigh in?
* Any potential design meeting topics?
* Any other concerns?

## Updates from active groups and projects

* [Project board](https://github.com/rust-lang/lang-team/projects/2)
* Active projects (design phase)
    * [Const evaluation](https://github.com/rust-lang/lang-team/issues/22)
    * [Async foundation](https://github.com/rust-lang/lang-team/issues/33)
    * [Const generics](https://github.com/rust-lang/lang-team/issues/51)
    * [Deref Patterns](https://github.com/rust-lang/lang-team/issues/88)
    * [Denying semicolons in expression macro bodies](https://github.com/rust-lang/rust/issues/79813)
    * [Safe transmute](https://github.com/rust-lang/lang-team/issues/21)
* Active projects (implementation)
    * [Tracking Issue for RFC 3086: macro metavariable expressions](https://github.com/rust-lang/rust/issues/83527)
    * [RFC 2229](https://github.com/rust-lang/lang-team/issues/50)
    * [Never type stabilization](https://github.com/rust-lang/lang-team/issues/60#issuecomment-814509681)
        * I'm currently using Infallible for `TryV2` (https://github.com/rust-lang/rust/pull/84092/files#diff-61f98b185ed55c4151035e7b82e7ac945377f8bfcb5133f16f9a595838e01a10R1653), but it sure would be nice to use `!`.
    * [FFI Unwind](https://github.com/rust-lang/lang-team/issues/19)
* Active projects (evaluation)
    * [Inline assembly](https://github.com/rust-lang/rust/issues/72016)
    * [Instruction set attribute](https://github.com/rust-lang/rust/issues/74727)
* Active projects (stabilization)
    * [Stabilize pat2015 but leave :pat2021 gated](https://github.com/rust-lang/rust/pull/83386)
    * [Tracking issue for X.., ..X, and ..=X](https://github.com/rust-lang/rust/issues/67264)
