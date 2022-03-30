---
title: "Design meeting 2022-03-30: Lint Policy"
tags: "design meeting"
---

# Lang Team: Lint Policy for Project, Language, and Compiler

tl;dr: Let's revise our thinking about what lints we, as the lang team, put into the language and compiler. 

It is healthy for the *Rust Project* (including specifically tools like Clippy) to be opinionated and make subjective choices on behalf of its developers, but Rust the language should be more conservative in its choices here.

## Introduction

Today, part of the Rust development experience is getting regular feedback from "lints". Lints are static analyses that
 1. can be enabled or disabled in a scoped manner, and 
 2. detect questionable coding patterns and issue diagnostics informing the developer of the presence of those patterns.

Why do we lint?

In particular: Rust treats violations of its static semantics as *errors* that cannot be ignored. So, playing devil's advocate: If lints are not soundness bugs, then why report them to the user?

One answer: While the divide between program acceptance and rejection should be tied to a well-defined static semantics, the Rust project also holds opinions about "good" and "bad" code, even though these notions are based on heuristics or subjective criteria.

Nonetheless, I decided to dig in a little deeper.

----

I identified four reasons tools like `rustc` and `clippy` issue lints today, and put them into the following categories.

1. [future-proofing][]: flagging code that a future release of Rust will reject (potentially tied to an edition),
2. [error-heuristic][]: flagging *sound code* that is nonetheless correlated with *programmer error*,
3. [uniformly-clean][]: enforcement of uniform coding style, and
4. [expressly-clean][]: educating the user about Rust features.

I elaborate on these categories a bit more below.

## Lint Categories

### future-proofing
[future-proofing]: #future-proofing

These lints flag code that we plan to start rejecting in the future, but have opted to not yet *start* rejecting, either because such rejection would break too much of the current ecosystem, or because such rejection would violate Rust's stability promise.

There are essentially two kinds of future-proofing lints:
* edition-compatibility lints for code patterns that stopped being supported in edition 20xx.
 * future-incompat lints for code patterns that are deprecated and will eventually be removed, for all editions, in a future version of the compiler.
     * *in principle* all of these cases should be instances of either outright unsound[^abstract] coding patterns, or constructs that the language team thinks were never meant to be assigned a runtime semantics (even if the currently assigned semantics is "sound"), or syntax that are being "repurposed", where we ["keep the general purpose of the syntax, but correct cases where users aren't getting the meaning expected or wanted"](https://hackmd.io/i1Ob4XS6TwuUv-rOVEoM4A#Repurposed-syntax).
     * in practice there have been (a few?) exceptions; see e.g. issues [#42326], [#48919].  

#### Edition Linting

Felix believes the future-proofing category of lints covers most of the cases that Niko laid out in their [lints and editions doc](https://hackmd.io/7YycPi5ORhiXbualfhaQRw).

This is the table from that document that lists those five cases and how they are to be handled with respect to linting and migrations:

name | Edition N | Edition N+1 | Migrations?
-----|-----------|-------------|-----------
repurposed syntax | opt-in warning | new meaning | mandatory
edition error | opt-in warning | error | recommended
soft deprecations | warn | warn | recommended
hard deprecations | warn then deny-by-default | deny-by-default | mandatory
bug fix | warn then error | warn then error | recommended

:::warning
Felix is not clear on the distinction between "edition error" and "hard deprecation". Obviously the two are supposed to be handled differently according to the above table, but Felix does not know how to know if a proposed lint falls into one category or the other. To be frank though, it seems like [Niko did not know either](https://hackmd.io/i1Ob4XS6TwuUv-rOVEoM4A?view#Edition-error). :smile:
:::

The main exception is "soft deprecations", which cover things like the `rust-2018-idioms` group. Those seem like they are more tightly associated with either the [error-heuristic][] or the [expressly-clean][] categories, each of which are described below.

[^abstract]: (at least in terms of violating our abstract memory model, even if there is no concrete witness of a segmentation fault or other runtime failure; see e.g. issue [#59159][])

[#42326]: https://github.com/rust-lang/rust/issues/42326
[#48919]: https://github.com/rust-lang/rust/issues/48919
[#59159]: https://github.com/rust-lang/rust/issues/59159

### error-heuristic
[error-heuristic]: #error-heuristic

These lints are for things that we believe to be correlated with programmer error, even if they are in principle sound coding patterns.

One can imagine some subcategories here. 

Programming errors such a passing the wrong argument to a function can sometimes be signalled by the `unused_variable` lint; likewise, an assignment to the wrong item can be signalled by the `unused_assignments` lint.

Clippy's `absurd_extreme_comparisons` lint catches cases like `min <= x` and `x > max` that are likely to represent bugs, because their boolean values never vary (regardless of the value of `x`).

Inefficient code can sometimes be detected by things like the compiler's `large_assignments` lint, or Clippy's `cyclomatic_complexity` lint.

:::info
The [error-heuristic][] category seems like it overlaps with Niko's ["hard and soft deprecations" cases of edition lints](https://hackmd.io/i1Ob4XS6TwuUv-rOVEoM4A?view#Hard-and-soft-deprecations). The distinction I am trying to draw is that the [error-heuristic][] lints detect code that is nonetheless sound, or at least the chance of outright unsoundness is low enough that we are not outlawing the syntax in the language itself (potentially via an edition change), but rather just trying to provide a "helping hand" to the user (which might *also* be tied to an edition change).

Thus, the "hard deprecations" remain under the [future-proofing][] category above, while the "soft deprecations" belong under [error-heuristic][].
:::

### uniformly-clean
[uniformly-clean]: #uniformly-clean

A *uniform* coding style can ease collaboration between multiple developers on a single code base, by reducing the noise caused by shifts in style between different developers. Reducing such noise can make it easier for reviewers to notice bugs.[^dogma]

[^dogma]: I cannot resist pointing out that dogmatic adherence to a coding style can, in some circumstances, make one's code *less readable*. I'm thinking specifically of cases where a programmer lays out their code in tabular or column format, to make it more apparent to the reader which parts of the code are entirely uniform and which parts are varying. Arguably this special case is just *another kind of uniformity*, one that automated style checks are often not well-equipped to handle without human intervention.

The [nonstandard syle lints](https://doc.rust-lang.org/rustc/lints/groups.html) are an example of this.

Another example of this is the `unused_parens` lint. (At least, to my knowledge, extra parens is not positively correlated with programming error.)

Unfortunately, the categorization within the compiler itself does not follow the breakdown proposed in this document: some elements of the [`unused` group](https://doc.rust-lang.org/rustc/lints/groups.html) fall under this [uniformly-clean][] lint category, while others fall under the [error-heuristic][] lint category. 


### expressly-clean
[expressly-clean]: #expressly-clean

An area that Rust takes very seriously is having *empathy* for its developers. We invest a lot of effort into making great diagnostics that do not presume the reader is already a Rust expert.

One corollary of that is that lints can be a great avenue to *educate* the developer about Rust features that they may not be familiar with, and advertise their use.

One example is the `while_true` lint, which detects instances of `while true { ... }` and suggests the developer replace it with `loop { ... }`. The former is probably more well-known to users of other languages, but the latter is more idiomatic Rust and also enables more features, such as the [`loop_break_value` feature](https://github.com/rust-lang/rust/issues/37339).

The edition idiom lints (e.g. `rust-2018-idioms`) are also examples of this.

:::warning
Question: Is this "just" a special case of the [uniformly-clean][] category?
:::

## Opinion Time

After reflecting on the above categorization, Felix has an opinion, which he will state in three different ways.

 * The T-lang design team is already over-extended, and should delegate whatever work it can. Many of these lints represent a rich opportunity for such delegation.
 * The Rust lang design team should not be in the business of deciding what lints should be provided for *all* the categories above.
 * We should not continue with our tradition of shipping all of the four above kinds of lints with the Rust compiler. We should narrow our focus, either to just the categories that are most strongly correlated with unsoundness, or to the categories that are correlated with both unsoundness and high chance of programmer error.

Specifically:

All of the [future-proofing][] lints ("flagging code that a future release of Rust will reject") remain a T-lang concern: they tie directly into what code is accepted/rejected, and *need* to be part of the compiler.

All three of the other categories seem a bit more "squishy."



The [expressly-clean][] lints ("educating the user about Rust features") serve an important purpose for pedagogy. But: Felix worries that we do not have bandwidth on our team to properly handle pedagogical concerns. Would these lints would be better handled via Clippy? Or via a different subteam with pedagogy as their core focus?
 * The main reason I can identify to add such lints to the Rust compiler itself, rather than Clippy, is simply because "its easy enough to do it."[^larger-audience] But that argument does not account for ongoing maintance costs, or language design costs.

[^larger-audience]: To be fair, there is also the argument that putting it pedagogical lints into the compiler inherently means it gets a larger audience than if they are deployed to Clippy alone.

The [uniformly-clean][] lints ("enforcement of uniform coding style") represent an inherently subjective matter. It is okay for the Rust Project as whole to present its own (informed) opinion on subjective matters such as coding style, via tools like Clippy or `rustfmt`. But deciding such opinions should not be part of the *language design* team's workload. Arguably style choices should not be embedded into rustc at all[^pretty], and should all be migrated to Clippy and `rustfmt`.

[^pretty]: I can hear you cry: "But what about the rustc pretty-printer? Style choices have to be made there!" I have no counter-argument to that.

The [error-heuristic][] lints ("flagging sound code correlated with programmer error") do sound like they are under T-lang domain. But: I do worry this is just as subjective as coding style issues: *why* do we think these patterns are correlated with errors? 

One might argue that if the signal-to-noise ratio is high, then an [error-heuristic][] lint is doing its job.
 * But are we doing anything to confirm that the signal-to-noise ratio *is* high and *remains* high?
 * We have no compiler telemetry, so what are we relying on? Absence of bug reports ? Angry or happy tweets from our users (i.e. anecdotes)?

In the end, Felix thinks that even if we cannot perfectly describe the correlations and error-cases of interest, there is still something inherently *non-subjective* about catching outright error.

This non-subjective (even if imperfect, heuristic-based) nature pushes Felix to the side of saying that it can remain a T-lang resposibility to make decisions about what [error-heuristic][] lints should remain in Rust the language and `rustc` the compiler.

But, in case you cannot tell, Felix could easily be swayed to the other side of this opinion.

## Proposal

Proposal 1 (mild): We should stop adding more [expressly-clean][] and [uniformly-clean][] lints to `rustc` (and likewise to the Rust language itself).

Proposal 2 (wild): we migrate all existing lints from the [expressly-clean][] and [uniformly-clean][] categories over to Clippy. I.e.: We disavow them as part of the language definition itself.

The main class of lints that Felix hesitates on applying the above reasoning to is the edition-idiom lint subset of [expressly-clean][]. Does it make sense to leave the edition-idioms as a T-lang responsibility? If so, *why*? (I.e., if edition-idioms remain a T-lang responsibility, then there is something wrong with the chain of logic presented above.)

:::warning
(p.s. it is entirely possible this whole document is a trojan horse to get `unused_parens` out of the compiler.)
:::

## References

* Niko's draft on [lints and editions](https://hackmd.io/7YycPi5ORhiXbualfhaQRw), which links to [edition and semver policy](https://hackmd.io/i1Ob4XS6TwuUv-rOVEoM4A) and [idiom lints](https://hackmd.io/HETreGqPSRezlN109vgCnQ)

## Meeting Notes (dequeued)

### inline doc-comments discussion

*Edition error* vs *hard deprecation*:

pnkfelix: Was hard deprecation meant to mean "future incompatible" stuff?

nikomatsakis: the intent was to have things that are *very very likely* to be bugs (overflowing literals) but we still have *very occasional* use. The reason to do warn-by-default first in older editions is to give people some time to adapt their code.

nikomatsakis: I'm not convinced that it carries its weight to tie this to editions at all.

pnkfelix: yes, though we did it for NLL and it was useful.

pnkfelix: niko confirms that removing parens is not a way to avoid bugs.

joshtriplett: I think it falls in the category of expressly clean. It's relevant for "this is people who think is a C-oriented language" to realize "I don't need to have parens here"

pnkfelix: agree. I came very close to filing it under expressly clean. There's maybe an argument where you'd have a version that only files for `if` ...

joshtriplett: If there are cases where it is much, much more subjective, e.g., this is normal rustc but you could take it or leave it. The top-level of an if or while, I think that's really valuable. To comment on the "expressly clean", part of it is empathy for developers, but also "gentle norms", we have things that are more than just compiler errors, but more like "welcome to a community of people who are trying to collaborate, here is how we do things around here". I see value in steering people towards a collaborative coding style in that way.

pnkfelix: Don't disagree, but curious what that implies. Still makes me think that should move to clippy.

joshtriplett: I think clippy has zero effect of norm'ing. It's to allow people to "opt in" to nitpicks that we don't put in the compiler because people would complain about them more often than they would appreciate them. It's rare to run it by default so I don't think it has a norming effect. The fact that there are warn-by-default things in clippy, but they have a lower signal-to-noise ratio, has the result that people don't want to run clippy and sometimes get annoyed when people run clippy on their project, since they don't want to take the churn packages. That is part of why things that don't have a low SNR below in rustc, precisely so that people will use them.

jlusby42 (in zoom): I left a comment to this effect

### nikomatsakis: at the risk of being too amazon, working backwards...

I am keen on the overall "sense" of this doc, but I think we should be working backwards somewhat more. In particular, I would like to identify the goals we have with lints (this doc is a good starting point, I think w can refine it slowly) and then we should ask ourselves how we think we can best help users achieve those goals. I would definitely be guided by the reliable / performant / supportive hierarchy (in that order).

As an example, I think it's a great idea to have "lints that help you to learn rust conventions". I've heard a LOT of new users talk about how helpful clippy is in this regard. That said, I've also heard a lot of teams who don't even know clippy exists -- does it make sense to require *extra steps* to opt into things that *help you learn*? I'm (legit) not sure! I'm not, tbh, entirely sure why clippy even exists as a separate command from rustc beyond historical reasons.

I guess another angle is the fact that we have some challenges, e.g., despite our efforts to retain the freedom to grow the scope of lints, people get annoyed when their code stops building outside of their control. That's probably a separate topic, but I think it would fall into us working through the workflow for various cases.

This is seemed like a big job though and I'm not sure whether it makes sense to pursue!

pnkfelix: I think this is what I was trying to do by opening with what we are trying to accomplish and trying to take a broader perspective beyond just the compiler. Only question is where things end up falling in terms of what things go where. I guess my first question then is "what do people think about the categories I came up with". One of scott's questions touches on a weakness in the categorization. What do other folks think, did I miss anything big?

mara: I think the edition linting part could overlap a lot with any of them. We have some lints that in the previous edition are always warning, in new editions go directly to hard error because it's not possible anymore. Might be tricky to separate editions from other categories.

pnkfelix: I tried to imply that the edition lints cannot be grouped together, there are big subcategories of edition lints. e,g. the idiom lints which are very different from compatibility lints.

jlusby: incase it helps, clippy has a lint categorization scheme that shares some similarities with the categories you came up with Felix https://github.com/rust-lang/rust-clippy

mara: as an example, the 2021 panic lint is always warning in older editions, not because of edition compatibility per se but because it is likely to be a bug. Doesn't fall into any category that is in the table. Currently 2018 as a warning, but in next edition they don't trigger, code doesn't compile.

pnkfelix: isn't that an error heuristic?

mara: I'm looking at this table ...

pnkfelix: ...that came from a different document, maybe I shouldn't have included it, I think is misleading.

joshtriplett: I found it a little bit confusing because the distinctions weren't clear. We've had two different categories. Cases where we only want to tie it to the edition, and cases where we want to "ramp it up". The latter I think isn't an edition lint, just one where we've decided to be more aggressive since people are opting in.

mara: looking at the list of four categories, I think it's both category 1 (future proofing) and 2 (error heuristic).

joshtriplett: I agree that future proofing often has overlap with the errors.

nikomatsakis: I think what's missing for me from the categories is *why we have the lints*, it's there, but not made very strong. For example if our goal is teaching Rust, that would 

joshtriplett: something else is signal-to-noise ratio, we care about that a great deal. But some of these things are subjective. Sometimes we say, effectively, we consider this to be 100% signal in that, yes, 90% of the time it's flagging something bad, and 10% it's flagging something that -- while not wrong -- we would prefer that you don't do.

joshtriplett: esp. in the "clean" categories, it's valid to say "your code would work", but it's valid to say they don't have noise, since we would you want to change the code anyway (paraphrased).

pnkfelix: while I do think the norming effect is important, it raises the question of "do we think that -Dwarnings should affect that?" I think that trying to educate users about norming...

joshtriplett: I think people are adding deny-warnings because they're used to -Werrors for other languages, but it's not obviously a good idea. I don't think we should put substantial effort into making deny(warnings) less painful to use, since you are opting in to "give me that pain". I think we could provide better categories like "deny class of warnings". If we had "style-related" items, I would absolutely be happy to see "deny-everything-that-is-not-a-norming-style-issue". If we feel like this is not about correctness, just "how we do things around here", I do think that's a different category. I do think, and I think Niko agrees, a lot of our lints are things where we might want to say "let's let you run your code and test suite even if you might not want it to pass CI/CD".

jlusyb42 (from zoom chat): related to josh's comment on signal to noise ratio: clippy::pedantic, clippy::nursery and clippy::restriction are all ways of communicating what kind of noise they have

jlusyby42 (from zoom chat): some sort of way to deny warnings that exist as of a certain lockfile version? (stops throwing out random proposals)

scottmcm: says to me we have another level of lint. I agree deny-warnings is largely not what you want but I also agree that we don't have that many deny-by-default warnings. I kind of want somewhere in between. People are turning our warnings into errors because there are things that are important, but they're not deny-by-default, but they're also not... but that "can I run my unit test category"... parens are important enough for CI and easy enough to get right for CI and also not I want to block my unit test from having too many parens.

joshtriplett: they're a failing test, not don't run my tests.

pnkfelix: I want to push back on this. In terms of a big collaborative group, my gut reaction, people turning on deny-warnings are expecting error or future incompat. Things that correlate with bug.

joshtriplett: I don't think it's just that. When I've seen -Werror turned on in large codebases, it's not just, oh all these are perfect, it's "if we don't force people to fix this, they might commit code thathas these warnings". If we do, we start treating those warnings as noise, so we miss important ones. It's very much "don't allow these to accumulate", it's "pay off debt as you accrue it".

pnkfelix: this ties back into our struggle with our warnings group

nikomatsakis: also fixing bugs in warnings

joshtriplett: don't want to get too far into impl space, but I think in most cases I've seen deny-warnings, I'd happily replace it with some kind of `#[test(warnings)]`. A test that fails if there are warnings. I'd happily use that so that when I run cargo test I get a failure. I would do that over deny-warnings. (yaahc note: I refuse to take the bait and dive further into the impl space (in this meeting))

nikomatsakis: that's a solid idea.

nikomatsakis: +1 to what josh said, and also, keep in mind that there are always new people coming in to every code base, so having something like clippy helps keep their PR cycles faster. Definitely an attitude also of "I may not agree with everythign but easier to just go with the flow".

pnkfelix: I keep thinking that teams that want to be getting that norm should be using clippy.

joshtriplett: I think any rust team that's primarily concern about things within their team might opt in within clippy, but rustc has that effect for the overall rust community. Here are our base line norms whether you know to opt into them or not.

nikomatsakis: I don't know why a user would want clippy as opposed to lint groups. But I think I agree that lang team should be focused on errors and compiler team should not be maintaining that code.

scottmcm: can we therefore agree that we don't want FCP on any lint?

pnkfelix: no! warning group can cause breakage.

joshtriplett: I would be quite happy for us to adopt a policy that breakage in deny-warnings is something we're not concerned about.

nikomatsakis: scott were you say "lang team never needs fcp on a lint" or "there are categories of lint that do not require fcp"

scottmcm: from the perspective of "a lint is never a 1-way door", at no point should we require FCP, though maybe there are categories where we should at least have a second

mara: I was going through lints to categorize them. Got stuck with the `...` and `..=` lint. Could argue it fits into category 4, 3, 2, or 1 (e.g., easy confusion, error, edition).

nikomatsakis: I think things often fit more than one, so categories are maybe not the right way to phrase this.

mara: I think it's almost never the case that they would fit in only one. Most of the time we find something cleaner is the "unclean version is easy to make mistakes with", and the reason...

pnkfelix: I think I agree that categories ought not to be considered mutually exclusive, and it's prob a waste of effort to try to find out if they are, and allow lints to have multiple categories. There might be a prioritization. Future proofing for example is maybe fundamentally more imp't, and so you would tag the most important, if 

jlusby42 (zoom): category tags!

jlusby42: minor point about being a 1-way door. Linting about "this will panic" is essentially a kind of stability promise. 

scottmcm: I just meant that in terms of "If there is a # Panic section" in the API, then I don't think you need to sign off on a lint that says "you passed 0, this will panic". If it's a deep inspection thing, that might be different. For things that are covered under documented guarantees already...

joshtriplett: In terms of sign-offs, I don't feel that historically we've had a massive work overload from reading and signing off on lints, it's more been a maintenance burden by way of "bad SNR ratio generates a lot of support issues" and if we don't know whether it does that's hard to review. I want to call attention to the point that Felix made that we should have better metrics for whether we should add it, whether it's working out well, in terms of whether it's learnability, etc. I do think we're doing guess-work that we could get more rigorous.

nikomatsakis: I can't argue that it's a 1-way door, but it feels like some categories are "part of the language" in a way that I still think belongs to lang team, have to think about that more.

scottmcm: We've seen people hit lints in CI and sometimes rolled it back in beta.

*at time*

pnkfelix: I think the answer is that in terms of follow-up items, the specific proposals I made are not going to happen, based on the general feedback of team. I've been trying to push up an idea that it's more important to post something than say nothing.

nikomatsakis: I think many elements of the prescriptions may be good, we're not quite right there.

nikomatsakis: I would enjoy brainstorming a bit

pnkfelix: maybe chat on zulip?

k

## Queue for (big) questions / discussion topics

### nikomatsakis: future compatibility grouping edition and not

More of a comment than a question: I have traditionally resisting grouping *future compatibility* lints with *edition lints* as they seem to me to be categorically different:

* Future compatibility lint: upgrading the compiler may cause this same code to stop compiling
* Edition lint: moving this code to a new edition (without using the appropriate tooling) would cause an error or change in semantics

pnkfelix: Interesting. I guess I colocated them because both of those are guarding against *different kinds of future incompatibilities*: one with compiler version, and one with source-code edition.

nikomatsakis: yes, my concern here is that I don't want to "muddy the waters" when it comes to edition messaging by conflating them.

### scottmcm: In general, I agree lang doesn't need to be involved

Lints are by definition not an irrevocable decision, so leaving it up to the diagnostics experts seems largely fine.  At the very least, it seems like it generally shouldn't ever need an FCP.

The one place where I can think it might at least deserve a `@rustbot second` kind of thing from a team is if it's a de-facto deprecation of something owned by that team.

For example, a lint that basically says "never use this function" should probably get a libs-api second.  A lint for "don't use `as`" might deserve a lang second.

But all the "here's a simpler way to do this" lints I don't think need approval.  Say a lint for "this will always panic given the arguments you passed" it denoting the specified behaviour of the construct, so I don't think would need a team sign-off at all.

yaahc: I would worry about making API stability guarantees in this example scenario

scottmcm: I only meant things like `.step_by(0)`, where it's a `# Panics` documented behaviour that yes, it always panics for `0`.


### scottmcm: Some of our naming lints help elsewhere

For example, we're subject to the same patten ambiguity as mentioned in https://www.cs.princeton.edu/~appel/papers/critique.pdf

We mitigate this largely by convention -- if you follow the naming conventions, then you're more likely to get warnings about bindings where you thought you wer matching a `const`, for example

nikomatsakis: I think this may be the distinction between what Felix terms "uniformly clean" and "expressly clean". The other distinction would be "things that affect users of your crate" -- e.g., non-standard method names have an impact on people who invoke your methods, but `while true` does not. I do think we should endeavor to draw some categories, and I'm not convinced that rustc should be (e.g.) enforcing `while true` lints, or at least not sure that it makes sense to do so without ALSO doing a bunch of other useful lints that clippy contains.

pnkfelix: I think scott has identified a case where we are using a lint to overcome a *weakness in the language* (in terms of there being footguns that one can run into if one doesn't follow existing naming conventions). That doesn't exactly fit the model I had for "expressly-clean" in my head, but I do think it seems like a case I should consider, and figure out which category it belongs in. (It *definitely* is one that should stay in the compiler and thus the language as well, until/unless the language is fixed so that the lint is no longer necessary.)

### scottmcm: I think there's a place for deny-by-default lints in rustc that aren't errors

My go-to example here is things like the "this is going to overflow at runtime" lint.  I think it's better for something like that to not be a *guarantee* of the language, especially since specifying exactly what it does or doesn't detect seems like it would be very difficult.  (Particularly when it starts doing smart things like flow-aware const prop.)

### josh: why is "unstable_name_collision" considered an exception?

This seems like exactly the same premise: warning about a compatibility issue in the future, because we've reclaimed syntax.

(Off-topic aside: I do think this lint has a different problem, namely that it suggests to people that they migrate their code to explicitly use the old version, which means people won't get any future indication to migrate to the new version. But that's off-topic for lint policy.)

### josh: is this policy meant to apply only to warn-by-default lints?

I'm hoping we can still integrate plenty of allow-by-default lints that people may wish to enable.

scottmcm: Can you elaborate on why these being in rustc in particular is important?  Anything that's "here's a kind of dialect" (because it's not something we tell everyone) seems like it's more fine for clippy.

josh: sure, see the point immediately below. :) And also, things in rustc rather than clippy are more readily accessible, which means a far higher percentage of people will actually use them.

scottmcm: I read the point below as being about on-by-default lints, rather than allow-by-default ones.

josh: True. I do think there's alue in having the allow-by-default ones for experimentation purposes as well; some of those get bumped up over time.

### josh: rationale for having these kinds of lints in rustc rather than clippy

Our lints have a norming effect on the Rust ecosystem. clippy lints *mostly* don't; they only affect people who opt into them. So, we push things into clippy that are either sufficiently heuristic that they might be wrong or sufficiently subjective that they're not worth the cost of attempting to enforce them. But lints in rustc do steer people in a direction, and it takes effort to disable them vs just go along with them. This has a substantial net win in unifying Rust style across the whole community. (e.g. naming and capitalization lints, unused lints...) We have what seems to be a near-universal "never ignore warnings" culture, and I think that has had great effects.

I agree with part of the premise, that we should avoid taking on too much work. I'm happy to delegate more. I think we should pay close attention to signal-to-noise ratio, and get better feedback. But I think we *should* have subjective lints in the compiler, and I think we get huge community benefits 


### mark: assumption about running Clippy?

My own experience with Clippy is that the default settings and upgrade story continues to cause CI failures in projects using it, which makes me unhappy to hear about lints migrating out to it -- I would rather see intentional work to adopt lints into rustc when they have a clear enough bar on noise to signal ratio.

pnkfelix: this is a good point. Is a reasonable response "should we figure out how to make clippy *easier to deploy*?"

josh: I would favor adopting lints in rustc that have near-perfect signal-to-noise.


### scottmcm: I think we should have more deny-by-default lints for certain things

The match issue I mentioned above (related to https://www.cs.princeton.edu/~appel/papers/critique.pdf) is a place where we're relying on getting a bunch of warnings for things that I would rather be deny.

So perhaps part of the reason people are doing `-Dwarnings` is that there are enough "this is almost certainly wrong" situations where we're *not* denying them today.





