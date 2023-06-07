---
title: Triage meeting 2022-10-18
tags: triage-meeting
---

# T-lang meeting agenda

* Meeting date: 2022-10-18

## Attendance

* Team members: Niko, Josh, Felix
* Others: simulacrum, Tyler, y86-dev, 

## Meeting roles

* Action item scribe: simulacrum
* Note-taker: nikomatsakis

## Scheduled meetings
- No pending proposals this time.

## Announcements or custom items

(Meeting attendees, feel free to add items here!)

### Any interest in joining (or having someone you know/work with join) a safe-ABI effort?

joshtriplett: Hoping to put together a draft for the motivation of a "Safe ABI" RFC in the next couple of weeks. If there's anyone on the team or folks you are working with who you'd be excited to point my way, let me know.

pnkfelix: "Safe ABI" or "stable" API?

joshtriplett: A bit of both. C ABI is stable. Looking for something a bit more capable than repr(C). Right now, the "common capabilities" you can get if you want to call from a shared library are "repr(C)" -- you can use C FFI. I'm looking to provide something more capable, so it can represent slices of given types, counted strings, etc. Basic extensions, what you should be able to pass between two safe languages without having to go by way of C.

Goal is for that to eventually become a new option for what you can export as the symbols from a library. It will take a while to get to that point but I hope we can get there eventually.

tmandry: Would you consider it in scope to be able to build stdlib this way eventually?

joshtriplett: yes, but not for first incarnation. I'm eventually hoping that, for example, trait objects could have a standard ABI, and that you could extract a trait object from a concrete type -- i.e., be able to pass around a `&dyn HashMap<K, V>`. You can't do a HashMap directly (that implies a representation), but you could pass around sonething that has same API and let people call into it.

## Action item review

* [Action items list](https://hackmd.io/gstfhtXYTHa3Jv-P_2RK7A)

## Pending lang team project proposals

None.




## PRs on the lang-team repo

None.




## RFCs waiting to be merged

None.




## Proposed FCPs

**Check your boxes!**
### "Support upcasting of `dyn Trait` values" rfcs#3324

- **Link:** https://github.com/rust-lang/rfcs/pull/3324
- [**Tracking Comment**](https://github.com/rust-lang/rfcs/pull/3324#issuecomment-1275115366): 
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
- [**Initiating Comment**](https://github.com/rust-lang/rfcs/pull/3324#issuecomment-1275115340): 
	> Discussing in the lang-team meeting. This proposal has been under discussion for some time, and is always something we expected to work, so even though the RFC was only recently opened, I am going to move to merge...
	> 
	> @rfcbot fcp merge
	> 
	> 

No sync discussion needed.

### "Add lang-team advisors team" rfcs#3327

- **Link:** https://github.com/rust-lang/rfcs/pull/3327
- [**Tracking Comment**](https://github.com/rust-lang/rfcs/pull/3327#issuecomment-1276536749): 
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
- [**Initiating Comment**](https://github.com/rust-lang/rfcs/pull/3327#issuecomment-1276536728): 
	> I'm going to go ahead and kick off the rfcbot fcp merge on this one. It's been [discussed on Zulip](https://rust-lang.zulipchat.com/#narrow/stream/213817-t-lang/topic/Draft.20lang.20team.20advisors.20rfc/near/301738882) and is largely procedural. 
	> 
	> @rfcbot fcp merge

No sync discussion needed.


## Active FCPs
### "make unaligned_reference a hard error" rust#102513

**Link:** https://github.com/rust-lang/rust/pull/102513

no comments or updates

### "Stabilize raw-dylib for non-x86" rust#102793

**Link:** https://github.com/rust-lang/rust/issues/102793

no comments or updates

## P-critical issues

None.




## Nominated RFCs, PRs and issues discussed this meeting

(none yet, move things from the section below as they are discussed)

### "Introduce a no-op FakeRead for `let _ =`." rust#102256

**Link:** https://github.com/rust-lang/rust/pull/102256

nikomatsakis: We had some discussion on this on Zulip. I'm not fully caught up. Seems like we may want to make a change for consistency.

What's being discussed was making the value at `<expr>` have to be accessible, regardless of the pattern, which is what matches do:

```rust
let mut <pat> = <expr>
```

joshtriplett: Does this affect when destructors run?

nikomatsakis: No, this is more of a language lawyer question.

joshtriplett: Would never affect runtime semantics?

(conversation after this point may be invalidated by XYZ below)
pnkfelix: Might effect if you can observe when memory accesses happen.

nikomatsakis: Could affect what's UB.

joshtriplett: Previously if you did `let _ = <memory-access>`, the memory access might not ever happen?

nikomatsakis: this is a total no-op today, but the proposal would cause a memory load:

```rust
let x: *const *const u32 = ...;
let _ = **x;
```

joshtriplett: Optimizer might cause this to have no effect, but it wouldn't be guaranteed anymore?

nikomatsakis: For safe references, for example, could optimize it out. Perhaps if you knew provenance of raw pointer. Unobservable to you as user.

pnkfelix: Part of what's confusing is that you have people like Ralf who no longer care about distinction between raw pointer and safe references. When Niko says "safe references you can optimize out".

nikomatsakis: You can optimize that out because if they were invalid references, it'd be UB, but they're still significant.

scottmcm: I think `_` doing a load is not what we want, although I agree it might be sort of the semantics of the PR, it's more that we want the place construction to happen without ever actually doing a load from the place. 

(XYZ; above may not be accurate) nikomatsakis: I think it wouldn't actually do a load, the way the PR is implemented, more like it's UB if a load wouldn't have been legal.

pnkfelix: is this observable? if you're allowed to do a memory load, why not say compiler could inject one?

nikomatsakis: this doesn't compile under the PR, also affects safe code. Even though you're not accessing all of `x`, all of `x` must be accessible:

```rust
let mut x = (String::new(), String::new());
drop(x.1);
let (a, _) = x;
```

The read doesn't come from the `_`, it comes from the RHS of the let, much like in a match:

```rust
let mut x = (String::new(), String::new());
drop(x.1);
match x {
    _ => 
}
```

we do that because we want this to be UB, and there are no patterns:

```rust
let mut x: ! = return;
match x {
}
```

`if let Some(_) = x` definitely isn't a read of the payload.

scottmcm: some of `_` not moving is a feature, e.g., `x.1` should still be accessible:

```rust
let mut x = (String::new(), String::new());
let (a, _) = x;
```

nikomatsakis: probably too deep for this triage meeting

scottmcm: wasn't this a discriminant read, not a fake read?

nikomatsakis: not sure, we should investigate

joshtriplett: sounds like there is general sentiment in favor of a fake-read mechanism, just a question of do we have the semantics?

scottmcm: I think FakeRead is the wrong representation. I think considering it a read is incorrect. There's a version of this PR where there's a MIR statement that's like "place mention" that's not a read in the borrow checking sense. The thing that's breaking is that it's breaking code for more than just "oh now we can see the ptr is null or there was a deref so it needs unsafe". Those breaking changes are ok, the changes that affect how `_` works in safe code I think is not what we want. 

joshtriplett: Agreed. It doesn't sound like you're advocating a change to the semantics of the PR, you're advocating renaming FakeRead?

scottmcm: I think I am advocating for a change to the behavior. Right now the PR is a breaking change.

pnkfelix: Most recent comments by cjgillot said "this version should not break borrow check anymore". I don't know if the cases above (193-195) breaks today under the PR (e.g., `let (a, _) = x`).

scottmcm: It's not clear to me that having a fake-read is right. Looks like current semantics might be what I expected.

joshtriplett: I was trying to advocate 

nikomatsakis: I want a good description of what PR does that is not defined by the code. I would like to know why `**p` would be UB but we accept `let (a, _) = x`. 

pnkfelix: my bet is that it falls out from the changes to dataflow

nikomatsakis: hoping for something less operational and more "how do I think about it"

joshtriplett: I hear two things--

* scottmcm's renaming
* niko's wanting of docs for net effect of how this works from language POV

joshtriplett: can we leave that feedback in the form of two comments
nikomatsakis: Yes
scottmcm: Yes -- Done at <https://github.com/rust-lang/rust/pull/102256/files#r998609886>


### "Add `cstr!` macro." rust#101607

**Link:** https://github.com/rust-lang/rust/pull/101607

> This adds a `cstr!` macro for easily creating `&'static Cstr`

Lang team question:

Should we add an actual feature instead of a macro?

We left room for it in the last edition.

> Nominating for lang team discussion, to discuss if we should add something like `c"abc"` to the language instead of adding a macro to the library.

pnkfelix: Motivation besides checking? Are there things that can only be provided at the lang level that a proc macro can't provide?

nikomatsakis: really short syntax?

scottmcm: shortist syntax is literals that can be coerced into multiple types.

pnkfelix: short syntax feels like it could be addressed by https://github.com/rust-lang/rfcs/pull/3267 e.g. `c!"foo"` if we named the macro `c`.

nikomatsakis: why *not* add it in though?

joshtriplett: if we think it's important enough to have that mechanism, shouldn't we just build it in?

nikomatsakis: it's general purpose?

joshtriplett: I don't think that mechanism should be used for general purpose.

nikomatsakis: summary seems to be some of us are open to adding this to lang, but some are skeptical?

joshtriplett: "might happen in future but possibly not quick enough"

pnkfelix: my opinion is if short macro were to go through, it seems obvious to me that this is better than c literals, but if there are objections to that, then indeed the macro forms don't seem all that palatable compared to native syntax.

joshtriplett: ...I thought we had FCP closed this... I remember us bringing this up in a libs meeting and it seemed like there wasn't much appetite for this. If sentiment has changed on that, maybe should discuss. I thought I was reflecting back whta we discussed overall many months ago.

simulacrum: is there a reason this has to be a macro and not a const fn? It's at least half implemented with one? Reason may be that you can't introduce a new compile-time allocation with a `\0` at the end.

joshtriplett: if what you want to do is make a static `Cstr` that this is not trivially done...

nikomatsakis: ...wait what is a `Cstr`? oh, that changes my opinion a little bit. I forgot we were not talking about a builtin type.

scottmcm: it's a weird DST.

simulacrum: there is a "from bytes with null" function.

joshtriplett: it also returns a result...you'd ideally get a compile error...

simulacrum: ...you would, if you call it from a const fn?

joshtriplett: you're proposing make this constable, have that fn statically concat a null on the end..?

simulacrum: I think concat is that the part we can't do. I claim writing `\0` is not that hard.

nikomatsakis: it seems annoying. I'd be annoyed if I had to write `\0` a lot.

scottmcm: from lang side of the decision, it sounds like there is no near term plan that "yeah we totally add syntax for this". if libs-api wants to have a macro, no lang concern about that.

joshtriplett: don't think we should block it, if we add a syntax, macro doesn't break.

nikomatsakis: I'm excited to find opportunities to make interfacing rust with C more ergonomic and easy, but I'm not sure about `CStr`, it doesn't seem so canonical to me?

scottmcm: I feel like you want the thing that's actually thin, right?

consensus: add the macro and see how popular it is.

scottmcm: I put a comment, I still think "type inferred literals" is the way forward for the nice APIs. If we could just have the normal string literal...

joshtriplett: ...seems like a longer conversation. Not disagreeing that would produce a really nice syntax but skeptical it won't break the universe.

nikomatsakis: one thought is that the way you specify a type inferred literal might be a trait with a const fn, so that's an interesting route to explore

scottmcm: I think it would only be a problem for places inferred to take multiple types? (referring to literal inference)

### "Use `token::Lit` in `ast::ExprKind::Lit`." rust#102944

**Link:** https://github.com/rust-lang/rust/pull/102944

https://github.com/rust-lang/rust/pull/102944#issuecomment-1277476773

nikomatsakis: I feel like this was discussed quite a bit

(Subsequent note: this comment was mistakenly confusing prefixes like `xyz#` with suffixes)
joshtriplett: Yes, we intentionally left the space so that macros can't use it for their own purposes, making is so that we can't use it in the future.

nikomatsakis: I'm not inclined to reopen the discussion given that it was highly delberated. 

https://rust-lang.github.io/rfcs/3101-reserved_prefixes.html

text from RFC (emphasis mine)

> Unless a prefix has been assigned a specific meaning by the language (e.g. `r#async`, `b"foo"`), Rust will **fail to tokenize** when encountering any code that attempts to make use of such prefixes. Note that these prefixes rely on the absence of whitespace, so a macro invocation can use `<identifier> # <identifier>` (note the spaces) as a way to consume individual tokens adjacent to a `#`.
    
joshtriplett: wait, this is suffixes, not prefixes.
    
Collectively: *oooooooh*
    
Conclusion. This is a semantic change, but it seems harmless. The danger of people writing custom macros can already happen. The only change is that cfg(false) code accepts them, but that's a feature, because the cfg might be "I have a new Rust".

y86-dev: `0x10.0` doesn't actually parse currently: https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=316edfc43c4ad955ee7b667883a10287 (also on nightly)
    
scottmcm: wait, the string suffix...not sure... the hexadecimal float also? What if we actually want them? Because you can pass it to a macro it's ok? Or does it tokenize as 3 things?
    
joshtriplett: aren't you allowed to do `number.method`?

nikomatsakis: I was under impression it's already a single token. 
    
scottmcm: I get hex float literal as not tokenizing on stable too

y86: everything that gets passed to macro is a single token.

scottmcm: I'm ok with "more digits in a literal". If I were to write my current impression, I'd have expected it accepted "u128".

nikomatsakis: it is a valid token already, but the change is to move it from erroring during Rust *parse* time to *typecheck* time.

scottmcm: is that a meaningful distinction?

nikomatsakis: doesn't affect what macros match I don't think, unless maybe it afects `$e:expr`

scottmcm: is there a meaningful distinction between "it doesn't parse because tokens are different vs grammar doesn't match".

[playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=8b6e67e3605773ed037662287cfb9622)

```rust
macro_rules! m {
    ($m:expr) => {
        
    }
}

fn main() {
    m!(1u129)
    /*
    error: invalid width `129` for integer literal
 --> src/main.rs:8:8
  |
8 |     m!(1u129)
  |        ^^^^^
  |
  = help: valid widths are 8, 16, 32, 64 and 128
  */
}
```

*back and forth niko missed*

joshtriplett: sounds like we are thinking "some yes, some -- is that how we actually want to lex"?

nikomatsakis: scottmcm, maybe you want to post questions on the PR?

scottmcm: y86 can post the link to the thread so we're working off correct data.

## Nominated RFCs, PRs and issues NOT discussed this meeting
### "RFC: Field projection" rfcs#3318

**Link:** https://github.com/rust-lang/rfcs/pull/3318

### "Support #[global_allocator] without the allocator shim" rust#86844

**Link:** https://github.com/rust-lang/rust/pull/86844

### "impl DispatchFromDyn for Cell and UnsafeCell" rust#97373

**Link:** https://github.com/rust-lang/rust/pull/97373

### "Stabilize raw-dylib for non-x86" rust#102793

**Link:** https://github.com/rust-lang/rust/issues/102793
