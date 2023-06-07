---
title: Triage meeting 2022-10-25
tags: triage-meeting
---

# T-lang meeting agenda

* Meeting date: 2022-10-25

## Attendance

* Team members: nikomatsakis, joshtriplett, pnkfelix
* Others: compiler-errors, y86-dev, vincent, Mark

## Meeting roles

* Action item scribe:
* Note-taker: nikomatsakis

## Scheduled meetings

- Tomorrow: lang team private, membership discussion

## Announcements or custom items

(Meeting attendees, feel free to add items here!)

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
### "Short Macro Invocation Syntax: m!123 and m!"abc"" rfcs#3267

- **Link:** https://github.com/rust-lang/rfcs/pull/3267
- [**Tracking Comment**](https://github.com/rust-lang/rfcs/pull/3267#issuecomment-1282943924): 
	> Team member @joshtriplett has proposed to close this. The next step is review by the rest of the tagged team members:
	> 
	> * [ ] @cramertj
	> * [x] @joshtriplett
	> * [ ] @nikomatsakis
	> * [ ] @pnkfelix
	> * [x] @scottmcm
	> 
	> No concerns currently listed.
	> 
	> Once a majority of reviewers approve (and at most 2 approvals are outstanding), this will enter its final comment period. If you spot a major issue that hasn't been raised at any point in this process, please speak up!
	> 
	> See [this document](https://github.com/rust-lang/rfcbot-rs/blob/master/README.md) for info about what commands tagged team members can give me.
- [**Initiating Comment**](https://github.com/rust-lang/rfcs/pull/3267#issuecomment-1282943902): 
	> We discussed this in a @rust-lang/lang meeting a long while ago, and discussed it again today. There was lukewarm sentiment towards having this as a general-purpose shorthand. In general, it felt like:
	> 
	> - The set of things we'd want to add this for are things that often want to be language features.
	> - We have language features planned for some things like this (e.g. prefixes on strings)
	> - The additional complexity and mental parsing overhead here (of having a non-delimited macro, and special whitespace and token-related rules, and questions about how far things might extend...) doesn't seem like something we want to add as a fully general-purpose mechanism.
	> 
	> With that in mind:
	> 
	> @rfcbot close

* 2 boxes are checked during meeting :)
* observation: we need to get more prompt with RFCs
    * we find it harder to "say no" definitively by fcp-close

### "Support upcasting of `dyn Trait` values" rfcs#3324

- **Link:** https://github.com/rust-lang/rfcs/pull/3324
- [**Tracking Comment**](https://github.com/rust-lang/rfcs/pull/3324#issuecomment-1275115366): 
	> Team member @nikomatsakis has proposed to merge this. The next step is review by the rest of the tagged team members:
	> 
	> * [ ] @cramertj
	> * [ ] @joshtriplett
	> * [x] @nikomatsakis
	> * [ ] @pnkfelix
	> * [x] @scottmcm
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

nikomatsakis: no recent comments, waiting on somebody's checkbox

### "Add lang-team advisors team" rfcs#3327

- **Link:** https://github.com/rust-lang/rfcs/pull/3327
- [**Tracking Comment**](https://github.com/rust-lang/rfcs/pull/3327#issuecomment-1276536749): 
	> Team member @nikomatsakis has proposed to merge this. The next step is review by the rest of the tagged team members:
	> 
	> * [ ] @cramertj
	> * [x] @joshtriplett
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




## Active FCPs

None.




## P-critical issues
### "function lifetime elision changed in 1.64" rust#103330

**Link:** https://github.com/rust-lang/rust/issues/103330

Now accepting code we did not used to accept.

There's a patch to change it back to the old behavior.

joshtriplett: Even if we think this is the correct behavior, we should consider that at our leisure, rather than doing it accidentally.

nikomatsakis: Obviously there's a hole in the test suite.

joshtriplett: Should approve for beta/stable backport. Does this merit a 1.64.1 point release?

simulacrum: I think the question is moot, it's not going to happen.

joshtriplett: Because we're so close to 1.65?

simulacrum: Yes.

joshtriplett: Time for a crater run?

simulacrum: Should take 24 hours -- but to some extent I would argue we should do it.

joshtriplett: More about reaching out to them.

Conclusion:

* mark as beta-accepted and stable-accepted

joshtriplett: Who will prepare the beta backport?

simulacrum: Release team, but it's really the compiler team that should approve backport. Let's get this reviewed though.

pnkfelix: We'll take care of it.

## Nominated RFCs, PRs and issues discussed this meeting

### "Add `cstr!` macro." rust#101607

**Link:** https://github.com/rust-lang/rust/pull/101607

joshtriplett: In context of libs team meeting, said "we'd be happy to add it, if lang team isn't going to add it."

simulacrum: I feel like last week we concluded not going to happen in short term.

joshtriplett: Which is to say, libs should prob go forward, and we can always make the macro resolve to it?

simulacrum: Yes.

joshtriplett: So we should author a comment that says "We might be interested in doing this but not in a sufficiently short time frame to discourage creation of a macro."

### "impl DispatchFromDyn for Cell and UnsafeCell" rust#97373

**Link:** https://github.com/rust-lang/rust/pull/97373

pnkfelix: Is this feature gated?

nikomatsakis: I'm not sure.

pnkfelix: see demo of what this is adding in its test: https://github.com/rust-lang/rust/pull/97373/files#diff-6ed6de2e88661cc70064bebf06f7324bc20faea05ddb601d1ff80e50adb2fb37R25

nikomatsakis: I think this is ok because you need arbitrary-self-types feature gate -- this PR would allow you to use `self: Cell<T>`, I believe, but that would still require arbitrary-self-types (and the motivation was allowing people to experiment).

joshtriplett: Sounds like the answer to this is -- please add test showing it is definitely feature-gated (i.e., a compile-error test that gives back an error because no feature gate is included).

nikomatsakis to write comment ([done](https://github.com/rust-lang/rust/pull/97373#issuecomment-1290971083))

### "function lifetime elision changed in 1.64" rust#103330

**Link:** https://github.com/rust-lang/rust/issues/103330

Discussed above under P-critical

### "Allow partially moved values in match" rust#103208

**Link:** https://github.com/rust-lang/rust/pull/103208

- Would like a design document and a design meeting

### "Introduce a no-op FakeRead for `let _ =`." rust#102256

**Link:** https://github.com/rust-lang/rust/pull/102256

- Would like a design document and a design meeting

### "Use `token::Lit` in `ast::ExprKind::Lit`." rust#102944

**Link:** https://github.com/rust-lang/rust/pull/102944

joshtriplett: scott commented. petrochenkov's response is "should've made that call, but didn't, so we shouldn't make the distinction here".

joshtriplett: there's a section on the reference on tokenization that talks about "literal forms that we tokenize, but don't accept in the Rust parser".

nikomatsakis: the distinction you're trying to draw scottmcm, I think, is between tokens that are "very similar" to the form of legal tokens (e.g., `3u33`) and other things that are currently lexed but which have no form (e.g., `0b0fffff` or `"foo"hello`).

scottmcm: we're inconsistent because things like `0x0000f32` is rejected at tokenization time.

joshtriplett: I have a suggestion for how to make forward progress. I realize a bunch of them are unsatisfying. Right now, the case where we already accept them are in macros, but the only kind of macros that can accept this is something which eats a whole token tree (won't parse as a number (or literal? --nikomatsakis). The only thing you can do with it is eat it and throw it away.

scottmcm: I'm very curious if we try to pass it to something that wants a literal.

joshtriplett: from a proc macro you could parse. 

nikomatsakis: maybe `stringify!`

joshtriplett: I don't think there's anything fundamentally preventing us from defining these things with a new meaning. And I don't think it makes it appreciably worse to say "just macros ignore these" to "also, cfg(false) will ignore these". In general, that would actually be useful for making it so that future versions of Rust has the option to introduce this syntax, with older versions ignoring it. My inclination is to accept these in its current form, on the theory that if we wanted to reject them, we should've already done so, and nothing stops us from assigning meaning in the future.

scottmcm: if you pass it as a literal it still fails.

joshtriplett: right, you could do a token eater, or a proc macro.

y86-dev: you can match against a big integer literal [demo](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=ce2a1cc7248b8948e5c2da1655638916)

scottmcm: can't match `0b022` because macro definition doesn't like it:

```rust
macro_rules! match_bad_literal {
    (0b022) => {}; // ERROR: error: invalid digit for a base 2 literal
}
```

joshtriplett: still doesn't mean we can't give a meaning, just that macro can do so too

scottmcm: I think I'm coming around to this being "tokenizer rules have weird cases and maybe we need an edition, but that's not this PR's problem".

*general mutters of agreement.*

scottmcm: so why *doesn't* the example above work?

nikomatsakis: it is weird, looks like macro-rules is doing some sort of validation there...?

joshtriplett: I want to knwo if it distinguish--- yes, it does, it you change the overlong integer, it won't match.

joshtriplett: for narrow case of this PR, shall we just do an FCP?

scottmcm: it's a breaking change to remove it, so it should be an FCP.

Josh started an FCP and un-nominated.

### "Support #[global_allocator] without the allocator shim" rust#86844

**Link:** https://github.com/rust-lang/rust/pull/86844

[previous minutes](https://github.com/rust-lang/lang-team/blob/master/minutes/2022-10-11.md#support-global_allocator-without-the-allocator-shim-rust86844)

[good comment](https://github.com/rust-lang/rust/pull/86844#issuecomment-894814365)

joshtriplett: That comment doesn't explain how the shim works. It would be nice to know if this worked on all platforms? Just on linux? I'm guessing it does not work everywhere.

nikomatsakis: To be honest, I thought of rlibs as only consumable by rustc, and I'm not sure I want to support that use case. 

joshtriplett: I think I share that concern, there's currently a proposal talking about stabilizing some version of the rlib format. But that's separate from this, where, it seems to me like this is useful for just thinking an rlib?

nikomatsakis: It sounds like we're missing some kind of object file that satisfies their needs, and I'd rather that people ask for that and use *that* than using an rlib. For example, could we still do mir-only rlibs this way?

joshtriplett: *elaborates on examples of how you can control deduplication* -- yeah, it's kind of like staticlib, except that staticlib doesn't work if you're not use.

nikomatsakis: I'm a little surprised compiler team seems fine with people using rlibs for this. I would expect people to have to say it explicitly somehow.

pnkfelix: I guess that's the question, is it lang or compiler?

joshtriplett: I think it's both. Part that is lang is: what does global allocator actually mean? Part that is compiler: how do we implement that? What tools exist on different platforms. As well as, what are the supported and unsupported use cases for using the compiler. It should like the lang consensus is that the portions within our purview are not seeming like things we're intending to block over, but the things that are maybe not in our purview, we want to raise the concern that rlib is not the right format for linking with other tools, that this should be an explicitly requested format?

pnkfelix: The thing I find funny here is the simultaneous question of "lang team thinks this is a compiler-team Q, not a lang team one; but also: lang team objects to the design of the feature". 

nikomatsakis: Maybe I'm just objecting with my compiler team hat on. I think we should enable the use case -- it should be possible to link.

joshtriplett: right, it should be possible to link rust code with things that are not rust.

scottmcm: right, I don't think the rlib format is part of the definition of rust, but...

pnkfelix: ...this PR did tee up for me the distinction between PR [#86844](https://github.com/rust-lang/rust/pull/86844) vs issue [#73632](https://github.com/rust-lang/rust/issues/73632), which is "formal support for linking rlibs using some other linker"? is there other utility for this PR beyond the issue?

joshtriplett says something nikomatsakis didn't catch.

pnkfelix: Let me try to rephase: There are cases where we would like rustc to do final linking. We also want to enable people to say explicitly that they are going to use their own linker. But enabling it for ALL rlibs is a 1-way door, once we do that, we won't be able to pull it back. 

joshtriplett: Right, if people have to opt in, to say "I want an artifact that you can link with another linker and expect it to work", then we have the opportunity to say "we are now adding a feature that we don't know how to support if we're not the linker, but that's ok, we can at least give you an error message".

nikomatsakis: Yes, and I do that this part of it -- reserving that freedom -- feels lang-ish. 

joshtriplett: Right, let's say, we don't know if we can commit to always creating an artifact that links with an arbitrary linker, but happy to see the option of producing such a format, as long as it is explicitly requested.

nikomatsakis: I agree with that.

pnkfelix: Definitely if we want to get involved with that we should comment on the [pre-rfc by pcwalton](https://internals.rust-lang.org/t/pre-rfc-stabilize-a-version-of-the-rlib-format/17558/54) linked from #73632

### MCP: Redefine dropck in terms of bound-like constructs compiler-team#563 

**Link:** https://github.com/rust-lang/compiler-team/issues/563

pnkfelix: Just drawing attention to it. It was initially closed as lang purview, but I think it's just a refactoring, and not a language change. I circled around with SoniL and they didn't intend to inject visible language features with this change. Of course, given our history...

nikomatsakis: ...types team?

pnkfelix: yep.

### "PhantomData: fix documentation wrt interaction with dropck" rust#103413

**Link:** https://github.com/rust-lang/rust/pull/103413

(see above, related to MCP 563)

## Nominated RFCs, PRs and issues NOT discussed this meeting

### "RFC: Field projection" rfcs#3318

**Link:** https://github.com/rust-lang/rfcs/pull/3318



