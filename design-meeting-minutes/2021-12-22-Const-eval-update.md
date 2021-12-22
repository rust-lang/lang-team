---
tags: rust
---

# Lang Team mtg 2021-12-22 Const eval status

Links with info, don't click except for background info, summary of status follows.

* [transient heap allocations MCP](https://github.com/rust-lang/lang-team/issues/129)
* https://github.com/rust-lang/const-eval
    * [skill tree](https://rust-lang.github.io/const-eval/)
    * [const heap issue](https://github.com/rust-lang/const-eval/issues/20)

## Status

### Unsafe const fn and features

Are getting stabilized with a steady pace. Mostly a libs concern, everything seems good here.

### Trait bounds, fn ptrs, dyn Trait

Non-const trait bounds on const fn are possible via workarounds that we missed two years ago with min const fn. https://github.com/rust-lang/rust/issues/90912 shows an example.

Out of this reason, as well as the fact that it would be inconsistent with const items (where trait bounds, fn ptrs and dyn Trait are legal since forever) wg-const-eval unanimously would like to just allow all three in const fn, too, but not allow calling methods or the function behind the pointer. The unstable impl has already been adjusted to reflect this. The placeholder syntax `~const` has been intoduced to mean const  trait impl required and guaranteed, but only if used in a const context. As an example:

```rust
// error, not allowed
const fn foo<T: PartialEq>(a: T, b: T) -> bool { a == b}
// ok, callers must provide type with const impl only if called from const context
const fn bar<T: ~const PartialEq>(a: T, b: T) -> bool { a == b } 
```

Considering that workarounds for the non-const case exist and it never had any problems, we should just allow users to write those bounds and figure out a new syntax for opting into being able to call methods or fn ptrs.

### Const heap

We need to differentiate transient heap allocations and heap allocations leaked into the final value of a constant.

Examples:

```rust
const TRANSIENT: () = {
    let mut x = vec![42];
    // stuff
};
const LEAK: &[i32] = {
    let mut x = vec![42];
    // stuff
    &{x} // move necessary, but not relevant here
};
```

we want to support both of these use cases in some form or another. It gets problematic when you end up with an owned type:

```rust
const BAD: Vec<i32> = vec![42];
```

as we can now do

```rust
fn main() {
    let x = BAD;
    let y = BAD;
}
```

which will (with the compiler impl right now, if we removed our post monomorphization sanity checks) cause not just a double free, but even just `x` by itself will try to call deallocate on static memory.

Long story short: this has been heavily discussed in the c++ community already, but their results don't apply to us (no duck typing, plus we have semverred libraries). We discussed this a lot, too, and came up with three solutions.

1. rely on post monomorphization errors.
    * convenient for library authors
    * new source of breaking changes for libraries (bodies of const fn are now essentially public)
    * annoying for users to figure out
    * users may not be able to fix the problem
2. A custom ConstAlloc allocator that doesn't free on drop.
    * We definitely want this
    * should be implementable on nightly right now
    * requires all libraries that allocate to become generic over the allocator before they can become const.
    * unclear how to clone a vec or box across allocators
    * Mostly relevant to libs as no language level changes are required
3. mark const fn that ~~may allocate~~ returns a type containing allocations with `#[const_alloc]`
    * "effect system"-ish
    * library authors need to add before being able to make their functions const
    * open question (like with const fn calls in general): how do we handle  trait objects method calls and fn ptr calls with this?
    * BUT: no need to make your entire library generic over the allocator used, just slap `const` on methods and impls and add the attribute where the compiler asks you to.

Now, solution 1 is necessary anyway, as we need to move the const allocations to static memory at codegen time. So we need to do the entire analysis, and can just error out if something is fishy this late in the process.

As stated, solution number 2 is desired, as it is "basically the right thing" from the type system perspective as far as I can tell.

It is not without flaws though, as you can return raw pointers to such allocations and mis-use them (e.g. by putting them into another allocator). That's already illegal for other allocators, but it could be an additional concern.

Solution number 3 is very minimal in library-author-side effort, but it's unclear if this is too much magic. It is also rather involved, as we need new send/sync style marker traits to make sure that we don't  end up leaking types containing such allocations. All the negatives are probably outweighed by the fact that this solution "just works" from a user perspective. Writing functions or libraries requires adding an attribute (and we could infer all of this in the future as a possibility so no changes are needed at all), but calling such code to compute a constant works without having to care about weird const heap interactione.

`#[const_alloc]` works like the `unsafe` modifier. Callers of functions marked with `#[const_alloc]` are expected to mark their own functions with `#[const_alloc]`, but the catch is they do not need to when we can prove that the return type does not contain any heap allocations. In order to make this sound, the original MCP proposes `ConstSafe` and `ConstRefSafe` marker traits to distinguish what return types (final type of a constant/return type of a function) are allowed *without* the `#[const_alloc]` marker if there are calls to `#[const_alloc]` functions.

## Questions

### niko: what is the "const trait" story exactly

> Considering that workarounds for the non-const case exist and it never had any problems, we should just allow users to write those bounds and figure out a new syntax for opting into being able to call methods or fn ptrs.

What exactly do you want to allow users to write? And what would it mean? I would like an overview of the `const trait` story as you would *ideally like* it to be (not necessarily what we have now). It seems like there are a few cases to consider, right?

* Requires methods that will be used in a const environment internally (always const)
* Needs const if the method is called in a const context (because it is a const fn) but not otherwise
* ... maybe other stuff ... 

#### Notes

oli: There was an RFC that we closed but did experimentation on nightly around this. The idea was that you can "implement a trait as const", meaning that all methods have to satisfy the `const fn` rules:

```rust

```

The trait bounds would then have to have a `const impl` as well. Inside the body you could call methods from your trait bounds, and if you do that, then you have to supply a `const impl`. But people found workarounds with this design:

[Example](https://github.com/rust-lang/rust/issues/90912)

```rust
trait IsMagic {}

const fn foo<T>() where T: IsMagic {} // Does not compile
const fn bar<T>() where (T,): IsMagic {} // Compiles
```

Niko: what happens if you actually use methods from `IsMagic`? ICE or something?

oli: No, but you can use those trait bounds to call "Static items", e.g., to get an associated constant or associated type. Can't use values of that type (`T`) as if they had the trait bound in a const way, but you can use `T::FOO` etc.

```rust
trait Something {
    const C: usize;
    fn foo();
    fn bar(&self);
}

const fn foo<T: Something>(t: &T) {
    let c = T::C; // OK
    
    T::foo(); // error: calling a non-const fn
    t.bar(); // error: calling a non-const fn
}
```

niko: How would I make the above work if I wanted to?

oli: idea before was that we would make that call work "out of the box" and that we would have a `?const` or something to opt-out. But this is different from how constant and generic initializers work. There you can already use bounds and get associated constants but can't use a method and call it.

oli: workaround makes this happen inside const fn too, but you need to go through a tuple or something.

cramertj: that's a bug, right?

oli: kind of? it's another nail in the coffin. We thought it would be nice and convenient to have an opt-out, but we could do that in an edition, and people are using it already, not even on purpose.

nikomatsakis: I'm hearing "that design was confusing and inconsistent and turned out to not be what we wanted".

oli: the workaroudns people made are hard to prevent and require complex analyses.

cramertj: why is it not just...you can't list trait bounds on a const fn

oli: we want to allow that at some point, but we want to differentiate between the ones we allow and not?

cramertj: but you're not supposed to be able to write bounds on a const fn, right?

oli: we blocked bounds on the generic parameters and not *all* bounds. Not sure exactly how we screwed that up, but the analysis was always fairly wonky.

dylan: I think the screw-up was due to implied bounds which weren't checked for.

nikomatsakis: can you give a code example of something it is inconsistent with?

oli: function pointers. You can use them in initializes as much as you want, but you can't call them. 

cramertj: we're just talking about what bounds should be, right? you don't ever list bounds in the body of an initializer?

.. collaboratively produced this..

```rust
trait Stuff {
    const SOMETHING: usize;
}

impl<T: Stuff> Foo<T> for Type {
    const BAR: usize = T::SOMETHING;
}

const fn foo<T: Stuff>() {
    T::SOMETHING
}
```

cramertj: but it makes sense to me that, unless we were writing a const impl, you wouldn't be able to use the methods, don't know if others have that same ...

oli: restricting generics has been a big restriction. The only reason for it was figuring out the "opt-out" syntax.

cramertj: I'm only making the point that the "right" thing would be that I can invoke methods from the `const fn` bounds (with opt-out). I think it's more usually the correct thing.

oli: that's not really clear. If we look at how it's been used when it was completely unstable, people used generics all over, even though they couldn't invoke const fns.

cramertj: That's because there was no syntax, right, everyone was doing it through associated constants...? I think if we have const impls, people will be less aggressive about their use of associated constants.

oli: sure, but we can't really plan for that. there will be more runtime style uses where you actually call methods. but we can switch between opt-out and opt-in via an edition if we realize it's a problem, and it wouldn't be that invasive

cramertj: in a world where "most fns that can be const are", it will be unfortunate if every method had all of their bounds with a `?` or `~const` or something.

oli: if we get to the point where most functions are const fn, we should probably opt out from const as well

cramertj: not sure the balance will ever shift *that* far, people write a lot of stuff with constexpr in C++, not more than half, but a lot.

oli: let's stash this for the moment.

nikomatsakis: what is `~const` supposed to mean?

```rust
const fn foo1<T: Debug>(t: &T) {
   // cannot use trait methods at all, but can call without a const impl
}

const fn foo2<T: ~const Debug>(t: &T) {
    // can use trait methods, but must with a const impl if in a const context
    //
    // cannot use methods in a const context
}

const fn foo3<T: const Debug>(t: &T) {
    // can use trait methods in any context
    //
    // must always supply a const impl
}

fn foo4<T: const Debug>(t: &T) {
    // same as above
}
```

cramertj: right, and I think the `foo2` will be what people will want the most, and which has the most awkward syntax

josh: how often is foo3/foo4 useful? 

oli: rare. associated constants can do a workaround. right now we have no proposal for const bounds syntax.

josh: the reason I'm bringing it up is cramertj's enxt question. We might want `~const` to be spelled `const`?

cramertj: I think `foo4` is the reason I don't agree with it. The places where `foo3` makes sense are the places where you might write `foo4`. You're wanting to use some behavior from `T` *in a const context* inside.

nikomatsakis: but both 3 and 4 may be uncommon.

cramertj: I can think of other traits where .. hmm. I guess it's pretty rare. I can't think of that many traits that would be useful in both const and non-const *and* are useful. I guess in that world we just wouldn't have a syntax for foo4?

nikomatsakis: or it could be something like `const(always)`, idk.

josh: `foo3` and `foo4` should get the less simple syntax, by a Huffman argument; `foo2` is the common case and should get the simpler syntax.

### cramertj: `const` bound syntax

Why `~const Trait` in bounds rather than `const Trait`? Is there an actual ambiguity here?

Is the `~const` == "const if being called in a const context?", and `const` would be "always must be const"? (answered: yes)

### cramertj: `const` + `Drop`?

> rely on post monomorphization errors
- What error is being suggested here? Nothing about the example issue seems specific to monomorphization to me:
```rust
const BAD: Vec<i32> = vec![42];

fn main() {
    let x = BAD;
    let y = BAD;
}
```
Is the idea that something about the `const`-evaluation of `Vec`'s code itself would detect that an allocation was leaking into runtime? This seems unnecessary if we ban `const`s with `drop` glue (modulo special cases like `const X: Option<Vec<...>> = None;`)

- It seems like there's a fourth answer which is just "don't allow anything with a `Drop` impl as a top-level `const` or `static`." e.g. you could have `const X: &Vec<i32>` but not `const X: Vec<i32>`. This seems the most sensible to me regardless, as even in non-allocating cases the semantics of `Drop` for `const`s and `static`s seem terribly unclear.

- It's not clear to me why one would want a `const` `Vec`-- is the idea just that it's a slice you don't have to call `.clone()` on manually?

#### Discussion

oli: The example as given doesn't interact with monomorphization, but if you have one with more generic types it might come up:

```rust
fn foo<T: Trait>() {
    T::BAD;
}

trait Trait {
    const BAD: Vec<i32>;
}

impl<T> Trait for T {
    const BAD: Vec<i32> = vec![T::SIZE; 42];
}
```

cramertj: I think post-monomorphization errors are cases where there are erorrs in the *body that is being generated* but it's only known until use time. We used to say we had no such errors but now "very few", I think, but in this case...isn't this an error at the caller? 

oli: well, see my example above, in that case you only know that `BAD` is *bad* because it could be an empty vec.

cramertj: I would've thought we would *always* reject `foo`, or, that it would fail earlier. I'd think you'd get an error in the trait where it creates a droppable thing.

nikomatsakis: wasn't...the whole idea to *accept* droppable things?

cramertj: I'm not clear why we want to accept droppable things in const, but the semantics feel unclear. I understand wanting to have allocations or reference data from an allocation, but that feels independent from allowing drop things.

oli: the problem with allowing alloc and not drop means we can't reuse types like box/vec and would split the ecosystem (Const types, non-const) which would be unfortunate.

cramertj: why use a vec and not a ref to a vec?

oli: imagine serde being completely const, it uses vectors internally.

nikomatsakis: I think what cramertj is saying "why would the final constant" be a droppable thing.

cramertj: right, you could use droppable things as intermediaries...but the actual *const* I would think would just return a reference, and now there is no "drop" that will get called

oli: that would be a breaking change

joshtriplett: does that fully solve the problem? even if you only allow references, not owned values, nothing stops the structure from holding a vec in a way that still looks like ownership, esp. if you have interior mutability

cramertj: you'd also have to require Freeze, but we have a lot of requirements on these things anyway, it seems like we're generally on the same page hta twe don't want types with random drop impls getting inserted

oli: but you can do this since 1.0

oli: can always create your own type, implement Drop, and create a constant that has a value of that type

cramertj: no...?

dylan: for the record, I agree with cramertj, I'd prefer to forbid types with actual heap alloc but we have a special case for things like empty vec. you could add attributes to the default constructor for vector to detect it. but harder than it sounds initially since we've already allowed stuff.

oli: example that is legal since 1.0

```rust
struct Foo;

impl Drop for Foo {
    fn drop(&mut self) {
        println!("dropped")
    }
}

const FOO: Foo = Foo;

fn main() {
    FOO;
    FOO;
}
```

nikomatsakis: but, dylan, it seems pretty nifty to be able to "leak" data from a const

dylan: I'd just like to make that leak explicit via `Vec::Leak` or what have you. Owning pointers allow you to mutate the underlying data in purely safe code. Returning a box directly that permits mutation.

cramertj: but not behind a shared reference, right?

dylan: Could also allow a box or owning type behind a shared value, but what's the advantage of that over `Box::leak`

cramertj: because you want to have structures that have owned values *in* them, e.g., a pre-existing struct with vecs and boxes inside of it, don't have an equivalent thing to "leak" (that has references as fields). They want to be able to *stuff* that structure into a constant. But I think what they *really* want is to stuff a `&'static` reference to the constant.

josh: I was imagining a number of uses where I would want a top-level structure and not a reference to a structure, but most any time I would want it, the structure is either roughly copy or I could have a ref and call `clone_in`.

cramertj: but oli's point about drop glue is good. Not something we can do with drop-glue or drop-glue + freeze. Have to do something allocation specific.

oli: you need to kind of check *values*, can't just rely on the type, already rely an `Option<Box>` in a constant, so long as it is always `None`. 

dylan: we have a framework for handling that, though?

josh: Becoming increasingly enamored of the `ConstAlloc` (or `StaticAlloc`) solution.

### niko: "bodies of const fn are public"

> "bodies of const fn are now essentially public"

We should be careful about what this means. I guess that being "public" here means that making changes can cause downstream crates to stop compiling, but I'd like to distinguish between (at least) two kinds of changes that seem relevant:

* Changes to the value that is returned
    * I believe this *already* can cause problems if you have e.g. `[T; foo()]`, right?
* Changes to the *way* the value is computed
    * This seems relevant to (for example) the `const Trait` bounds, right? i.e., calling more methods than we did before -- or any methods -- may make you need const bounds that weren't required before.

> Now, solution 1 is necessary anyway, as we need to move the const allocations to static memory at codegen time. So we need to do the entire analysis, and can just error out if something is fishy this late in the process.

I don't really understand what "solution 1" means here.

One other thing to consider is the "layering" story. i.e., is there a way that we start with something "correct but heavyweight" (like custom allocator) and then make convenient syntax for it later?

#### Other notes

* oli: it's what you wrote, the *way* the value is computed is public, because there are const fn that take arguments
* niko: like, when something takes a usize and it may not be "addable"?
* cramertj: overflow is an easy example
* oli: what I mean is "imagine if Vec::new suddenly allocated a zero-sized thing internally"
    * now you have an allocation
    * that would be a breaking change if we allow heap allocations in const fn
    * it would only fail once you get the value of the constant and analyze it to realize that there's an allocation
* cramertj: but isn't that a change in the *return value*, you're saying it's an impl detail, but I see it as an impl detail of the return value
* niko: you've added private fields to the structs, presumably, to store this data?
* cramertj: you don't have to modify the structure, you might be just returning a different value for some pointer that already existed
* cramertj: are there any plans here for supporting containers that actually mangle their pointers?
    * e.g. I know of a handful of crates that use the "last N bits are zero" trick
* oli: we can support some of that in const eval
* oli: would require some weird cases, have to make sure we really want that
    * "yes we can do it, it's annoying"

### josh

>     * requires all libraries that allocate to become generic over the allocator before they can become const.

This seems like a potentially useful prompt for people to do that, to the extent libraries aren't already generic over the allocator. (Also, do we need to make it *easier* to be generic over the allocator by default, since the vast majority of code won't *actually* care what allocator a `Vec` or similar uses?)

#### josh 2

What does it look like to write a function that's callable in `const` context and returns a `ConstAlloc`, and also callable in non-`const` context and returns a non-`ConstAlloc`? That's not fully generic over allocators; that's a function that returns `ConstAlloc` in const contexts and some other allocator (likely the default allocator) in non-const contexts.

Do we need a *generalized* version of conditional const (e.g. `const ? ConstAlloc : GlobalAlloc`)? Or, should we expose an allocator alias that has a different type at const and non-const time?

#### josh 3

ConstAlloc seems generally useful for more purposes; perhaps we should call it `StaticAlloc`, and make it possible to use in more cases (for instance, manually constructing the internals of a structure that is normally heap-allocated, and knowing it won't be freed).

#### Notes

josh: seems like it'd be useful to do this generically over an allocator, what would it look like to have somethign that "when const returns something in the const alloc allocator, but otherwise uses global alloc"?

josh: we might have an alias, for example, "const or global alloc" that becomes the default?

cramertj: the return type then would change...?

oli: well, if we infer this from the `const` on the function... could ... kinda work?

dylan: how do I go from a vec that was allocated with const-alloc to global-alloc

josh: you'd use the functions for cloning between allocators

cramertj: skeptical of a route in which this is our only way, feels limiting, people doing serialize/deserialize have to have all their types being generic over an allocator, which sounds uncomfy

niko: yeah we'd have to improve our story

josh: that motivated my question of "do we need to make it easier to be generic over an allocator"

dylan/oli: if only someone were writing a big blog post about capabilities (subtweeting tmandry's post)

cramertj: I'm not convinced that the right thing to do is to make whole crates generic, such that you don't get codegen until the very end

nikomatsakis: I think you're conflating two things

josh: codegen aside, it seems useful to be generic

oli: nudging people to be allocator capable is possible with const stuff, but I'm not sure if everyone is behind it

josh: given a good story for "const-alloc in const, global-alloc otherwise" would be really useful, all the behavior that differs is kind of wrapped in "what it means to allocate something"

cramertj: we need to be able to coerce references to these types. In order to have a useful use in a runtime context, you need to be able to pass a reference to it into a thing that expects a reference to ...

```rust
struct MyType {
    v: Vec<u32>
}

const MY_TYPE: &MyType = &MyType {
    v: vec![2]
};

fn foo(x: &MyType) { 
    //     ^^^^^^ this would have to be generic over an allocator too, right, but only when it may be used with a value that came from a constant
}

foo(MY_TYPE);
```

cramertj: or creating a list that has multiple references, some of which came from constants, and some of which didn't

cramertj: if you can know some way to say that the type's *representation* isn't dependent on the generics...or something...or normal method calls...we could permit coercions... this is sounding very complicated to me comparatively. 

fee1-dead: the allocator could be a reference to a trait object and we decide which one to use based on the context.

dylan: but now we have 3 choices, right?

### dylan: something for lang team to think about

Something for lang team to be thinking about. We only deal with `const` trait bounds at the level of a const fn, but I think you want to be able to write trait impls that are "const if they bounds on the input parameters are also const".


```rust
impl<T: ~const Eq> const PartialEq for Vec<T> {
    fn eq() { /* ... */}
}
```

feel-dead: that is supported 

oli: doesn't matter where the bounds are

dylan: ok, we're starting to tread on a parametric effect system here

oli: following the rules fairly straightforwardly

## Stuff we didn't get to in the meeting

### josh

where is the non-placeholder version of `~const` syntax being discussed? Is there an issue specifically for the syntax discussion, and/or a Zulip topic? Would be good to nail that down (*not* in this meeting) soon, so that it doesn't end up being a blocker for stabilizing that syntax.
Reiterating: *not* looking to discuss the syntax here, just looking for a pointer to where that discussion can happen or is already happening.

### josh - What precisely is the intended semantic of `#[const_alloc]`?

What does it mean when a function marked `#[const_alloc]` returns a `Vec<T>`?

### Mark - what is our expected "long-term" future for const?

That is, is roughly everything expected to be const-compatible in one way or another, modulo io::stdin and other external APIs? If this is the case, I am wary of forcing code to get (re)written or to constantly have to make sure we're adding the right const or generic-over-allocator bounds in all the places.

### niko: where does "valtree" and the questions around equality fit into this story?

We had a previous meeting -- one which I would like to revisit! -- where we talked about structural equality and the like. I'm surprised not to see that or valtrees arise in this document. What's the status there?

### 

### 









