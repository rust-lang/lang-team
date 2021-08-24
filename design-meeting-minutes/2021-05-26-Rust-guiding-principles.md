---
tags: design-meeting
---

# 2021-05-26: Rust's Guiding Principles

## What is this?

A rough draft for a charter and set of "guiding principles" for Rust. These principles arose out of the Async Vision Doc. Initially I was trying to draft principles just for Async Rust, but I found that they were evolving into something that seemed applicable to all of Rust. My expectation is that we will take these principles and write a *specialization* of them for different domains. I expect to shop these around to other teams as well. --nikomatsakis

## Rust's charter

Rust empowers everyone to build reliable and efficient software, by building an **accessible language for systems programming** and a **healthy community and ecosystem around that language**.

### Guiding principles of Rust's design

The charter declares what we are trying to achieve. This section lists the design principles that tell you how we do it. These principles govern the design of the Rust language, its libraries, and the tooling built around it. These principles are ordered: in the case of unresolvable conflict, we favor things earlier in the list.

* **Correctness first.** We strive to make Rust hard to misuse[^rr]. Safe Rust code is memory safe, data-race free, and free of other kinds of [undefined behavior](https://en.wikipedia.org/wiki/Undefined_behavior). What's more, the Rust language, library, and tooling encourage robust, reliable programming patterns. If your program compiles, it does what you expected it to do (usually...).
* **Fill all niches, efficiently.** Rust targets programs that work closely with hardware, control low-level details, and make careful use of available resources. Idiomatic, natural Rust is highly efficient and parallel friendly. We expose the full capabilities of your machine so that we can fill all niches, from embedded systems to kernel modules to distributed servers to web and WebAssembly to GPUs.
* **Wears well over time.** Rust programs are meant to last a long time. We encourage quality craftsmanship and long-term maintenance.
* **Productive and accessible.** Rust is productive and a joy to use: common operations are easy, and Rust offers top-notch tooling with friendly, accessible interfaces. Rust avoids overly complex designs and is meant to be understood and used.[^aspirational] 
* **Portable and interoperable by default.** By default, Rust programs work well across all common operating systems and architectures. They adopt the native conventions of their environment and can be integrated into existing programs, whether that be through through linking, IPC, networking protocols, or other means.
* **Enable exploration.**[^py] Nobody has a monopoly on good ideas. We encourage people to try new approaches and alternatives. We believe in a rich ecosystem of cooperative, interoperable crates; however, we also recognize that sometimes the best way to enable exploration is to standardize (judiciously).

[^py]: Also known as, "Crates are one honking great idea, let's do more of those."
[^rr]: Hat tip to [Rusty Russell](https://ozlabs.org/~rusty/index.cgi/tech/2008-03-30.html).
[^aspirational]: Aspirational at times. We do our best!

### Guiding principles of Rust's community

TBD! I'm still in the brainstorming phrase here. 

## Examples

These principles were derived in part by working through various examples and tradeoffs that we have struck in the past. This section works through some examples and also shows the power -- and limits -- of having named principles. It's not that they make the decisions for us, but they give us a vocabulary to use when discussing the tradeoffs.

### Bounds checks and iterators

We insert bounds checks on accesses to arrays because "correctness first", and that matters most. However, that rules out some domains or extreme cases, so we do expose unsafe methods that let you skip the bounds checks; further, we encapsulate those unsafe methods within iterators and encourage safe Rust to use those ("fill all niches, efficiently"). We don't do fancier things like requiring you to statically prove your accesses are in-bounds because we value "productivity and accessibility" and that would be complex.

### Auto traits and auto trait leakage

We chose to make `Send` and `Sync` automatically derived because idiomatic Rust is parallelizable by default ("fill all niches, efficiently"). This is in tension with long-term maintenance ("wears well over time"), since changes to the fields of structs can make things no longer `Send`.

The Copy trait was initially an auto trait as well, but it was changed to be "opt-in" precisely because of "wears well over time". The semver effects were thought to be too great, and the benefits too few.

### Copy, clone, and auto-clone

Early in Rust's design we were focused on what operations needed to be explicitly acknowledged, particularly around copying data. Experience from C++ copy constructors had shown that implicit, non-trivial copies could be a real performance footgun; at the same time, we couldn't come up with a good rule for how much memory was "too much". We opted for *any memcpy is ok* but more complex things require an explicit operation.

This is clearly a tradeoff between "fill all niches, efficiently" and "productive and accessible", but it's not a *clean* one. The current rules do not always encourage the most efficient thing: creating a Rc or Arc might be much more efficient, but it's syntactically penalized. Moreover, the current rules clearly work against productivity and accessibility from time to time (balancing `clone` calls can be tedious; clone seems to be unidiomatic, but often it's the right thing).

### Small standard library

We intentionally aimed for a small standard library because we wanted to "enable exploration". Experience from Python had shown both the *power* of its "batteries included" philosophy, which favors "productivity and accessibility", but also its [downside](https://leancrew.com/all-this/2012/04/where-modules-go-to-die/). I think this decision has worn well over time -- if anything, the standard library should probably have been smaller still.

At the same time, one of the most common complaints about Rust and cargo is that it is too hard to find crates or figure out which one to use. The current design I think has over-rotated on "enable exploration" and could use to be tilted back towards "productivity and accessible". 

In the case of async, "enable exploration" is not being well served. Building new libraries is too hard because of the lack of traits for interoperability.

### Coherence and enable exploration

The coherence rules support "correctness first" (a primary motivator was subtle bugs where there are two distinct hashing implementations for a given type) but clearly lean heavily against "enable exploration". Can we find a better balance?

## Things to discuss in the meeting

* The principles themselves, and their ordering.
* Examples of prior decisions, particularly those that demonstrate the ordering *or* contradict it.
* Implications of these principles on ongoing design work.

# Question queue

* Felix: does "correctness first" being first on list *actually* align with how we actually prioritize work? Consider e.g. the number of soundness bugs that we let lie unaddressed because they don't appear to affect people in practice. (Or is "correctness" modulo utilization? If a bug is not observed, it does not get same priority level?)
   * Josh: This does suggest a tradeoff between correctness-first and compatibility. Should backwards-compatibility (there will never be a Rust 2.0) be somewhere on this list? "Wears well over time" could address it, but I think in practice backwards compatibility as a value belongs higher than 3rd on this list. Correctness is one of the *rare* cases where we will consider (carefully) breaking compatibility though.

* Felix: The "guiding principles of Rust's community" seems as good a place as any for me to give a shout-out to the (in-progress) [compiler contributor guidelines](https://hackmd.io/TYGRjVIbSBmxpbcfDzll-w).

* Josh: Is it intentional that "Enable exploration" is lower priority than "Portable and interoperable by default"? I can think of cases where those would come into conflict, and it's not always clear that "portable and interoperable" should take priority. (For instance, it should be easy to write and experiment with target-specific code, and you shouldn't be forced to create things that work on all platforms. We make it reasonably straightforward to build target-specific crates.) Are there examples where, in practice, we would prioritize "portable and interoperable" over "enable exploration", and what would those look like?

* Niko: A question I had while reading was "we prioritize being able to do things in libraries, where is that?" but I realized that it's an aspect of "enable exploration". Maybe worth highlighting in the text.
    * and/or "accessible"
    * and/or "maintainable"

* Niko: I think the order of "Wears well over time" should maybe be below "productive and accessible" in practice. One could consider autotraits to also be a kind of accessibility/productivity enhancement, as an example. How much do we truly prioritize semver -- and how much *should* we? Also maybe "wears well over time" should be "maintainable over time"?

* Scott: Is it worth also describing things that we're *not* worried about?  Or, attempting to sketch it positively, something like "we expect to provide useful errors and warnings that people will follow".  I'm thinking of things like "we're usually not worried about saving a few tokens here and there".
    * Perhaps a separate list of guidance might be useful? Pithiness not always being a virtue could be one of them.

* Josh: Should we discuss the concept of how these tradeoffs interact with the passage of time? I think there isn't a conflict between "enable exploration" and the size of the standard library, for instance, as long as it's clear that there's a natural progression from people exploring to things solidifying in the crate ecosystem to things becoming part of the library or language.

* Felix: Re: "Copy, clone, and auto-clone", I think the story there is not yet finished. E.g. on the basis of one user story, Oli added a lint to catch moves over a certain byte size, which helps in some of the `memcpy` cases.

* Scott: How does this relate to things like https://blog.rust-lang.org/2017/03/02/lang-ergonomics.html#how-to-analyze-and-manage-the-reasoning-footprint ?  I like that post, but I don't know how to fit it in the 

# Notes

* Correctness first
    * Niko: Surprise is key factor to avoid
    * scott: Ties in with UB and lack of surprise due to effects at a distance.
        * I like the "effects at a distance" phrasing, Mark 👍  Reminds me of https://manishearth.github.io/blog/2015/05/17/the-problem-with-shared-mutability/ and "My intuition is that code far away from my code might as well be in another thread, for all I can reason about what it will do to shared mutable state."
* What does efficiency mean?
    * we can be used for 'all things'
    * phrasing could be improved, targeting 'all places', rather than 'niche' in the sense of low usage
* Exploration vs. standardization (judiciously)
    * Common problem in Rust, constantly fighting this balance.
    * Want to provide a foundation
    * How to standardize: identifying already shared territory, expanding on what can be explored
* Document is aspirational -- what we want to be doing


Questions pulled down:

* Felix: does “correctness first” being first on list actually align with how we actually prioritize work? Consider e.g. the number of soundness bugs that we let lie unaddressed because they don’t appear to affect people in practice. (Or is “correctness” modulo utilization? If a bug is not observed, it does not get same priority level?)
    * Niko: There are two separate things here: designs we make vs. implementation
        * e.g., an unsound design doesn't work at all
        * but engineering effort may not actually target soundness bugs specifically
    * Josh: Rust Language and Libraries must not allow UB (in safe code), but need to evaluate against the backwards compat correctness
        * scott: But backwards compat may not be correctness
        * (agreement this may not be right category)
    * Mark: Maybe our soundness bugs aren't in practice actually surprising
    * Niko: Part of what is missing is 'what is practical' -- for example, we care about what people encounter in practice
        * Productive & Accessible might cover that
    * Felix: There are some cases where we have standard-library-specific patches
        * e.g., may_dangle, for example
       
* Josh: This does suggest a tradeoff between correctness-first and compatibility. Should backwards-compatibility (there will never be a Rust 2.0) be somewhere on this list? “Wears well over time” could address it, but I think in practice backwards compatibility as a value belongs higher than 3rd on this list. Correctness is one of the rare cases where we will consider (carefully) breaking compatibility though.
    * Niko: Trying to keep the list short.
    * Josh, Felix, Niko: Feeling like this is part of wears well over time.
    * Josh: Seems like we prioritize wears well over time over the 'fill all niches, efficiently'
        * we slowly expand if we're not sure on interfaces
    * Josh: Feels like correctness > wears well over time, but not filling niches 2nd.
    * Niko: The lack of adding C features all immediately is because we prioritize fill all niches
        * trying to find examples for all priority pairs

* Josh: Is it intentional that “Enable exploration” is lower priority than “Portable and interoperable by default”? I can think of cases where those would come into conflict, and it’s not always clear that “portable and interoperable” should take priority. (For instance, it should be easy to write and experiment with target-specific code, and you shouldn’t be forced to create things that work on all platforms. We make it reasonably straightforward to build target-specific crates.) Are there examples where, in practice, we would prioritize “portable and interoperable” over “enable exploration”, and what would those look like?
    * Niko: I interpreted "fill all niches" as being able to access all OS-specific functions
        * I'm not sure as to the order on portability and exploration
    * Felix: usize / pointer interconversions are an active topic of discussion
    * Mark: I think the order makes sense *for the language* rather than the ecosystem
        * we usually don't land even unstable features if they're not possible to expand portably
    * Scott: Meta-question: how does this map onto lang vs. compiler and other teams?
        * Seems like the interface should largely be common, but the internal implementation of e.g., compiler, std, tools
        * Niko: would like to log cases where practical speed doesn't match this list, look at experience users are having

* Niko: A question I had while reading was “we prioritize being able to do things in libraries, where is that?” but I realized that it’s an aspect of “enable exploration”. Maybe worth highlighting in the text.
    * Early rust had lots of stuff built-in to the language
    * we moved lots of this out of the language into libraries
    * Josh: Small language / large library -- makes language & library easier to learn.
        * Some features are easier if it's diagnostic of compiler
    * Mark: Feels like this is not an exploration, rather, we put things in libraries because we fill all niches
        * Niko: Maybe we don't want exploration on the list at all, given this thinking
        * Felix: Maybe a meta-point of filling niches by distributed work (e.g., letting users do things in libraries lets us not concentrate that work)


* Scott: Is it worth also describing things that we’re not worried about? Or, attempting to sketch it positively, something like “we expect to provide useful errors and warnings that people will follow”. I’m thinking of things like “we’re usually not worried about saving a few tokens here and there”.
    * e.g., why not a pow operator? semicolon after a struct?
    * Niko: There seems to be a contrast: e.g., trailing separators for macros and diffs
    * Scott: Maybe a predictable look / not too many options?
    * Josh: "Not too many abbreviations" etc. falls out of productive and accessible & wears well over time
        * we want to prioritize long-lived maintenance
        * Niko: e.g., type annotations on functions and lack of that inference
    * Felix: Accessibility seems like a key piece of our story, where is that?
        * Niko: Correctness sort of targets this
    * Scott: Something about how we make decisions based on the behaviour of *incorrect* code may fit in here too -- thought of this based on the inference conversation, since part of why we require type annotations (when technically we wouldn't have to) is to give better error messages

* Josh: Should we discuss the concept of how these tradeoffs interact with the passage of time? I think there isn’t a conflict between “enable exploration” and the size of the standard library, for instance, as long as it’s clear that there’s a natural progression from people exploring to things solidifying in the crate ecosystem to things becoming part of the library or language.
    * Standardize lower layers to permit exploration on higher levels
    * How does the tradeoff between permitting exploration and wanting stabilization change over time?
        * Josh: Exploring in the frontier, while maintaining productivity in the common layer
        * example: future -> async -> (maybe) stream, asyncread/write

* Felix: Re: “Copy, clone, and auto-clone”, I think the story there is not yet finished. E.g. on the basis of one user story, Oli added a lint to catch moves over a certain byte size, which helps in some of the memcpy cases.
    * broad agreement that this isn't finished


* Scott: How does this relate to things like https://blog.rust-lang.org/2017/03/02/lang-ergonomics.html#how-to-analyze-and-manage-the-reasoning-footprint ? I like that post, but I don’t know how to fit it in the
    * Niko: I would argue that any particular dimension taken too high violates one of these principles
        * All of these are about correctness (Felix: or maybe accessibility, but that might be there too)
        * Arguing for how can we know the impact  on correctness is significant?

Next steps:
* Niko would like to talk to Josh to get a matrix
* Maybe talk to libs, tools
    * Mark: Would suggest not biting off too much -- focus on getting something for lang first.
    * Scott: would like more examples
    * Felix: a full 6x6 matrix, or just pairs?
        * Niko: Would like more, etc.
        
* Portability of the day decision was made