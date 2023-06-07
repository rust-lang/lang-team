---
title: Triage meeting 2022-11-08
tags: triage-meeting
---

# T-lang meeting agenda

* Meeting date: 2022-11-08

## Attendance

* Team members: nikomatsakis, pnkfelix, cramertj, joshtriplett
* Others: y86-dev, nbdd0121

## Meeting roles

* Action item scribe:
* Note-taker: pnkfelix (and niko)

## Scheduled meetings
- "Meeting proposal: Rust Specification" [lang-team#179](https://github.com/rust-lang/lang-team/issues/179) 
- "Contracts and Automated Reasoning for Rust" [lang-team#181](https://github.com/rust-lang/lang-team/issues/181) 

## Announcements or custom items

### possibility of scheduling `let _ = ...` discussion for tomorrow

pnkfelix: would like to have `let _ = ...` as a design meeting tomorrow

## Action item review

* [Action items list](https://hackmd.io/gstfhtXYTHa3Jv-P_2RK7A)

## Pending lang team project proposals

None.




## PRs on the lang-team repo

None.




## RFCs waiting to be merged
### "Support upcasting of `dyn Trait` values" rfcs#3324

**Link:** https://github.com/rust-lang/rfcs/pull/3324

Gary Guo: concern raised about potential space overhead. Once upcasting is globally enabled, space overhead will remain for all users.

cramertj: folks do rip out existing object metadata in some cases. not clear how much more data this would be.

nikomatsakis: how many cases could be addressed with LTO-- i.e., whole program optimization?

cramertj: would need access to LLVM bytecode if you invoked rust code that was compiled to some other artifact

joshtriplett: could statically determine if something does upcasts and potentially throw away metadata

cramertj: it would change the layout of the vtable, right?

nikomatsakis: no, could add a null pointer etc

garyguo: concern is the pointee of the trait vtable

nikomatsakis: I feel like we want upcasting and we are going to want to allow libstd to do it

cramertj: maybe we avoid it from core?

joshtriplett: I do see it being useful in core in some specific places

cramertj: I see this as an analogue to some of the lengths people go to avoid using format machinery

joshtriplett: Mara is working on a proposal for subsetting the stdlib, she was talking about partial no-std cases, but you could imagine doing it for core too on a lang feature basis

pnkfelix: two variants of dyn? upcastable and not? is that absurd?

cramertj: we're talking about users who are willing to use a subset of rust, but prob do want core.

garyguo: all use cases where space is concerned. typically doesn't include std, but could include core/alloc

cramertj: if you're pulling in some embedded std traits that folks would use. Having all of them be not workable in this new world. Or maybe they're not usable via dynamic ....

nikomatsakis: ...if there were a dyn2 ...

garyguo: ...I think opt-in upcasting is a better approach. Most of the use cases have a workaround.

pnkfelix: opt-in upcasting is analogous to those workarounds, the question is whether people have to anticipate that they're going to need upcasting.

cramertj: I don't think you can push it all the way down to the usage site, since it doesn't have creation.

Three proposals on the table:

* put an "opt-in" attribute on the trait to permit upcasting, or possibly per supertrait
* put an "opt-out" attribute on the trait to forbid upcasting, or possibly per supertrait
* two variants

Only case of concern:

* multiple supertraits
* and you use the dyn

Some cases where you use a trait as a trait alias. If you apply upcasting to that trait, you get a lot of vtables being generated.

cramertj: I didn't realize the scope of problem was so restricted. I'm less concerned given that its only about multiple supertraits, since I don't expect to come up so often.

cramertj: Oh, but then there's `trait Error : Debug + Display`

nikomatsakis: i'm okay with making this an Unresolved Question. It would be good to get data. Maybe we can ask people to help gather it, it shouldn't be hard.

### "Add lang-team advisors team" rfcs#3327

**Link:** https://github.com/rust-lang/rfcs/pull/3327



## Proposed FCPs

**Check your boxes!**
### "Restrictions" rfcs#3323

- **Link:** https://github.com/rust-lang/rfcs/pull/3323
- [**Tracking Comment**](https://github.com/rust-lang/rfcs/pull/3323#issuecomment-1298977805): 
	> Team member @joshtriplett has proposed to merge this. The next step is review by the rest of the tagged team members:
	> 
	> * [ ] @cramertj
	> * [x] @joshtriplett
	> * [ ] @nikomatsakis
	> * [ ] @pnkfelix
	> * [ ] @scottmcm
	> 
	> Concerns:
	> 
	> * syntax-unresolved-question (https://github.com/rust-lang/rfcs/pull/3323#issuecomment-1298982762)
	> 
	> Once a majority of reviewers approve (and at most 2 approvals are outstanding), this will enter its final comment period. If you spot a major issue that hasn't been raised at any point in this process, please speak up!
	> 
	> See [this document](https://github.com/rust-lang/rfcbot-rs/blob/master/README.md) for info about what commands tagged team members can give me.
- [**Initiating Comment**](https://github.com/rust-lang/rfcs/pull/3323#issuecomment-1298977766): 
	> We discussed this in today's @rust-lang/lang meeting.
	> 
	> We definitely want this feature. We're still debating some of the syntax, and we may want to change it. But at the same time, the *rest* of this feature seems great, and we can always change the syntax before stabilization. Thus:
	> 
	> @rfcbot merge


joshtriplett: Concern is to explore syntax, especially for the most common cases (e.g., "sealed" instead of `impl crate`). I don't want to block the RFC on bikeshedding.

nikomatsakis: having feature available would help us see how it's actually being used.

### "Remove drop order twist of && and || and make them associative" rust#103293

- **Link:** https://github.com/rust-lang/rust/pull/103293
- [**Tracking Comment**](https://github.com/rust-lang/rust/pull/103293#issuecomment-1293411354): 
	> Team member @joshtriplett has proposed to merge this. The next step is review by the rest of the tagged team members:
	> 
	> * [ ] @cramertj
	> * [x] @joshtriplett
	> * [ ] @nikomatsakis
	> * [ ] @pnkfelix
	> * [ ] @scottmcm
	> 
	> Concerns:
	> 
	> * ~~consistency-with-if-let~~ resolved by https://github.com/rust-lang/rust/pull/103293#issuecomment-1293451766
	> 
	> Once a majority of reviewers approve (and at most 2 approvals are outstanding), this will enter its final comment period. If you spot a major issue that hasn't been raised at any point in this process, please speak up!
	> 
	> See [this document](https://github.com/rust-lang/rfcbot-rs/blob/master/README.md) for info about what commands tagged team members can give me.
- [**Initiating Comment**](https://github.com/rust-lang/rust/pull/103293#issuecomment-1293411326): 
	> @rfcbot merge
	> 
	> @rfcbot concern consistency-with-if-let
	> Per https://github.com/rust-lang/rust/pull/103293#issuecomment-1293408574

Really weird edge case in the drop order of temporaries created with `&&` and `||` operators:

```rust
f(1).g() && f(2).g() && f(3).g() && f(4).g()
```

drops 2, 3, 4 and then 1. Changing an `&&` expression by pre-pending something will substantially change drop order.

Seems accidental and unlikely to be by design.

Crater run to find if anything relied on it. Net result is: we found one crate with a one-line diff, it was fixed.

simulacrum: crater run is evidence of lack of compilation failure, but runtime semantics do change.

joshtriplett: true, but if someone were relying on the locking, bear in mind they would be dropped in reverse order of having been acquired

nikomatsakis: I buy the argument that this is equally likely to fix latent bugs as to cause them.

### "Unreserve braced enum variants in value namespace" rust#103578

- **Link:** https://github.com/rust-lang/rust/pull/103578
- [**Tracking Comment**](https://github.com/rust-lang/rust/pull/103578#issuecomment-1298942489): 
	> Team member @joshtriplett has proposed to merge this. The next step is review by the rest of the tagged team members:
	> 
	> * [ ] @cramertj
	> * [x] @joshtriplett
	> * [ ] @nikomatsakis
	> * [x] @pnkfelix
	> * [ ] @scottmcm
	> 
	> No concerns currently listed.
	> 
	> Once a majority of reviewers approve (and at most 2 approvals are outstanding), this will enter its final comment period. If you spot a major issue that hasn't been raised at any point in this process, please speak up!
	> 
	> See [this document](https://github.com/rust-lang/rfcbot-rs/blob/master/README.md) for info about what commands tagged team members can give me.
- [**Initiating Comment**](https://github.com/rust-lang/rust/pull/103578#issuecomment-1298942413): 
	> We discussed this in today's @rust-lang/lang meeting, and we agree that we should make this change. If a concrete proposal to use these arises, we can change it back in a future edition.
	> 
	> @rfcbot merge

## Active FCPs

None.




## P-critical issues

None.




## Nominated RFCs, PRs and issues discussed this meeting

### "`Arc::ptr_eq` does not always return "true if the two Arcs point to the same allocation" as documented" rust#103763

**Link:** https://github.com/rust-lang/rust/issues/103763

Question: should `a == b` where `a, b: *const dyn Foo` be

```rust
a.data == a.data && a.vtable == b.vtable // today
a.data == a.data // proposal A
```

pnkfelix: is it true that you can't observe the difference? i.e., vtables may differ, but they must be equivalent? with unsafe code, you could do things.

garyguo: two ZST values of different types that implement the same trait?

pnkfelix: I'm immediately opposed based on that argument.

Property that `==` has today:

* If it is true, then whether I call the method on `a` or `b` is equivalent.

Property that `==` does not have today:

* If it is false, then `a` and `b` represent different values.

Arguments where just comparing data could go wrong:

* ZST values (same data pointer regardless of type)
* unsafe code constructing two `*const dyn Trait` for the same data pointer with observably distinct vtables.

vtables can differ even for the same type (e.g. two copies from the linker)
vtables can be the same for different types (merging)


Q: is there precedent for such a guarantee from `==`? E.g. we have a `trait Eq`, but its not `unsafe`, so people can freely violate this rule for their `==`.

pnkfelix: I see this as a business logic concern, not a safety concern. For *safety*, one cannot rely on this property, yes.

Discussion of "What is notion of "same" that we are trying to achieve via `==`?"

garyguo points out that  https://doc.rust-lang.org/std/ptr/fn.eq.html says "compares pointer address" but it has the same behavior of also comparing vtables.

Some claims:
 * If we make this change, you can no longer make useful conclusions from `==` returning `true` nor `false`.
     * backtracking: For two non-ZSTs, it *can* be useful to just compare the data pointers alone.

niko: I wouldn't be opposed to a lint against these kinds of comparisons, since they can be easily misused. (With the reasonable alterantive being some standard library methods like `data_table_eq` that spell out some of the concerns e.g. the potential for the linker to duplicate vtables or merge them)

CONCLUSION: pnkfelix to comment on the issue.

### "Restrictions" rfcs#3323

**Link:** https://github.com/rust-lang/rfcs/pull/3323

Discussed in the context of FCPs, un-nominated.

### "Remove drop order twist of && and || and make them associative" rust#103293

**Link:** https://github.com/rust-lang/rust/pull/103293

Discussed in the context of FCPs, un-nominated.

### "Parse error recovery is obversable by macros in several cases" rust#103534

**Link:** https://github.com/rust-lang/rust/issues/103534

joshtriplett: Talked this through last week, gave a response, does this need to be nominated?

consensus: no

### "Remove const eval limit and implement an exponential backoff lint instead" rust#103877

**Link:** https://github.com/rust-lang/rust/pull/103877

Problem: fixed step count threshold is very sensitive to MIR lowering details.

nikomatsakis proposal: count backedges + function calls.

nikomatsakis: I would like to keep a limit, but make it predictable, and allow you to set it at the const fn definition.

joshtriplett: I would like to remove the limit and separately permit some kind of watchdog timer that kills rustc.

nikomatsakis: benefit?

joshtriplett: I don't think we can make a predictable strategy. If you went with function call inlines, can't we tell?

nikomatsakis: you have to do work, remember fn entry and the backedges that arose from sourcre code

joshtriplett: and now if people change their code, it could timeout where it didn't before, right?

nikomatsakis: yeah but that's true with wallclock, and worse because it's machine dependent.

pnkfelix: I want to come back to the idea of how important it is to guarantee termination.

pnkfelix: what about proc macros, they may not terminate?

nikomatsakis: ok, that's a strong point.


## Nominated RFCs, PRs and issues NOT discussed this meeting

### "RFC: Field projection" rfcs#3318

**Link:** https://github.com/rust-lang/rfcs/pull/3318

### "impl DispatchFromDyn for Cell and UnsafeCell" rust#97373

**Link:** https://github.com/rust-lang/rust/pull/97373

### "Panic on invalid usages of `MaybeUninit::uninit().assume_init()`" rust#100423

**Link:** https://github.com/rust-lang/rust/pull/100423

### "Stabilize default_alloc_error_handler" rust#102318

**Link:** https://github.com/rust-lang/rust/pull/102318

### "PhantomData: fix documentation wrt interaction with dropck" rust#103413

**Link:** https://github.com/rust-lang/rust/pull/103413

### "Refine `instruction_set` MIR inline rules" rust#104121

**Link:** https://github.com/rust-lang/rust/pull/104121

### "update aliasing rules section of the reference" reference#1290

**Link:** https://github.com/rust-lang/reference/pull/1290


