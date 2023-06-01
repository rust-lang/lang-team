# Type Alias impl Trait (TAIT) Stabilization (design meeting)

We accepted [RFC 2515] in 2019.  That RFC proposed that `impl Trait` syntax be allowed in type aliases and in associated types.  This is a long-awaited feature.  It helps API designers to not leak unwanted implementation details and it closes certain fundamental expressiveness gaps in Rust.  We would like to stabilize this feature quickly if possible.

This meeting will be a success if everyone walks away with a clear understanding of the motivations for this feature and the key problems it addresses, the details of the proposed plan for stabilization, and how we can move forward.

In this document, TAIT stands for "type alias `impl Trait`", and ATPIT stands for "associated type position `impl Trait`.

[RFC 2515]: https://github.com/rust-lang/rfcs/blob/master/text/2515-type_alias_impl_trait.md

## Motivation 1: Closing the expressiveness gap on unnameable types

In Rust, there are types that cannot be named directly such as the type for each closure and future.  These types can only be described by the traits that they implement.  Type alias `impl Trait` allows type aliases to contain these (and other) types by using type inference.  This is similar to `impl Trait` in return position (RPIT), but unlocks new use cases by allowing hidden types to appear in more places.

### Example: Sending futures down a channel

Consider this example which might be part of a job queueing system:

```rust
use std::future::Future;
use tokio::sync::mpsc;

type Iter = impl Iterator<Item = u8>;

type Fut = impl Future<Output = impl FnOnce() -> Iter>;

struct Job {
    id: u64,
    fut: Fut,
}

async fn send_job(tx: mpsc::Sender<Job>) {
    let iter = std::iter::once(42u8);
    let k = move || iter;
    let fut = async move { k };
    let job = Job { id: 0, fut };
    let _ = tx.send(job).await;
}
```

In this example, we can see that:

- We can place a `Future` within a `struct` without boxing.
- We can use this "hidden type" in argument position, which allows us to send this type down a channel.
- `impl Trait` can appear multiple times within one type alias, which allows our `Future` to resolve to an unboxed closure.

Without TAIT, we would need to use boxing and trait objects to achieve a similar result.

### Example: Replacing Tokio's `ReusableBoxFuture`

In async Rust, when implementing `Future::poll` for a type, we often want to update some inner future.  For example:

```rust
struct RecvManager {
    inner: ??? // This is a `Future`, but without TAIT we can't name it.
}

async fn make_inner(mut rx: Receiver<T>) -> (Option<T>, Receiver<T>) {
    let v = rx.recv().await;
    (v, rx)
}

impl Future for RecvManager {
    type Output = ();

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        let (v, rx) = ready!(self.inner.poll(cx));
        self.inner = make_inner(rx);
        // ...
    }
}
```

If we just naively box the `inner` future, then we would have to allocate a new `Box` every time the outer future is polled.  Clearly that's undesirable.

To solve this problem, both internally and for its users, Tokio implements a [`ReusableBoxFuture`].  The implementation does a lot of `unsafe` magic.  With TAIT, the future returned by `make_inner` can be named, so all this clever trickery can simply be avoided.

[`ReusableBoxFuture`]: https://docs.rs/tokio-util/0.7.8/tokio_util/sync/struct.ReusableBoxFuture.html

## Quick aside: Why it's important to close expressiveness gaps

The problem with an expressiveness gap in a language is that users often only hit it after having made a substantial commitment to an architecture or code factoring.  The engineer has built an entire wall, and only upon trying to lay the last brick does it become apparent that, though the design is logically sound, it cannot be expressed in the language, so the whole wall has to be torn down and built some other way.

What may be worse is the long-term effects of these expressiveness gaps on language experts.  Because we've deeply internalized these expressiveness gaps, we *stop thinking* of architectures that would be better for the problem because we subconsciously know we'll run into the gap.  The expressiveness gap can start to *stunt our thinking*.

## Motivation 2: Closing the expressiveness gap on captured lifetimes

RPIT captures the lifetimes of any generic type parameters that are in scope.  This can cause the returned hidden type to have undesirably tight bounds.  For example, this code does not work, even though the returned hidden type clearly does not capture any references:

```rust
fn foo<T>(_t: T) -> impl Sized {
    () // Returned hidden type captures lifetimes in T.
}

fn bar<'x>(x: &'x ()) -> impl Sized + 'static {
    foo(x) // Type captures 'x.
}
```
```
error: lifetime may not live long enough
  |
  | fn bar<'x>(x: &'x ()) -> impl Sized + 'static {
  |        -- lifetime `'x` defined here
  |     foo(x) // Type captures 'x.
  |     ^^^^^^ returning this value requires that `'x` must outlive `'static`
```

There are no good ways to work around this in stable Rust today.  As we'll see below, TAIT gives us a way to express the correct bounds for this hidden type.

## Motivation 3: Cleaner APIs with `impl Trait`

Currently, using RPIT in a function exposed in an API is discouraged.  This is because for callers of a function behind an API, it's painful to receive a return value whose type cannot be named.  Such a type cannot be stored unboxed in any data structure.  This reduces the usefulness of RPIT.

TAIT fixes this problem.  Because the type alias or other type (e.g. a `struct` or `enum`) containing the hidden type or types can be exposed via the API, callers to the API can name the types that the API returns.  These types can be placed in data structures and used anywhere any other type may be used.

## Motivation 4: Doing what we said we would do

Back when [RFC 1951] was being debated, these very same problems and motivations were raised.  The feeling was that these were important and should be addressed.  Members of the lang-team and others raised serious concerns about the design in that RFC because of these issues.  The RFC resolved these concerns and justified the expressiveness restrictions that it imposed by leaning heavily on the assumption that we would later stabilize a fully explicit syntax.  It's now many years later.  It's time that we take a step in the direction of fulfilling that assumption.

[RFC 1951]: https://github.com/rust-lang/rfcs/blob/master/text/1951-expand-impl-trait.md

## How it works: Desugaring

### A hypothetical `existential type` syntax

To help with understanding `impl Trait`, let's suppose that Rust supported this explicit syntax (not part of this proposal):

```rust
existential type H: Default; // Not part of this proposal.
```

This would introduce a *type parameter* `H` and add a trait bound such that `H` must implement the trait `Default`.  As with other type parameters in Rust, `H` may be used in places where we would expect to find a type, but the type is opaque.  We can only assume that it implements the traits in the bound (with the exception of leaked auto traits, described below).  We can say, e.g., `H::default()`.

This type parameter is *existential* in the sense that no monomorphization is performed as with type parameters in argument position.  `H` must represent exactly one concrete hidden type.

We can extend this syntax to support type and lifetime parameters:

```rust
existential type H<'t, T>: Iterator<Item = &'t T>; // Demonstration only.
```

This means that, for each type `T`, there is *exactly one* concrete hidden type `H<'_, T>`.  Formally:

```
Ɐ T. Ǝ H. H: Iterator<Item = &'_ T>
```

This is read as: *"for all `T` there exists an `H` such that `H` implements an `Iterator` that returns items of type `&'_ T`"*.

### `impl Trait` in type aliases

The TAIT proposal differs from the explicit syntax above in that the hidden type (`H` above) is anonymous and can not necessarily be named explicitly.  Let's desugar the first example to show how this works:

```rust
type Iter = impl Iterator<Item = u8>;
type Fut = impl Future<Output = impl FnOnce() -> Iter>;

// Desugars to:

existential type _H0: Iterator<Item = u8>;
existential type _H1: FnOnce() -> _H0;
existential type _H2: Future<Output = _H1>;

type Iter = _H0;
type Fut = _H2;
```

We can see that's there's actually nothing special about the *type alias*.  The type alias is just a normal type alias.  Each use of the `impl Trait` syntax simply causes a new anonymous type parameter to be introduced.

### Quick aside: `impl Trait` everywhere

We can see from the above desugaring why it's normal and natural to want `impl Trait` everywhere.  There's no conceptual distinction between a use of `impl Trait` in a type alias and in a struct or enum.  I.e.:

```rust
type S = (impl Default, impl Default);
// Is conceptually similar to:
struct S(impl Default, impl Default); // Not part of this proposal.
```

In both cases, `S` is just a normal type that has "holes punched in it" that are later filled in with concrete types.

This is not part of the proposed stabilization.

## How it works: Constraining the hidden type

As described above, each hidden type represents *exactly one* concrete type.  The concrete hidden type is chosen by type inference within the scope of whatever item contains the `impl Trait`.  Typically this is going to be a module, but it could also be a function or any other kind of item that may contain other items.

Within that scope the hidden type may be used in two different ways:

1. It may be used only for the traits that it implements; or
2. It may be used in a way that constrains it to a particular hidden type.

Outside of that scope, it may only be used for the traits that it implements.

Inside of that scope, the hidden type may be constrained more than once.  But if it is, all of those uses must constrain it to the exact same concrete type.  All items that constrain the hidden type must fully constrain that type.  An item cannot partially constrain a type and rely on other items to fill in the gaps.

For a function to constrain the hidden type, the hidden type must appear in the signature of the function -- in the type of the function's return value, in the type of one or more of its arguments, or in a type within a bound.

Note that we talk here about constraining the hidden type, *not* about constraining "the TAIT" or the type alias.  It's important to remember the hidden types are anonymous, that there can be more than one of them in a single type alias, and that these hidden types can be constrained through encapsulating tuples, `struct`s, `enum`s, etc.

Let's look at some examples of how these hidden types may be constrained.

### Example: Return position

A hidden type that appears in return position may be constrained by the function returning a value of some concrete type:

```rust
type Foo = impl Default; // Let's call the hidden type `_H`.

fn foo() -> Foo { () } // Constrains _H to ().
```

Note that if we simply pass through the hidden type, it has not been constrained:

```rust
type Foo = impl Default; // Let's call the hidden type `_H`.

fn foo(x: Foo) -> Foo { x } // Does not constrain _H.
```

To preserve our ability to make backward compatible changes in the future, this and other non-constraining functions within the scope in which the hidden type was introduced nonetheless are treated by the implementation *as if* they could constrain the hidden type, with an error being thrown later if needed, as we detail in [an appendix](#Appendix-A-Signature-restriction-details).

### Example: Argument position

A hidden type that appears in argument position may be constrained by the function using the hidden type in a way that coerces that hidden type to some concrete type.  For example:

```rust
type Foo = impl Copy; // Let's call the hidden type `_H`.

fn foo(x: Foo) {
    let _: () = x; // Constrains _H to ().
    let _ = x as (); // Also constrains _H to ().
    std::convert::identity::<()>(x) // Also constrains _H to ().
}
```

### Example: Trait bound

A hidden type that appears in a bound may be constrained by its use within the function or by the function's return type.  For example:

```rust
type Foo = impl Sized; // Let's call the hidden type `_H`.

// Constrains _H to ().
fn foo<I: Iterator<Item = Foo>>(i: I) -> impl Iterator<Item = ()> { i }
```

### Example: Statics and constants

A hidden type may be constrained by using it in the type of a static or constant:

```rust
type Foo = impl Default; // Let's call the hidden type `_H`.

const C: Foo = (); // Constrains _H to ().
static S: Foo = (); // Also constrains _H to ().
```

We can see here that it's OK for `_H` to be constrained multiple times, as each use constrains it to the same concrete type.

### Example: Constraining through encapsulation

Remember, we're constraining the hidden type, not the type alias, so it's totally OK to constrain the hidden type through some other type that encapsulates it.  For example:

```rust
type Foo = impl Default; // Let's call the hidden type `_H`.

struct Bar(Foo);

fn foo() -> Bar { // Constrains _H to ().
    Bar(())
}
```

We say here that the hidden type appears in the signature of `foo()` because `Bar` contains within it the existential type parameter `_H` (which is anonymous and cannot be written explicitly).

### Example: Constraining hidden types separately

All hidden types that are introduced within the scope of an item must be constrained within that scope.  However, a function, `const`, or `static` that constrains one of the hidden types does not need to constrain all of them, even if those hidden types are all contained within the same outer type.  For example:

```rust
// Let's call the hidden types `Result<_H0, _H1>`.
type Foo = Result<impl Default, impl Default>;

const FOO_OK: Foo = Ok(()); // Constrains _H0 to ().
const FOO_ERR: Foo = Err(()); // Constrains _H1 to ().
```

This is fine as each item fully constrains each hidden type.

### Example: Cannot partially constrain hidden type

An item that constrains the hidden type must fully constrain that type.  It cannot partially constrain it and rely on other items to fill in the gaps.  For example, this does not work:

```rust
type Foo = impl Sized;

fn foo_ok() -> Foo { Ok(()) } // Error.
fn foo_err() -> Foo { Err(()) } // Error.
```

These items individually try to constrain the hidden type without fully constraining its type.  This results in an error that type annotations are needed.

Note carefully the difference between this and the earlier example.  We can separately constrain two hidden types contained in one concrete type.  But we cannot constrain a hidden type without constraining all of the contained hidden types.

## How it works: Leaked auto traits

Return position `impl Trait` (RPIT) hidden types leak auto traits that are not specified in the bounds.  While callers cannot see the concrete hidden type behind an opaque type returned by the function, they can see whether it implements these auto traits.  For example:

```rust
fn foo() -> impl Sized { () } // We only promised `Sized`

fn is_send<T: Send>(_t: T) {}

fn test() {
    is_send(foo()); // But we can see that it's also `Send`.
}
```

This was a conscious design decision.  On the one hand the behavior is convenient and useful, but on the other it can require the compiler and associated tooling to do more work and it can present SemVer hazards.  These tradeoffs were considered and accepted during the design and stabilization of RPIT.

Type alias `impl Trait` exactly matches the RPIT behavior with respect to leaked auto traits.

## How it works: Associated type position

Everything said here about `impl Trait` in type aliases is also true about `impl Trait` in associated type position, and associated type position `impl Trait` (ATPIT) is part of this stabilization proposal.  For example:

```rust
struct S;
impl Iterator for S {
    type Item = impl Future<Output = ()>; // Let's call the hidden type `_H`.

    fn next(&mut self) -> Option<Self::Item> {
        Some(async {}) // Constrains _H to an unnameable Future.
    }
}
```

For ATPIT, only methods and associated constants on the same impl can constrain the hidden type.  Functions and other items nested within the methods of the impl cannot constrain it because of our existing restriction against using generic parameters from outer scopes in inner items.

## How it works: Capturing in-scope type parameters

Hidden types in RPIT implicitly capture the lifetimes within all generic type parameters in scope when the hidden type is introduced with `impl Trait`.  Hidden types in TAIT work in exactly this same way.  For example:

```rust
mod m {
    //  _H implicitly captures the lifetimes of any references in T.
    pub type Foo<T> = impl Sized; // Let's call the hidden type `_H`.
    pub fn foo<T: Sized>(t: T) -> Foo<T> { t } // Constrains _H to T.
}

fn bar<'x>(x: &'x ()) -> impl Sized + Captures<'x> {
    m::foo(x) // The return type from foo captures 'x.
}
```

[Playground link](https://play.rust-lang.org/?version=nightly&mode=debug&edition=2021&gist=d0c78cb47af81016c6b307f4dc03b6e0)

Similarly for ATPIT:

```rust
struct S<T>(Option<T>);
impl<T: Clone> Iterator for S<T> {
    // _H implicitly captures the lifetimes of any references in T.
    type Item = impl Clone; // Let's call the hidden type `_H`.

    fn next(&mut self) -> Option<Self::Item> {
        Some(self.0.clone()) // Constrains _H to T.
    }
}

fn bar<'x>(mut x: S<&'x ()>) -> impl Clone + Captures<'x> {
    x.next().unwrap()
}
```

[Playground link](https://play.rust-lang.org/?version=nightly&mode=debug&edition=2021&gist=c7ff8fdc0734e13499aab60892e0b27b)

Note, however, that the lifetimes of generic type parameters that are only in scope of where the hidden type is *used* are not captured.  Therefore this code works:

```rust
mod m {
    // No type parameters are in scope, so the hidden type
    // does not capture any.
    pub type Foo = impl Sized; // Let's call the hidden type `_H`.

    pub fn foo<T>(_t: T) -> Foo { () } // Constrains _H to ().
}

fn bar<'x>(x: &'x ()) -> impl Sized + 'static {
    m::foo(x) // Hidden type does not capture 'x.
}
```

This allows for expressing correct bounds on hidden types in a way that is not possible in stable Rust today.

## How it works: Generic parameters must be used generically

When the hidden type captures a type parameter, that type parameter must be used generically.  It's an error to fill it in with a concrete type.  For example, this code is invalid:

```rust
type Foo<T> = impl Sized;

fn foo() -> Foo<()> { () }
```
```
error[E0792]: expected generic type parameter, found `()`
  |
  | type Foo<T> = impl Sized;
  |          - this generic parameter must be used with a generic type parameter
  |
  | fn foo() -> Foo<()> { () }
  |                       ^^
```

## How it works: The signature restriction

When a function constrains a hidden type to a particular concrete type, the hidden type must appear somewhere in the signature of the function -- in the type of the function's return value, in the type of one of its arguments, or in a type within a bound.

Note that, as in many of the examples above, a hidden type may be nested arbitrarily deeply within other types, and those outer types appearing in the signature satisfy this requirement.  For example:

```rust
type Foo = impl Sized; // Let's call the hidden type `_H`.
struct Bar(Foo);

fn foo() -> Bar { // Bar contains _H, so this is OK.
    Bar(()) // Constrains _H to ().
}
```

When a function contains a nested inner function, the inner function may not constrain hidden types introduced outside of the outer function unless the hidden type appears in the signature of the outer function.  For example, this is invalid:

```rust
type Foo = impl Sized;

fn foo() {
    fn bar() -> Foo { () }
}
```

## Stabilization proposal summary

In summary, we propose for stabilization:

- Type aliases and associated types may contain multiple instances of `impl Trait`.
- Each `impl Trait` introduces a new hidden type.
- The hidden type may be constrained only within the scope of the item (e.g. module) in which it was introduced, and within any sub-scopes thereof, except that:
  - Functions and methods must have the hidden type that they intend to constrain within their signature -- within the type of their return value, within the type of one or more of their arguments, or within a type in a bound.
  - Nested functions may not constrain a hidden type from an outer scope unless the outer function also includes the hidden type in its signature.
- The hidden type may be constrained by functions, methods, constants, and statics.
- The hidden type leaks its auto traits; these can be observed through the opaque type.
- The hidden type implicitly captures the lifetimes within all generic parameters in scope.
- Generic type parameters captured by the hidden type must be used generically.

## Next steps

Here are the proposed steps for moving forward:

- Start FCP on [#107645] "TAIT defining scope options" (or simply close it).
- Oli will submit PR with final TAIT/ATPIT semantics.
- Merge that PR.
- Post stabilization report to tracking issue [#63063] and start FCP.
- Stabilize TAIT/ATPIT.

[#63063]: https://github.com/rust-lang/rust/issues/63063
[#107645]: https://github.com/rust-lang/rust/issues/107645

## Appendix A: Signature restriction details

### Implementation considerations

The signature restriction that's part of this proposal simplifies the implementation and improves its performance.

Because of this restriction, outside of a function, we can check the opaque types for which that function may register hidden types by just looking at its signature.  Using this information, we can avoid computing typeck for that function when we need to resolve hidden types that the function cannot possibly register.

Inside of a function, because we know the opaque types for which we may register hidden types before we start typeck, we can create a single inference variable for the hidden type ahead of time.  That allows us to perform inference across all use sites of the opaque type within that function.  This is what RPIT already does across the `return` statements and the trailing `return` expression.

The net result is that we can use better caching and generally be more performant because we don't have to carry information about the current function into all cache keys for the various things that we try to prove during typeck.

### Cycle errors

If a function defined within the scope in which an `impl Trait` hidden type is introduced does not register that hidden type, but does ask whether its corresponding opaque type implements an auto trait (e.g. `Send`), we need to compute the concrete hidden type to check that bound.  That involves further type checking.  If we have to assume that this function could itself register the hidden type, then this will produce a cycle error.

For code affected by this, the workaround is either to avoid checking for the implementation of an auto trait on the opaque type or to move the hidden type into a more narrow scope.

### Planning for backward compatibility

With the signature restriction in this proposal, we *could* eliminate some cycle errors.  Because we can determine before we start typeck the opaque types for which a function may register hidden types, within the function we could reveal the leaked auto traits for those hidden types that the function could not possibly register.  We are proposing not to do this for now.

Under this proposal, for the purpose of computing cycle errors, we conservatively assume that all functions in the scope might register the hidden type *and* we throw any errors needed according to the rules of the signature restriction.  This produces the maximum possible number of cycle errors.  We do this to preserve the maximum scope for making future changes to this feature in a backward compatible way.  The rule is that we need to know before running typeck which opaque types a function may register hidden types for implicitly in the future.

Here's how the implementation works:

- We scrape the hidden types in the signature of each function in the same module or in a child module of where the hidden type was introduced.
- For ATPIT, we scrape mentions of the associated type from the signatures of the methods and associated functions.  This is done iteratively by looking for associated types, then normalizing each to its concrete type and repeating the process.
- If a function within this scope tries to constrain a hidden type that did not appear in the signature, we throw an error.
- If a function within this scope that does not constrain the hidden type uses the opaque type in such a way that we would need to leak its auto traits, we throw an error.

If we were to decide at any point to commit to never removing the signature restriction, we could accept more correct code without cycle errors.

## Appendix B: The IDE concern

To maximize responsiveness, IDEs try to minimize the amount of work that they need to do to infer types and provide completions.

Without the signature restriction proposed here, because `impl Trait` opaque types leak auto traits, IDEs would have to check more function bodies than they do today to provide correct completions and type inference annotations.  The problem the IDEs face is similar to the problem faced by the compiler itself when performing incremental compilation.

Because of the proposed signature restriction, this is not an issue.  The IDEs do not have to type check a function body to determine whether a hidden type may be constrained by that function.

@matklad, the author of rust-analyzer, has confirmed that with the signature restriction, TAIT is equivalent to RPIT in terms of the complications it presents for the IDE.

Even if we were to remove the signature restriction, the following considerations mitigate the IDE concern:

- Even today, because of leaked auto traits and RPIT, rust-analyzer presents incorrect completions and type inference hints.  TAIT cannot break anything that is not already broken.
- Stable Rust today allows items to be nested in ways that, even without RPIT or TAIT, require IDEs and other tooling to check all function bodies to correctly infer all types.  There is a proposal (as-yet not accepted) that would add restrictions to Rust in a future edition to remove this requirement.
- [RFC 1522], which decided to leak auto traits, explicitly acknowledged and accepted the additional burdens that this decision put on all tooling.

[RFC 1522]: https://github.com/rust-lang/rfcs/blob/master/text/1522-conservative-impl-trait.md

## Questions

### How do I leave a question?

ferris: How do I leave a question?

Like this! Make a new section, summarize, and then give details

### "limited to other generics"

tmandry asks... 

> What about functions nested in other items, like modules? It seems inconsistent to apply this rule only to functions that contain other functions, when nested functions don't otherwise interact with their parent function's generic params.

...

> and are we enforcing it only on things that actually constrain, or things that could constrain?

```rust
type Foo<A> = impl Sized;

fn bar() { 
    let f: Option<Foo<u32>> = None;
}

fn baz<A>(x: Foo<A>) { 
    let f: Option<Foo<u32>> = None;
}
```

compiles: https://play.rust-lang.org/?version=nightly&mode=debug&edition=2021&gist=b513dbabbfe151eca5b9f10069e7903b

```rust
#![feature(type_alias_impl_trait)]

type Foo<A> = impl Sized;

// fn bar() { 
//     let f: Option<Foo<u32>> = None;
// }

fn baz<A>(mut x: Foo<A>) { 
    let f: Option<Foo<u32>> = None;
    x = 22;
}

fn main() { }
```

if we were introducing a constraint by using `Some`, we would get 

```rust
#![feature(type_alias_impl_trait)]

type Foo<A> = impl Sized;

// fn bar() { 
//     let f: Option<Foo<u32>> = None;
// }

fn baz<A>(mut x: Foo<A>) { 
    let f: Option<Foo<u32>> = Some(22);
    x = 22;
}

fn main() { }
```

gets error

```
error[E0792]: expected generic type parameter, found `u32`
  --> src/main.rs:10:31
   |
3  | type Foo<A> = impl Sized;
   |          - this generic parameter must be used with a generic type parameter
...
10 |     let f: Option<Foo<u32>> = Some(22);
   |                               ^^^^^^^^
```

### fns within fns

tmandry: What about functions nested in other items, like modules? It seems inconsistent to apply this rule only to functions that contain other functions, when nested functions don't otherwise interact with their parent function's generic params.

oli-obk: this is not current behavior, but it's meant for IDE convenience.

nikomatsakis: IDE authors don't want to have to *parse* fn body

oli-obk: right, but as long as we allow impls in there, it's a moot point.

nikomatsakis: I think you're saying oli:

* we can permit it in nested functions now
    * means that IDE authors have to parse but they have to anyway because impls etc
* then in an edition we take that out

oli: or we can trivially make it an error and then decide to permit it later, just by erroring out more often, this would avoid an edition-dependent behavior later

nikomatsakis: important that this works, I think

```rust
fn foo() {
    type Bar = impl Debug;
    fn bar() -> Bar { }
}
```

nikomatsakis: common trick that I've seen in derives etc to make names that don't appear outside is to do this...

```rust
const _: () = {
    
}
```

...so what about something like this?

```rust
type MyData = impl Debug;
const _: () = {
    fn foo() -> MyData { 22 }
    
};
```

oli: would be an error now, though you could do `const _: MyData = ...`.


```rust
type MyData = impl Debug;
const _: fn() -> MyData = {
    fn foo() -> i32 { 22 }
    foo
};
```

nikomatsakis: I've seen this pattern used in e.g. synstructure crate, where it always wraps with `const _: () = {}` for "poor man's hygiene"

nikomatsakis: any modificaiton that requires you to name the TAIT will require some modifiation to constrain

### "Constraining through encapsulation"

nikomatsakis: This is interesting and somewhat unexpected to me. It appears that the rule descends through all types reachable from the fields? How do we define the limits here? For example:

```rust
mod foo {
    pub struct Ptr<T> {
        p: *mut (),
        d: PhantomData<T>,
    }
}

mod bar {
    type Bar = impl Debug;
    
    fn something() -> crate::foo::Ptr<Bar> { }
}
```

I imagine partial motivation was things like `Option<T>`? I think that we could also say "all TAITs that appear in the signature" and include types appearing in generic arguments of other types, as a "privacy-respecting" alternative. However, I'm not able to come up with a way that a scheme that says *either* appearing directly *or* reachable from fields can violate privacy per se?

This has *some* implication for IDEs, they must traverse type definitions, but they also have to do trait matching so that doesn't seem like the most important thing.


TC: this is important for "impl trait everywhere" to work as you expect, since the opaque type in `struct Foo(impl Trait)` requires `Foo`.

TC: The code above works only if you annotate the type of the PhantomData:

```rust=
#![feature(type_alias_impl_trait)]
#![feature(impl_trait_in_assoc_type)]

use std::marker::PhantomData;

mod foo {
    use super::PhantomData;
    pub struct Ptr<T> {
        pub p: *mut (),
        pub d: PhantomData<T>,
    }
}

mod bar {
    use super::PhantomData;
    type Bar = impl Sized;
    
    fn something() -> crate::foo::Ptr<Bar> {
        crate::foo::Ptr {
            p: &mut () as *mut (),
            d: PhantomData::<()>,
        }}
}
```

[Playground link](https://play.rust-lang.org/?version=nightly&mode=debug&edition=2021&gist=d54918e382900a533faf4236594549b1)

what about this example:


```rust
mod foo {
    pub struct Iteratee<T: Iterator> {
        d: <T as Iterator>::Item;
    }
}

mod bar {
    type Bar = impl Debug;
    
    fn something<X>(x: X) -> Iteratee<X>
    where
        X: Iterator<Item = Bar>
    {
        Iteratee { d: 'x' } // constraints `Bar` to be `char`?
    }
}
```

[Playground link](https://play.rust-lang.org/?version=nightly&mode=debug&edition=2021&gist=507759b33f1e68d8bfac2284d47c7d93)

TC [after meeting]: Yes, that's correct.

### ATPIT

nikomatsakis: In the section on associated type position impl trait (ATPIT), it says...

> only methods and associated constants on the same impl can constrain the hidden type.

...and I think this means "and only when they meet the same requirements that would be needed for a top-level function", is that correct?

TC: Yes.

### signature restriction

nikomatsais: This text...

> When a function constrains a hidden type to a particular concrete type, the hidden type must appear somewhere in the signature of the function

...I think also means "or an associated type reference that normalizes to it", right? It's interesting because it's a case where fixing a bug or limitation in the compiler such that it is able to do more normalization than it was could cause previously out of scope uses to become in-scope. I think it's ok, but it's a good thing to be aware of, once of those cases where more complete trait selection can cause breakage.

### behavior in a function

nikomatsakis: When a functon constrains an opaque type `Foo`, can it "see through" the opaque type during trait matching?

```rust
type Foo = impl Debug;

fn test() -> Foo {
    let x: Foo = 22_u32;
    println!("{x:?}"); // OK or error?
    x
}
```

TC: It never "sees through" the opaque type, but the code above is not (logically) an error (as @oli says, it is an error now).  It's more correct to say that in that example the function is seeing the concrete type it has selected.

oli: it is an error right now, but there are avenues for making it pass that may be desirable from an impl perspective.

### closing, stabilization report

tmandry: I'd like to see a table listing out the positions being stabilized and rationale















----
### current diagnostics

(low priority topic; getting semantics straight is more important)

pnkfelix: how are we feeling about quality of the current diagnostics? I'm specifically worried about "error: concrete type differs from previous defining opaque type use" without pointing to the introduction of the impl Trait itself, though I might just be [misreading some current diagnostics](https://play.rust-lang.org/?version=nightly&mode=debug&edition=2021&gist=e06d75ceebac75146fe47c984ab09a43). See [also](https://play.rust-lang.org/?version=nightly&mode=debug&edition=2021&gist=f02970b4687f58cc6541f893e0ade107).

---

(Leaving space above so people can add comments in parallel by taking different existing blank lines)

TODO: table with examples linked to rationale
