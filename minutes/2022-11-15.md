---
title: Triage meeting 2022-11-15
tags: triage-meeting
---

# T-lang meeting agenda

* Meeting date: 2022-11-15

## Attendance

* Team members: Josh, nikomatsakis, pnkfelix, scottmcm
* Others: Mark, Mara, Tyler

## Meeting roles

* Action item scribe: pnkfelix
* Note-taker:

## Scheduled meetings
- "Meeting proposal: Rust Specification" [lang-team#179](https://github.com/rust-lang/lang-team/issues/179)
    - Nov 16
    - josh: we want to keep this project member focused, but do we feel we have enough representation from compiler?
    - pnkfelix: I'll be there
    - nikomatsakis: maybe ping Wesley
    - Mara: Ralf said he'd join as well
    - nikomatsakis to create a separate invite to avoid too many people and keeping meeting productive; ping if interested
- "Contracts and Automated Reasoning for Rust" [lang-team#181](https://github.com/rust-lang/lang-team/issues/181)
    - Dec 7

No more meetings planned until end of year.

Holiday schedule

* Nov 22 -- meet if you can (nikomatsakis not coming :)
* Nov 23 -- skip
* Dec 20 (Triage) -- optional, meet if you can
* Dec 21 (Design) -- skip
* Dec 27 (Triage) -- skip
* Dec 28 (Design) -- skip
* Jan 3 (Triage)
* Jan 4 (Planning)

## Announcements or custom items

Zip. Nada. Zilch.

## Action item review

* [Action items list](https://hackmd.io/gstfhtXYTHa3Jv-P_2RK7A)

## Pending lang team project proposals

None.




## PRs on the lang-team repo

None.




## RFCs waiting to be merged
### "Support upcasting of `dyn Trait` values" rfcs#3324

**Link:** https://github.com/rust-lang/rfcs/pull/3324

### "Add lang-team advisors team" rfcs#3327

**Link:** https://github.com/rust-lang/rfcs/pull/3327



## Proposed FCPs

**Check your boxes!**
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
	> No concerns currently listed.
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


scottmcm: how does this interact with parallel compilation RFC? more of a compiler problem.

joshtriplett: I feel we should change this to "exponent backoff progress indicator" and leave the mechanism to the compiler.

pnkfelix: idea of using lint mechanism to specify constraints.

joshtriplett: lints don't take parameters, how would you do that?

pnkfelix: I just meant allow/deny

scottmcm: deny means .. machine dependent?

nikomatsakis: I continue to be unenthusiastic about this approach, I think we could have a predictable limit that we can just raise, which avoids these complications.

joshtriplett: If we can have a limit that is independent from optimization or machine, then by all means, let's talk about it. in the meantime, the current fuzzy limit is something we should drop. I don't think we should tie those two together.

nikomatsakis: we do have a reasonable plan, but once we drop the limit, we can't go back there, right?

pnkfelix: changing the metric now is a breaking change, right?

nikomatsakis: we're counting every step today, we'd be counting a strict subset of those steps, right?

scottmcm: I thought we didn't even run CTFE on optimized MIR?

joshtriplett: what if we change how we desugar things?

nikomatsakis: what we're counting is loops and function calls, it's not really something that we change

pnkfelix: if we do that and keep the end the same, I agree we can assert it will accept strictly a superset of what is accepted before, but whether that's the correct threshold is a question?

joshtriplett: I'm concerned because now number of backedges is a semver breaking change. Any kind of limit is a breaking change.

nikomatsakis: Given timeouts and limits also kind of true, but yes, that's true.

pnkfelix: current situation where you can't adjust the limit is worst of both worlds.

scottmcm: a bit of a law vs fact thing. Anything that depends on time is so much more breaking than anything we could do that depends on the code you're running. If what we're trying to solve: if there's a limit, and we make it advisory, not something you can deny, we can use a smarter limit that's not time, that provides more information, be more consistent.

joshtriplett: if we're pointedly not giving a deny, only giving a progress indicator, I think time is the correct threshold. Every minute of compile we say "hey, still doing CTFE...".

pnkfelix: some people want the ability to deny, e.g., fuzzers. 

joshtriplett: maybe the fuzzer wants to timeout...?

joshtriplett: can we get it down to what is compiler vs lang?

nikomatsakis: I feel the lang part is "will there be a limit" (and to a secondary degree, what is limit).

pnkfelix: if we say that whether there'll be a metric is lang team, and we are also going to say we don't want to allow for any breaking change here, ie., the threshold might be something we could strictly raise now, my concern now is that if we're going to say lang team responsibility to define that we need a stable metric, it's also up to us to decide if we are willing to accept a breaking change?

nikomatsakis: is this motivated by one specific thing, or by general streak of annoying breakages?

pnkfelix: one specific customer that has deliberately downgraded.

nikomatsakis: can they make a workaround today?

simulacrum: I think they have an attribute but it's not exposed on stable today

tmandry: goal of this is to prevent infinite loops?

nikomatsakis: basically, yes

pnkfelix: some folks have expectation that compilation will return a result. I've been pushing to say an answer here is to provide right level of feedback to say "hey, code is taking a long time, here is how you can adjust things so that you force it to finish with an error, or let it keep going".

nikomatsakis: ok, I do feel like this solution meets all the hard constraints, even though I don't love it...

* some kind of stable count (e.g., backedges) that is used to trigger warnings by default
* but we can set a hard limit e.g. for fuzzers, perhaps with a compiler flag to avoid it becoming a semver hazard

the main argument in favor of this is---

* number of backedges is not part of your semver guarantee

the main argument against is---

* we can't reliably say when compilation has gone awry, some processes can take forever (e.g., complicating IDEs)
    * this is already true during macro expansion because procedural macros
    * and if limit is high you'd have to wait a long time anyway

scottmcm: the macro limit doesn't seem to cause big problems in practice; being "pretty close" is probably ok

pnkfelix: can add warn, but not hard error by default, that's part of this, right?

scottmcm: could add deny-by-default technically, but prob shouldn't, though we've done it before

tmandry: is it possible to write code that infinitly loops? do we prevent that?

scottmcm: It's easy; this is stable today:
```rust=
pub const fn demo() -> ! { loop {} }
```
<https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=de37822bc57e7d880e89d6f3335be1ee>

nikomatsakis: whether code terminates is highly relevant to proving things, but the limit doesn't really help there

pnkfelix: our official suggestion is that we don't approve of using lints...?

joshtriplett: we leave it up to compiler to provide appropriate progress indicators, but lints are not a great fit, as we expect them to be available even for stuff in crates

scottmcm: I think we're saying "no deny-able time-based limit"

joshtriplett: can't have a deny limit based on an unpredictable metric that varies from system to system, at the very least, and don't have consensus on whether we have a denyable lint even if it is, since it wouldn't be consistent from library to library

scottmcm: the more we approach that we have a defined lowering from rust to mir the more this becomes predictable, in the future we'd have more consistency about it, would be close enough to be stable as any other lint -- we always can change them between compiler versions, right?

simulacrum: I know a lot of people experience the lints changing from version to version as very painful at scale. I think this would be annoying if you upgraded your compiler and you have to regularly bump some limit. 

joshtriplett: I've seen the case made that we should consider being stricter about whether lints becoming stricter is a breaking change. I've seen people make the case that warning should be a breaking change.

scottmcm: leads to the C problem...

joshtriplett: not arguing with it, just observing 

pnkfelix to report consensus to the PR and request change from "lint" to "progress indicator from the compiler"

## Active FCPs
### "Restrictions" rfcs#3323

**Link:** https://github.com/rust-lang/rfcs/pull/3323

### "Remove drop order twist of && and || and make them associative" rust#103293

**Link:** https://github.com/rust-lang/rust/pull/103293

### "Unreserve braced enum variants in value namespace" rust#103578

**Link:** https://github.com/rust-lang/rust/pull/103578



## P-critical issues

None.




## Nominated RFCs, PRs and issues discussed this meeting

### "Remove drop order twist of && and || and make them associative" rust#103293

**Link:** https://github.com/rust-lang/rust/pull/103293

Removing nomination.

### "RFC: `c"…"` string literals" rfcs#3348

**Link:** https://github.com/rust-lang/rfcs/pull/3348

josh: Came up in a previous meeting and said "well, vaguely in favor of this, but prob not quickly enough for libs to skip the macro". Mara took that as a challenge and wrote the RFC.

scottmcm: How do we feel about fat cstr? Seems totally fine but I'd be sad if `c"foo"` becomes the wrong type later?

josh: if we get thin cstr, should be the same cstr

scottmcm: are we sure we can do that?

mara: could block stabilization

scottmcm: or use an edition, wouldn't be fun though. Don't know the state of extern type though.

mara: I'm hoping that if we do this RFC with the c string literals, would give motivation to work on `&CStr` being a thin pointer.

josh: I left a pointer that we should make it block stabilization. We should either decide we are not making it a thin pointer (and change what C string literals mean) or decide that we are.

mara: regardless, if this will happen, we can say no to the `cstr!` macro.

nikomatsakis: I feel like this thin question can be resolved post RFC, I think the real question is whether we want this syntax for explicit C string literals.

scottmcm: which I also want, but FFI is imp't enough 

nikomatsakis: even if we make them inferred, we prob want some form of explicit syntax anyway

scottmcm: perhaps type ascription

joshtriplett: would be nice to have `p` string literals for `Path` literals

mara: complicated because not in core, but yes

(conversation switched to [rfc#3349](https://github.com/rust-lang/rfcs/pull/3349) "UTF-8 characters and escape codes in (byte) string literals")

mara: someone proposed they should accept both utf-8 and non-utf-8, there's a related RFC bringing that to byte string literals. If we do that, there is no longer a difference between escape codes, which makes parser/macros easier, and we can enforce it later.

scottmcm: given our previous thing we FCP'd around tokens should accept whatever and make it a semantic error, at very least I think we'd accept "token can have whatever escapes you want and it's only when you use it as a non-token that it matters"

mara: tokenizer does need to understand where the string *ends*

scottmcm: I meant it would understand all the escape sequences and not emit errors until you use it as a non-token

nikomatsakis: in other words, all the escape sequences that any string supports, not all *possible* escape sequences

mara: roughly what I propose here but also not rejecting it later on, even, at least for byte strings

scottmcm: just splitting it in my brain since first part is comparatively easy. second part is more of a conversation.

joshtriplett: for future work, should we have some mechanism for macros to convert b-string to c-string and vice versa?

mara: my future plan is to fix the concat macro in the next edition, make it always use the string type of the first argument, so you can use do `concat!(b"", ...)` to concat whatever.

joshtriplett: ah, interesting. *digresses a bit on whether this is a bug* anyway, ok, we can handle that later.

(back to rfc#3348)

joshtriplett: anybody see a reason not to FCP this once we feel ready?

scottmcm: seems fine for someone to pfcp it

joshtriplett: can Rust source files contain embedded literal null characters inside a string?

mara: ummmmmmmm, let's try it out

joshtriplett: I hope you can't just a null ..?

scottmcm: seems no worse than a nbsp.

joshtriplett: at very least, for same reason we disallow `\0` in a c string, can we also explicitly disallow a literal null inside a c string?

mara: I want to stop giving rules about what escape codes are allowed, and start giving rules about what *bytes* are allowed (e.g., disallow invalid utf-8).

mara: it already says that null byte is disallowed.

joshtriplett: ok.

### "Raw pointer alignment checks in debug" compiler-team#570

**Link:** https://github.com/rust-lang/compiler-team/issues/570

nikomatsakis: this adds checks to catch deref of unaligned pointers

joshtriplett: debug-assertions are related to debug builds of crates?

nikomatsakis: yes.

joshtriplett: this would make debug builds check raw pointer alignment before dereferencing ...?

joshtriplett: right now you can turn on/off integer overflows without changing other things

pnkfelix: but integer overflows have a semantics (in Rust)

nikomatsakis: is the motivation for turning off performance of debug builds or is it I want to be able to deref aligned pointers?

joshtriplett: not performance. I'm not too worried about this making debug builds too slow, prob not a big deal, we may be able to optimize away check sometimes too. I'm thinking of existing code that does this and isn't actually wrong to do so.

nikomatsakis: I don't think that if code does this, it is wrong

joshtriplett: I think it's false to always deref an unaligned pointer if you're on a platform

scottmcm: but it's still wrong because we tell LLVM that

joshtriplett: how do we stop doing that?

scottmcm: you use the "read unaligned" function, which tells it to be alignment 1

joshtriplett: we've already made default raw pointer reads commit to be aligned, even when it's unnecessary. when did we decide *that*?

nikomatsakis: we decided it pre-1.0 but in an ad-hoc way

joshtriplett: feels like there's a meaningful difference between we made this decision intentionally or this happens to be the case and we're not questioning it. I'd like to stop having that pattern 

pnkfelix: I think that's an unfair characterization. I'm sure this decision was made on the basis of "Rust is going to raise the bar on how languages work"

joshtriplett: (Was going based on niko's characterization as "ad-hoc".)

nikomatsakis: I meant there wasn't an RFC, it was discussed, but prob on a GH issue.

simulacrum: matches the semantics of all pointer dereferences in rust, right? they're generally required to be aligned unless you use the intrinsic, no real distinction for raw pointers.

scottmcm: clang will also make unaligned derefs UB even if your platform supports it.

simulacrum: fwiw, I don't know about raw pointers, but for regular references LLVM uses it to SIMD optimize etc.

joshtriplett: I am on board with regular references being mandatory aligned, and I'd like to even use those for niches, just talking about *raw* pointers.

simulacrum: my point is that slice-iter, vector, if we wanted to go back here, we'd want to use a read-aligned intrinsic to help LLVM along. In theory it might be able to infer that the pointer originated in a reference.

scottmcm: I would say dropping to a pointer shouldn't make your performance worse.

joshtriplett: I'm hoping LLVM should be able to track where it came from. Anyway, i'm not proposing that I definitely want to change this back, I'm proposing we have historical ad-hoc things like this that the default is the status quo, even though we haven't had a language level evalution. 

nikomatsakis: (5 minutes over)

scottmcm: this one is so core, this isn't some corner case that basically nobody's thought of for 2 years, this is like ralf's models of everything include this, miri has checked this for as long as miri has checked UB, etc. I don't think of this one as "oh, that's weird", this is a lots of people have seen that for many years. Some things in miri that it allowed through for a while because we weren't sure but this doesn't fall into that category.

joshtriplett: we should wrap up, but I do think we should figure out why this pattern keeps occurring.

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

### "Implement a lint for implicit autoref of raw pointer dereference " rust#103735

**Link:** https://github.com/rust-lang/rust/pull/103735

### "Refine `instruction_set` MIR inline rules" rust#104121

**Link:** https://github.com/rust-lang/rust/pull/104121

### "update aliasing rules section of the reference" reference#1290

**Link:** https://github.com/rust-lang/reference/pull/1290


