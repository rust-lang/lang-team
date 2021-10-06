---
title: "Planning meeting 2021-10-06"
tags: planning-meeting
---

# T-lang planning meeting agenda

* Meeting date: 2021-10-06

## Attendance

* Team members: pnkfelix
* Others: simulacrum, Mara, Esteban

## Meeting roles

* Action item scribe: simulacrum
* Note-taker: pnkfelix

## Proposed meetings
-  "Never allow unwinding from Drop impls" [lang-team#97](https://github.com/rust-lang/lang-team/issues/97) 
-  "Dyn upcasting, safety considerations" [lang-team#119](https://github.com/rust-lang/lang-team/issues/119) 
-  "where the where: GAT syntax" [lang-team#120](https://github.com/rust-lang/lang-team/issues/120) 

## Initiative updates

### Async fundamentals update

[Async fundamentals update](https://rust-lang.github.io/async-fundamentals-initiative/updates/2021-oct.html)

* MVP is first half of subbullet beneath the first bullet.
* Mark: How do you see your goals for stakeholders? 
* Niko: These are production users that we want to make sure we do not overlook. So we ask for a two-way relationship, where we talk them and they talk to us.
* Niko: its not that these are the only voices we care about. But we are asking for a commitment.
* Niko: Tyler is sending out mail to certian people.
* Mark: This feels like a new thing. If people find out via random doc, saying "this was developed via chat with random people we chose", could lead to "why wasn't i consulted" effect.
* Niko: Yeah its possible. Not really concerned since there is so much happening in the open
* Niko: But it would be good to have a post that describes the plan.
* Niko: Goal is to get good sample of input. Also, the RFC process hasn't generated a lot of feedback historically, and they come rather late. The goal in other words is to include *more* voices, not less.
* Mark: A blog post or some similar material, a T-lang page, could be good. "Here's why"; i.e. an RFC for why we're using a stakeholder process.
* Niko: Yeah. I want to write a lang-team update for Inside Rust blog; this would be good for that.
* Niko: also, there will be a traditional RFC.
* Niko: We're in the process of authoring the MVP RFC. At least in theory.


### Impl trait initiative update

[Impl trait initiative update](https://rust-lang.github.io/impl-trait-initiative/updates/2021-oct.html)

* Niko has a last minute update here from earlier today :) 

> In a module-level type alias: the defining scope is the enclosing module.

* Q: Is this module "and children"?
* Niko: honestly I forget
* Niko: two implicit questions: Does it include child modules. 2. Does it include modules in function bodies, and all that funky things we can do?
* Niko: I don't think we've tested the latter yet.
* Niko: Confident theres more tests to be added
* Niko: I think it includes child modules, but I'd need to check.
* Niko: Reasonable people could disagree here.
* Felix: maybe we need to allow people to express either.
* Esteban: If you don't allow crossing files, that precludes a bunch of refactors you want to do.
* Niko: More likely than wanting to include submodules, is that you list 2 or 3 functions that you want to include, and exclude the others?

&nbsp;

* Q: "we have a large suite of test cases" -- by what metric? How does one measure coverage here?
* Niko: Yeah. We tried to think of everything we could think of.
* Niko: We did the exercise of thinking of everything we want to test apart from the implementation.
* Niko: Maybe we should be utilizing the grammar. Open to ideas here.
* Felix: yeah, that's black box. You might want white box.
* 
&nbsp;

* Q: is `impl Trait1 + Trait2` part of the MVP?
* Niko: Absolutely, yes.
* Felix: is it tested?
* Niko: laughter.
* Mark: Not true today?
* Niko: No it works.
* Felix: you're thinking of dyn
* Mark: oh maybe i'm thinking that you can only use one lifetime?
* Niko: No, you can use multiple lifetimes. But there are cases, where if you have lifetimes (... named but not in interface (?)), inference can get stuck.
* Niko: Looking to add intersection of lifetimes to support this. Won't be in surface syntax, but should help address this. Or wait, ... (discussion of captures and what they address)

&nbsp;

* Niko: Update: Oli and I discussing.
* Niko: Inconsistency between let-binding RFC and hidden types.

```rust
fn bar1() {
    let x: impl Debug = 22_u32;
    /* `x` has opaque type here */
}

type Foo = impl Debug;
fn bar2() {
    let x: Foo = 22_u32;
    /* `x` has hidden type here */
}
```

 * Niko: Investigating changes that would make the second case above `bar2` behave like the first (`bar1`.
 * Niko: This change makes scope questions less relevant.

```rust=
type Foo = impl Debug;
fn bar3() {
    let x: Foo = 22_u32;
    println!("{:?}", x);
}
```

 * Niko: people should care less about whether something is "in scope", because there's less impact on what you can/cannot do.
 * Niko: wrinkle is auto-trait leakage. Auto-traits are still a pain.

### Dyn upcasting initiative update

[Dyn upcasting initiative update](https://rust-lang.github.io/dyn-upcasting-coercion-initiative/updates/2021-oct.html)

 * Niko: A hypothetical breaking change was discovered: unsizing coercion can cause deref coercions today, but having them include upcasts will cause the deref coercion to not be included. 
 * Niko: But main use case for the deref coercions here is to emulate the feature that is being implemented here.
 * Niko: Plan is to emit a future compat warning, and then just make the change.

### Generic associated type initiative update 

[Generic associated type initiative update](https://rust-lang.github.io/generic-associated-types-initiative/updates/2021-oct.html)

&nbsp;

* Esteban: is there interaction between GATs and TAIT?
* Niko: Not to my knowledge
* Felix: what are you expecting?
* Esteban: e.g. `type T<'a> = impl Trait + 'a;`
* (discussion/debate of what is/isn't linted)
* Felix: sounds like it needs a test. lets move on.

### Let else update

[Let else update](https://github.com/rust-lang/rust/issues/87335#issuecomment-933672440)

* Lets try it out!

### Deref patterns update

[Deref patterns update](https://github.com/rust-lang/lang-team/issues/88#issuecomment-935056996)



### Never type stabilization update

[Never type stabilization update](https://github.com/rust-lang/lang-team/issues/60#issuecomment-935233842)

 * Felix: what is our response to point that expressing RFC's here as "edits" is nigh-impossible due to lack of docs/specs of type inference.
 * Mark: type inference system is quite delicate; lots of cases of people getting surprising breakage from two line changes in their code.
 * Niko: it would be nice to be able to write out our changes in a structured way. I don't know how to get there now, though.

### Other

No known update and/or missing from initiatives list:

* inline assembly
* project safe transmute
* const evaluation
* RFC 2229 -- en route to stabilization!
* const generics

## Design meetings

https://github.com/rust-lang/lang-team/issues?q=is%3Aopen+is%3Aissue+label%3Ameeting-proposal+-label%3Ameeting-scheduled

* October 13 -- GAT syntax
* October 20 -- dyn upcasting
* October 27 -- never allow unwinding in drop impls
    * Amanieu -- prepare doc 

&nbsp;

* GAT syntax (where clause syntax)
* dyn upcasting, safety
* never allow unwinding in drop impls (20, 27)
    * [amanieu gathered data](https://github.com/rust-lang/lang-team/issues/97#issuecomment-927266622)

## Pending proposals on the lang-team repo

### "Prototype Sync & Async Iterator Items (Minimal generators)" lang-team#121

**Link:** https://github.com/rust-lang/lang-team/issues/121


### "Async fundamentals initiative" lang-team#116

**Link:** https://github.com/rust-lang/lang-team/issues/116

### "negative impls integrated into coherence" lang-team#96

**Link:** https://github.com/rust-lang/lang-team/issues/96

* Update: Niko needs to merge

### "Deprecate target_vendor " lang-team#102

**Link:** https://github.com/rust-lang/lang-team/issues/102

### "Attribute for trusted external static declarations" lang-team#118

**Link:** https://github.com/rust-lang/lang-team/issues/118
