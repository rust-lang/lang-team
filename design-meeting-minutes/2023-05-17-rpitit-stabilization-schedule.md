---
tags: design-meeting
---


# Return Position Impl Trait in Trait Stabilization

[RFC #3425] proposes return-position impl trait in Trait. This is a long awaited feature and in particular would be helpful as a stepping stone towards async functions in trait. We would like to stabilize it quickly if possible. The goal of this meeting is to make sure that everyone on the lang team is aware of the key points in the design as well as the concerns that have been raised thus far. Ideally, we'll emerge with alignment about the correct way forward.

[RFC #3425]: https://github.com/rust-lang/rfcs/pull/3425

[RFC #3245]: https://github.com/rust-lang/rfcs/pull/3245

## Motivation and stabilization strategy

Impl trait is allowed in return types everywhere except for traits and impls. This makes it inconvenient to author traits that return iterators, closures, futures, or other instances of combinator traits. Generic associated types make it possible to create traits that return complex values, but the resulting traits are [verbose and confusing](https://github.com/rust-lang/rfcs/pull/3425#issuecomment-1539067273).

### Example: Iterators that return traits

The classic example where `-> impl Trait` is useful is returning an iterator:

```rust
trait Factory {
    fn widgets(&self) -> impl Iterator<Item = Widget> + '_;
}
```

Examples like this "just work" under the MVP proposed below.

### Example: Expressing async fn that don't capture their arguments

The current [Tower service trait](https://docs.rs/tower/latest/tower/trait.Service.html) does not capture `self` in the resulting future and thus cannot use the `async fn` notation. It could be expressed with `impl Trait` like so:

```rust
trait Service {
    fn serve(&mut self, request: Request) -> impl Future<Output = Response>;
    //                                      -------------------------------
    // NB: the result does not capture                        |
    // the lifetime from `&mut self` -------------------------+
}
```

### Example: Expressing async fn that must be send

Some traits would like to express the idea of an async function whose future is guaranteed to be `Send`. 
This can be expressed like so

```rust
trait Provider {
    fn provide(&self) -> impl Future<Output = Result> + Send + '_;
    //                                                  ----   --
    //                                                    |    |
    //     Guarantee that future returned is `Send` ------+    |
    //                                                         |
    //     Capture the `&self` --------------------------------+
}
```

In cases like this, it is desirable that people can *implement* the trait using `async fn` sugar:

```rust
impl Provider for MyType {
    async fn provide(&self) -> Result { 
        // Use the design below, this works for *this* trait.
    }
}
```

Given the design below, this example compiles even though the `-> impl Future` produced by `async fn` is different from the signature that appears in the trait. For more complex functions that take more references as arguments, the differences between `async fn` capture rules and `-> impl Future` capture rules become more challenging to express, but that issue is orthogonal to the questions at hand today.

## Key highlights of the design in the RFC

The RFC permits `impl Trait` to be used in return position in a trait:

```rust
trait IntoIntIterator {
    fn into_int_iter(self) -> impl Iterator<Item = u32>;
    
    // nb. Just as with other functions,
    // `-> (impl Iterator, impl Iterator)` and
    // `-> impl Iterator<Item = impl Trait>` are also supported,
    // but not `impl PartialEq<impl Debug>`.
}
```

Intuitively, this desugars to a (potentially generic) associated type -- or multiple, for more complex types like `(impl Debug, impl Debug)`:

```rust
trait IntoIntIterator { // desugared (approximately)
    type $: Iterator<Item = u32>;
    fn into_int_iter(self) -> Self::$;
}
```

Users cannot directly name this associated type. This is analogous to the way that `impl Trait` in argument position desugars to an anonymous generic parameter that cannot be directly referenced.

### Impls of traits using `-> impl Trait`

In [RFC #3245][], we decided that impls should meet the declared interface from the trait unless annotated with `#[refine]`. In accordance with this, implementations of methods using `impl Trait` in return position must use `-> impl Trait` with equivalent bounds by default:

```rust
impl IntoIntIterator for Vec<u32> {
    // OK
    fn into_int_iter(self) -> impl Iterator<Item = u32> {
        self.into_iter()
    }
}

impl IntoIntIterator for Vec<u32> {
    // ERROR: Impl bounds are stronger than trait
    fn into_int_iter(self) -> impl ExactSizeIterator<Item = u32> {
        self.into_iter()
    }
}

impl IntoIntIterator for Vec<u32> {
    // ERROR: Impl uses precise type but trait uses `impl Trait`
    fn into_int_iter(self) -> std::vec::IntoIter<u32> {
        self.into_iter()
    }
}
```

This is equivalent to a desugaring based on impl Trait in associated type position, roughly as follows:

```rust
impl IntoIntIterator for Vec<u32> {
    // OK
    type $ = impl Iterator<Item = u32>;
    fn into_int_iter(self) -> Self::$ {
        self.into_iter()
    }
}
```

In particular, an implication of this desugaring is that auto traits "leak" through the `impl Iterator` value, so while callers cannot observe exactly what sort of `Iterator` is returned, they can test if the iterator is `Send`:

```rust
fn is_send_iter<T>(t: T) where T: Send { }
is_exact_size_iter(vec![22_u32].into_int_iter()) // OK
```

### Impls with `#[refine]`

Impls can also specify more precise types for the return type by using `#[refine]`. For example, the impl of `IntoIntIterator` for `Vec<u32>` could go so far as to specify the *precise* return type:

```rust
impl IntoIntIterator for Vec<u32> {
    #[refine] // OK
    fn into_int_iter(self) -> std::vec::IntoIter<u32> {
        self.into_iter()
    }
}
```

This is equivalent to a desugaring as follows:

```rust
impl IntoIntIterator for Vec<u32> {
    type $ = std::vec::IntoIter<u32>;
    fn into_int_iter(self) -> Self::$ {
        self.into_iter()
    }
}
```

Callers of `into_int_iter` can rely on this return type:

```rust
let x: std::vec::IntoIter<u32> = vec![22_u32].into_int_iter();
```

Refine can also be used to strengthen the bounds without revealing the exact type:

```rust
impl IntoIntIterator for Vec<u32> {
    #[refine]
    fn into_int_iter(self) -> impl ExactSizeIterator<Item = u32> {
        self.into_iter()
    }
}
```

This example simply desugars to `type $ = impl ExactSizeIterator<Item = u32>`.

```rust
fn is_exact_size_iter<T>(t: T) where T: ExactSizeIterator { }
is_exact_size_iter(vec![22_u32].into_int_iter()) // OK
```

### Interoperability of `impl Future` and async fn

One of the goals for this RFC is that users should be able to convert from `async fn` into equivalent functions using `-> impl Future`, either on the trait *or* the impl side, and everything should continue to work. 

Example:

```rust
trait AsyncProvider {
    async fn provide(&self) -> Value;
}

impl AsyncProvider for MyType {
    fn provide(&self) -> impl Future<Output = Value> + '_ {
        async move { self.provide() }
    }
}
```


```rust
trait AsyncProvider {
    async fn provide(&self) -> Value;
}

impl AsyncProvider for MyType {
    fn provide(&self) -> impl Future<Output = Value> {
        async move { self.provide() }
    }
}
```

### Differences from what is implemented on nightly

The implementation of this feature on nightly does not respect the `#[refine]` semantics -- all methods act as though `#[refine]` has been provided, so they are free to refine the trait interface.

## Stabilization proposal

The proposal is to stabilize a specific subset of the above content:

* `#[refine]` would not be stabilized
* traits that use `-> impl Trait` would require impls to use `-> impl Trait` (or, equivalently, `async fn`) and meet the following conditions
    * the actual type from the impl must meet the bounds in the trait
        * So if the trait says `-> impl X0 + ... + Xn`, the impl must return a type that implements `X0 + ... + Xn`.
        * Given that the impl must have a return type of the form `impl Y`, this implies that for each `i`:
            * `Xi` must be an auto trait that "leaks" from the actual type used in the impl
                * e.g. `async fn` in the impl but `-> impl Future + Send` in the trait
            * `Y => Xi`
                * e.g. `impl Eq` in the impl but `impl PartialEq` in the trait (this would cause an error below, however)
            * or `Xi` is provable for all types for which `Y` is true
                * e.g. `impl Debug` in the impl but `impl Debug + Foo` in the trait where `impl<T> Foo for T`
    * the bounds from the trait must imply the bounds from the impl
        * `-> impl Eq` in the impl and `-> impl Eq` in the trait is ok
        * `-> impl PartialEq` in the impl and `-> impl Eq` in the trait is ok by this check (but errors in the check above)
        * `-> impl Eq` in the impl and `-> impl PartialEq` in the trait is an error by this check (but ok by the check above)

The goal is to let people use `-> impl Trait` in traits and impls, and interchange it with `async fn` when that becomes stable, but not to stabilize the "refine" mechanism yet. Instead, we only permit impls whose semantics would not change when "refine" is used.

## Key questions for this meeting

### Are we ok to stabilize this MVP?

Major concerns raised ([complete list](https://github.com/rust-lang/rfcs/pull/3425#issuecomment-1539038582)) and some brief responses:

> On their own, RPITITs work against maximum flexibility, as compared to normal associated types, because there is no way to add bounds to them. Using them in widely used libraries is likely to be a kind of anti-pattern.

This is true, but it's also consistent with impl Trait elsewhere. For example, best practice for widely portable libraries remains to use newtypes rather than RPIT, so that people can name the types returned by your functions, for example for use in structs. Similarly, impl Trait in argument position is less flexible, as people cannot use turbofish. It would be good to address these shortcomings in future work, but the future stands up on its own just as much as impl Trait in other positions stands up.

[more](https://github.com/rust-lang/rfcs/pull/3425#issuecomment-1539085009)

> RPITITs are different from other "opaque" uses of impl Trait (e.g., RPIT, TAIT), in that they desugar, on the trait side, to an associated type (on the impl side, they are exactly analogous, I believe). This weakens the "more consistent language" story, since it's not just a matter of applying the exact same desugaring we already have elsewhere.

It's true that, if you start with the implementation, impl Trait is "inconsistent" in that it can desugar in different ways. But the intent of the impl Trait syntax, as a user, is that any place you use it it winds up desugaring to "some type that implements Trait" in a sensible way. So in fn argument position, that is a fresh generic type parameter. In (inherent) return position, it's an inferred opaque trait. And yes, under this RFC, in trait return position, it's an associated type.

[more](https://github.com/rust-lang/rfcs/pull/3425#issuecomment-1539067273)

> If we stabilize TAITs or uses of impl Trait in associated type values (AssocIT), that -- combined with GATs -- lets users model RPITIT explicitly. It'd be a better place to start.

Progress on this RFC isn't blocking us from making progress on those features and experience with those features won't help us resolve any of the interesting questions about the contents of this RFC. The core question, really, is whether we believe we will want people to be able to write `-> impl Trait` in traits and to have it desugar to some kind of associated type -- if so, this RFC represents a conservative step in that direction.

[more](https://github.com/rust-lang/rfcs/pull/3425#issuecomment-1539088510)

### Should we use `#[refine]`?

The `#[refine]` attribute was proposed in [RFC #3245][]. The premise was that impls should be able to provide stronger guarantees that the trait requires, but that this should be a conscious decision. For example, traits with `unsafe` methods currently impls to also declare those method as `unsafe`. [RFC #2316] proposed accepting impls that elide unsafe and then allowing callers to rely on that, if the self type is sufficiently known. [RFC #3245][] clarified that while we thought this was useful, we felt impls should use `#[refine]` to acknowledge that they are intentionally committing to the method being safe, as otherwise users might not realize what is going on. A similar logic holds here: users might not realize they are comitting to stronger bounds when they list those bounds in the impl.

[RFC #2316]: https://rust-lang.github.io/rfcs/2316-safe-unsafe-trait-methods.html

On the other hand, including `#[refine]` makes the desugaring more complex. We could instead simply desugar `impl Trait` on the trait side to an associated type whose value is the exact return type given by the user. This would mean for example that this impl would be accepted:

```rust
impl IntoIntIterator for Vec<u32> {
    // ERROR: Impl uses precise type but trait uses `impl Trait`
    fn into_int_iter(self) -> std::vec::IntoIter<u32> {
        self.into_iter()
    }
}
```

and moreover that callers could observe the actual return type, even though the trait is declared with `-> impl IntoIntIterator`:

```rust
let x: std::vec::IntoIter<u32> = vec![0_u32].into_int_iter();
```

Is this a problem? Not necessarily. The main concern would be that users are not aware that when they write an impl like the one above, they are making a semver commitment that `<Vec<u32>>::into_int_iter`  will always return that exact type.

Other relevant examples include `unsafe`:

```rust
trait SomeTrait {
    // Only callable when X can be safely dereferenced.
    unsafe fn foo(x: *mut u32);
}

impl SomeTrait for NoOp {
    fn foo(x: *mut u32) {
        // Did user intend to semver promise that `foo` will never be `unsafe`?
    }
}
```

## Comments and questions

### How do I leave a question?

ferris: just do make a section and type stuff! Put your name first.

### is `#[refine]` a macro, and is async keyword required?

vincenzopalazzo: wondering if we should we establish uniform requirements for async functions, while granting macros the capability to decipher and refine them?

nikomatsakis: refine is not a macro, just an attribute, but the intent is that async fn and `-> impl Future` are interchangeable (so long as you get the bounds *just right*, see next question)

### stabilizing RPITIT faster?

eholk: Is the idea that we could stabilize RPITIT quicker than AFIT, so we should go ahead and do that while we are finishing up AFIT?

nikomatsakis: it's a separate feature that I think holds its own. I'd like to stabilize async fn together with send bounds, but it's a nice benefit that RPITIT would let you write traits that are forwards compatible with using `async fn`.

eholk: agreed.

### arguments for more general (?) stuff like TAITs+GATs etc

pnkfelix: Just wanted to check: are the people who are arguing that RPITIT doesn't hold its own weight; if it comes to fruition that they end up being right, will the main outcome be that rustc and/or clippy starts linting against the use of RPITIT and advises programmers to use the preferred construct (be it TAIT or whatever)? I'm trying to imagine whether that future is even all that bad. :)

nikomatsakis: I think the worst case is that best practices for writing libraries advise against the use of `-> impl Trait` and/or people find themselves stuck.

```rust
trait IntoIntIterator {
    type Iterator: Iterator<Item = u32>;
    
    fn into_iter(self) -> Self::Iterator;
}

impl IntoIntIterator for Vec<u32> {
    type Iterator = vec::iter::IntoIter<u32>;
        
    fn into_iter(self) -> ... { }
}

fn foo<T: IntoIntIterator>(t: T)
where
    T::Iterator: ExactSizeIterator
{}
```


```rust
trait IntoIntIterator {
    fn into_iter(self) -> impl Iterator<Item = u32>;
}

impl IntoIntIterator for Vec<u32> {
    type Iterator = vec::iter::IntoIter<u32>;
        
    fn into_iter(self) -> ... { }
}

fn foo<T: IntoIntIterator>(t: T)
where
    T::Iterator: ExactSizeIterator // can't write this
{}
```

one reason that e.g. lcnr was opposed is that the feature that's been proposed to close this gap (RTN, see below), lcnr doesn't like:

```rust
trait IntoIntIterator {
    fn into_iter(self) -> impl Iterator<Item = u32>;
}

impl IntoIntIterator for Vec<u32> {
    type Iterator = vec::iter::IntoIter<u32>;
        
    fn into_iter(self) -> ... { }
}

fn foo<T: IntoIntIterator>(t: T)
where
    T::into_iter(): ExactSizeIterator,
{}
```

...but I think that while I do want to solve this problem, we don't have to solve it with RTN, and the feature stands on its own imo.

eholk: there is a "more than one way to do it" argument

pnkfelix: yes, but that ship has sailed. The argument was raised about impl trait in argument position at the time but clearly in hindsight it holds its weight.

eholk: now that we have impl trait anywhere, the more places you can have it the better. 

### anyone in the room who wants to argue we should not have it?

nikomatsakis: see above

*crickets* (nobody speaks)

### interconverting async fn and impl trait

eholk: Isn't there something subtle about capture rules for `async fn` that make this not always straightforward?

### `#[refine]` and elided lifetimes

tmandry: Prior to this we discussed some subtle cases where you could accidentally refine a signature with elided lifetimes, are those in scope for this discussion?

### send interaction

```rust
trait Foo {
    async fn bar(self);
}

impl Foo for u32 {
    // is #[refine] needed here because of the Send bound?
    fn bar(self) -> impl Future<Output = ()> + Send {
        
    }
}
```

eholk: arguably you could see async fn vs impl future as a kind of refinment?

nikomatsakis: no, the rfc explicitly takes the opposite position.

eholk: ok, but with send...

nikomatsakis: then it's different. and the example above would require `#[refine]`

compiler-errors: but vice versa doesn't require `#[refine]`

```rust
trait Foo {
    fn bar(self) -> impl Future + Send;
}

impl Foo for u32 {
    async fn bar(self) {
        // #[refine] not needed, it leaks    
    }
}
```

eholk: it seems weird to me

nikomatsakis: yes, but consistent with how async fn works elsewhere, right? when I write an async fn I expect it to be send if it needs to be

tmandry: consistent with auto trait leakage, which admittedly is weird

eholk: I'd like to get rid of auto trait leakage

### Should we have to spell `#[refine]` at all?

TC: Is it so bad to just not require `#[refine]` to be said at all?  It seems like something people will just add after the compiler complains, just like people do today with `mut` on stack variables.

tmandry: valid question and I don't think we know the answer. There are some places where it seems like a footgun, e.g. with `unsafe` or maybe writing a concrete type with `impl Trait`. Places where there are hazards of semver compatibility that we want you to be conscious of, and I am hopefully refine will help.

TC: I feel like auto traits mean semver compat ship has sailed.

tmandry: I was wondering if we could do something on publicly accessible impls or something where we lint against leaking details.

nikomatsakis: you could imagine having a lint against things that are a public impl but doesn't have `#[refine]`

tmandry: ...and do that for auto-trait leakage, i.e., if you have leaked a `Send`...

nikomatsakis: oh, interesting

pnkfelix: talking about adding over an edition boundary if there's somehow been auto-trait leakage of some kind?

```rust
// In Rust 2021
pub async fn foo() { }

// In Rust 202N you have a suggestion to annotate...
pub async(Send) fn foo() { }
```

pnkfelix: I don't see the equivalence to `mut`, that's a local thing that isn't part of the public API. Being told to write `#[refine]` because I've got a change that's part of my public API seems good?

TC: the equivalence I was trying to draw is that people may dump it in without really understanding what they are committing to.

pnkfelix: yes, they could be adding `unsafe` blocks; we can't stop people from doing that?

TC: I'm bringing it up here -- when you add `unsafe`, I hope people understand waht they're doing there. When you have to add `#[refine]` is subtle though, even we were a bit unclear. I feel like there will be a lot of cases that people just do it.

nikomatsakis: I think the equivalence to unsafe is a bit funny, because e.g. there will definitely be a "auto-apply" for `#[refine]`, and unsafe has a heavier lift.

nikomatsakis: I am definitely of two minds about refine. I am not sure it carries its weight. If people did their desugaring in their heads, I think it's natural that you have specified the value of the associated type. But I'm not sure if people will be doing that. This is why I wanted to stabilize only the portion that is compatible with `#[refine]` (i.e., that would not need refinement) but not stabilize `#[refine]` yet.

TC: I hope they do the desugaring, we should be teaching them what it means. Part of why I don't like `#[refine]` is that it is surprising if you think about the desugaring.

pnkfelix: If the way to teach it is to teach the desugaring, I think we've failed in some way, it should be a meaningful abstraction that you can understand on its own terms.

TC: I think that people shift down

pnkfelix: yes, but it's crucial we don't force them to do so

scottmcm: Obligatory mention of the [Dialectical Ratchet](https://github.com/rust-lang/rfcs/pull/2071#issuecomment-329026602).

TC: then the abstract has to work consistently, as soon as it doesn't work, you have to delve into "what's different about this". If you want people to think "I can write `impl Trait` and it can infer a type here", then it should work everywhere you might expect it to.

nikomatsakis: in some sense, you don't need refine for it to make sense without the desugaring, you just have to be aware of the meaning of what you're committing to; but if you remove refine, the desugaring feels natural and the overall feeling is smaller.

tmandry: if you think of refine as more of a lint...

*time's up!*

eholk: I'm +1 on getting things out and leaving room to choose about `#[refine]` later.

