---
title: Triage meeting 2022-10-04
tags: triage-meeting
---

# T-lang meeting agenda

* Meeting date: 2022-10-04

## Attendance

* Team members: nikomatsakis, joshtriplett, scottmcm, pnkfelix
* Others: simulacrum, Lokathor, dtolnay, y86-dev, tmandry

## Meeting roles

* Action item scribe: simulacrum
* Note-taker: nikomatsakis

## Announcements or custom items

### Merge process

nikomatsakis: I merged the process PR but opened a PR going back to the [liaison terminology](https://github.com/rust-lang/lang-team/pull/176).

I also started going through pending proposals and closing them and encouraging folks to consider experiments where appropriate.

## Action item review

* [Action items list](https://hackmd.io/gstfhtXYTHa3Jv-P_2RK7A)

## Pending lang team project proposals
### "Support platforms with size_t != uintptr_t" lang-team#125

**Link:** https://github.com/rust-lang/lang-team/issues/125

joshtriplett: would like to see this unblocked (and help)

### "Interoperability With C++ Destruction Order" lang-team#135

**Link:** https://github.com/rust-lang/lang-team/issues/135

### "allow construction of non-exhaustive structs when using functional update syntax" lang-team#143

**Link:** https://github.com/rust-lang/lang-team/issues/143

### "Initiative: `?` traits, `try` blocks, `yeet` exprs, oh my" lang-team#160

**Link:** https://github.com/rust-lang/lang-team/issues/160

### "Add const evaluatable `where const { <block> }`" lang-team#163

**Link:** https://github.com/rust-lang/lang-team/issues/163

### "#[repr(Interoperable_2024)]" lang-team#165

**Link:** https://github.com/rust-lang/lang-team/issues/165

joshtriplett: would like to see this as an experiment

### "add `#[never_call]` attribute" lang-team#170

**Link:** https://github.com/rust-lang/lang-team/issues/170



## PRs on the lang-team repo
### "Note design constraints on hypothetical `DynSized`" lang-team#166

**Link:** https://github.com/rust-lang/lang-team/pull/166

nikomatsakis still on the hook for this

### "Document membership criteria and expectations" lang-team#174

**Link:** https://github.com/rust-lang/lang-team/pull/174

nikomatsakis: rebased this, before it had process changes mixed in

### "s/champion/liaison/" lang-team#176

**Link:** https://github.com/rust-lang/lang-team/pull/176



## RFCs waiting to be merged

None.




## Proposed FCPs

**Check your boxes!**
### "Document membership criteria and expectations" lang-team#174

- **Link:** https://github.com/rust-lang/lang-team/pull/174
- [**Tracking Comment**](https://github.com/rust-lang/lang-team/pull/174#issuecomment-1256506230): 
	> Team member @nikomatsakis has proposed to merge this. The next step is review by the rest of the tagged team members:
	> 
	> * [x] @cramertj
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
### "make const_err a hard error" rust#102091

**Link:** https://github.com/rust-lang/rust/pull/102091

### "Interoperability With C++ Destruction Order" lang-team#135

**Link:** https://github.com/rust-lang/lang-team/issues/135

nikomatsakis: inclined to close, can reach out to cramertj, but I don't think we have an experienced team member to drive it

pnkfelix: wait, work is being done? there is a tracking issue.

nikomatsakis: oh, didn't realize that. maybe we'll just link to the tracking issue for this one.

### "allow construction of non-exhaustive structs when using functional update syntax" lang-team#143

**Link:** https://github.com/rust-lang/lang-team/issues/143

joshtriplett: started out trying to liaise this but found that I did not have time.

### "Initiative: `?` traits, `try` blocks, `yeet` exprs, oh my" lang-team#160

**Link:** https://github.com/rust-lang/lang-team/issues/160



## P-critical issues

None.

## Nominated RFCs, PRs and issues discussed this meeting

### "Introduce a no-op FakeRead for `let _ =`." rust#102256

**Link:** https://github.com/rust-lang/rust/pull/102256

- This is introducing an op in MIR to detect things like unsafe accesses
- Should this be handled in MIR or THIR? Isn't unsafety checking moving to THIR?
- nikomatsakis: I am nervous about this, I think we calibrated this carefully...

compiles today...

```rust
let mut x = 3;
let y = &mut x;
let _ = x;
drop(y);
```

...as does this (which is why we put the rules how they are)...

```rust
let mut x = 3;
let y = &mut x;
|| { let _ = x; };
drop(y);
```

joshtriplett: we did decide that we wanted `let _ = x` to be unsafe though, right?

pnkfelix: it's possible that, the way this is implemented, this is a different kind of fake-read?

scottmcm: I think we decided that some analyses were based on dataflow and some were lexical scope. Variable initialization is dataflow not scope. But unsafe is a block and makes more sense as lexical scope, so doing it on MIR was not a good fit.

pnkfelix: This seems like it solves a real problem, do we really want to delay this waiting for THIR work?

nikomatsakis: This doesn't seem like our call, this is a compiler team implementation question.

nikomatsakis: We should specify requirements we care about, and then ask the compiler team to meet those. Proposed requirements

* if it doesn't disrupt a borrow, closure doesn't capture (as above)
* if it's unsafe in one, it's unsafe in the other

pnkfelix: I'll leave a omment (ACTION ITEM)

### "Constify `Location` methods" rust#101030

**Link:** https://github.com/rust-lang/rust/pull/101030

scottmcm: I was supposed to write a comment and have un-nominated this.

### "make const_err a hard error" rust#102091

**Link:** https://github.com/rust-lang/rust/pull/102091

FCP already started previously, nomination dropped.

### "impl DispatchFromDyn for Cell and UnsafeCell" rust#97373

**Link:** https://github.com/rust-lang/rust/pull/97373

Need documentation comment in source code, not comment on github. Niko to un-nominate and write clarifying request.

### "Stabilize `half_open_range_patterns`" rust#102275

**Link:** https://github.com/rust-lang/rust/pull/102275

[ehuss points out](https://github.com/rust-lang/rust/issues/67264#issuecomment-1260072214) that this gates two kinds of patterns:

* `..=X`
* `X..`

and points out that `X..` was kept unstable to potentially allow for this kind of thing, as documented [in this test case](https://github.com/rust-lang/rust/blob/master/src/test/ui/half-open-range-patterns/slice_pattern_syntax_problem0.rs)...

```rust
    let [first_three @ ..3, rest @ 2..] = xs;
```

because we currently permit

```rust
    let [a, b, c, rest @ ..] = xs;
```

scottmcm: I don't recall us discussing this slice pattern thing at all. The easy answer is "ok let's keep `X..` gated for now".

joshtriplett: I agree, that seems like something we need to settle at some point rather than kicking the can down the road, but for right now, we did an FCP, and it wasn't intended to cover slice patterns, just the range context.  We should go ahead and make it clear that the FCP was covering "half open range patterns" and not "slice patterns".

nikomatsakis: :thumbsup: 

joshtriplett: I can write a comment to that effect (ACTION ITEM).

### "`as` cast of non-`Copy` enum is no longer a move" rust#102389

**Link:** https://github.com/rust-lang/rust/issues/102389

joshtriplett: this sounds familiar, didn't we just make a change...?

scottmcm: yes we did, noted in the issue, but the notes on that talk a lot about the drop behavior. We were intentionally changing the drop behavior. This is the weird case of a non-copy fieldless enum. Used to be a move, now it's a copy of a type that's not copy.

```rust
pub enum Enum { A, B, C }
pub fn func(inbounds: &Enum, array: &[i16; 3]) -> i16 {
    array[*inbounds as usize]
}
```

scottmcm: was this intentional?

joshtriplett: the change we intentionally made was to say you could no longer as cast an enum that had a drop trait

nikomatsakis: doesn't feel consistent to me. saying you can't cast if it has drop. 

joshtriplett: sounds like more than one thing may have been regressed by [this PR](https://github.com/rust-lang/rust/pull/96862), e.g., [#102303](https://github.com/rust-lang/rust/issues/102303)

scottmcm: it was the conversation on #102303 that caused me to notice this was a thing

joshtriplett: do we agree this is a bug?

nikomatsakis: I think it's a bug

joshtriplett: presumably if this is not copy, you did that on purpose, and you want the compiler to catch cases where you treat it as it were not copy. 

nikomatsakis: there's a perspective where this is valid, e.g., it's like calling [`discriminant`](https://doc.rust-lang.org/std/mem/fn.discriminant.html) on this.

joshtriplett: there is time sensitivity on this. It's not P-critical, but code will start compiling.

pnkfelix: def'n of P-critical I usually use is "is this a release blocker"

scottmcm: too late to be a release blocker

pnkfelix: relevant question is "Would this motivate a point release".

joshtriplett: If we could fix this, I would say yes

pnkfelix: I think the way to view stable->stable regr is, if it motivates a point release, it should be P-critical.

joshtriplett: If we had broken working code, we'd call it P-criticial, if we've gone the other direction and are accepting things we shouldn't, it's not wildly less priority. Would anyone object to being P-critical?

scottmcm: I think once it's shipped, the bar does change a bit. Is it important to get this fix out 3 weeks earlier than if we do something and backport to beta?

pnkfelix: I think the most imp't thing is that there is a fix for 1.65.

nikomatsakis: if we're going to fix it, now's the time to do it, before there's stuff relying on this.

joshtriplett: posted a comment, marked P-critical

### "Document `label_break_value` in the reference" reference#1263

**Link:** https://github.com/rust-lang/reference/pull/1263

joshtriplett: pnkfelix, you pushed an updated change, are you happy with it now?

pnkfelix: yes but github is being mean to me

*nikomatsakis and joshtriplett approve*

*pnkfelix pushes the big green button*

### "Elaborate supertrait bounds when triggering `unused_must_use` on `impl Trait`" rust#102287

**Link:** https://github.com/rust-lang/rust/pull/102287

nikomatsakis: today, an `impl Iterator` will warn, but `impl ExactSizeIterator` will not. 

scottmcm: I don't think this makes sense for "every type that implements the trait".

nikomatsakis: if the type implements `ExactSizeIterator`, it already implements `Iterator`, so it would already be warning, elaborating supertraits wouldn't affect that.

Some examples to clarify:

```rust
#[must_use]
trait Foo { }

trait Bar { }
impl<T: Foo> Bar for T { }
```

`-> impl Bar` would not warn

```rust
#[must_use]
trait Foo { }

trait Bar<T: Foo> { }
```

neither would `-> impl Bar<X>`

joshtriplett: Seems like a very safe `#[must_use]` extension, compared with some of the more complex ones we've considered in the past. This is an "is-a" relationship. Only case I can think of is if you had some magic type that was iterator but you had some other impl semantics--

nikomatsakis: --but then you wouldn't use impl Trait, the whole point of that is to not rely on the precise type.

scottmcm: what about dyn trait??

nikomatsakis: I'll leave a comment.


### "Inference failure with `type_changing_struct_update`" rust#101970

**Link:** https://github.com/rust-lang/rust/issues/101970

scottmcm: No real plan to move forward yet. People are trying to figure out what to do. We have a change that breaks a lot of inference right now. Obvious questions

* We're allowed to make inference breakage, would lang team be up for it?

If answer is no, then do we want to do an edition? Don't know how funky that is. Or do we want to try and do some bigger change, adding more syntax, something I don't know.

joshtriplett: I see in the opening comment the example of `Default::default`, sounds like this would break the case where you have a generic parameter that was previously determined, but now isn't. 

scottmcm: right

joshtriplett: Obvious question is...this would be pretty widespread breakage...I don't think we want to special-case that in its current form (though we may want some other special defaulting form). Otherwise question is, would we choose to do this over an edition?

scottmcm: Part of why I was exploring with other syntax etc (see thread), if we add something that better handles the `Default::default` case, then making this change over an edition is more feasible. We can say "we'll migrate the old things to this new thing". If we get that migration 95%-99% accurate, great. But if we have to force it in some strange way then because there isn't a good answer, I'm not sure I'd want to do it over the edition.

nikomatsakis: I agree, I wouldn't want to do it on its own, it's too much breakage and the pattern is too useful.

scottmcm: We can't even just change it to `Foo::default()`, you'd have to do `<Foo<T>>::default()` to set the right type parameters.

scottmcm: We have the broad conversation, esteban's RFC that I'm hoping we get about struct field default values, as well as the recurring conversation about FRU using a desugaring that's not what a lot of people expect, if we had an option to "opt-in to the other desugaring", it always works, because `Default::default` isn't a place, so it's doesn't have the breaking problem.

nikomatsakis: the other desugaring being?

scottmcm: (using English words)

```rust!
{
    let mut x = base;
    x.f1 = ...;
    x.f2 = ...;
    x
}
```

joshtriplett: Sounds like nobody has appetite for doing this without an edition. Even doing it over an edition, we need a migration for what people should do. So the questions are

* is there anything other than Default which needs a migration path
* do we have an alternate syntax for Default?

scottmcm: I think Default might be "not everything" because even `Foo::new()` would have this problem, if `Foo` had type parameters.

scottmcm summarizes 

* not willing to take this breakage
* need a plan to make migration path reasonable over an edition
    * and don't consider `<Foo<T, U, V>>::default()` as the right path

## Nominated RFCs, PRs and issues NOT discussed this meeting
### "RFC: Field projection" rfcs#3318

**Link:** https://github.com/rust-lang/rfcs/pull/3318

