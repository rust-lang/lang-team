---
title: Triage meeting 2022-11-22
tags: triage-meeting
---

# T-lang meeting agenda

* Meeting date: 2022-11-22

## Attendance

* Team members: scott, josh
* Others: mark, dtolnay, Gary Guo (nbdd0121), y86-dev, pnkfelix (30min late)

## Meeting roles

* Action item scribe: 
* Note-taker: scottmcm

## Scheduled meetings
- "Meeting proposal: Rust Specification" [lang-team#179](https://github.com/rust-lang/lang-team/issues/179)
    - Closing since the meeting has happened
- "Contracts and Automated Reasoning for Rust" [lang-team#181](https://github.com/rust-lang/lang-team/issues/181)

## Announcements or custom items

## Action item review

* [Action items list](https://hackmd.io/gstfhtXYTHa3Jv-P_2RK7A)

## Pending lang team project proposals

None.




## PRs on the lang-team repo

None.




## RFCs waiting to be merged
### "Support upcasting of `dyn Trait` values" rfcs#3324

**Link:** https://github.com/rust-lang/rfcs/pull/3324

scott: I'll take that

### "Add lang-team advisors team" rfcs#3327

**Link:** https://github.com/rust-lang/rfcs/pull/3327

josh: I'll take this one

## Proposed FCPs

**Check your boxes!**
### "RFC: `c"…"` string literals" rfcs#3348

- **Link:** https://github.com/rust-lang/rfcs/pull/3348
- [**Tracking Comment**](https://github.com/rust-lang/rfcs/pull/3348#issuecomment-1317925033): 
	> Team member @joshtriplett has proposed to merge this. The next step is review by the rest of the tagged team members:
	> 
	> * [ ] @cramertj
	> * [x] @joshtriplett
	> * [ ] @nikomatsakis
	> * [ ] @pnkfelix
	> * [ ] @scottmcm
	> 
	> No concerns currently listed.
	> 
	> Once a majority of reviewers approve (and at most 2 approvals are outstanding), this will enter its final comment period. If you spot a major issue that hasn't been raised at any point in this process, please speak up!
	> 
	> See [this document](https://github.com/rust-lang/rfcbot-rs/blob/master/README.md) for info about what commands tagged team members can give me.
- [**Initiating Comment**](https://github.com/rust-lang/rfcs/pull/3348#issuecomment-1317925024): 
	> @rfcbot merge

scott: I'm interested in splitting out part of the second one, https://github.com/rust-lang/rfcs/pull/3349#discussion_r1023256002

josh: is the proposal that it's a valid tt or a valid liter?

scott: that it's literal too, I think

josh: oh, not just a token?

scott: I thought that we accepted invalid literals as literals in https://github.com/rust-lang/rust/pull/102944

Gary: invalid literals are valid as tokens but not valid when lowered to HIR (will not error as input to proc macro).

josh: you should make fcp+concern 3349, scott!

### "Stabilize `#![feature(target_feature_11)]`" rust#99767

- **Link:** https://github.com/rust-lang/rust/pull/99767
- [**Tracking Comment**](https://github.com/rust-lang/rust/pull/99767#issuecomment-1320937299): 
	> Team member @joshtriplett has proposed to merge this. The next step is review by the rest of the tagged team members:
	> 
	> * [ ] @cramertj
	> * [x] @joshtriplett 
	> * [ ] @nikomatsakis
	> * [ ] @pnkfelix
	> * [ ] @scottmcm
	> 
	> No concerns currently listed.
	> 
	> Once a majority of reviewers approve (and at most 2 approvals are outstanding), this will enter its final comment period. If you spot a major issue that hasn't been raised at any point in this process, please speak up!
	> 
	> See [this document](https://github.com/rust-lang/rfcbot-rs/blob/master/README.md) for info about what commands tagged team members can give me.
- [**Initiating Comment**](https://github.com/rust-lang/rust/pull/99767#issuecomment-1320937292): 
	> @rfcbot merge

josh: I proposed it.  Some discussion about current status, seems ready to be considered.

### "Remove const eval limit and implement an exponential backoff lint instead" rust#103877

- **Link:** https://github.com/rust-lang/rust/pull/103877
- [**Tracking Comment**](https://github.com/rust-lang/rust/pull/103877#issuecomment-1310223336): 
	> Team member @pnkfelix has proposed to merge this. The next step is review by the rest of the tagged team members:
	> 
	> * [ ] @cramertj
	> * [ ] @joshtriplett
	> * [ ] @nikomatsakis
	> * [x] @pnkfelix
	> * [ ] @scottmcm
	> 
	> Concerns:
	> 
	> * dont-use-lints-for-this (https://github.com/rust-lang/rust/pull/103877#issuecomment-1315785467)
	> 
	> Once a majority of reviewers approve (and at most 2 approvals are outstanding), this will enter its final comment period. If you spot a major issue that hasn't been raised at any point in this process, please speak up!
	> 
	> See [this document](https://github.com/rust-lang/rfcbot-rs/blob/master/README.md) for info about what commands tagged team members can give me.
- [**Initiating Comment**](https://github.com/rust-lang/rust/pull/103877#issuecomment-1310223314): 
	> Before this change, there was a hard error associated with hitting the internal const-eval limit. And thus there was a notion that every rust compilation is "always" guaranteed to terminate. (Note this was a flawed notion, since proc-macros do not currently have any such limit imposed on their execution.)
	> 
	> This PR changes that by turning this limit into a lint that can be allowed by the user.
	> 
	> I want lang team to sign off explicitly on the idea of introducing *some kind* of user-exposed limit that 1. allows the user to change the limit on how long const-eval can proceed (where the values it can change to in this PR are limited to either âˆž via allow or 1,000,000 via deny) and 2. can introduce a new way for the compiler to infinite loop (i.e. by the user turning off the limit)
	> 
	> Things I want to discourage as being "out of scope" for this fcp discussion:
	>  * This PR also changes the underlying mechanism that drives the diagnostic machinery, namely by using exponential backoff, but I want to treat that as a detail associated with the lint=warn case; the lang team currently should focus solely on the question on whether we expose this as a user-controllable thing at all. In other words, the focus for this fcp should be on the fact that the lint can be set to `allow`.
	>  * The specific details of what metric is used is not under fcp. (@nikomatsakis and I agree that counting terminators executed is not a good metric to expose to users in the long term (it is too coupled to the internal details of MIR).)
	>  * This fcp is not committing us to never exposing a finer-grain control. I.e. it may be a good idea to revise this lint to allow someone to override the threshold, rather than keeping the hard-coded limit of 1,000,000. But I do not want to side-track this fcp with discussions of that; I think exposing the ability to control the specific number should only come after the metric itself has been revisited.
	> 
	> (I am willing to consider "what should this lints default be: is it allow-by-default or deny-by-default" as something in scope for lang team discussion.)
	> 
	> @rfcbot fcp merge
	> 
	> 

josh: last week's discussion led to a concern that Felix filed.  Appears to have been updated, so Felix will want to look at it.

### "Stabilise inline_const" rust#104087

- **Link:** https://github.com/rust-lang/rust/pull/104087
- [**Tracking Comment**](https://github.com/rust-lang/rust/pull/104087#issuecomment-1315946147): 
	> Team member @scottmcm has proposed to merge this. The next step is review by the rest of the tagged team members:
	> 
	> * [ ] @cramertj
	> * [ ] @joshtriplett 
	> * [ ] @nikomatsakis
	> * [ ] @pnkfelix
	> * [x] @scottmcm
	> 
	> No concerns currently listed.
	> 
	> Once a majority of reviewers approve (and at most 2 approvals are outstanding), this will enter its final comment period. If you spot a major issue that hasn't been raised at any point in this process, please speak up!
	> 
	> See [this document](https://github.com/rust-lang/rfcbot-rs/blob/master/README.md) for info about what commands tagged team members can give me.
- [**Initiating Comment**](https://github.com/rust-lang/rust/pull/104087#issuecomment-1315946122): 
	> Thanks for the report!  It looks like the current state is enough to allow [this to work](https://play.rust-lang.org/?version=nightly&mode=debug&edition=2021&gist=4c95f0a9ec92f4b1084f4f4021d87393):
	> ```rust
	> pub fn create_none_array<T, const N: usize>() -> [Option\<T\>; N] {
	>     [const { None }; N]
	> }
	> ```
	> which is the use that [keeps coming up](https://users.rust-lang.org/t/is-it-possible-to-imply-copy-for-option-none/78892/3?u=scottmcm) on URLO.  And this *also* keeps coming up ([1](https://internals.rust-lang.org/t/do-const-fn-s-always-be-inlined/17732/5?u=scottmcm) [2](https://internals.rust-lang.org/t/simple-dependent-types-in-const-contexts/17593/3?u=scottmcm)) on IRLO as needed (or at least wanted) for various things.
	> 
	> So let's do it!
	> 
	> @rfcbot fcp merge
	> 
	> cc Tracking Issue https://github.com/rust-lang/rust/issues/76001, which also covers patterns so isn't closed by this PR.

scottmcm: Is https://github.com/rust-lang/rust/issues/86730 a blocker?

gary:
```rust
match () {
    () => const {
        
    }, // Whether do we need ","?
}
```

josh: which case are we in?

gary: current requires a comma

josh: seems mildly surprising, as people will see the braces.

gary: `async` behaves like this

scott: is it breaking to remove the requirement?

josh: if there 

gary: `if Expression [except struct expression] BlockExpression` (https://doc.rust-lang.org/reference/expressions/if-expr.html#if-expressions)

josh: so whichever way we go, `if const { foo } { }` will work.

scott: which I know works today

gary: if it's not a breaking change then we can just accept it now and change it later


scott: are there any weird things that could come up like `const { a } - b` that would change meaning?

josh: would this change meaning if we removed the `;` requirement?
```rust
const { 2 }
- const { 1 }
```

gary: yes, this would change behaviour. `{ 2 } - { 1 }` errors today.

josh: so it behaves like `async {}` not `{}`, and removing the requirement *would* make it behave like `{}`?

scott: sounds like yes, so is it useful as a statement?

gary: yes, for "static assertions"
```rust=
const { assert!(...) };
```

josh: so you need a statement for `async{}`

scott: async block as a statement does nothing since it's never polled.

gary: could we change `async` too?

josh: we could certainly add a lint

scott: maybe `must_use` already does?

scott: this today is two statements, not a subtraction expression, so expecting this from `const{}` seems reasonable to me.
```rust=
if b { 1 } else { 2 }
- if b { 3 } else { 4 }
```

needs to be unit to throw it away without a semicolon:
```rust=
error[E0308]: mismatched types
 --> src/main.rs:2:22
  |
1 | fn main() {
  |           - expected `()` because of default return type
2 | if true { 1 } else { 2 }
  |                      ^ expected `()`, found integer
```

`unsafe` and `if` don't need the semicolon.

scott: I particularly like the `unsafe` analogy because it's the same kind of "uncarried effect".

gary: it's more like a closure (at least internally in rustc, e.g. for typechecking)

felix: as a user it feels like a syntactic detail constraining the language.

gary: if you put unsafe outside the const block, it will not propagate into it.

gary: `unsafe { const { require_unsafe() }}` doesn't work, similar to `unsafe { [0; require_unsafe()] }`.

scott: I have a concern now

josh: I'm very surprised that unsafe doesn't propagate into an array size expression.  I would expect them to be consistent that it *does* propagate

felix: the argument is that things are happening at compile-time vs runtime?

gary: it's like a const item, but more powerful.

scott: but the more power is sortof the point -- like accessing locals in a closure.

scott: we could change how it works in array lengths, and it'd just add more lints for unnecessary unsafe blocks

felix: I feel like we need a design meeting about this stuff now, before we lift restrictions to make then uniform.

gary: only thing needing a decisions here is the with or without block, since we can't change that.

josh: does it need a meeting or just a proposal?

scott: I can file a concern about without-block.

josh: let's treat them separately, but file concerns for both

scott: do we need a concern on *this* RFC for the `unsafe` behaviour?  Is it a blocker?

josh: how hard would it be to fix?

gary: shouldn't be bad -- much easier than the type inference.  It works much like a closure, which also propagate, so should be feasible.  array size might be harder, since it feeds back into the type system.



## Active FCPs
### "Restrictions" rfcs#3323

**Link:** https://github.com/rust-lang/rfcs/pull/3323



## P-critical issues

None.




## Nominated RFCs, PRs and issues discussed this meeting

### "Panic on invalid usages of `MaybeUninit::uninit().assume_init()`" rust#100423

**Link:** https://github.com/rust-lang/rust/pull/100423

(I forgot to take notes but)

comments about how adding something not in the code is scary, but also it's already UB so that's maybe not surprising?  How does this compare to things like `ud2`s from `unreachable_unchecked()` or not-actually-diverging `->`s?  Should there be a lang FCP for this?

## Nominated RFCs, PRs and issues NOT discussed this meeting
### "RFC: Field projection" rfcs#3318

**Link:** https://github.com/rust-lang/rfcs/pull/3318

### "impl DispatchFromDyn for Cell and UnsafeCell" rust#97373

**Link:** https://github.com/rust-lang/rust/pull/97373

### "Stabilize default_alloc_error_handler" rust#102318

**Link:** https://github.com/rust-lang/rust/pull/102318

### "PhantomData: fix documentation wrt interaction with dropck" rust#103413

**Link:** https://github.com/rust-lang/rust/pull/103413

### "Implement a lint for implicit autoref of raw pointer dereference " rust#103735

**Link:** https://github.com/rust-lang/rust/pull/103735

### "Stabilise inline_const" rust#104087

**Link:** https://github.com/rust-lang/rust/pull/104087

### "Refine `instruction_set` MIR inline rules" rust#104121

**Link:** https://github.com/rust-lang/rust/pull/104121

### "More deriving on packed structs" rust#104429

**Link:** https://github.com/rust-lang/rust/pull/104429

### "update aliasing rules section of the reference" reference#1290

**Link:** https://github.com/rust-lang/reference/pull/1290

### "Raw pointer alignment checks in debug" compiler-team#570

**Link:** https://github.com/rust-lang/compiler-team/issues/570


