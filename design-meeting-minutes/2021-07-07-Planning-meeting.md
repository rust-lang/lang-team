---
title: Planning meeting 2021-07-07
tags: planning-meeting
---

# T-lang planning meeting agenda

* Meeting date: 2021-07-07

## Attendance

* Team members: nikomatsakis, Josh, Felix
* Others: simulacrum

## Meeting roles

* Action item scribe: simulacrum
* Note-taker:

## Announcements and other updates

(Anyone can feel free to add things here, tag it with your username.)

- nikomatsakis: Josh and I plan to write up the new "initiative" process that we discussed as a series of edits to our lang-team web page. I am also going to make various proposals around the current set of active initiatives as we go.

## Proposed meetings
-  "Structural equality" [lang-team#94](https://github.com/rust-lang/lang-team/issues/94)
-  "Never allow unwinding from Drop impls" [lang-team#97](https://github.com/rust-lang/lang-team/issues/97)
-  "Lang team process, part 2" [lang-team#104](https://github.com/rust-lang/lang-team/issues/104)

* July 14 -- no meeting (also, Felix has a conflict)
* July 21 -- Lang team process, part 2
* July 28 -- Structural equality (depending on Oli's scheduling constraints; may move to the 27th)
* August 4 -- next planning meeting, Niko on PTO
* August 11 -- Niko on PTO

* Action item: update the issues and announce the meetings

### Notes

* For lang-team#97:
    * For this meeting to be a success, we would need some exploration of what the alternatives are and what kinds of backwards compatibility concerns exist in the wild.
        * Are people relying on the current behavior?
        * Can we estimate how many latent bugs exist *because of* the current behavior?
            * Anecdotally: Rayon for example found it too hard to manage and just aborts.
            * Libs team has found this painful in the std library.
        * Josh to summarize.
* For lang-team#94:
    * oli can prepare the write-up, nikomatsakis is response to ensure it gets done
    * Problem is how to deal with matching on constants, const generic equality, etc in a coherent way
    * Scheduled to July 21
* For lang-team#104:

## Active initiatives

* [Planning board](https://github.com/rust-lang/lang-team/projects/2)

* Umbrella initiatives:
    * 

### "const-evaluation" lang-team#22

**Link:** https://github.com/rust-lang/lang-team/issues/22

* [Comment](https://github.com/rust-lang/lang-team/issues/22#issuecomment-873364101)
* Inline constants [#76001](https://github.com/rust-lang/rust/issues/76001) have not had any substantive progress
    * This is a blocker for const support in inline assembly
    * What does it take to make forward progress here?
    * What work has been done here?
    * Action item: follow up on the status and discuss next steps (nikomatsakis)

### "async foundations" lang-team#33

**Link:** https://github.com/rust-lang/lang-team/issues/33

* [Comment](https://github.com/rust-lang/lang-team/issues/33#issuecomment-875752762)

### "const-generics" lang-team#51

**Link:** https://github.com/rust-lang/lang-team/issues/51

* [Update](https://github.com/rust-lang/lang-team/issues/51#issuecomment-874798421)
* [Vision doc in progress here](https://rust-lang.github.io/project-const-generics/)
* Upcoming design meeting:
    * Structural equality

### "project-safe-transmute" lang-team#21

**Link:** https://github.com/rust-lang/lang-team/issues/21

* [Update](https://github.com/rust-lang/lang-team/issues/21#issuecomment-870514068)

### "Deref patterns" lang-team#88

**Link:** https://github.com/rust-lang/lang-team/issues/88

* [Update](https://github.com/rust-lang/lang-team/issues/88#issuecomment-874359588)
* nrc recently opened a bunch of threads and seems to have an interest

### let-else statements

**Link:** https://github.com/rust-lang/rfcs/pull/3137

* Discussed in triage yesterday

### macro metavariable

**Link:** https://github.com/rust-lang/rust/issues/83527

* [Update](https://github.com/rust-lang/rust/issues/83527#issuecomment-870392547)
* Ping author regarding "vacation period"

### "RFC 2229" lang-team#50

* [Update](https://github.com/rust-lang/lang-team/issues/50#issuecomment-874650087)
* [Closure size, after optimization](https://docs.google.com/spreadsheets/d/1zOOeTFz4AvprEVsiPNFlkt4x_93N7WQe2uv2pX3BzXQ/edit#gid=545238654)
    * [Closure size, before optimization](https://docs.google.com/spreadsheets/d/1U5vaXu490wn8Ae1LB1mSfoP_EZrGRHbUU6ADbeY8bn4/edit#gid=545238654)
* Maybe automate the capturing of data instead, rather than asking people to submit
    * Then we can use crater instead

### "never type stabilization" lang-team#60

**Link:** https://github.com/rust-lang/lang-team/issues/60

* [Update (1/2)](https://github.com/rust-lang/lang-team/issues/60#issuecomment-870126162)
* [Update (2/2)](https://github.com/rust-lang/lang-team/issues/60#issuecomment-874351239)
* Plan to produce some sort of write-up about what we do
    * One question is to what extent we want to document the algorithm
    * Also, where? There isn't much to fit it into
* Action item: Mark to kickoff a conversation about the documentation aspect

### "ffi-unwind" lang-team#19

**Link:** https://github.com/rust-lang/lang-team/issues/19

* [Update](https://github.com/rust-lang/lang-team/issues/19#issuecomment-875772875)

### Tracking Issue for denying trailing semicolons in expression macro bodies #79813 

**Link:** https://github.com/rust-lang/rust/issues/79813

* Update: Ongoing crater run

### "inline-asm" lang-team#20

**Link:** https://github.com/rust-lang/lang-team/issues/20

* [Update](https://github.com/rust-lang/rust/issues/72016#issuecomment-874373791)
* There is a subset that could be stabilized, modulo:
    * clobber support
    * namespacing decision being finalized

### Tracking Issue for #[instruction_set] attribute (RFC 2867) #74727 

**Link:** https://github.com/rust-lang/rust/issues/74727

* Hasn't been changes in a while
* [Update from April 6](https://github.com/rust-lang/rust/issues/74727#issuecomment-814538151) suggests:
    * The implementation works but
    * Generates suboptimal code owing to ungreat inlining
* Seems ready to stabilize *if we wanted to*
    * Minor bikeshed of `#[repr(instruction_set = "...")]` as an alternative spelling
* What would be the plan to get more feedback

### Tracking issue for X.., ..X, and ..=X (#![feature(half_open_range_patterns)]) #67264 

* Has a stabilization PR in FCP