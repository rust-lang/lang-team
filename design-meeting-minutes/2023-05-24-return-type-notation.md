---
tags: design-meeting
---

# Return Type Notation

## The "send bound" problem

Users of async fn in traits quickly hit the problem that they need a way to add "send bounds" to the futures returned by methods in the trait. We call this the "send bounds" problem. In fact, the problem is more general, and applies to any `-> impl Trait` method defined in a trait. In this doc though we will focus on examples of async functions and send bounds, with only minimal discussion of other examples and applications.

The async WG conducted a series of case studies in which we found that stabilizing async functions in traits *without* a solution to the send bounds problem is not sufficient for many desired use cases. Therefore, to meet our goal of shipping usable (if not perfect) support for async fn in traits in 2023, we must solve the send bound problem in some way.

## Proposed solution: return type notation

We are proposing a solution dubbed "return type notation". There is a [draft RFC][] available[^name] that covers a minimal subset. The core idea is simple. Users can add where clauses like `T::m(): Send` to bound the return type of invoking the method `m` on a value of type `T`. Alternatively, users can leverage [associated type bounds][] (RFC #2289) notation like so `T: Trait<m(): Send>`. Only this second form is implemented, presently, so we will focus on that.

[associated type bounds]: https://github.com/rust-lang/rust/issues/52662

[^name]: The RFC uses the term "associated return type", but the name seems unpopular and doesn't seem particularly better, so I'm giving up on it. --nikomatsakis

[draft RFC]: https://hackmd.io/@wg-async/H1qTVUMgn

## How it works: HRTB

A where clause like `T: Trait<m(): Send>` expands to a higher-ranked trait bound. Let's work through an example:

```rust
trait HealthCheck {
    async fn check(&mut self, server: Server);
}

fn spawn_check<HC>(hc: HC)
where
    HC: HealthCheck<check(): Send> + Send + 'static,
    //              -------------    --------------
    //                   |                  |
    // What we are interested in;           |
    // says that future returned by         |
    // check() is Send.                     |
    //                                      |
    //                   What will, unfortunately, also
    //                   be necessary much of the time.
    //                   We intend to ship a proc-macro
    //                   to reduce boilerplate in the
    //                   short-term (discussed below)
    //                   and pursue other solutions in
    //                   longer term.
{}
```

As discussed in the [RPITIT RFC][], the `Healthcheck` trait is "desugared" into a trait with an (anonymous) associated type `$`, like:

[RPITIT RFC]: https://github.com/rust-lang/rfcs/pull/3425

```rust
trait HealthCheck {
    type $<'a>: Future<Output = ()>;
    fn check(&mut self, server: Server) -> Self::$<'a>;
}
```

We can express the `HealthCheck<check(): Send>` bound as a higher-ranked trait bound on `$` like:

```rust
where
    HC: HealthCheck,
    for<'a> HC::$<'a>: Send,
```

The bound `check(): Send` then says: no matter what types `check` is invoked with, the result is `Send`.

## Limitation: only RPITIT methods, only lifetime parameters

In the interest of minimality, the current [draft RFC][] limits RTN to RPITIT methods (including async functions) that only have lifetime parameters. The following methods would not (yet) be supported:

```rust
trait Foo {
    // Not supported: Not async nor RPITIT
    fn foo(&self) -> String; 
}

trait Bar {
    // Not supported: type parameter T
    fn bar<T: Display>(&self, t: T) -> impl Display; 
}

trait Baz {
    // Not supported: anonymous type parameter from 
    // `impl Display` in argument position.
    fn baz(&self, t: impl Display) -> impl Display;
}
```

The first limitation (just RPITIT) is easiest to lift.

The second requires two things. First, support for higher-ranked trait bounds parameterized over types ([available on nightly but still experimental](https://github.com/rust-lang/rust/issues/108185)) with "where clauses" (not yet implemented). Second, some decision about whether to include other default bounds. For example, consider `X: Bar<bar(): Send>` with the example above. Should that be `for<T: Display>`, i.e., return type is send for all `T`, or do we want `for<T: Display + Send>`? The latter seems too hard-coded, but the former will almost certainly not be true. We choose to defer this question until we have more concrete use cases to work from.

## Experience

As part of the async WG's case studies, we evaluated how well RTN works in practice. We found that it was easy to explain and it works "OK" for simple cases, like the one shown above, though writing `HealthCheck<check(): Send> + Send + 'static` still felt suboptimal (how many times do we have to say `Send`?). For more complex cases where there are multiple methods being invoked, it simply doesn't scale.

The workaround is to introduce an "trait alias":

```rust
trait SendHealthCheck: 
where
    Self: HealthCheck<check(): Send>,
    Self: Send + 'static
{}

impl<T> SendHealthCheck for T
where
    T: HealthCheck<check(): Send>,
    T: Send + 'static
{}
```

Now users can write 

```rust
fn spawn_check<HC>(hc: HC)
where
    HC: SendHealthCheck,
{}
```

A number of the developers we spoke with described plans to create aliases of this kind. Tower, for example, expected to make the alias the *primary* trait (i.e., `Service`) and then have the non-send variant be called `LocalService`.

To resolve some of the boilerplate, a [proc macro has been implemented](https://github.com/google/impl_trait_utils).

## Other uses for RTN

RTN can be used for things beyond the "send bound" problem. Consider the `IntoIntIterator` trait introduced in the [RPITIT RFC][]:

```rust
trait IntoIntIterator {
    fn into_iter(self) -> impl Iterator<Item = u32>;
}
```

Users could write a function that requires a `DoubleEndedIterator` (and hence can be reversed) like so:

```rust
fn gimme<T>(t: T)
where
    T: IntoIntIterator<into_iter(): DoubleEndedIterator>
{
    for i in t.into_iter().rev() { ... }
}
```

(If we wind up stabilizing `#[refine]`, this function would only be callable for impls that `#[refine]` the trait to promise the stronger bounds.)

## Concerns

A number of concerns have been raised about RTN. This section walks through them.

### Doesn't scale to larger examples

> The syntax as defined constraints the return of only one method at a time. But most traits have more than one method. Aren't users going to be immediately frustrated?

Yes, many users will definitely be unsatisfied with send bounds on their own, which is why we want to provide a proc-macro to alleviate the boilerplate of creating an alias. In the future a solution that applies to many methods at once would be a nice extension.

### Confusing, confusable with const generics

> The syntax `foo()` looks like an expression but is actually a type; it could be confusing that no actual function call occurs, particularly given that const generics permit `{foo()}` notation, in which the function is called at compilation time. It's also confusing that `foo()` works no matter how many arguments `foo` has.

It's true that under this proposal the interpretation of `T::m()` would depend on whether it appeared in a type or expression context. It's also true that `T::m()` (as a type) is only permitted in a very limited set of contexts right now, specifically the subject of where bounds like `T::m(): Send` or `T: Foo<m(): Send>`. Note that nested usages (e.g., `where Foo<T::m()>: Send` or `T::m()::Item`) are not included, nor are types outside of where bounds like `let x: T::m() = ...`. These ideas are discussed under the Futures Possibilities section, but there are some significant complications for the latter (the former is possible without great trouble, if ill-motivated).

It's not obvious though that having types and expressions look like one another is a problem. In fact, we consciously *tried* to make constructors and their types line up in other cases, e.g. `(id1, id2)` or `[id1; n]`. While no doubt some users will find notation like `T: HealthCheck<check(): Send>` confusing, we've not seen a lot of evidence of this so far in our ancedotal experimentation. 

One particularly interesting example is type vs const generics. For example `T<foo()>` and `T<{foo()}>` would have quite different interpretations. Today the proposal does not permit one to write `T<foo()>`, but one could imagine it, perhaps only in where-clauses like `where Rc<T::m()>: Something`. In cases like this, the compiler can easily detect the mistake and suggest adding (or removing) the `{}`.

### Too general, YAGNI.

> The syntax as proposed permits users to place all kinds of bounds, not just `Send` bounds or other auto-traits. Do we really need that? Can we do something more tailored to `Send` and `Sync`?

Answer: There are also reasons that you might want to place other bounds when you consider RPITIT, as we demonstrated with the `IntoIntIterator<into_iter(): DoubleEndedIterator>` example.

### Too narrow, where is this going?

> As currently proposed, RTN can only be used in where clauses, applies only to RPITIT trait methods, and always constraints the return type for all possible argument types. As currently *implemented*, RTN can only be used in associated type bound position (i.e., `HC: HealthCheck<check(): Send>` and not `HC::check(): Send`). The future possibilities section below walks through ways that all of these limitations could be lifted, but each of them has tradeoffs of their own (even ignoring implementation complexity), and it's not obvious that we want to do them. If we never accept any of those ideas, won't this feature feel half-baked forever?

Half-baked is of course a mater of opinion, but it makes sense for us to think about how far we want to take this feature. The future possibilities section outlines a number of extensions we could consider that loosely build on one another:

Using in more places:

* Use `HC: HealthCheck<check(): Send>` (implemented now)
* Permit use as top-level type in a where-clause, e.g.
    * `where HC::check(): Send`
* Permit use as a type elsewhere in where-clauses, e.g.
    * `where Foo<HC::check()>: Send` 
* Allow `HC::check()` as a type in other contexts, e.g.
    * `let x: HC::check() = ...`
    * `fn foo<HC>(x: HC::check()) where HC: HealthCheck`

Expanding range of things you can name: 

* Permit naming and constraining free functions like `some_function()`
* Permit local variables and other expressions like `c()`

Niko's estimate is that the following extensions are "relatively straightforward" to implement:

* Permit use as top-level type in a where-clause
* Permit use as a type elsewhere in where-clauses
* Allow `HC::check()` as a type in other contexts
* Permit naming and constraining free functions like `some_function()`

...but this one is pretty dang hard and is likely out of scope for all time:

* Permit local variables and other expressions like `c()`

## Alternatives

Here are some of the alternatives that have been proposed and 

### type-of

The compiler currently supports a `typeof` operation as an experimental feature (never RFC'd). The idea is that `typeof <expr>` type-checks `expr` and evaluates to the result of that expression.

It might appear that `typeof`  can be used in a similar way to RTN, but in fact it is significantly more complex. Consider our first example, bounding the `HealthCheck` trait:

```rust
fn start_health_check<H>(health_check: H, server: Server)
where
    H: HealthCheck + Send + 'static,
    H::check(): Send, // 👈 How would we write this with `typeof`?
```

To write the above with `typeof`, you would do something like this

```rust
fn dummy<T>() -> T { panic!() }

fn start_health_check<H>(health_check: H, server: Server)
where
    H: HealthCheck + Send + 'static,
    for<'a> typeof H::check(
        dummy::<&'a mut H>(),
        dummy::<Server>(),
    ): Send,
```

`typeof` on its own fails the "ergonomic enough to use for simple cases" threshold we were shooting for. But it's also a significantly more powerful feature that introduces a *lot* of complications. We were able to implement a minimal version of ART in a few days, demonstrating that it fits relatively naturally into the compiler's architecture and existing trait system. In contrast, integrating `typeof` would be rather more complicated. To start, we would need to be running the type checker in new contexts (e.g., in a where clause) at large scale in order to normalize a type like `typeof H::check(x, y)` into the final type it represents.

With `typeof`, one would also expect to be able to reference local variables and parameters freely. This would bring Rust full on into dependent types, since one could have a variable whose type is something like `typeof x.method_call()`, which is clearly dependent on the type of `x`. This isn't an impossible thing to consider -- and indeed the same could be true of some extensions of ART, if we chose to permit naming closures or other local variables -- but it's a significant bundle of work to sort it out.

Finally, while `typeof` clearly is a more general feature, it's not clear how well motivated that generality is. The main use cases we have in mind are more naturally and directly handled by ART. To justify `typeof`, we'd want to have a solid rationale of use cases.

### `::Output` and higher-ranked defaults

Instead of special syntax for function calls, we could allow lifetime elision in more places and make it translate to higher-ranked bounds.

This would mean given a trait like

```rust
trait MyTrait<'a> {
    async fn some_method(&i32);
}
```

(given associated type bounds) you could write a bound like

```rust
where T: MyTrait<some_method::Output: Send>
```

which would translate to something like

```rust
where for<'a> T: MyTrait<'a>,
      for<'b> <T::some_method as Fn<(&'b i32,)>>::Output: Send,
```

Our reasons for not recommending this approach are:

1. It is not well-scoped to the particular problem we are trying to solve, and aren't sure if this is something we want more generally.
2. RTN syntax gives us an extension point for things that might make more sense for function calls (like specifying argument types), analogous to the `Fn()` bound syntax that we already have.
3. It is less ergonomic to write than RTN, though not that much so.

Finally, we can always make RTN desugar to this kind of bound in the future if we choose to generalize the pattern beyond function traits.

### `::return` notation

## Future possibilities

As currently proposed, RTN can only be used in where clauses and applies only to trait methods. There are a number of ways that the feature could be extended.

**Listing a possibility is not endorsement.** In fact, some of the authors of this document (nikomatsakis, at least) do not favor most of these options at this time.

### Specifying the types of arguments

Currently the RTN proposal only supports `foo(): Send` which constraints the return type of `foo` for *all possible* argument types. One could imagine wanting to be more specific  about what arguments will be provided. For example, `where HC: HealthCheck<check(&'a mut HC, Server): Send>` might specify that that the first argument will *specifically* be a `&'a mut HC` and thus not require a higher-ranked bound. As this example suggests, providing specific arguments raises some questions:

* Do we wish to list `self` type explicitly? Is there a difference between `HC::check(_, _)` in this regard?
* Do we have to repeat "non-varying" types, e.g., `Server`? Can we use `_` instead?
* Would this ever be useful?

On the final point, in the case of `HealthCheck`, the lifetime of `self` is unlikely to influence whether the result is `Send`. However, for functions that have `impl Trait` arguments, those arguments correspond to generic type parameters that would be captured by the resulting `impl Trait` return type, e.g.

```rust
impl Trait for X {
    fn foo(x: impl Foo) -> impl Bar
}
```

Here, the `impl Bar` return type captures the `impl Foo` API. So if you want to say that `X::foo(): Send`, you would have to ensure that the `impl Foo` type is `Send`.  A notation like `for<T: Send> X::foo(T): Send` would be one way to do that. Another would be to find some way to specify that `impl Foo` can be used via turbofish and support `for<T: Send> X::foo::<T>(): Send`.

### Embedded RTN in where-bounds

We could permit RTN embedded within types that appear in where bounds, such as the following

* `Foo<T::m()>: Send`
* `T::m()::Item: Send`
* `T: Trait<T::m()>`

or even multiple RTN

* `T: Trait<T::m(), U::n()>`

Each of these can be translated in a relatively straightforward way. However, it may be surprising to permit `T::m()` in these "type-like" positions without going "all the way" to permiting `T::m()` in other type contexts. This latter extension is covered in the next section.

### RTN as a type

Building on the previous point, we could permit RTN to be used wherever types are used, for example:

```rust
fn foo<HC: HealthCheck>(mut hc: HC) {
    let x: HC::check() = hc.check();
}
```

This raises up some interesting questions. When one writes `HC: HealthCheck<check(): Send>`, it means *for all possible calls to `check`*. But when writing `let x: HC::check()`, it would presumably mean *for some call to `check()`* -- i.e., it would presumably expand to something like `HC::$<'_>` -- at which point it represents *some value* returned by `check` and not *every possible value* (as it does in the where-clause position). This is not necessarily a problem, in fact, it is in some sense quite natural and could align well with lifetime elision -- the idea being that `'_` in where-clauses (currently an error) could represent "any lifetime" but in function bodies it represents "some lifetime". There are some details to iron out if we want this to work, e.g., what happens in arguments like `impl Trait<'_>`, where the `Trait<'_>` is both a bound and an argument type. 

### RTN for free functions

Building yet more on the previous point, we could extend RTN so that it is usable not only for trait methods (`X::foo()`) but also for *free functions* (`foo()`). One could then write `where foo(): Send` to bound the return type of a function like:

```rust
async fn foo(data: &[u8]) -> u32 { ... }
```

Unlike with traits, it's not so clear why you would want to do this. Send bounds leak, so the where-clause doesn't add any information. And you can't use this notation with anything that is not an auto trait because `-> impl Trait` specifies the full bounds that you can rely on.

### RTN for closures

Another concern is that supporting RTN for free functions suggests it could also be used for closures or local variables:

```rust
let x = || 22;
let y: x() = x();
```

This is *very very* close to full-blown dependent types, as the type of `y` can reference a local variable `x`; that said, unlike other dependently typed languages, this would not permit capturing or constraining arbitrary values, and the result is *really* just a projection of the `Output` associated type, so it likely doesn't make types flow dependent in a new way (for example, if `x` were mutable, users would presumably still not be able to change the return type by overwriting `x` with a new value). Still, despite seeming like a small extension, this is a non-trivial extension to the 
type system.

### RTN as an alternative to TAIT

On its own, RTN for free functions doesn't make much sense, but if you combine it with the ability to use RTN in types, one could write

```rust
async fn foo(data: &[u8]) -> u32 { ... }

...

let x: foo() = foo();
```

This is effectively an alternative form of TAIT. The same idea is possible with TAITs, but it requires modifying the definition of `foo`:

```rust
type Foo<'a> = impl Future<Output = u32>;
fn foo(data: &[u8]) -> Foo<'_> { async move { ... } }

...

let x: Foo<'_> = foo();
```

(There may be some trick that avoids modifying the definition of `foo`, but if so, I've not thought of it yet. --nikomatsakis)

## Questions and comments

### A note on framing

Yosh: Something I haven't really seen mentioned here I think, but might be important to ask: what role do we expect this feature to fill? Is it an ergonomic high-level feature we expect everyone to use? Or is it more of a building block, that represents a fundamental Rust feature but we expect will only see specialized use?

Yosh: At least the way I've been thinking about ART/RTN is that it necessarily represents a fundamental building block where ergonomics should be "good enough", but more important is that it provides the features we want.

nikomatsakis: I believe goal is to have a building block first and foremost, but I would also like it to be usable for simple cases and explainable.

nikomatsakis: whatever successor we do is going to take 6 months - a year to be stabilized, so it's something we're going to have to live with for a bit.

tmandry: and there may not even be a successor feature for a while, that's always possible.

### Big question: RTN vs `X::return` or `X::Output`

nikomatsakis: Josh, who is unfortunately not here, had a long discussion with me in which he advocated three positions:

* We should not support `HC: HealthCheck<check(T): Send>` as the way to constrain or specify the value of parameters, but we should rather have some way to make `impl Trait` arguments be listed in the turbofish listing.
* The `foo()` notation used in RTN was going to lead to user confusion because it yields the return type and doesn't execute the function particularly when contrasted with const generics or if permitted in positions like `let x: foo() = ...` where expressions would otherwise occur.
* RTN is too verbose to be widely typed anyway and people are going to use trait aliases, so it'd be ok to do something even more verbose, e.g. `HC: HealthCheck<check::return: Send>`

I was rather persuaded of the 1st one, but after reflection feel unconvinced on the other two. But I want to surface this question first and foremost before going into details of RTN.

nikomatsakis: my take is roughly that I want to be able to teach this without being embarassed. I see that teaching `::Output` would be nice in that it's reusing a concept, so I like that, but `::return` feels like the worst of both worlds to me. Special *and* complex.

scottmcm: `::return` feels very long, I agree

pnkfelix: I'd prefer not to have an argument based on terseness because it's coupled to `(..)` vs `()`, which I don't want.

scottmcm: also comes to how much this is the desugared form vs all the time form, as Yosh was asking earlier.

eholk: I like `::Output` in that it's reusing an existing concept, we already have `Fn` traits with `Output`. I do think terseness is important though. Already the `()` is stretching the limits of what we can type. It's why we got rid of the `..`, which was very painful to write. If we end up going with `::Output` it's really important that people never write this but use macros, like the one google just released.

nikomatsakis: is it on github?! :tada: 

tmandry: yes! not crates.io. Needs docs.

pnkfelix: another reason is that these will be used in bounds, right? So it's not just `foo::return` it's `foo::return: Send`, right? that's terrible.

yosh: feels semantically different from how `Future` uses `Output`. It could be confusing that `Output` refers to the future in some cases but the value of awaiting in others.

pnkfelix: how does one distinguish future being returned vs the future? would it be `foo().await`?

nikomatsakis: that's certainly an option. 

pnkfelix: seems like we're working our way towards a typeof?

nikomatsakis: yes, somewhat, but things that desugar to associated type references are pretty different from actual expressions.

pnkfelix: Agree, but I'm thinking about the mental model we expect people to have. I don't mind `().await`, it's actually pretty natural, but I do want us to be clear on what trajectory we expect to be on.

nikomatsakis: I don't know how often that will come up.

TC: I don't like the verbosity of `::return`, but reading through the document I see suggestion that `()` will be a problem in terms of generalizing this to more type positions. If there were a syntax that didn't cause a problem in generalizing this to more positions, maybe we'd extend it to more positions faster, and we might wind up with a more consistent language overall.

nikomatsakis: are you saying there are *technical* problems from the `()` or *readability* problems?

TC: I anticipate social problems, just based on how much discussion this occupies in the document.

nikomatsakis: that came out from my conversation with Josh but I assume his take is representing some segment of users.

TC: I don't have a problem with the parens, but I do have a problem with stabilizing but not generalizing to all type positions. I feel like we'll end up with irregularity.

nikomatsakis: certainly a known failure mode of Rust.

nikomatsakis: anyone who wants to argue strongly in defense of another option?

pnkfelix: `::Output` seems natural, but only because I know the trait well, and most users prob don't. Seems elegant if you know the language in depth.

tmandry: I agree to some extent. Also think that functions are special, e.g., `Fn` trait syntax, coupled with the potential for confusion that Yosh mentioned. My feeling is "yeah if we want a really regular language we can spread this `::Output` thing and generalize it to all bounds". That might be a nice story to tell. But in practice I think you want an extra syntax that makes it easier.

nikomatsakis: it is worth pointing out that `()` could also be a sugar in the future for `::Output`. 

TC: because of the future thing that was mentioned, if you wanted to constrain the value of what was returned by the future, you would do `HC::check()::Output`, which makes sense to me.

nikomatsakis: agreed that would be the obvious thing.

### Using `foo()` to mean "any possible args" is too constraining

pnkfelix: I *strongly* recommend that you do not use the notation `foo(): Send` as it is currently shown. I recommend you consider some alternative like `foo(...): Send`.

yosh: heh, that's what the syntax initially was proposed as - before it was changed to be more ergonomic after feedback that RTN was a bit.. verbose to type? I agree tho.

pnkfelix: My reasoning is as follows: In a future world where we do want to support more specific descriptions of when the Send bound applies (e.g., one where you have some generic parameter to `foo` and thus you end up with `foo(format!("a string"")` being non-send via `foo(32)` being send, and you'd like to specific this via saying e.g. `foo(i32): Send`, for example, we now have the problem that `foo()` itself looks like it denotes an invocation of `foo` on zero arguments at all.

pnkfelix: I admit, after reflecting on what I typed above, we don't actually *have* this problem today due to other weaknesses in Rust. Namely, our lack of varargs  support means that `foo()` is "never" confusable with `foo(String)` nor `foo(i32)`. But I'd like to imagine a future where that is addressed via some means (i.e. a future where we *do* have varargs support), and I'd like the syntax we use here for "any invocation of foo" to actually bring that to mind.

scottmcm: I'm reminded of `Foo::Blah(..)` patterns to explicitly not care about the params.

nikomatsakis: so my argument has been that I couldn't come up with an actual bug. If you had zero arguments you could be saying either "all" or "none". But when you change to one argument, either it's true no matter the type, in which case why make you specify it, or it's not, in which case it doesn't compile, so what's the problem?

pnkfelix: if `foo(): Send` were interpreted as something like `for<T: Send> foo(T): Send` then it would be narrower, and maybe not what you wanted.

nikomatsakis: true. I'm generally skeptical of those kind of bounds.

Yosh: for reference [Jules Berthold made this same case Felix made here](https://rust-lang.zulipchat.com/#narrow/stream/187312-wg-async/topic/associated.20return.20types.20draft.20RFC/near/356826987) - this apparently has interactions/prior art with "unsized fn params"

pnkfelix: *explains his point about varargs*  i.e., if the arity of the function were something you had to specify in the bound.

pnkfelix: there are many other syntaxes that could sidestep the problem, e.g., you have to give a name to the argument.

pnkfelix: I demote my objection from "strongly recommend" to "weak"

eholk: what happens with `::Output`?

pnkfelix: I don't know how we would express this with `::Output`?

scottmcm: would it include the parens so you could specify the argument types?

nikomatsakis: Josh's argument was that it would be better for impl Trait to go in turbofish. I see that point, but I also feel like, in expr position, you *have* a way to specify the value of impl trait, by supplying an expr of the type you want, and the type position feels like the natural extension of it.

```rust
fn foo(x: impl Debug)
let x = foo::<u32>(22) // error

fn foo(x: #[positional] impl Debug) // "imagine"
```

tmandry: you could require `..` for variadic functions, if we add them? but then it changes the meaning of `foo()`

nikomatsakis: it seems like it'd be better if `foo()` was always forall and there was an explicit way to say none...

scottmcm: `foo(,)`!!

nikomatsakis: :scream: 


### Does this need UFCS versions?

scottmcm: If something implements multiple traits, does it need something like a `<T as Iterator>::intersperse()` form?

nikomatsakis: yes, I've just been assuming we would support that.



### do we need a way to group methods?

pnkfelix: I'm trying to predict whether people will need to list out these `foo(): Send` constraints for many methods on a trait, not just one or two, and if we'll need some way to write down that group of methods succinctly. Did this ever come up in the user studies?

nikomatsakis: wanting all methods came up, not arbitrary subgroups

https://smallcultfollowing.com/babysteps/blog/2023/03/03/trait-transformers-send-bounds-part-3/

Tl;DR is `Send Trait` as a shorthand for `Trait<fn1(): Send, fn2(): Send> + Send`, more or less.

macro defines a trait alias like `SendTrait` that is equiv to above

### Associated type bounds are unstable

This doc uses 
```rust
where
    HC: HealthCheck<check(): Send> 
```
but
```text
error[E0658]: associated type bounds are unstable
 --> src/lib.rs:1:31
  |
1 | fn foo<T>() where T: Iterator<Item: Send> {}
  |                               ^^^^^^^^^^
  |
  = note: see issue #52662 <https://github.com/rust-lang/rust/issues/52662> for more information
```

scottmcm: Are we confident that we want to use that syntax here if we've not stabilized it for normal associated type bounds?

nikomatsakis: I have been missing it when I use GATs, because right now you need separate where clauses.

```rust
fn foo<J>()
where
    J: JvmOp,
    for<'jvm> J::Output<'jvm>: AsRef<Foo>,
```

with bounds could do this, which is better

```rust
fn foo<J>()
where
    J: for<'jvm> JvmOp<Output<'jvm>: AsRef<Foo>>,
```

better still would be to remove the for, but that's orthogonal

```rust
fn foo<J>()
where
    J: JvmOp<Output<'_>: AsRef<Foo>>,
```

nikomatsakis: compiler-errors looked into the impl...

compiler-errors: it's basically ready to go, there is a question around implied bounds, need to file an issue about it. The whole question of what's required to be implied vs copied is a bit vague. But they don't ICE, they work. I'd like to see them stabilized.

nikomatsakis: `where T: Iterator<Item = impl Bar>` vs `where T: Iterator<Item: Bar>`, depending on where the binder went for this `impl`, they could *kind* of be the same thing.  `where T: Iterator<Item = (impl Bar, impl Bar)>`

but it would have to desugar to something we don't really have now...

`where exists<X: Bar> Iterator<Item = X>`

scottmcm: Then could this feature be something like the following?
```rust
where
    HC: HealthCheck<check(): Send> 
```
vs
```rust
where
    HC: HealthCheck<check() = impl Send> 
```
(Not sure I'd want it, but figured it was worth discussing.)

TC: I have functions that return two tuples, using this syntax, you could distinguish those...?

tmandry: might be leaning a little too hard on impl trait syntax?

nikomatsakis: it is leaning awfully hard, also very long...

tmandry: ...new meaning, in a new context...? You could imagine solving the tuple problem in other ways, e.g., making `::0` project.


