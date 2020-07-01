# design meeting 2020-07-01

* [Recording available](https://youtu.be/3aw-5Fcyo7s)
* [Meeting proposal](https://github.com/rust-lang/lang-team/issues/26)
* Starting from safe transmute but generalized to specialized traits that let you determine when types are laid out in particular ways and can be interconverted etc
* Two main approaches
    * ZeroCopy -- not very granular
    * 'Typic' -- Expose type layout to the type system in a "perfect" way (or close to perfect)
* Most of the time, the traits from ZeroCopy are enough, and it'd be nice to be able to ask simple questions like "can this be zeroed"
* Probably want a combination. Use typic as the foundation but have simple traits exposed to users for common use cases.
* Question: Can this handle endian-ness cleanly?
    * typic -- not presently
    * zerocopy -- sort of
    * this kind of amounts to inserting types from byteorder crate, which will do conversions to/from native endianness as you do your accesses
* Is there a question for us to ponder/answer today?
    * John: I want to show how typic works
    * Ryan: I'm fairly convinced this functionality belongs in the stdlib.
        * Error messages are gnarly.
        * It's awfully unsafe and offers the ability to remove unsafe operations from library types.
        * Overall area is quite large, not quite clear exactly what are the smallest pieces we can start moving into stdlib, to ultimately get to a goal with a 1-line safe transmute?
* Is expectation that moving some part of this to stdlib would benefit from compiler support?
    * Yes.
* typic tries to solve an ambitious goal --
    * deprecating `mem::transmute`, such that if you find yourself calling it, you are probably doing something fundamentally unsound
    * but realizing that vision requires global knowledge of types, which typic lacks
    * so long term, realizing typic would really require compiler support
    * compiler has that ground truth of what the layout looks like anyways, so it'd be nice to build on that
* Josh: is there a clean obvious line like "if the compiler did this much, we could build on that in ecosystem" or is it necessarily the case that compiler/lib support wants to go in simultaneously?
    * We'll get to that.
    * Promising path might be:
        * Unsafe zerocopy traits with a marker attribute
        * Free to add an intelligent, automatic version, but until then libs can pick up some of the slack
* Screen share time!
* "As easy as 1..2..repr(C)" (not an actual quote, but should've been)
* How complex do we want?
    * zerocopy is based around "can this be made to bytes". Combined with dynamic alignment checks, this can cover a lot of cases, but not as many as typic. This covers e.g.:
        * interconvering a `&[Point]` to `&[f32]`, for example (hmm, adjust length?)
    * Examples typic solves that don't fit into this category
        * showing that two types with padding are interconvertible because they have padding in the same places
        * can permit interconversion between enums, understanding the discriminant values and so forth
* Some present shortcomings
    * typic doesn't understanding that `Option<&u32>` and `*const u32` are equivalent
        * public interface is fine for this
        * private perspective: sounds hard
* If the high-level interface of this is sufficiently capable, then do you believe that some of the internal details would be easier given compiler assistance behind the scenes?
    * Yes. I see the future of typic being that the compiler exposes some kind of unstable intrinsic with a predicate (`can_transmute<T, U>` that returns a boolean).
    * And a public facade implemented and based on that.
        * e.g., being able to do `where { can_transmute<T, U> }`
* Can in shorter time frame expose zerocopy traits like
    * `AsBytes<O: TransmuteOptions>`
    * `FromBytes<O>`
    * `Unaligned<O>`
    * we could move those traits to the standard library and annotate them with the marker attribute
    * stdlib can evolve how smart it is and until that time people can implement manually
    * carrying an options `O` type is essential
* What would next step for this be?
    * Josh: This approach seems single, it would make sense to have a path for implementing a nightly only prototype in the compiler to demonstrate that this gets easier with compiler support.
    * John: Having a `can_transmute` const fn predicate would be useful, as it would allow us to test our implementations on compiler ground truth
* Scott: How much do we want to do the whole from-bytes/to-bytes thing earlier?
    * Could be done in a library today with auto-traits?
    * Taylor: you need repr-C / padding checks?
    * Scott: `FromBytes` doesn't need padding, right, since you can overwrite padding with arbitrary values?
    * Josh: That sounds right
* John: I think having `FromBytes` and friends in the core library without anything backing them up would be helpful
    * Niko: They do have to obey the coherence rules, right?
    * John: Crates already have to manage that because they use derive
    * Niko: I just meant stuff like `u8`, slices, references
    * Ryan: I have the mem-markers repo...
* scottmcm: There is also "all bytes are valid and no padding" -- we could expose a trait that is implemented for such types -- those are the core constraints that a library might use to implement support for transmutes etc, but that becomes a marker trait with no methods at all. Up to some other library to make from-bytes...
    * John: At that point, the compiler needs to do some fairly in-depth reasoning at type-check time, so it's a small leap from there to just having the full safe transmute.
    * scottmcm: In other words, not that much harder to do "between arbitrary types" than this special case?
    * John: Right.
* scottmcm: Maybe there is some kind of compile-time reflection?
    * typic doesn't expose the byte-level repr of type layouts. It is changed all the time.
    * Hard to envision infinitely into a future-proof way to represent a type layout.
* Some questions to maybe follow up on:
    * What did those comments mean about needing a "global view" in the beginning? -- Niko
    
