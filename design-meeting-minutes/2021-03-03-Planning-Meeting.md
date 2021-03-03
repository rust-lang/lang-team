# Planning meeting 2021-03-03

* [Watch the meeting](https://youtu.be/l4EQmsowvU0)

## Attending

* Team: nikomatsakis, scottmcm, taylor, josh, pnkfelix
* Other: simulacrum
* Action item scribe: simulacrum
* Minutes scribe: scottmcm

## Updates

### pub macro

* Covered in triage meeting, copy notes from there

### const eval

- oli looking into "value tree"
    - more advanced representation for thigns like comparing non-scalars for equality
- summary: work proceeds
- const UB RFC: discussion going on.

### Async Foundations

- quiet on the language side
- async vision doc effort trying to get started
    - trying to make a few-years vision and using that to decide how to address today's problems.
- https://rust-lang.github.io/wg-async-foundations/vision.html
- trying to get the community to write about experiences and wishes

### Type Ascription Expressions

* old plan: change behavior of type ascription in reference contexts:

```rust=
let x = &(y: &u32);
```

* new plan: forbid type ascription in reference contexts
    * it always coerces, always produces a new value
* BN is preparing a revised RFC to reiterate the design
- Are people still into it?  How do people feel about syntax?
    - Generally interested
    - simulacrum: not a fan of the syntax, otherwise interested
        - maybe something postfix?  `.type::<i32>`?
    - scottmcm: patterns using `:` is really strong precedent
        - could be nice to have something that chains
        - I kinda wish it could be `as`, with all the non-ascription `as` things moved to library methods.
            - With safe-transmute especially, this might open new things.
    - josh: type ascription in patterns may satisfy some of the desire to be more specific to avoid the questions about match ergonomics.  Might be able to use `: &_` or similar to avoid `ref` and such (details TBD, would want to be able to match `&` and prevent `&&` without necessarily specifying `&ExactType`).
    - patterns vs expressions being coercing or not?
        - Are pattern types non-coercing?
        - array-to-slice, for example


```rust=
match foo {
    x: &[u32] => {
    
    }
}

let foo: &[u32; 3] = &[1,2,3];
match foo {
    x => {
        let x: &[u32] = x;
    }
}
```

### Never type

* Current status is still [#79366](https://github.com/rust-lang/rust/pull/79366)
* Looking into whether to float a proposal where we do fallback only on edition
* simulacrum is reading into it - likely update by next triage
- interesting related to `Try`
    - Both for the use and because `never_type` stabilizing would make the RFC a breaking change (though having it on nightly would be enough)

### Const generics

- min_const_generics in 1.51 🎉
- weekly meetings ongoing
* Looking to build on min const generics
    * Remove ty/const param ordering restriction
        * Lifetimes would still have to go first, at least for now
        * https://internals.rust-lang.org/t/const-generics-and-defaults/14138/14?u=scottmcm
    * Permit const parameter defaults
        * e.g., `struct Foo<const N: usize, const M: usize = N>`
        * maybe worth an RFC to restate and clarify the rules
            * The rules for type defaults (see `HashMap`'s hasher) aren't obvious to me, at least, so I don't know if those things happen with `const` too.
    * Permit `_` as a const generic argument, in contexts where `_` would be accepted as a type argument
        * Includes: `foo::<u64, _>`, for example
        * Excludes: `const T: [u64; _] = [1, 2, 3]`
        * Similar examples:
            * `fn foo<const T: ()>` -- `foo::<()>` is treated as a type
            * `()` and `ZST` where `ZST` is some `struct ZST;` like thing
    * Can we find a way to not need to list *all* the generic parameters?
        * Related to what happens with changing ordering restrictions
        * if the ordering changes, could allow `foo::<{N}, ..>` or something
* Plan:
    * Stabilization reports as they get close -- or maybe RFCs so that we have a good place to lay out the rules
    * Should there be a vision doc?  An updated skill tree?

### Declarative macro repetition counts

* Opened RFC
    * https://github.com/rust-lang/lang-team/issues/57
* How is it different from the previous "`$self` for path" conversation?
    * This one doesn't break the universe
    * This isn't hygiene, so doesn't have the "should be macros 2.0" thing.
    * So the previous conversations don't apply here, and this seems reasonable.

### Instruction set update

I've been catching up on the T-lang meetings, and `#[instruction_set(?)]` has been on the agenda with no info for a month or two, apparently. Here's the story.

The thing with the `#[instruction_set(?)]` attribute is that the way rustc passes this to LLVM it is that it sets or unsets a target feature attribute on the individual function (`thumb`, in the case of ARM, I'm not sure the MIPS name). This is *similar to* the `#[target_feature(enable="foo")]` attribute on a function. The primary difference is that `instruction_set` can enable or *disable* a target feature, and the normal `target_feature` attribute can only enable target features.

So this brings us to all the fun with cross-feature function calls.

* LLVM *will not* inline a function across an "incompatible" target feature barrier.
  * If you have function `a()` that uses `avx` and function `s()` that uses `sse`, then `s()` can be inlined into `a()`, but `a()` cannot be inlined into `s()`.
  * On ARM, having `thumb` either on **or** off is incompatible with the opposite state.
* Actaully I just lied to you a bit because `inline(always)`, which maps to `forceinline` in LLVM, apparently can actually *force* the inline to happen, even across a feature barrier.

So *when* does this matter? Well, sometimes, but not always:
* If your function body is just rust code, then that code could naturally have worked in either instruciton_set.
* If your funciton body is inline assembly which is written for A but then gets inlined into a funciton using T, then you're kinda hosed.

So that's one concrete problem: **instruction_set and inline assembly behave badly in the presence of inline(always)**

(Note: of course, inline assembly is unsafe and so this isn't a *soundness* issue, it's just a *footgun* issue)

But here's the other problem, which is actually possibly just as much of an issue: remember how rust has all these "zero cost" abstractions that rust loves? like iterators? and smart pointers? and stuff like that? that stuff all depends on functions that get inlined. But when `core` and the rest of the program are being compiled using one instruction set, and then your little alternate instruciton set function is being compiled using another instruction set, then your little alternate instruction set function can't inline anything into itself. Even a single volatile access is a function call to the volatile method, then the single access instrction, and then that returns. Runtime performance falls into a deep, deep hole in the ground, worse than the worst debug performance you've ever faced. Which is not... *wrong* exactly, it's not *unsound*, but it is *very bad*.

And you can "fix" this by making your instruction_set function with an alternate instruction set immediately just call to whatever other fuction written in the default instruction set for the program. However, that's... rather limiting and unfortunate.

So that's the other concrete problem: **instruction_set has poor runtime performance in the absence of either inline(always) or inline assembly**

And that's the status report.



- Why do we expect this to be common?
    - embedded platforms with both arm and thumb
- Could they be made to work in combination?
    - Can inlining a function compile it to the appropriate instruction set?
- Is this lang or compiler right now?
    - is it implementable?
    - is this the correct abstraction?
    - can we stabilize and improve later?
    - does `inline(always)` need to get fixed to avoid UB?
        - "always" is just a hint, so arguably can't be relied upon...

### Try

- Stealing some time here to ask about edition thoughts
- https://github.com/rust-analyzer/rust-analyzer/pull/7735
- Is it worth doing an edition change here?
    - Is the option<->result interconversion unwanted enough to do an edition change?
    - This is about clarity, not efficiency.
    - It was never intentionally stabilized, and the previous RFC was trying not to allow it.

### ffi-unwind

* The PR adding "C-unwind"` is r+'d and close to landing, solving some niggling testing issues
* Active discussion about longjmp
* If we want to permit longjmp out of a function `f`, should we require that `f` is annotated?
* If we don't, we lose the ability to do some optimizations that we would otherwise be able to do
    * in particular, we lose the ability to move actions after the call to `f`, since we can't be sure that a longmp won't cancel them out
* Current status: awaiting a better doc and write-up 

## Meeting proposals

* March 10 -- no pnkfelix
* March 17 -- Backlog Bonanza
* March 24 -- Lang team org work meeting
* March 31 -- #82

General rules:

* Doc has to be written and sent out by Tuesday for the Wednesday meeting
    * or it is cancelled
    * no expectation people will read until the meeting

### Pending proposals

- Backlog bonanza (nikomatsakis)
    - revisit the RFC bonanza and double check that we did everything
    - and/or revisit decisions as needed
- Lang team org work meeting (nikomatsakis)
- How to dismantle an `&Atomic` bomb. [#82](https://github.com/rust-lang/lang-team/issues/82)
    - What does it mean to have a reference to something that gets deallocated?
    - The atomic API surface is large, so duplicating to pointers is non-ideal.
    - Should it be a tweak to the rules for `UnsafeCell`?
    - What does this need?  Meeting?  RFC written?
    - Customers are trying to write code like this and aren't clear what's legal.
    - Needs a doc; Felix to make sure one gets written. Mark can help.
- Discuss the amount of oversight that lang should have for lints [#72](https://github.com/rust-lang/lang-team/issues/72)
    - Also needs a proposal.
    - scott won't be writing one this month, at least.
- Discuss the possibility of denying `bare_trait_objects` in 2021 edition #65
    - no meeting, close -- we've been over this in sufficient detail
- What do we do about macros?
    - We've had many things come up recently, should we have a "let's make a plan for macros" meeting?
    - Should we enable a general extension point, and what things should we add to that extension point?
    - (This should not turn into a macros 2.0 meeting, unless someone is prepared to stand up and say "I'm going to implement macros 2.0".)
    - Need a concrete proposal and someone prepared to drive it