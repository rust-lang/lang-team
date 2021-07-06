---
tags: design-meeting
---

# 2021-05-19: May status updates


## Attending

* Team: nikomatsakis, cramertj, joshtriplett, scottmcm
* Other: simulacrum
* Action item scribe: simulacrum
* Minutes scribe: nikomatsakis

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
    * [FFI Unwind](https://github.com/rust-lang/lang-team/issues/19)
* Active projects (evaluation)
    * [Inline assembly](https://github.com/rust-lang/rust/issues/72016)
    * [Instruction set attribute](https://github.com/rust-lang/rust/issues/74727)
* Active projects (stabilization)
    * [Stabilize pat2015 but leave :pat2021 gated](https://github.com/rust-lang/rust/pull/83386)
    * [Tracking issue for X.., ..X, and ..=X](https://github.com/rust-lang/rust/issues/67264)

## Update from docs team

I would like help with reviews if anyone can find the time.

https://github.com/rust-lang/reference/pull/1010 — Revert "Temporarily remove pat_param."
https://github.com/rust-lang/reference/pull/1011 — Rearrange HRTB grammar.
https://github.com/rust-lang/reference/pull/1016 — Add crate and module to glossary.
https://github.com/rust-lang/reference/pull/1022 — Expand on Unicode identifiers.
https://github.com/rust-lang/reference/pull/1026 — Fix type_length_limit example.

Some of these should be very easy.

## Notes and minutes

### const evaluation

What is a val tree?

* Tree whose leaves are integers
* Represent different Rust aggregate types:
    * `(1, 2, 3)` becomes
        * ()
            * 1
            * 2
            * 3
* Val trees may be important for "structural equality"
    * what does it mean when you match vs have const generics etc
* Could be a nice inside rust blog post
    * Action item: Niko to suggest authoring an Inside Rust blog post
* Changing arguments to intrinsics to be required to be constants
    * You sometimes need inline constants
    * but constant literals, named constants work fine

### async foundations

* Vision doc has ended its brainstorming period
* Trying to build up a kind of skill tree of ideas
* Had generator meeting

### const generics

* Working towards a document on plans in this area
* Documenting structural equality vs. partial equality etc.
    * Ralf is working on a proposal here
* const parameter defaults stabilization
    * some concern around moving ahead without some discussion or RFC
    * Taylor notes that type param defaults aren't quite what is desired
    * But types and constants working differently may be very surprising
    * Consts may be 'interesting' where types are not, but we don't know if this is true
    * syntactic vs. inference based defaults
    * C++ uses types(?) vs. value generics for some stuff, but [I am not following]
    * May want to request an RFC
        * may be short
        * identify use cases
    * ordering of types and consts playing into defaults
    * Action item: want an RFC, at least for guide-level section
        * cover at least:
            * ordering (types vs consts)
            * examples of uses
            * ...
        * scottmcm: I'd love to see, for example, a description of how a `SmallVec<T, N = .....>` would be implemented, what kinds of things can go in the default, when that default is used vs inferred, etc.  (All the kinds of problems people hit with `HashMap`'s `S` and how that's relates here.)
* Default [T; 0] 
    * Currently implemented for zero-length arrays without a T: Default bound
    * Would like default with T: Default for all N lengths
    * proposed hack to avoid extending specialization, via a lang item
    * currently only Default is being discussed because other traits were not implemented specially for N=0
    * Hack could be applied elsewhere if needed, but sort of iffy
    * There's likely some unblocking of usage, but we don't know
    * Is there a super compelling motivational use case like serde?
* Issue for doing it for all the other ones: Most core trait impls for empty arrays are over-constrained https://github.com/rust-lang/rust/issues/52246


# denying trailing semicolons

* Now denying lint in std/compiler, crater run in progress

# safe transmute

* No update this month

# deref patterns

* Possibly stuck around syntax of the pattern
* The list of stdlib types last brought for consideration was 
    - `Box`
    - `Arc`
    - `Rc`
    - `Vec`
    - `String` -- the only one we might caveat, because of the latent desire to have zero-capacity static strings, but also high motivation
        - For `DerefMut` I think it would be observable, even if it's lazy.
    - `Pin<P>`
* cramertj: I'd like to be able to match on string literals
* josh: matching against literals feels like you don't want to have to write `box`, but creating variables makes sense
    * (it's ambiguous, in some sense)
* vec is also very appealing for the same reason

Example:

```rust=
let v: Vec<u32> = vec![1, 2, 3];
let w: Box<[u32]> = v.clone().into_boxed_slice();

match (v, w) {
  ([..], [..]) => { }
}

let s: String = "xyz".to_string();

match s {
    "string literal" => todo!("this should just work"),
    my_str => todo!("this should require syntax if you want my_str to be &str rather than String"),
}

let b = Box::new(...);
match b {
    box inside => { } // need to clarify the distinction here
}

match b: Box<String> {
    a @ "string_literal" => /* do you want the String or the Box<String>? (or the &str?) */
}

match b: &&Foo {
    a @ Foo { .. } => /* a: &&Foo? */
}
```

* Consensus that the above examples should work
* Suggestion: Move ahead with non-explicit syntax for the time being, tackle that question later
* Binding mode examples

```rust=
let b = Box::new(SomeStruct { ... });

match b { 
  SomeStruct { x, y } => { /* moves */ }
}

match b { 
  SomeStruct { ref x, ref y } => { /* borrows */ }
}

match &b { 
  SomeStruct { x, y } => { /* borrows */ }
}
```
Josh: Would prefer syntax for this (as opposed to the above examples with literal values)
Scott: existing binding modes feature is clear precedent (in the sense of "something done or said that may serve as an example or rule to authorize or justify a subsequent act of the same or an analogous kind") for this, though there are certainly those not fond of that feature.  Perhaps the same things being pondered there (like we've discussed `SomeStruct { x: &Foo }` in the past) could help here too.
Josh: For reference, binding modes was not originally presented as a generalized concept that should be extended in the future, so much as a specific proposal. The premise seemed to be "it shouldn't matter if you have a reference"; that doesn't necessarily support the idea that "it shouldn't matter that you have a Vec/Box/Rc/Arc".
(further discussion deferred)

* Action item: Mark to write up a doc or something and we will discuss it

## RFC 2229

* Improved diagnostics
* Migrations detect changes in trait impls
* working on insignificant destructors (e.g., those deallocating memory)
    * Mark: Maybe also Arc/Rc?
* will work on good statistics, but estimate is ~80% of closures don't change size
    * in principle didn't have to change size at all
    * but that would be harder: this implementation didn't change MIR or borrowck really
    * some digging into actual cases, some investigation suggests that simple optimizations like capturing the whole thing would be easy enough
* Any analysis on async block sizes?
    * Taylor would like to get tests and some size estimates

## Never type

* Still experimenting with algorithms that try to avoid breakage
* It's really painful, no clear solutions yet

## ffi-unwind

* C-unwind issues remain top priority
* longjmp etc have not received much attention

## inline asm

* no official update this month


## instruction set attribute

* work still ongoing
* no clear next steps

## stabilized without documentation

* extended key value attributes didn't have much here
    * suggestion: file an issue on the reference tracking this
* stabilization report template with a "known person" who will document it
* Ember has an additional stage after stabilization, "recommended", where you stabilize, and then documentation etc happens
    * Possible problem: stabilized but not great in practice, this is not great
    * Ember uses editions here
* Stage for "stabilized but not satisfied with docs.


## typeid bug experiments

* No actual update, may initiate a project here