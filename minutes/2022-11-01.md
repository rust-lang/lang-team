---
title: Triage meeting 2022-11-01
tags: triage-meeting
---

# T-lang meeting agenda

* Meeting date: 2022-11-01

## Attendance

* Team members: Josh, Niko, Felix
* Others: Mark, dtolnay, Nilstrieb, y86-dev, Tyler Mandry, Vincent

## Meeting roles

* Action item scribe: Mark
* Note-taker: Mark

## Scheduled meetings
- No pending proposals this time.

- Design meeting -- planning meeting tomorrow (2022-11-02).

## Announcements or custom items

(Meeting attendees, feel free to add items here!)

## Action item review

* [Action items list](https://hackmd.io/gstfhtXYTHa3Jv-P_2RK7A)

## Pending lang team project proposals

None.




## PRs on the lang-team repo
### "Fix link and newlines in meeting proposal template." lang-team#180

**Link:** https://github.com/rust-lang/lang-team/pull/180



## RFCs waiting to be merged

None.




## Proposed FCPs

**Check your boxes!**
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




## Active FCPs
### "Short Macro Invocation Syntax: m!123 and m!"abc"" rfcs#3267

**Link:** https://github.com/rust-lang/rfcs/pull/3267

### "Support upcasting of `dyn Trait` values" rfcs#3324

**Link:** https://github.com/rust-lang/rfcs/pull/3324

### "Add lang-team advisors team" rfcs#3327

**Link:** https://github.com/rust-lang/rfcs/pull/3327

### "Use `token::Lit` in `ast::ExprKind::Lit`." rust#102944

**Link:** https://github.com/rust-lang/rust/pull/102944



## P-critical issues

None.

## Nominated RFCs, PRs and issues discussed this meeting

### "Parse error recovery is obversable by macros in several cases" rust#103534

**Link:** https://github.com/rust-lang/rust/issues/103534

- Nilstrieb: originally a compiler issue. Macro matching can vary depending on if parser does recovery.
- https://github.com/rust-lang/rust/issues/103534#issuecomment-1291035057 has two code blocks that should (or should not) compile

```rust
macro_rules! blah {
    ($expr:expr) => {};
    (not $expr:expr) => {};
}

fn main() {
    blah!(not 1);
}
```

* `not 1` is "recovery" parsed as a 'broken' expr "token".

```rust=
macro_rules! what {
    ($e:expr) => { compile_error!("no") };
    ($($tt:tt)*) => {};
}

fn wh() {
    what! {
        match 1;
    }
}
```

```rust=
macro_rules! x {
    (a b) => {};
    (a c) => {};
}

x! { a c }
```

* Neither of these code blocks compile today.
* Question: Should these work?
    * Each arm is matched until it fails, and then the next one is tried
    * However, if the arm has a NT (e.g., expr), then whether this arm is used is based only on the first token being in the 'start set' of the NT.
        * If that fails, then the whole macro invocation fails to compile -- no fallback to next arm.
* Niko: originally, one of the alternative design was to conservatively 'expand' the scope of NTs to some rough grammar, but then actually parse with current expr
    * Probably can't adopt this today. But maybe we should be taking an opportunity to expose less of the grammar...
* Josh: Can we do something partial, incremental today? Like not applying parser recovery here?
* Niko: I *do* think that the first macro (with not $expr) shouldn't work in the fullness of time (i.e., that's the kind of thing that exposes the details of our expr grammar)
    * But I agree we should probably avoid parser recovery now anyway. That's how the system works now...
* Josh: In the past we did pat_or to extend here. Maybe we fix this long-term the same way.
* Josh: Do we need an FCP to determine that error recovery shouldn't play a role here?
    * Seems like the answer is 'no', this (error recovery playing a role) is just a compiler bug.
* Felix: Is the feedback that we want both to work right? Long-term Niko at least seems like we want something different.
* Niko: I think this is a bug relative to the current behavior; lang would be open to RFCs/proposals/exploratory work changing the status quo (over e.g. edition boundaries).

### "Unreserve braced enum variants in value namespace" rust#103578

**Link:** https://github.com/rust-lang/rust/pull/103578

* https://github.com/rust-lang/rust/pull/103578#issuecomment-1292594900
* Seems like no current plans
* Josh: Seems like we could have some kind of mem::discriminant on 'unfilled' enums, rather than requiring dummy values
* Josh: We would need an edition to get this back, but seems fine.
* Starting an FCP merge.

### "update aliasing rules section of the reference" reference#1290

**Link:** https://github.com/rust-lang/reference/pull/1290

* Niko: A function with &T argument can assume the &T starts and stays valid during its execution
* Josh: But you can temporarily make it dangling?
    * Niko: No, I don't think so. You might be able to 'uninitialize' the memory it points to?

> For references, the liveness duration is upper-bounded by the syntactic lifetime assigned by the borrow checker.

* Josh: Not sure what this means.
* Niko: I think this means that the reference isn't living *longer* than what the borrow checker dictates. Something like "replacing all pointers with references" is *good enough* (if it works). It not working is not necessarily indicative of problems though.
* Niko: I think the goal here is to give minimal area rather than an actual definiton we *want*.

* Josh: https://github.com/rust-lang/rfcs/pull/3336, seems like should be considered together?
* Mark: No, seems like these are very separable. The reference PR documents what is *true* today.
* Josh: It seems like we ought to not document something as UB unless we're *also* providing an alternative.
* Josh: Suppose:
    * Would it be legal to take a &mut, turn it into a *mut, free it, reallocate something at the same address, and then use the &mut?
    * Josh: It seems like the letter of the reference says this is illegal, but I think this "should" be fine.
    * Nilstrieb: Stacked borrows says this is illegal today.
* Niko: In async context, we want to reuse stack memory after moving out, so I do think we should draw some lines. Maybe can relax later.
* Josh: I think you're saying that we should write these rules and then I can write an RFC to avoid giving these hints.
* Josh: I believe people are making things temporarily invalid today?
    * What does *dangling* mean?
    * It's not well-defined here.
* Mark: It seems like if we ask for an exact definition we get a 100 page thesis.
* Niko: I agree with Mark that right line is hard to find, but seems like we ought to be able to give some guidance.
    * For example, LLVM has a fairly specific definition.
    * At least can note this is a point of confusion.
* Josh: No objections to documenting current behavior (and considering future extensions)
* y86: Note that dangling is defined in the doc (see rendered link).


* Felix: Josh, what is the layout/visiblity/details of the T in &mut T? When people are doing this?
    * Josh: e.g. &mut [u8]
* Felix: Asking to avoid being unable to fulfill between Niko's async example and this case.
* Josh: .... something about moving these in different directions ...

* Niko: Would like some consensus on what we do with this PR
    * Forward-ref to the dangling definiton
    * ...
* Josh: If we're just documenting the current state and we can change it in the future, then seems OK.
* rfcbot merge planned (to be started by Niko)

### "Restrictions" rfcs#3323

**Link:** https://github.com/rust-lang/rfcs/pull/3323

* Will discuss in a bit...
* Jacob won't join us today.
* Josh: Proposal gives us sealed traits and read-only fields.
    * Is there anything we can do better?
    * `pub mut(self) field: type` -- this is publicly read-only
    * this is relatively verbose, so maybe
* Mark: Do we really want to answer this question *now*?
    * Josh: It seems like we shouldn't treat *this* RFC as experimental, since the work isn't really in surface syntax.
* Niko: "Let's land this, but let's still dive into some more experimentation with syntax".
    * Not really clear what the experimentation looks like.
    * Don't really like *this* syntax, but not sure I have alternatives either at this time.

* Josh to propose fcp, can sort out via concerns/comments.

## Nominated RFCs, PRs and issues NOT discussed this meeting
### "RFC: Field projection" rfcs#3318

**Link:** https://github.com/rust-lang/rfcs/pull/3318

### "impl DispatchFromDyn for Cell and UnsafeCell" rust#97373

**Link:** https://github.com/rust-lang/rust/pull/97373

### "PhantomData: fix documentation wrt interaction with dropck" rust#103413

**Link:** https://github.com/rust-lang/rust/pull/103413


