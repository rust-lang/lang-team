# Lang team roadmap

Rust 1.0 was released in 2015. Since that time, we've seen Rust grow from a small language used for a handful of prominent projects into a mainstay in use at virtually every major tech company. As we start the Rust 2024 development cycle, it's natural to ask "what's next" for the language. This roadmap provides insight into that question by describing what we, as members of the lang team with input from other Rust teams, would like to prioritize.

The goal of Rust has always been to **empower everyone to build reliable and efficient software**. Achieving this goal requires not only designing and implementing a great language with great libraries and great tools, it requires building a great and supportive community. Widespread adoption of Rust creates a positive cycle by ensuring that there is a broad base of knowledge about best practices in Rust and a wide set of libraries available on crates.io to make building programs easy. It also means that we have more people contributing ideas for how to make Rust better. Everybody wins!

As we grow, we face increasing challenges in how we can scale the ways in which we empower people to an increasing number of people. This roadmap presents three general themes we plan to focus on, which will help the overall ecosystem scale:

* **Help Rust's users help each other**
    - Empower library authors so they can---in turn---empower their users.
* **Flatten the (learning) curve**
    * Make Rust more accessible to new and existing users alike, and make solving hard problems easier.
* **Help the Rust project scale**
    * Develop processes to scale to the needs and use cases of a growing number of users; evaluate and finish projects we've started.

We intend this roadmap to evolve over the course of the Rust 2024 development cycle. In 2022, we expect to establish foundations for further work, and start conversations and initiatives to develop the remainder. The document is focused more on the *themes* of work that we think are important than the specific items within them. For each theme, we give some examples of specific work items that fit that theme, but we've left things deliberately open-ended; the items representing foundational work for 2022 are more detailed than those representing later work. Some of these items we have concrete plans for; others require more extensive design work. Our hope is that people who want to contribute to Rust will use this document to get ideas for good "starting off" points in terms of what kinds of work we think is most needed, and what kinds of work we're most willing to mentor and review.

These themes are not meant to be an "exhaustive list" of all the things that the lang/libs team would consider doing. In particular, we always welcome opportunities to make small improvements to the language. If you have an idea that you think has promise, please feel free to [open a proposal](https://lang-team.rust-lang.org/initiatives/process/stages/proposal.html) and tell us about it! It's easy and we'll let you know if we think that this idea is a good fit for Rust right now.

## Theme: Help users help each other

* *The vision:*
    * Library authors (and consumers) have tools to manage portability.
    * Library authors have tools to manage their development lifecycle.
    * Library authors can tailor the developer experience.
* We achieve this by empowering library authors with new language capabilities, including those previously reserved for the compiler and standard library, such as lints, diagnostics, and stability attributes.

Libraries are key part of how Rust empowers its users, but developing and maintaining a Rust library should be easier. The standard library and compiler have access to a number of unique capabilities, ranging from diagnostics integration to the stable/nightly development cycle, that other Rust libraries cannot take advantage of. For Rust 2024, we want to empower library authors with new capabilities that make it easier to author and maintain Rust libraries.

We encourage people to experiment and explore in the library ecosystem, building new functionality for people to use. Sometimes, that new functionality becomes a foundation for others to build on, and standardizing it simplifies further development atop it, letting the cycle continue at another level. However, some aspects of the Rust language (notably coherence) make it harder to extend the Rust standard library or well-established crates from separate libraries, discouraging experimentation. Other features (such as aspects of method resolution) make it hard to promote best-in-class functionality into the standard library or into well-established crates without breaking users of the crates that first developed that functionality. For Rust 2024, we want to pursue changes that enable more exploration in the ecosystem, and enable stable migration of code from the ecosystem into the standard library.

### Examples

Here are some currently active initiatives that full under this theme:

* The async working group's [portability](https://www.ncameron.org/blog/portable-and-interoperable-async-rust/) initiative (which builds on the work to support [async fn in traits](https://rust-lang.github.io/async-fundamentals-initiative/)) will help the async ecosystem to grow by enabling more interoperability.
* RFC 3240 proposes [edition-based method disambiguation](https://github.com/rust-lang/rfcs/pull/3240), to support moving extension methods from external crates into the standard library.
* [Negative impls in coherence](https://rust-lang.github.io/negative-impls-initiative/) allows for more flexibility in the coherence check by permitting crates to explicitly declare that a given type will never implement a given trait.
* There are numerous core extensions to Rust's type system that permit richer traits to be developed. Often the lack of these features prohibits people from writing general purpose libraries because they can't get sufficient reuse:
    * [Async fn in traits](https://rust-lang.github.io/async-fundamentals-initiative/)
    * [Const generics](https://github.com/rust-lang/lang-team/issues/51) and [constant evaluation](https://github.com/rust-lang/lang-team/issues/22)
    * [Type alias impl trait](https://rust-lang.github.io/impl-trait-initiative/explainer/tait.html)
    * [Generic associated types](https://rust-lang.github.io/generic-associated-types-initiative/)

Here are some examples of initiatives that we would like to see happen, if someone were to step up to lead them (and to help resolve the numerous hard problems that are involved in each one):

[RFC 2492]: https://github.com/rust-lang/rfcs/pull/2492

* Support "global services" like allocators or async runtimes, perhaps via an approach like [RFC 2492], and perhaps extending to something like [scoped contexts and capabilities](https://tmandry.gitlab.io/blog/posts/2021-12-21-context-capabilities/).
* Revive the stalled [portability lint](https://github.com/rust-lang/rfcs/pull/1868) or pursue an alternative design (a recent suggestion is that the "platform" might be a global service, similar to [RFC 2492], permitting one to use where clauses to designate portable code)
* Allow libraries to provide custom lints for their users.
* Allow libraries to control or customize Rust diagnostics, especially for trait resolution failures.
* Adopt a standard way to write performance benchmarks (perhaps simply adopt `criterion`).
* Make stability markers available to ecosystem crates.
* Constrained dynamic linking - support generics via runtime dispatch and trait objects
    * Inspired by Swift's approach
* Variadic tuples - avoid manual (slow) macros that implement things for every size of tuple
* Find ways to extend coherence across crates in a workspace or to designated "joiner" crates.
* The expressiveness of Limits of trait expressiveness that prevent people from writing libraries, particularly in async space
    * Async fn in trait
    * [Type alias impl trait][tait]
    * [Generic associated types][gats]
    * Variadic generics

## Theme: Flatten the (learning) curve

* *The vision:*
    * Rust helps you focus more on the "inherent" complexity of your problem, without added complexity from Rust such as the need to copy-and-paste where clauses or insert `&` or `&*` in strategic locations to guide the type checker.
    * Rust helps you manage the "inherent" complexity of key domains like Async I/O or embedded programming, and makes the most natural development path lead to maintainable, high-performance code.
* We achieve this by better integrating Async I/O, improving the compiler's ability to leverage information it already has (e.g. trait bounds, `Copy` types, type sizes), and making existing language features work smoothly together (e.g. `-> impl Trait` and trait functions).

Rust is consistently beloved by those who have already learned it, but that doesn't make Rust easy to learn. We know that some people hit Rust and bounce off. And even those who have learned Rust still run into cases where they've had to learn patterns and idiosyncracies to accomplish some complex task, where we could instead make that task obvious and straightforward. Our goal is to make Rust no harder to learn than other languages.

Over the last few years, the Async and Embedded working groups have worked to enable Rust users in those domains. We have made a lot of strides, and these two domains are large growth areas for Rust. Our goal for Rust 2024 is to make working in those domains not only *possible* but *straightforward and delightful*.

### Examples (subject to change)

[initiatives]: https://lang-team.rust-lang.org/initiatives.html

Current [initiatives] that fit in this category, where we've done substantial design or implementation work:

* [Dyn upcasting coercion initiative](https://github.com/rust-lang/dyn-upcasting-coercion-initiative/issues/6): Allow upcasting `dyn trait` objects from `&dyn Subtrait` to `&dyn Supertrait`.
* Many parts of the [async roadmap](https://rust-lang.github.io/wg-async/vision.html), including
    * [Async fundamentals](https://rust-lang.github.io/async-fundamentals-initiative/): Support `async fn` in trait, async drop, and async closures. (This, in turn, enables simpler standard `AsyncRead` and `AsyncWrite` traits.)
* [Extending impl trait](https://rust-lang.github.io/impl-trait-initiative/): Permit `impl Trait` syntax in more places, such as [type aliases](https://rust-lang.github.io/impl-trait-initiative/explainer/tait.html), [fn return types in traits and impls](https://rust-lang.github.io/impl-trait-initiative/explainer/rpit_trait.html), etc.
* [More precise borrow check](https://github.com/rust-lang/polonius/): Non-lexical lifetimes were a big stride forward, but the [polonius project](https://github.com/rust-lang/polonius/) promises to improve the borrow check's precision even more.
* [let-else](https://github.com/rust-lang/rust/issues/87335): Direcly express the "match this variant or `return`/`continue`/etc" pattern.

Examples where we have an idea of what we want to accomplish, but the plan is either unformed or in need of design:

* [Implied bounds](https://rust-lang.github.io/rfcs/2089-implied-bounds.html): Remove the need for redundant or boilerplate where clauses from `impl` blocks.
* [Deref patterns](https://github.com/rust-lang/lang-team/issues/88): Permit matching types with patterns they can dereference to, such as matching a `String` with a `"str"`.
* **Autoref, operators, and clones**: Generic methods that operate on references sometimes necessitate types like `&u32`; since `u32` is `Copy`, we could automatically make it a reference. We've historically had some hesitance to add more reference-producing operations, because it can lead to types the user doesn't expect (e.g. `&&&str`). We have some ideas to simplify those cases and avoid unnecessary double-references.
* **Perfect derive**: determine the precise conditions for generic type parameters based on the types of a struct fields (e.g., `#[derive(Clone)] struct MyStruct(Rc<T>)` would not require `T: Clone`, because `Rc<T>` can be cloned without it).

## Theme: **Help the Rust project scale**

* *The vision:*
    * Rust users can tell the status of any Rust feature, to evaluate whether you can depend on it and how you can help with it.
    * We empower developers to improve Rust for their use cases, scaling and delegating language development without compromising quality, stability, and coherence.
* We achieve this by refining our processes for [managing initatives](https://lang-team.rust-lang.org/initiatives.html), [making decisions, handling dissent](https://lang-team.rust-lang.org/decision_process.html), and scaling the team.

Rust has a lot going on. As of this writing, there are [618 open tracking issues on rust-lang/rust alone](https://github.com/rust-lang/rust/issues?q=is%3Aopen+is%3Aissue+label%3AC-tracking-issue), [176](https://github.com/rust-lang/rust/issues?q=is%3Aopen+is%3Aissue+label%3AC-tracking-issue+label%3AT-lang) of which are for the lang-team. At the same time, there are currently only [5 members of the lang team](https://www.rust-lang.org/governance/teams/lang), which limits both the speed at which we can resolve issues and the number of features that we can personally help to mentor. The result is that Rust development often feels like it is moving both ridiculously fast and glacially slow at once.

To improve the situation, we aim to do a combination of four things for Rust 2024:

* Develop decentralized, experimentation-friendly processes like the [lang-team initiative process](https://lang-team.rust-lang.org/initiatives.html). The goal is to empower people to contribute to the Rust language without having to be a member of the lang team; it also gives us a better "on-ramp" to discover and recruit new members.
* Improve our processes for incorporating feedback so they are more focused and less overwhelming for participants (RFC threads have a bit of a reputation). [Handling dissent]
* Create visualization tools, like the [Initiative Project Board](https://github.com/orgs/rust-lang/projects/16/), that identify the ongoing work that the lang team is doing.
* Through processes like [Backlog Bonanza](https://blog.rust-lang.org/inside-rust/2020/10/16/Backlog-Bonanza.html), reduce and unstable features. For features that don't have a path to stabilization, remove the code from the language and compiler. For issues that do, we should be able to easily state what is blocking it from making progress.



# Meeting discussion notes

## What do I do now?

After everyone is doing reading the doc, we will have questions and discussion. Please write your comments here as a fresh `##` section. Put the question details and your name, and we'll take notes there when we discuss. Thanks!

## Q: Purpose of roadmap

nikomatsakis: It occurs to me that it would be good to clarify -- for ourselves, but also potentially publicly -- the purpose of this roadmap. In my view it is a combination of

* letting people know what we are focused on (and that includes ourselves)
* giving people ideas of the sort of initiatives and proposals that are wanted
* giving a "signal boost" to some of the specific examples, without necessarily committing to them :)

pnkfelix: goals seem good, but maybe we can do better at achieving them

joshtriplett: yes, that is the right general idea, the themes are more important than any one specific goal listed as an example, but I think some of the examples are important, themes are more important

pnkfelix: one question I had about doc itself-- first comment I had was "What is the scope of this thing"? Lang? Libs? 2024? 2025? 

nikomatsakis: targeting 2024. focused on what lang can do, but a lot of the themes are themes I would like to see the Rust project pick up, and which cannot be "truly done" just by lang alone

joshtriplett: lang focused; the scope describes things that will happen in other teams, but the items are things that we can do to enable it.

pnkfelix: specific places where I got confused was benchmarking. e.g. criterion.

joshtriplett: portion is the `#[bench]` attribute, but it is primarily a tooling question. There are aspects in which it's integrated in the language.

nikomatsakis: I thought about removing that when integrating the list, though I agree with what josh said.

scottmcm: the whole distributed slice-- an attribute that collects a bunch of things would be helpful. Maybe `#[bench]` is not the lang feature, but an attribute that collects instances of a trait or whatever would be a general enabler.

nikomatsakis: re: team, I don't want to get stuck in deciding whose team is whose, but also I think we can't address user needs without cross-cutting multiple teams, not sure the best way to thread that needle.

scottmcm: the themes seemed familiar from other roadmaps.

pnkfelix: themes that are here that aren't in this picture.

nikomatsakis: the way I saw this was that it's a moment in time -- these are the biggest, most important themes for 2024. If we make great progress, we can add more themes in 2023 (or even later in 2022), but I'd rather make that decision later on when we know more about how much we got done. If I had to pick 3 themes.

nikomatsakis: so it seems like we reach a consensus, should we saying more of this explicitly?

pnkfelix: I think yes.

## Q: Ordering the themes?

Mark: Are the three themes intended to be ordered? I might want to reverse them (help us > let users help each other > learning curve), since if we can't scale the latter two seem hard.

Josh: They weren't intended to be ordered. Which raises the question, *should* they be? I think they shouldn't, but this seems like a topic for discussion.

lcnr: Even if not intended, I do feel like there is some unavoidable prioritization implied from the ordering. For me it tends to feel like the first theme is the most important, which at this point probably should be "help us"?

nikomatsakis: I think the vast majority of readers are Rust *users*, not contributors, and so I would want to emphasize "what will happen to them". But also I think things have improved somewhat in terms of desperation. :)

joshtriplett: put another way, if you want to know how you can help, the *things to help with* are the first items, helping "make the project scale" is not as much something many people can or will contribute to. The items in the first lists are the ones that we want people to contribute to.

simulacrum: This may be a distinction in wording-- the way I see it is, the team of people wokring on solving coherence, is helping Rust scale, enabling Rust project to care about that issue. Whereas the first two seem more user-oriented. Helping Rust users help each other, we can't do that if the Rust project can't scale.

joshtriplett: Agree, I think we can expand the 3rd theme to make it clear that we aren't going to accomplish what we want if we don't make it so that the project can make it scale. But I want to make sure that we put front and center the things that the audience cares about and says "oh btw to get this done we are going to have to make ourselves scale better".

simulacrum: I guess I disagree, if this doc is being read by high-level management at a company, and they see the third point, they may decide "we don't need to hire people to work on rust project because they don't think that this is a priority".

joshtriplett: I don't know. If we think that we could actually get 20 people helping with Rust process...I don't really think we have work for 20 people on Rust process right this second, or, we probably do, but we're not ready to accept useful contribution at that scale. But there's work for 50 people on the first two points right now.

pnkfelix: I'm torn, on the one hand, I see Josh's point. In the compiler roadmap I said we won't discuss process changes. Having said that, while I don't think the audience is going to care about helping with this problem, I do think the audience will care about the visibility that is enabled by those process shifts. Depends a bit on what Mark said, whether we'd see investment. "Recognize there's a problem, don't have easy way to get visibility"

nikomatsakis: Could we thread the needle? Describe it in the intro-- to reach those goals, we will need people to help, and that's why we have included goals to improve the processes.

joshtriplett: at minimum could add something to the 3rd section, stating that there is some work we will need to do to make it possible for others to contribute in this area, but we need to prioritize that work and get people contributing.

simulacrum: main thing I think is missing, maybe other things help, is that -- if I read this -- it sounds like "oh the first two, the rust project already has a team of people who will really be able to solve the first two, but the third problem, yeah, they have some process, they'll work through it". No real discussion of "yes actually we want 20 people to work on coherence" or whatever. Including some of that will help us go to people and say "you will come".

joshtriplett: Sounds like a problem for all 3 sections and the intro. We should explicitly invite help across the board. This roadmap says "this isn't just work we are going to be doing", but we should spell that out more explicitly, "these are things we want people to drive". We don't actually really want people making a proposal first and foremost, 

nikomatsakis: it feels like want a "Help wanted" section. Feels like good feedback.

pnkfelix: One thing I think was really important in compiler roadmap was spelling out concrete goals vs aspirations (things that have no allocated time). But *everything* had an owner, even aspirations-- here is the person you contact if you are interested in helping. Some of the things in this document don't yet have owner.

nikomatsakis: could at least say, for everything we list, "if you'd like to push on this, here's the next step".

## meta-Q: How much should we talk about the examples?

Scott: In some sense it feels like the examples are non-normative, but they also give some officialness to them.  How much should we talk about specifics of them in this meeting?

joshtriplett: in this meeting, let's not go into the specifics of any one example. If there are aspects of a theme that is missing, let's talk about that. If something seems way off, let's discuss it, but not in the technical details of any one thing.

nikomatsakis: they're here for a reason, to spark some conversation about whether they are good ideas, so I would like to have notes if people say that there are concerns .

pnkfelix: anti-examples could be potentially useful, we did that in the compiler roadmap, in terms of diverting people from a rabbit hole.

nikomatsakis: what would a good anti example be?

pnkfelix: effect systems? I don't know.

nikomatsakis: one example, to be controversial, might be const generics in 2022.

scottmcm: variadic generics feels like "not this year".

## Q: Niko notes a few things about the final section

I was thinking just before the meeting that I would like to revise the final section in its emphasis to be more about "painting a picture" of the end goal we are trying to reach, which I think is basically

* project board clarifying what is ongoing and its status
* clearer entry points plus "path to team"
* ability for team to easily delegate our and run initiatives in parallel
* (maybe?) relatively narrow time from "start to finish" for things -- i.e., we don't have a ton of things hanging out in limbo
    * this is an interesting one to debate about

## Q: Which active initiatives are missing?

nikomatsakis: Glancing at the [project board](https://github.com/orgs/rust-lang/projects/16/views/1), and scraping my memory, here are active efforts that are not listed. Should they be?

* ffi-unwind -- not sure if/where this fits, actually, maybe nowhere (and that's ok)
* try blocks, error handling -- feels like "help users help each other" but also "ergonomics" to me
* never type -- ?
* safe transmute -- seems like it fits a slightly different theme, the "safer unsafe code" theme, but we left that out because it felt like something that we weren't ready to commit to in 2022 (it may be a late addition in 2023, in my mind, if we lay enough groundwork this year)
    * Might fit in the category of making harder areas more feasible, if we expand embedded/kernel to embedded/kernel/systems or similar? Don't want to expand too much though.
* generators -- ergonomic win definitely, but I think kind of stalled out right now


## Q: Users helping each other: "...enable stable migration of code from the ecosystem into the standard library."

Mark: I might want us to gesture toward a *broader* set of options, e.g., maybe std never has regex, but regex is first-class supported (perhaps even more so than today, with rmeta binaries shipped with Rust).

Scott: +1.  It might make it less lang, but I'd prefer some phrasing around discoverability and easy of use of crates instead.

Josh: I think there's a self-similarity here: we want to make it easy to migrate things from crates into std, but also make it easy to migrate things from experimental crates to well-established crates. This isn't just about extensions to the standard library, it's also about extensions to libraries in general.

nikomatsakis: agree with all that, just want to hold the position that we're not always so aware of the costs of NOT standardizing, I do want some language to make clear that we find that useful. 

yaahc: I know Mara has been talking about a library ecosystem team, and this seems to fall somewhat under that umbrella. I agree with Niko's earlier comment that probably this is a libs thing.

nikomatsakis: Mara and I have argued about this a fair number of times too. It's definitely possible to go too far in either direction. I think we're too far on one side for async, for example, but adopting a single, standard runtime feels too far on the other. Traits might be a good.

## Q: Learning curve: "...We achieve this by better integrating Async I/O"

Mark: I am surprised to see async called out specifically here, rather than the broader class in the problem description -- is async intended to be special here? (At the very least, I would not expect *i/o* to be special.)

Josh: :+1: for dropping "I/O" as opposed to async in general. async is both a specific instance and a member of the general category of hard problems that are now possible but could be easier.

nikomatsakis: I believe async is special, though, as an area that needs custom attention and where we have a LOT of new users in particular.

## Inspired by the perfect derive note and a bit about giving libraries easier tools

Scott: Maybe we could make the "easy" cases of derives not require proc macros somehow?  They might always be needed for things like the fanciest diesel stuff, but the "normal map reduce" derives feel like they could be way easier

nikomatsakis: something like a "metaderive"? I've thought about this from time to time. 

pnkfelix: see also "polytypic programming"

## C++ interop?

pnkfelix: I saw some rumblings on twitter about how important C++ interop is (and "has always been") for Rust's success. I couldn't identify where it falls in any of the given three themes, nor was it explicitly listed. Is this just "yet another feature" (or set of features to support that use case), and does not need to be called out in this roadmap?

joshtriplett: If it were going to go anywhere, I think it falls under "helping users help each other", in so far as it makes it easier to write libraries that build on C++, makes it easier to reuse code written in C++, feels like it's in that category.

pnkfelix: If you squint, but then what doesn't?

joshtriplett: I don't think it's another category on par with those themes, but it's "do we want to add that as an item".

simulacrum: I think the cxx library and bindgen are both examples of "mostly working with caveats" tools today that, with some lang/compiler changes, I could imagine improving their workflow and/or standardizing on one that enable those to be best-in-class solutions. I think it fits very well under the first theme.

yaahc, in chat: also, doesn't cxx make assumptions about std ABIs?

nikomatsakis: I think it wants to be a working group if we're going to really make progress, it's kind of cross cutting, but I agree with Mark that most of the work to be done (from a lang perspective especially) is about enabling people in the ecosystem do that work. 

pnkfelix: I'm on board with that viewpoint that Mark expressed.

nikomatsakis: The whole point of that bullet, now that I think about it, was to try and minimize what has to be done in the rust-lang org and enable the ecosystem to move faster. I'm not sure if that point got lost a bit in the narrative. 

pnkfelix: case study here could help.

## Ensuring doc accomplishes its goal(s)

pnkfelix: We established three over-arching goals for the doc. E.g. raising awareness and encouraging engagement/volunteer efforts. If we're serious about wanting to encourage engagement, then need to do exercise of putting selves in reader's shoes: "Wow, this is exciting! What do *I* do next if I want to help?" From reading this document, I don't think someone knows what to do next. (Compare against compiler-team roadmap, where we attached owners to specific goals, and also included a FAQ spelling out what readers should do next if they want to get involved.)

nikomatsakis: yes, and I think we talked about 

pnkfelix: we also made a channel for people to chat 

nikomatsakis: we could play up a bit more the short term part of it

pnkfelix: spelling out what is short term and what is not would be a big help, pulling up the process changes is a good eaxmple, doesn't help to spend 3 years.

nikomatsakis: maybe we can block out a rough chart of "active", "could be active now", "blocked but will start", "long term"

pnkfelix: logarythmic scales!!


