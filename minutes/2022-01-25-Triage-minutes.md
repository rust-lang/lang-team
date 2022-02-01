---
title: Triage meeting DATE
tags: triage-meeting
---

# T-lang meeting agenda

* Meeting date: 2022-01-25

## Attendance

* Team members: Taylor, nikomatsakis, joshtriplett, pnkfelix, scottmcm
* Others: Jane, David, simulacrum

## Meeting roles

* Action item scribe: simulacrum
* Note-taker: nikomatsakis

## Scheduled meetings

- *Maybe* a "lang team initiative" meeting tomorrow, else we'll fall back to backlog
    - Niko and Josh didn't do their prep, so if they don't do it soon, we'll fall back

## Announcements or custom items

* David Barsky interested in better debugger support, creator of lldb is interested in improving Rust support :tada:, though perhaps that is more of a compiler thing
    * Will cover at end of agenda

pnkfelix: Josh, you made a point about roadmaps...I did some discussion with compiler team, is there a thread about the lang team roadmap?

Josh: there isn't one, we should probably start one, I suspect the right answer is a combination of a zulip thread and a hackmd. Start with a zulip thread and capture vague consensus and hammer it into some semblance of a roadmap. I think there's a possibility it will help direct resources, but it's not a given, not suggesting we spend days and days. Just a bit of time writing down things we would really like to have happen but somebody needs to put effort into. I have a few things for that list: e.g., cfg-accessible.

Niko: This sounds a bit different from a roadmap, which I think is actually good, it seems more like a "help wanted" list. We've talked about that before and I think it's a good idea.

Josh: Yes, I've heard from companies that people want to contribute to Rust in areas that they feel confident are high impact. One way to do that is to look at a roadmap. They're putting a huge amount of stock into "what have we explicit said we want". We've done both kinds of roadmaps. More recently we've done "here is what we think is going to get done", which is useful, but small, and doesn't solve this problem. In the past we did more ambitious things, and only a fraction got done. But in this context that is the kind of roadmap that has been requested. "What do you want to have done that will get done better". That's a thing we can do and it feels lower effort, because all we have to say is "here are things we want" and be vaguely realistic.

Felix: I am not sure a pure wishlist is ideal. I think being explicit about what is wishlist vs planned is really important. If we just focus on making a wishlist and don't show what we have planned, I think people will be more engaged if people are actively investing in aligned things. They'll feel a lot more welcome and get more return if they're working in the same area as other people.

Josh: sure. I think among other things for isntance, the planned vs wishlist could be "here are the things we've already accepted MCPs, they have owners, vs here are things we know we want to have done -- like accepted RFCs or things other teams want from us -- that don't seem likely to happen unless somebody helps". It's not just "here's our wish list", it's "here's especially valuable things". 

Niko: Zulip thread seems like a good start, seems like we may need more than that.

Felix: Time is of the essence, we should get something up soon rather than the perfect thing mid-year.

Josh: Doesn't have to be this week, but the next 4-6 weeks would be good. 

Niko: I was pushing on terminology, I think wishlist might be too weak, roadmap too strong. But I think we should definitely have a good invitation to "go ahead and get started".

Josh: Needs to be things where we are very excited about. 

Niko: It might be that some items are like "investigate X", which isn't a promise to accept X.

Josh: I suspect more practical, less research-y things would also be good. I'd have a bias towards things that need code written. Design work of the architectural style is fine, design work that requires a PhD seems less plausible.

---

* async: niko getting interested in generalized coroutines :)
   * in summary, I've noticed that there are a lot of cases where we need `&mut` that is not held across an await
   * I'm doing a bit of thinking about it, plan to publish a blog post

josh: this doesn't have higher priority than async in traits, right?

niko: no, no. that's more important.

josh: just on the "reasonably important" list. I do think I'd like to be able to write a generator instead of hand-writing iterators and async iterators.

niko: primarily I'm talking about the ability to take input when you get re-awoken.

cramertj: I have absolutely wanted that but also never able to figure out a coherent API surface. Also lots of places where you want it to be an `&mut` and not a thing you pass around by value. You wind up with this weird "NLL++" style thing, where you have an `&mut` you can only touch on either side of an await, and it understands that. The details are funky.

niko: agreed

---

* negative impls: spastorino landed a key PR, still a few more tweaks but probably ready for an RFC soon-ish --nikomatsakis
    * feeling pretty good about state of implementation, hoping to move that to an RFC soon-ish

---

* Libs team interested in having a "things we'd really like from lang (and other teams)" session; how do we want to handle that?
    * Specifics: cfg(accessible), scoped traits, various other things

josh: I thought that design meeting would be good, though CTCFT could be good too, I suspect highest level of productivity is a design meeting. There was a long list, Jane do you want to add anything?

cramertj: for cfg-accessible, is there lang work left? What I remember is that we have an accepted RFC, and cfg-accessible in a weird "we don't know how to implement this" state, not a "blocked on design" state?

josh: correct, but one of the things we talked about was that it would be feasible to have a subset of cfg-accessible that works. The big problem was "how do you handle the case where you have weird interactions between macros generators module imports" and an accessible that's going and searching through those imports to find what's available. There are some cycles that have to be broken and that requires some joint compiler/lang work. But if we had a subset mechanism, whether that be cfg-accessible or a dedicated cfg-accessible for the stdlib, that would be solving a great many of the problems already and ignoring some of the details of the namespace that you are in. Have the reason people want this is for the stdlib.

cramertj: is the concern that we might not want to standardize cfg-accessible for the stdlib because it might imply that it exists in more generality, or do we just need lang team sign-off, or what?

josh: not sure that we can use cfg-accessible as spec'c to handle just the stdlib. As spec'd it should handle the case where you "use other crate as std", for example. But compiler team doesn't know if we can handle that. If you ignore that case completely and just say we magically know about std/core/alloc and similar, then that's easy to support in the compiler, but not obvious we should call it cfg-accessible. Maybe  we want another name?

cramertj: I see.

josh: key detail, this is something libs would really like to enable. There are a lot of crates in the crates.io ecosystem that really want to say "I'm providing this thing but if std provides it, I will not". Things like extension methods and trait impls. If there's any way for us to provide this mechanism and compiler to implement it, libs can add things that would otherwise break.

jane (in chat): itertools intersperse is a good example.

cramertj: I've lost a bit of track. The specific motivation from libs, if it's libraries that have dedicated maintainers, is there a reason the path we carve out for them be cfg-version? I know that cfg-version is basically done and was FCP'd and didn't stabilize I think because of concerns about cfg-accessible. It sounds like the design space has grown significantly larger. I guess we're getting into the meeting now.

josh: I'm the one with the blocking concern, and my concern was that once ecosystem starts adopting cfg-version, it'll be hard to move, and experiences from C/JS is that version checking is much more painful, I'd love for people to have both options. I'd have liked full cfg-accessible to work, 

mark: did we decide we wanted a design meeting? I think it would be helpful for libs to talk not about features but the problem. It sounds like e.g. cfg-accessible would have other solutions.

josh: yes, I think it's worth floating "here's the problem that we need".

niko: I think a design meeting is good, and I second mark that "interesting problems that come up a lot and some possible solutions" is a great starting point.

## Action item review

* [Action items list](https://hackmd.io/gstfhtXYTHa3Jv-P_2RK7A)

## Pending lang team project proposals

Niko started playing with github projects to represent these, will try to finish the write-up this week:

https://github.com/orgs/rust-lang/projects/16

there are alternate ways to view this:

https://github.com/orgs/rust-lang/projects/16/views/1?layout=board

josh: one point made on compiler team was that MCPs that don't progress are not a problem. it's ok for them to just get forgotten about. We could potentially take a similar approach around proposals.

pnkfelix: the [T-compiler Zulip thread](https://rust-lang.zulipchat.com/#narrow/stream/131828-t-compiler/topic/What.20happens.20to.20MCPs.3F) includes a comment by Mark that suggests that lang/compiler team have distinct goals for their proposals vs MCPs and maybe we should deviate.

### "Deprecate target_vendor " lang-team#102

**Link:** https://github.com/rust-lang/lang-team/issues/102

### "Async fundamentals initiative" lang-team#116

**Link:** https://github.com/rust-lang/lang-team/issues/116

### "Attribute for trusted external static declarations" lang-team#118

**Link:** https://github.com/rust-lang/lang-team/issues/118

### "Prototype Sync & Async Iterator Items (Minimal generators)" lang-team#121

**Link:** https://github.com/rust-lang/lang-team/issues/121

### "Support platforms with size_t != uintptr_t" lang-team#125

**Link:** https://github.com/rust-lang/lang-team/issues/125

### "Positional Associated Types" lang-team#126

**Link:** https://github.com/rust-lang/lang-team/issues/126

### "Heap allocations in constants" lang-team#129

**Link:** https://github.com/rust-lang/lang-team/issues/129

### "Interoperability With C++ Destruction Order" lang-team#135

**Link:** https://github.com/rust-lang/lang-team/issues/135

## PRs on the lang-team repo

None.


## RFCs waiting to be merged

None.

## Proposed FCPs

**Check your boxes!**
### "Tracking issue for naked fns (RFC #1201)" rust#32408

**Link:** https://github.com/rust-lang/rust/issues/32408

Josh: Quick update, this RFC #1201 was revised to "constrained naked functions" that limit to a single assembly block. Original RFC was underspecified. What I'm proposing is that we FCP close this tracking issue and say "we won't do unconstrained naked functions, we are going to proceed with constrained naked functions and likely stabilize them, and if people want more mechanism (e.g., multiple asm blocks), that can be done in a followup".

Niko: I can't quite tell what npmcallum is saying but I *think* he's agreeing with you. 

Josh: Yes, he's explaining the history and talking about some of the implementation details.

### "Make `unused_lifetimes` lint warn-by-default" rust#92386

**Link:** https://github.com/rust-lang/rust/pull/92386

Josh: Update. Crater run was done, it had thousands of lints, and current state is that it has enough people to pass FCP, but the blocking concerns on the basis of "it should be possible to underscore away lifetimes" and "dealing with semver issues". We discussed it previously and don't need to retread that territory, anything to add before we go on? *crickets*

### "Allow `impl Fn() -> impl Trait`" rust#93082

**Link:** https://github.com/rust-lang/rust/pull/93082

Josh: Last proposed FCP is allowing "impl Fn -> impl Trait". There are a couple of potential issues. My original reason for starting the FCP was helping us make the consensus decision on the associativity of the `->` so you can have a function returning a function and have it do something sensible. I want to know what the nexts steps are.

Niko: I was persuaded by comments on the thread that we're not treading new ground here, especially given that we support `Foo<Output = ...>` position.

cramertj: The thing I found annoying was that because you can use it in "argument position" for other traits, it should be consistent. So you can have `impl Foo<impl Bar>` and it would be reasonable to expect that `impl Fn(impl Bar)` behaves the same, especially when we get to a world where we have access to the generic fn traits. 

scottmcm: This makes me slightly sadder about impl meaning both things, it's a bit less obvious what thing impl should mean once you're inside a few places. Apparently you can have `impl Iterator<Item = impl Debug>` and you could probably convince me either way way...

josh: ...you can also do `impl Iterator<Item: Debug>`...

cramertj: the point is that we don't have an impl-style syntax for generic closures and type-level HRTBs. I anticipate the day will come when we are writing 

nikomatsakis: I have wanted to make `'_` and `impl Trait` kidn of equivalent, e.g. "impl binds to the same spot a `'_` would", and we do already distinguish `impl Fn(&u32)` from `impl Foo<&u32>`.

cramertj: I find that moderately convincing, I still think I might've preferred other behavior in some other cases, but I think that ship has sailed.

josh: To be clear, you're concerned about the fact that `impl Fn(impl Bar)` could mean either `impl Fn(T)` where `T: Bar` or `for<T: Bar> impl Fn(T)`?

cramertj: yes.

josh: which one are we currently going forward with?

cramertj: in this proposal, neither, we're only talking about return position impl trait here.

pnkfelix: did someone actually explicitly state this?

cramertj: yes, but I'll confirm.

josh: doesn't return position still have that ambiguity?

cramertj: no, because impl trait in return position always means that "some type that implements the trait is being returned". Conceptually it's an output type. The counter example is things like `collect`.

Josh: So if you write `fn func(f: impl Fn() -> impl Debug)`, you can pass that `fn f() -> impl Debug`, but can you pass it `fn f() -> u32`?

```rust
fn func(f: impl Fn() -> impl Debug) { }

// becomes:
// fn func<T>(f: impl Fn() -> T) { }
// fn func<F, T>(f: F) where F: Fn() -> T
// fn func<F, T>(f: F) where F: Fn<(), Output = T>

// as opposed to something like this (which I think nobody wants, but there is some concern around `'_`):
// fn func<T>(f: impl Fn() -> T) { }
// fn func<F>(f: F) where for<T> F: Fn() -> T

// cramertj wants it to mean this:
// fn func<F>(f: F) where F::Output: Debug

fn foo() -> impl Debug { 22 }

fn bar() -> u32 { 44 }

fn quuz() -> impl Debug { String::new("hi") }

fn main() {
    func(foo);
    func(bar); // both of these compile today
    func(quuz); // fsk: okay this compiles too then.
}
```

pnkfelix: does the ambiguity in interpretation arise for `fn foo(f: impl Fn() -> impl Ret)` ? (ha ha josh and felix had same thought)

nikomatsakis: `fn(&u32) -> &u32` gives me something from the input

nikomatsakis: `impl Fn(&u32) -> impl PartialEq<&u32>`

```rust=
fn foo<'a, T>(f: impl Fn(&u32) -> T)
where
    T: PartialEq<&'a u32>
```

nikomatsakis: not totally sure what I think it *should* be

cramertj: is that observable today? you can't turbofish.

nikomatsakis: in terms of errors, yes, and maybe in other ways.

cramertj: It might be unambiguous to only allow `-> impl Fn() -> impl Trait`, and not allow it in parameter position at either level.

Summary: nikomatsakis to raise a blocking concern about lifetimes, and another about impl ambiguity

## Active FCPs
### "Tracking issue for #[cfg(target_has_atomic = ...)]" rust#32976

**Link:** https://github.com/rust-lang/rust/issues/32976

### "Positional Associated Types" lang-team#126

**Link:** https://github.com/rust-lang/lang-team/issues/126

### "Interoperability With C++ Destruction Order" lang-team#135

**Link:** https://github.com/rust-lang/lang-team/issues/135



## P-critical issues

None.




## Nominated RFCs, PRs and issues

### "`no_mangle`/`used` static is only present in output when in reachable module" rust#47384

**Link:** https://github.com/rust-lang/rust/issues/47384

### "Warn about dead tuple struct fields" rust#92972

**Link:** https://github.com/rust-lang/rust/pull/92972

* nikomatsakis: I did feel the last time not that I didn't want to do this change, but that I wanted a better way to roll this change *out*.
* josh: we could start by having a separate sublint, make it allow-by-default, and put out an announcement "this is a thing you might want to turn on", and then consider enabling it in the future on the basis of folks being forewarned.
    * if it's a separate lint, people can disable it very easily, as well.
* nikomatskais: that seems very good, giving people time to opt-in, and then an easy (1-line) opt-out, will reduce the pain a lot.

### "Tracking Issue for scoped threads" rust#93203

**Link:** https://github.com/rust-lang/rust/issues/93203