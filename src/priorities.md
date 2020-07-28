# Priorities

This page describes the current lang team priorities and explains
their motivations. This page is typically updated as part of the
yearly Rust roadmap process. They are derived from a combination of
the Rust survey results, feedback from users, and other soruces. When
new project proposals are created, they can cite priorities listed in
this page.

Each priority also lists lang-team members who typically prefer to
liaison for issues in this area. This can give you an idea of who you
might reach out to if you wish to discuss a project proposal.

**Last updated:** 2020-07-28

* Async I/O
    * **What?** Continued improvements with ergonomics and productivity related to Async I/O.
    * **Why?** Shows up heavily on the survey, this is an obvious area where a lot of Rust developers are working.
    * **Who:** [nikomatsakis], [withoutboats], [cramertj]
* C Parity, interop, and embedded -- these often overlap in 'low level capabilities'
    * **What**? Extending Rust's low-level capabilities to do "things otherwise only possible in C or assembly", as well as enabling smooth, ergonomic FFI between other languages and Rust.
    * **Why?** Embedded is a large factor in the survey.
    * **Why?** Our ability to act like "native C" is a differentiating capability for Rust. We've seen a lot of traction integrating into big companies on this basis, as a C++ replacement. It's clear that doing this requires the ability to do piecewise adoption.
    * **Who:** [joshtriplett]
* Const generics and constant evaluation
    * **What?** Supporting 
    * **Why?** Heavily requested feature and important for key areas.
    * **Who:** [nikomatsakis], [withoutboats]
* Trait and type system extensions
    * **What?** Specifically impl Trait, GATs, and specialization
    * **Why?** Long-standing areas that affect a lot of domains, including async
    * **Who:** [nikomatsakis]
* Error handling
    * **What?** Combination of [library related improvements](https://blog.yoshuawuyts.com/error-handling-survey/#conclusion) that consolidate "best practices" into standard library, documentation to describe how it works, as well as possible language improvements to leverage those changes (try blocks, `yeet`/`throw` keyword, etc).
    * **Why?** Cross-cutting productivity concern, and a persistent problem that makes working with Rust code more difficult than it should be.
    * **Why?** Anecdotally, something that comes up for a lot of people (see e.g. [nrc's #rust2020 blog post](https://www.ncameron.org/blog/rust-in-2020-one-more-thing/))
    * **Who:** [withoutboats], [joshtriplett]
* Borrow checker expressiveness and other lifetime issues
    * **What?** Think Polonius, RFC 2229, RFC 66, and other ideas like knowing which fields of `self` are used by particular methods.
    * **Why?** Learning curve remains a stubborn problem, and the best way to improve it is to make the compiler smarter.
    * **Who:** [nikomatsakis], [pnkfelix]
* Unsafe code capabilities and reference material
    * **What?** Document the rules for legal unsafe code and add features that either add required capabilities or make correct code easier and more ergonomic to write.
    * **Why?** Growing base of unsafe code, changes here are getting harder, this represents a kind of "reputation risk". We really want to be "better than C" here.
    * **Who:** [nikomatsakis], [pnkfelix]
* Targeted ergonomic wins and extensions
    * **What?** Small additions or improvements to make Rust easier to use.
    * **Why?** These will never rise to the "top of the list", but they often have outsized impact on people's enjoyment of Rust.
    * **Who:** [scottmcm]
* Soundness holes to try and correct
    * **What?** 
    * **Why?** Rust's appeal rests on safety. We have to make steady progress on these points. It's often hard to prioritize them compared to "whiz-bang" features. Also, long-running safety issues can cause fallout when fixed, weakening our stability guarantees.
    * **Who:** [nikomatsakis]

[nikomatsakis]: https://github.com/nikomatsakis
[withoutboats]: https://github.com/withoutboats
[cramertj]: https://github.com/cramertj
[joshtriplett]: https://github.com/joshtriplett
[scottmcm]: https://github.com/scottmcm
[pnkfelix]: https://github.com/pnkfelix
