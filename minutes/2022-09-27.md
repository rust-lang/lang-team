---
title: Triage meeting 2022-09-27
tags: triage-meeting
---

# T-lang meeting agenda

* Meeting date: 2022-09-27

## Attendance

* Team members: Josh, Felix
* Others: dtolnay, Lokathor

## Meeting roles

* Action item scribe: Felix
* Note-taker: Felix

## Scheduled meetings
- Provenance meeting October 5
- Planning either September 28 or October 12

## Announcements or custom items

(Meeting attendees, feel free to add items here!)

## Action item review

* [Action items list](https://hackmd.io/gstfhtXYTHa3Jv-P_2RK7A)

## Pending lang team project proposals
### "Support platforms with size_t != uintptr_t" lang-team#125

**Link:** https://github.com/rust-lang/lang-team/issues/125

### "Positional Associated Types" lang-team#126

**Link:** https://github.com/rust-lang/lang-team/issues/126

### "Interoperability With C++ Destruction Order" lang-team#135

**Link:** https://github.com/rust-lang/lang-team/issues/135

### "allow construction of non-exhaustive structs when using functional update syntax" lang-team#143

**Link:** https://github.com/rust-lang/lang-team/issues/143

### "Async fns in traits" lang-team#150

**Link:** https://github.com/rust-lang/lang-team/issues/150

### "dyn* trait" lang-team#158

**Link:** https://github.com/rust-lang/lang-team/issues/158

### "Initiative: `?` traits, `try` blocks, `yeet` exprs, oh my" lang-team#160

**Link:** https://github.com/rust-lang/lang-team/issues/160

### "Initiative: Ghost types and blocks" lang-team#161

**Link:** https://github.com/rust-lang/lang-team/issues/161

### "Keyword generics" lang-team#162

**Link:** https://github.com/rust-lang/lang-team/issues/162

### "Add const evaluatable `where const { <block> }`" lang-team#163

**Link:** https://github.com/rust-lang/lang-team/issues/163

### "#[repr(Interoperable_2024)]" lang-team#165

**Link:** https://github.com/rust-lang/lang-team/issues/165

### "add `#[never_call]` attribute" lang-team#170

**Link:** https://github.com/rust-lang/lang-team/issues/170



## PRs on the lang-team repo
### "Note design constraints on hypothetical `DynSized`" lang-team#166

**Link:** https://github.com/rust-lang/lang-team/pull/166

### "replace initiatives with a newer, lighterweight process" lang-team#173

**Link:** https://github.com/rust-lang/lang-team/pull/173

### "Document membership criteria and expectations" lang-team#174

**Link:** https://github.com/rust-lang/lang-team/pull/174



## RFCs waiting to be merged

None.




## Proposed FCPs

**Check your boxes!**
### "Stop using `Reveal::All` before borrowck" rust#101478

- **Link:** https://github.com/rust-lang/rust/pull/101478
- [**Tracking Comment**](https://github.com/rust-lang/rust/pull/101478#issuecomment-1249693297): 
	> Team member @lcnr has proposed to merge this. The next step is review by the rest of the tagged team members:
	> 
	> * [x] @compiler-errors
	> * [ ] @cramertj
	> * [ ] @jackh726
	> * [ ] @joshtriplett
	> * [x] @lcnr
	> * [ ] @nikomatsakis
	> * [ ] @oli-obk
	> * [ ] @pnkfelix
	> * [ ] @scottmcm
	> * [ ] @spastorino
	> 
	> No concerns currently listed.
	> 
	> Once a majority of reviewers approve (and at most 2 approvals are outstanding), this will enter its final comment period. If you spot a major issue that hasn't been raised at any point in this process, please speak up!
	> 
	> See [this document](https://github.com/rust-lang/rfcbot-rs/blob/master/README.md) for info about what commands tagged team members can give me.
- [**Initiating Comment**](https://github.com/rust-lang/rust/pull/101478#issuecomment-1249693270): 
	> I lowered some of the changes to a future compat lint which should hopefully allow us to merge this PR quickly without figuring out our exact goals for opaque types. 
	> 
	> ---
	> 
	> This pull request changes the compiler to stop using `Reveal::All` at any point until `borrowck` is completed. Using `Reveal::All` means that it is possible to inspect
	> - the underlying type of opaque types
	> - the value of a "defaulted" associated item (requires `feature(specialization)`)
	> 
	> In cases where this is known to break existing code, we retry the failing code with `Reveal::All`, emitting a future compatibility lint in case that succeeds. The cases where we emit a future compatibility lint are const evaluation for type system constants and `transmute` checking. **We currently do not retry for patterns, this is a breaking change, even if not discovered using crater.** See `in-ctfe/match-arm-exhaustive.rs` for an example. In case this causes an actual regression, we could extend the future compatibility lint to this area as well.
	> 
	> This PR is necessary as using `Reveal::All` before `borrowck` is incorrect and blocks work on implied bounds. Even if we want to fully embrace opaque types in these places - which is not trivial, see https://github.com/rust-lang/rust/pull/101478#issuecomment-1246610070, the approach then should not use `Reveal::All`. So even then, this PR will be "correct". I do not want to wait with my implied bounds work until this is figured out.
	> 
	>  
	> 
	> A crater run found 2 impacted crates:
	> 
	> ## [`mijit`](https://github.com/apt1002/mijit/)
	> 
	> Transmute in a test between an opaque type and the underlying concrete type. Opened PR fixing this in https://github.com/apt1002/mijit/pull/51.
	> 
	> ## [`name-it`](https://github.com/GoldsteinE/name-it)
	> 
	> This crate cannot really be fixed to avoid the lint, as it [fundamentally requires this pattern](https://github.com/GoldsteinE/name-it/blob/47544a062a950279fc49c8c3832b7b8e13febb8d/src/lib.rs#L104-L114):
	> ```rust
	> use std::mem;
	> fn returns_opaque() -> impl Sized {
	>     0u8
	> }
	> 
	> struct NamedOpaqueType {
	>     data: [mem::MaybeUninit\<u8\>; size_of_fut(returns_opaque)]
	> }
	> 
	> const fn size_of_fut\<FUT\>(x: fn() -> FUT) -> usize {
	>    mem::size_of::\<FUT\>()
	> }
	> ```
	> @rfcbot fcp merge


### "replace initiatives with a newer, lighterweight process" lang-team#173

- **Link:** https://github.com/rust-lang/lang-team/pull/173
- [**Tracking Comment**](https://github.com/rust-lang/lang-team/pull/173#issuecomment-1256506373): 
	> Team member @nikomatsakis has proposed to merge this. The next step is review by the rest of the tagged team members:
	> 
	> * [ ] @cramertj
	> * [ ] @joshtriplett
	> * [x] @nikomatsakis
	> * [ ] @pnkfelix
	> * [ ] @scottmcm
	> 
	> No concerns currently listed.
	> 
	> Once a majority of reviewers approve (and at most 2 approvals are outstanding), this will enter its final comment period. If you spot a major issue that hasn't been raised at any point in this process, please speak up!
	> 
	> See [this document](https://github.com/rust-lang/rfcbot-rs/blob/master/README.md) for info about what commands tagged team members can give me.
- [**Initiating Comment**](https://github.com/rust-lang/lang-team/pull/173#issuecomment-1256501398): 
	> @rfcbot fcp merge
	> 
	> I'd like to get folks to review this and agree to it. We discussed it some in the meeting, and I don't think I really deviated from that, but nonetheless.


### "Document membership criteria and expectations" lang-team#174

- **Link:** https://github.com/rust-lang/lang-team/pull/174
- [**Tracking Comment**](https://github.com/rust-lang/lang-team/pull/174#issuecomment-1256506230): 
	> Team member @nikomatsakis has proposed to merge this. The next step is review by the rest of the tagged team members:
	> 
	> * [ ] @cramertj
	> * [ ] @joshtriplett
	> * [x] @nikomatsakis
	> * [ ] @pnkfelix
	> * [ ] @scottmcm
	> 
	> No concerns currently listed.
	> 
	> Once a majority of reviewers approve (and at most 2 approvals are outstanding), this will enter its final comment period. If you spot a major issue that hasn't been raised at any point in this process, please speak up!
	> 
	> See [this document](https://github.com/rust-lang/rfcbot-rs/blob/master/README.md) for info about what commands tagged team members can give me.
- [**Initiating Comment**](https://github.com/rust-lang/lang-team/pull/174#issuecomment-1256499859): 
	> @rfcbot fcp merge
	> 
	> I think this is largely documenting existing practice, but I've also made a few changes (e.g., suggesting a brief write-up for a proposed new member). I'd like to get people to review and agree to the plans formally.




## Active FCPs
### "Tracking Issue for asm_sym" rust#93333

**Link:** https://github.com/rust-lang/rust/issues/93333

### "Allow transmutes between the same types after erasing lifetimes" rust#101520

**Link:** https://github.com/rust-lang/rust/pull/101520

### "Positional Associated Types" lang-team#126

**Link:** https://github.com/rust-lang/lang-team/issues/126

### "Interoperability With C++ Destruction Order" lang-team#135

**Link:** https://github.com/rust-lang/lang-team/issues/135

### "allow construction of non-exhaustive structs when using functional update syntax" lang-team#143

**Link:** https://github.com/rust-lang/lang-team/issues/143

### "Async fns in traits" lang-team#150

**Link:** https://github.com/rust-lang/lang-team/issues/150

### "Initiative: `?` traits, `try` blocks, `yeet` exprs, oh my" lang-team#160

**Link:** https://github.com/rust-lang/lang-team/issues/160

### "Initiative: Ghost types and blocks" lang-team#161

**Link:** https://github.com/rust-lang/lang-team/issues/161

### "Keyword generics" lang-team#162

**Link:** https://github.com/rust-lang/lang-team/issues/162



## P-critical issues

None.




## Nominated RFCs, PRs and issues discussed this meeting
 

### "Document `label_break_value` in the reference" reference#1263

**Link:** https://github.com/rust-lang/reference/pull/1263

* (discussed amongst Josh + Felix: suggested change is about content presentation, not about a semantic change to the language itself. It is also readily revertible. thus does not need to go through a "heavy-weight" process like FCP merge for approval.)

### "Constify `Location` methods" rust#101030

**Link:** https://github.com/rust-lang/rust/pull/101030

 * should we delegate decision here to wg-const?
 * scott's concern: seems like it might enable one to observe things like filename at compile-time.
 * josh: wouldn't that be blocked on nightly features?
 * dtolnay: there are other functions that may well become const fn that would enable that logic. Or just put  `#[track_caller]` on another const fn that you define.
 (specifically https://doc.rust-lang.org/nightly/std/panic/struct.Location.html#method.caller)
 * scott: The scary part for me is `const { assert!(location.line(), 10) };`
 * josh: sure, but you can shoot yourself in foot in lots of ways. Can already do that at runtime too.
 * scott: seems like different category, to have a runtime error vs a compile-time error.
 * felix: my concern is more about developers assuming these values will be "reasonable", when the original intention was that `Location` information is for debugging, and could well imagine swapping in dummy values for file names and line/column numbers.
 * josh: could indeed imagine a compilation mode that swaps in such dummy values...
 * josh: for short term: Would documenting that "these values are solely for debugging and should not be relied on for program logic", but without adding a mode to *enable* that (e.g. something that swaps in dummy values); would that suffice here? Or do we need the latter?
 * scott: the argument you can already shoot self in foot at runtime resonates. but I want to ask: when *do* you want to actually use this during const-eval? (in the sense of the specific values, rather than, say, a `{:?}` to show someone).
 * josh: presumably for embedding source info for developer feedback
 * josh: this feature strikes me as in the category as "both useful and easily misused"
 * lokathor: The docs themselves say that the location info is about the location of a panic, which is itself not entirely correct.
 * josh: Anyone up for doing some documentation work here? Scott?
 * scott: sure.
 * josh: does this deserve actual lang FCP, or just some lang review and libs-api FCP?
 * scott: this is const unstable, so we can just merge the PR, we don't need to go through those layers.

### "Constify `TypeId` ordering impls" rust#101698

**Link:** https://github.com/rust-lang/rust/pull/101698

 * josh: similar to aforementioned topic: Powerful and easily misused.
 * josh: What is motivation?
 * scott: people were trying to transmute TypeIds to u64 precisely to be able to sort them.
 * josh: ah
 * scott: note that the hashes and string names here are available at const-eval, though the memory addresess are not available.
 * scott: There are plans/ideas for changing representation of TypeId to, instead of being 128-bit, make it some inline hash bits plus a pointer to something out-of-line.
 * josh: that would definitely make *this* not work.
 * scott: compilation units can already have different memory addresses, which means this already needs to be a fast-accept for same address but *not* fast-reject for differing addresses.
 * scott: But you still cannot compare the address since its the address of a const...
 * scott: in any case you cannot e.g. rely on linker to intern all the possible objects here, so you *have* to handle the different addresses case via the expensive traversal.
 * josh: why isn't the 128 bits representation enough here?
 * (accidental vs adversarial questions re what is the threat model)
 * josh: how is this used? For debugging?
 * scott: its used in Any. Downcasting.
 * lokathor: on most Tier-1 platforms, 256 bits "isn't that bad"
 * josh: Const-ifying this one is different from Location, because doing so may close off implementation possibilities (w.r.t. the aforementioned use of memory addresses for fast accept)
 * scott: Another concern I have here is what *is* "ordering" supposed to mean. E.g. can one assume that the type-id of `u32` will always be less than the type-id of `u64`? Or is this just enough to order for a data structure.
 * josh: Even ignoring Ord and just look at Eq, two types that are the same must have equal TypeId, and type types that are not the same must have non-equal TypeId.
 * josh: That seems like relevant decision criteria here.
 * josh: namely it will outlaw certain implementation options
 * felix: does it? What if we had a way to specialize on `const`/non-`const`, and leveraged that to provide differing implementations depending on which context we are in?
 * scott/josh: We ... could ... (even going so far as to have differing structural representations at const-eval vs runtime)
 * josh: scary though
 * josh: or we could have TypeId be the thing that is committed to the variant that could not be optimally designed for runtime performance, and people would have to add a separate FastTypeId that you could always convert TypeId to.
 * josh/scott: This is const unstable, so we could for now say that that we are approving this, while also updating the tracking issue with a unresolved question describing the potential one-way door here that rules out certain TypeId implementations.
 * josh: does anybody feel like this should not go in, even unstable? or should the associated feature gate be marked as "experimental"?
 * scott: probably not. and I don't know if you can do that for library features anyway.
 * josh: can anyone follow up with a comment to the above effect?
 * scott: I can do that, it aligned with the other thing I'm following up on.
 * ...
 * scott: related note, do we have any plan in place for avoiding having to re-derive every trait in the standard library?
 * josh: there's some idea of doing an opt-in e.g. `#[derive(PartialEq(const))]` or `#[derive(const PartialEq)]` that would provide the const-derived form.
 * scott: so someone should try to propose a policy such that we stop ripping out the derives (replacing them with ugly manual impls that have the constified forms), and instead work on providing whatever the above feature is..
 * lokathor: I can help! (e.g. proposing something)
 * scott: ideally it would follow Niko's new proposal that is meant to make initiatives lighter weight.
 * lokathor: BTW, is there an inclination one way or the other re the bikeshed `#[derive(const Trait)]` vs `#[derive(Trait(const)]`
 * (people provide reasons in both directions, and Felix thinks he hears scott suggest a third alternative, `#[const_derive(Trait, ...)]` ...! (to avoid repeating the word "derive" ad nauseum for N traits))
 * [Zulip Thread](https://rust-lang.zulipchat.com/#narrow/stream/213817-t-lang/topic/Request.20For.20Bikeshed.3A.20Deriving.20Const.20Trait.20Impls) for discussion

### "make const_err a hard error" rust#102091

**Link:** https://github.com/rust-lang/rust/pull/102091

* josh: advantage of switching this from future-incompat lint to hard error, is that it will enable removing the effectvely dead-code.
* josh: definitely needs an FCP, as well as a crater run (where the latter has already been done and reviewed by RalfJ).
* josh: any objection to starting FCP and handling rest asynchronously?
* scott: go for it.
* josh: okay. done. Boxes ready for checking.


### "Inference failure with `type_changing_struct_update`" rust#101970

**Link:** https://github.com/rust-lang/rust/issues/101970

* scott: This is a "we need a plan" kind of thing. We cannot cover it in remainder of this meeting.

## Nominated RFCs, PRs and issues NOT discussed this meeting
### "impl DispatchFromDyn for Cell and UnsafeCell" rust#97373

**Link:** https://github.com/rust-lang/rust/pull/97373


### "Introduce a no-op FakeRead for `let _ =`." rust#102256

**Link:** https://github.com/rust-lang/rust/pull/102256

