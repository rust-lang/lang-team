# 2022-08-31 Lang team GATs meeting

This is an interim report to bring folks up to date on GAT stabilization and summarize the questions and comments that have been raised.

## History and links

The main stabilization report

* [Stabilization PR](https://github.com/rust-lang/rust/pull/96709#issuecomment-1129311660) by jackh726
* [Stabilization proposal](https://github.com/rust-lang/rust/pull/96709#issuecomment-1129311660) by nikomatsakis

Since the stabilization proposal there have been a number of comments. None of the comments have raised new issues that were not addressed in the stabilization proposal, but some folks have taken issue with the recommendations therein. 

## The major options

Ultimately there is a choice of various paths. We'll lay them out here and expand on some of these arguments in detail below. Those items that are covered in detail are marked with § and a link to the discussion. 

* Stabilize now and incrementally improve
    * Areas that need improvement:
        * Syntax is awkward (easy implementation, but it's syntax).
        * Implied bounds (lengthy implementation, but highly impactful). [§][IB]
        * Polonius (lengthy implementation, but impactful in a broad sense, needed for a filter method on lending iterator).
        * Tweaking the required where clauses rules. The current rules are too conservative. [§][RWC]
    * Arguments in favor:
        * Tons of people are using this today and getting a lot of value out of it (the GATs initiative has a [list of examples](https://rust-lang.github.io/generic-associated-types-initiative/design_patterns.html) and [this comment](https://github.com/rust-lang/rust/pull/96709#issuecomment-1170276610) also accumulated a few; there were also a [few](https://github.com/rust-lang/rust/pull/96709#issuecomment-1171645336) [user](https://github.com/rust-lang/rust/pull/96709#issuecomment-1180200266) [stories](https://github.com/rust-lang/rust/pull/96709#issuecomment-1182645568) written expressing their experiences). 
        * In particular frameworks like bevy and embassy are asking for it to help build nicer user-facing abstractions (which often don't explicitly make use of GATs).
            * This makes sense: those frameworks need to be able to abstract over interfaces, and if those interfaces involve methods that borrow from self, generic methods, or methods where the return type depends on `Self`, they can easily require GATs.
            * For these use cases, having something stable that will improve is fine.
* Delay stabilization until implied bounds are fixed (and possibly other specific improvements)
    * Arguments in favor:
        * The limits around implied bounds in particular causes a number of surprising errors [§][IB] and can make GATs less useful than the existing stable workarounds. We should fix them before we land.
    * Arguments against:
        * Jack has been exploring diagnostics and has a prototype that is able to pinpoint the use of GATs as a problem.
* Stabilize lifetime-only GATs and explore alternatives to type GATs
    * Arguments in favor:
        * Lifetime-only GATs come up *very often* and are needed throughout the standard library. Type GATs come up less often, and often in more abstract "framework-y" code, such as the [many modes]() pattern. Many modes used GATs to author functions that could execute in different modes, such as "check whether a grammar is met" versus "construct an AST". The counterargument to many modes is that this pattern might be better left unsolved.
    * Arguments against:
        * Type-based GATs come up less often, but they still frequently arise. Whenever you are abstracting over an interface and the functions in that interface return different types that depend on Self, you need an associated type. That associated type may have to be generic if the return type *also* depends on a generic parameter of the method. [Many modes][mm] is one such example, but [generic scopes][gs] is another, and of course the desire to be abstract over [pointer types][pt] comes up a lot (though we've not elaborated that example).
        * In general, placing arbitrary limitations (this only works for lifetimes) doesn't actually help with learning. Rather, it introduces more edges that can make things more complex. On the other hand, the primary argument here is more about the second-order effects of enabling this sort of abstraction: that it will give rise to more and more complex crates that use traits in complex ways. To that point, many commenters have cited examples where they use GATs precisely so they can offer simpler interfaces to the users, who don't directly interact with GATs.
* Rethink GATs entirely
    * We could abandon GATs and just offer sugar that makes use of them, like async fn in traits or return-position impl trait in traits. At this point, nobody is arguing this position, but it's worth pointing that there are a number of patterns (e.g., `LendingIterator`) where it is very natural to want to give a name to the associated typre (e.g., `Item<'_>`, in that case). Limiting ourselves to RPITIT doesn't support that very naturally, particularly since in that case the method is something like `fn next() -> Option<impl Sized + '_>`, so it's the return type of `next` is distinct from the `Item` type.

[IB]: XXX
[RWC]: XXX
[gs]: https://rust-lang.github.io/generic-associated-types-initiative/design_patterns/generic_scopes.html
[mm]: https://rust-lang.github.io/generic-associated-types-initiative/design_patterns/many_modes.html
[pt]: https://rust-lang.github.io/generic-associated-types-initiative/design_patterns/pointer_types.html


## Review of what is proposed for stabilization

This section is meant to convey the precise "syntactical surface area" that is being stabilized.

### Associated types in traits/impls can have type parameters, where-clauses

The primary language feature stabilized here is the ability to have generics on associated types, as so. Additionally, where clauses on associated types will now be accepted, regardless if the associated type is generic or not. These where clauses must be proven whenever the associated type is used (e.g., `<T as NoGats<'b>>::Assoc` is legal if the where-clause `T: 'a` is satisfied).

```rust
trait LendingIterator {
    type Item<'a>
    where
        Self: 'a;
}

trait NoGats<'a> {
    type Assoc
    where
        Self: 'a;
}
```

Just as with generic methods, when implementing the trait, you repeat the generic parameters that appeared in the trait.

```rust
struct MyIterator<T> { /* ... */ }

impl<T> LendingIterator for MyIterator<T> {
    // Recommended syntax appears after the value of the type. This scales better to more complex types.
    //
    // The where-clauses in the impl don't have to match the trait *exactly*, but they do have to be
    // implied by the trait. In this case, the trait has a where-clause `MyIterator<T>: 'a`, which implies
    // that `T: 'a` by Rust's existing rules, hence `T: 'a' is accepted.
    type Item<'a> = &'a T
    where
        T: 'a;
}
```

### Where-clauses in impl must be subset of those in trait (just as with generic methods)

Note that where clauses are allowed both after the specified type and before the equals sign; however, the latter is a warn-by-default deprecation ([decided in this issue](https://github.com/rust-lang/rust/issues/89122)):

```rust
struct MyIterator2<T> { /* ... */ }
impl<T> LendingIterator for MyIterator2<T> {
    // Deprecated syntax (accepted but warns by default):
    type Item<'a> where T: 'a = &'a T
}
```

Just as with a method, when you implement a trait with a GAT, the where-clauses that appear in the impl must be implied by the where-clauses on the trait. Hence the following example is illegal:

```rust
struct MyIterator3<T> { /* ... */ }
impl<T> LendingIterator for MyIterator3<T> {
    // Illegal because `T: 'static` is not required by the trait;
    // trait only requires `MyIterator<T>: 'a`:
    type Item<'a> = &'a T
    where
        T: 'static;
}
```

### Referencing GATs from where-clauses

You can use a where-clause to specify the value of a normal associated type like

```rust
fn iterate<T>(
    iterator: T
) where
    T: Iterator<Item = u32>
```

The syntax for GATs is similar. Here is an example one could use for a lending iterator:

```rust
fn iterate<T>(
    iterator: &mut T
)
where
    T: for<'a> LendingIterator<Item<'a> = &'a [u32]>,
{
    while let Some(d) = iterator.next() {
        println!("data: {d:?}");
    }
}
```

Note that we wrote `for<'a>` to say that, whatever lifetime `'a` is used, the item will be a `&'a [u32]`. This demonstrates that the generic parameters to the GATs (e.g., `'a` in `Item<'a>`) are *references* to names that are bound elsewhere. **This is a common source of confusion for people**, but it's consistent with other bits of Rust syntax (e.g., the need to write `impl<T> Foo for Bar<T>` instead of just `impl Foo for Bar<T>`). For example, the following function would give an error:

```rust
fn iterate<T>(
    iterator: &mut T
)
where
    T: LendingIterator<Item<'a> = &'a Data>
    //                      ^^ Error: undeclared lifetime `'a`
{
    while let Some(d) = iterator.next() {
        println!("data: {d:?}");
    }
}
```

## Known limitations

This section describes the known limitations of what is being stabilized with some notes on how we expect to address these problems over time. 

### Known suboptimal point: HRTB syntax

**This syntax is known to be suboptimal in a few ways:**

* It's verbose to write the `for`, and `for` is generally confusing.
* People often get confused about when they are defining new things and when they are referencing existing things (e.g., around impls).

nikomatsakis believes that a partial fix would be to extend and define the `'_` syntax in this position, so that one could write

```rust
T: LendingIterator<Item<'_> = &Data>
```

as a shorthand for the `for` notation. However, that would require a more fully articulated proposal. Until now we have avoided stabilizing `'_` in where-clauses precisely to give ourselves room to figure out questions like this.

### Known limitation: HRTB inference interactions can be unreliable

As we noted above, most consumers of GATs wish to use `for<'a>` syntax to specify the value of the GAT, as shown here. (You can view a complete running example of this syntax on [this playground](https://play.rust-lang.org/?version=nightly&mode=debug&edition=2021&gist=03f08b36ba383cbd687affb5aad9c33e).)

```rust
fn iterate<T>(
    iterator: &mut T
)
where
    T: for<'a> LendingIterator<Item<'a> = &'a [u32]>,
{
    while let Some(d) = iterator.next() {
        println!("data: {d:?}");
    }
}
```

The compiler however has a lot of known limitations around HRTB, and one of them interacts poorly with GATs. A reasonable variation on the `iterate` function might be to make it generic over the kind of item that is produced, requiring only that the `Item` be `Debug`:

```rust
fn iterate<T>(
    iterator: &mut T
)
where
    T: LendingIterator,
    for<'a> for<'a> T::Item<'a>: std::fmt::Debug
{
    while let Some(d) = iterator.next() {
        println!("data: {d:?}");
    }
}
```

But if you [try this](https://play.rust-lang.org/?version=nightly&mode=debug&edition=2021&gist=1ee7b86a5c8b48375ac5a744c337d861), you'll find you get a mysterious error that seems to indicate the compiler is failing to do some basic type inference:

```
error[E0277]: `<_ as LendingIterator>::Item<'a>` doesn't implement `Debug`
  --> src/main.rs:46:13
   |
46 |     iterate(&mut buffer);
   |     ------- ^^^^^^^^^^^ `<_ as LendingIterator>::Item<'a>` cannot be formatted using `{:?}` because it doesn't implement `Debug`
   |     |
   |     required by a bound introduced by this call
   |
   = help: the trait `for<'a> Debug` is not implemented for `<_ as LendingIterator>::Item<'a>`
note: required by a bound in `iterate`
  --> src/main.rs:16:47
   |
11 | fn iterate<T>(
   |    ------- required by a bound in this
...
16 |     for<'a> <T as LendingIterator>::Item<'a>: std::fmt::Debug
   |                                               ^^^^^^^^^^^^^^^ required by this bound in `iterate`

```

This turns out to be an HRTB bug, [#89196](https://github.com/rust-lang/rust/issues/89196), where we fail the normalization because type information is not known early enough. You can sidestep it by using turbofish to explicitly specify the value of `T` ([link](https://play.rust-lang.org/?version=nightly&mode=debug&edition=2021&gist=12d066ee1679857fa92b3cfdc5c59c5c)):

```rust
fn main() {
    let mut buffer = Buffer { data: vec![1, 2, 3]};
    iterate::<Buffer<u32>>(&mut buffer);
}
```

This is a good representative of the kinds of papercuts that GAT users experience today. The error is not clear and figuring out what is going wrong require searching the issue database. (Interestingly, that same example works ok with an `&self` receiver, [see here](https://play.rust-lang.org/?version=nightly&mode=debug&edition=2021&gist=bceb31ecb49f945695baf97ce08b2f0c).)

### Known limitation: implied bounds of HRTB

Continuing with the previous example, there is a deeper problem with how the compiler manages HRTB today called "implied bounds". The question is how one interprets a where-clause like `for<'a> T::Item<'a>: Debug`. Today, the compiler interprets it to mean that `T::Item<'a>` must be `Debug` for *literally any lifetime `'a`*. But that is often not true. Consider this generic function that takes a `Buffer<T>` and invokes `iterate`:

```rust
fn buffer_iterate<A>(
    buffer: &mut Buffer<A>
)
where
    A: std::fmt::Debug,
{
    iterate::<Buffer<A>>(buffer)
}
```

If you try to compile this, you will get a rather unhelpful error ([link](https://play.rust-lang.org/?version=nightly&mode=debug&edition=2021&gist=9c8c33cf16b8d27d6a893b8abb776fc8)):

```
error: `A` does not live long enough
  --> src/main.rs:29:5
   |
29 |     iterate::<Buffer<A>>(buffer)
   |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^
```

The root cause of the problem is because the compiler sees that, to call iterate, it must show

```rust
for<'a> <Buffer<A> as LendingIterator>::Item<'a>: Debug
```

To do that, it must normalize `<Buffer<A> as LendingIterator>::Item<'a>` to `&'a [A]`. But to do *that*, it has to prove the where-clauses on the GAT hold. The where-clauses in this case are `Self: 'a`, which means the compiler must show `for<'a> Buffer<A>: 'a`, which is not true unless `A: 'static`. Indeed, adding `A: 'static` to that function [makes it compile](https://play.rust-lang.org/?version=nightly&mode=debug&edition=2021&gist=5390099a4c25fc1853fce8d0dd1b49a4). But that bound shouldn't really be necessary.

jackh726 is working on better errors here that at least identify the HRTB bound as the culprit, but they're not quite ready yet. The real fix is that we have in mind is to change what `for<'a>` means. The new meaning of `for<'a>` would be essentially "for *any suitable* `'a`" (not *any* `'a` at all). In other words, `for<'a> T::Item<'a>: Debug` would mean "for any lifetime `'a` for which `T::Item<'a>` is well-formed, `T::Item<'a>: Debug`". This kind of change is relatively trivial for chalk and mir-formality, but much harder for rustc. We are working on an implementation plan.

**But in the meantime, the net effect is that:**

* Defining GAT abstractions works ok.
* Defining code that invokes those GAT abstractions with fully known types like `Buffer<u32>` *mostly* works, particularly if those types are `'static` (modulo HRTB inference bugs).
* But trying to write generic functions like `buffer_iterate` that invoke those GAT abstractions with partially known types often hits limitations and requires unnecesdary `'static` annotations.

### Known limitation: Required where-clauses in trait to help ensure well-formedness

Remember that where-clauses in impls must be implied by those of the trait. There is a known confusing pitfall around GATs -- in order for the types in the impl to be well-formed, they often require where-clauses, but those where-clauses must be implied by the trait definition. If they are missing from the trait definition, reasonable impls of the trait will not work. The intution for what goes wrong is that GATs, more often than not, represent the return type of a method, and so they wish to have the where-clauses and implied bounds from that method "available":

```rust
trait LendingIterator {
    type Item<'a>
    where
        Self: 'a; // <-- this is an implied bound of the `next` method, made explicit

    fn next(&mut self) -> Option<Self::Item<'_>>;
    //      --------- `self: &'a mut Self`, which implies that `Self: 'a`
}
```

It is tempting to add those as default where-clauses in the trait that don't have to be stated explicitly. However, that would make the trait less general than it might need to be, and we don't know *exactly* the right set of rules to add by default anyway. At the same time, WF bounds are a common source of user-confusion, so doing nothing to catch this common pitfall is likely to be a major source of confusion. We decided in [#87479](https://github.com/rust-lang/rust/issues/87479) to have *required where clauses* where we require users to write the where-clauses on the trait explicitly. This leaves room for us to make them defaulted later. It also ensures that if those defaults would cause problems for people, we'll find out.

**The current rules for what where-clauses are required are not perfect.** In fact, we have some improvements in mind already that would make fewer where-clauses required. However, we also believe that the existing rules are conservative, i.e., they require more where-clauses than we would like, not less. We'll discuss the rules, some of their known shortcomings, and other examples below.

For a GAT `<P0 as Trait<P1..Pn>>::Item<Pn...Pm>` that is referenced in exactly one function within the trait (like `Item<'a>` above), the rules are as follows:

* For any parameter Pi and some other lifetime parameter Pj (e.g., `Self` and `'b`, above) where `Pj != 'static`
    * If `Pi: Pj` holds on the method definition, 
    * If `Pj` is one of the trait parameters (`P0..Pn`):
        * Check whether `Pi: Pj` is provable on the GAT:
            * If not, a where-clause must be added.
    * If `Pj` is one of the GAT parameters (`Pn..Pm`):
        * Find the corresponding lifetime parameter `Pj'` on the GAT (`'a`, in our example)
        * Check whether `Pi: Pj'` is provable on the GAT:
            * If not, a where-clause must be added.

There are at least two ways that the rules are (likely) too conservative:

1. They apply to cases where no GAT parameter is involved:
    * This was intentional, because similar kinds of problems can occur even without GATs, but for trait-level parameters the problems are far less severe because where-clauses can be added to the impl.
2. They apply to cases where the parameter values are fully known, e.g., `where <Self as Foo<'a>>::Bar<()>`
    * we can prove that `(): 'a` but this doesn't mean that the GAT should have a corresponding where-clause

Working through some examples (credit to pnkfelix)...

```rust
#![feature(generic_associated_types)]

pub trait LifetimeGenericConcreteMethod<'a> {
    type Item<'c>;
    //            ~~~~~~~~+~~~~~~
    //                    |
    //            so far so good; no need to relate 'a and 'c.
    //
    fn f(&self) -> Self::Item<'static>;
}

pub trait LifetimeGenericConcreteMethodWhereSelf<'a> {
    type Item<'c> where Self: 'a;
    //            ~~~~~~~~+~~~~~~
    //                    |
    //            required

    fn f(&self) -> Self::Item<'static> where Self: 'a;
    
    // Commentary: This requirement is likely not desirable. It is an example of Case 1.
    // Specifically, `Self::Item<'static>` here is actually short for
    // `<Self as LifetimeGenericConcreteMethodWhereSelf<'a>>::Item<'static>`.
    // We see that `Self` and `'a` are parameters (but not GAT parameters!)
    // and that `Self: 'a` is provable on the method, but not on the GAT,
    // so we require the where-clause.
    
}

pub trait LifetimeGenericConcreteMethodWhereItem<'a> {
    type Item<'c>;
    //            ~~~~~~~~+~~~~~~
    //                    |
    //            look ma, no where clause here
    //
    fn f(&self) -> Self::Item<'static> where Self::Item<'static>: 'a;

    // Commentary: This seems right. The goal is to capture
    // where-clauses that might affect whether the value of `Item<'c>`
    // is well-formed. This outlives clause affects whether the value is a valid
    // thing to return, but not whether it's well-formed.
}

pub trait TypeGenericConcreteMethod<'a> {
    type Item<T> where T:'a;
    //           ~~~~+~~~~~
    //               |
    //           required because of ...?
    //
    // (How is this different from LifetimeGenericConcreteMethod?)
    //
    fn f(&self) -> Self::Item<()>;

    // Commentary: This where-clause probably should not be required
    // and is an example of Case 2. Specifically, we see a reference to
    // 
    // `<Self as TypeGenericConcreteMethod<'a>>::Item<()>`
    // 
    // and we see that two of those parameters can be relatedin the method (`(): 'a`)
    // but the generic version doesn't hold in the GAT (`T: 'a` doesn't hold).
    //
    // The changes nikomatsakis has in mind would exclude this case because it doesn't
    // involve any generic types that appear on the method, and `(): 'a` could be proven
    // at the GAT level, but it would be useful to see more examples of how GATs are used
    // before finalizing those rules.
}

pub trait TypeGenericParametericMethod<'a> {
    type Item<T>;
    //           ~~~~~~~~~+~~~
    //                    |
    //           look ma, no where clause here
    //
    fn f<T>(&self) -> Self::Item<T>; // where Self::Item<T>: 'a;

    // Commentary: this seems reasonable, the method has no particular
    // known bounds on T.
}

pub trait TypeGenericParametericMethodWhereItem<'a> {
    type Item<T>;
    //           ~~~~~~~~~+~~~
    //                    |
    //           look ma, no where clause here still
    //
    fn f<T>(&self) -> Self::Item<T> where Self::Item<T>: 'a;

    // Commentary: again reasonable, the bounds on method don't
    // affect whether the value of `Self::Item` would be well-formed.
}

pub trait TypeGenericParametericMethodWhereSelf<'a> {
    type Item<T> where Self:'a;
    //           ~~~~+~~~~~~~~
    //               |
    //            required because of this: \
    //                              --------+-----
    fn f<T>(&self) -> Self::Item<T> where Self: 'a;

    // Commentary: This is Case 1 -- no GAT parameters are involved,
    // so perhaps not wanted (in particular, it would be possible
    // to add a requirement to the impl that `Self: 'a` and make values
    // of `Item<T>` well-formed).
    // 
    // This is a good example where it's not *exactly* clear whether
    // the where-clause should be defaulted or not, it may be that implementors
    // don't want to require that `Self: 'a` in the general case, but only
    // when `f` is invoked. At the same time, it seems also like an example
    // where it would be fine if the user had to write it themselves
    // and we didn't figure it out automatically.
}
```

## Clarifications

This section includes some clarifications to specific questions that were raised. 

### Where-clauses vs bounds on the type parameter

Question: Are the following two traits equivalent?

```rust
trait WidgetFactory {
    type Item<T: Debug>;
}

trait WidgetFactory {
    type Item<T> where T: Debug;
}
```

Answer: Yes.

### Where-clauses vs ensures bounds

Question: Does this trait...

```rust
trait WidgetFactory {
    type Item<T>: Display;
}
```

...have a distinct meaning from this one?

```rust
trait WidgetFactory {
    type Item<T>
    where
        Self::Item<T>: Display;
}
```

Answer: Yes. With GATs, we draw a distinction between the *ensures bounds* on the GAT (e.g., the `Display` in `type Item<T>: Display`) versus the *where clauses*.

This is in contrast to normal associated types. For historical reasons, normal associated types behave differently. The following two traits are considered equivalent by rustc today:

```rust
trait NonGat {
    type Item: Display;
}

trait NonGat
where
    Self::Item: Display
{
    type Item;
}
```

This is supported by some fairly special-case code in the compiler that searches the list of where-clauses for entries of this form. The reason this code exists is because we used "desugar" ensures clauses into where-clauses of this form, but we eventually refactored the compiler to track ensures clauses separately. Unfortunately, there was extant code that depended on the older behavior.

Nikomatsakis considers this unfortunate because it complicates the question of whose responsibility is to prove what, and in particular it interacts poorly with the ongoing efforts to make it possible to write "false where-clauses". We've an [accepted RFC](https://rust-lang.github.io/rfcs/2056-allow-trivial-where-clause-constraints.html) that allows  users to write functions that include "counterfactual" where-clauses, like `where String: Copy`, with the end-result being that the function simply cannot be called. This is very useful in many code-generation situations where the code-generator often doesn't know whether the trait bounds holds or not, but it would like to generate code for the case where it does hold. However, permitting trivial where-clauses mixes very poorly with GATs and ensures clauses. Consider this example:

```rust
trait WidgetFactory {
    type Item<T>
    where
        Self::Item<T>: Display;
}

impl WidgetFactory {
    type Item<T> = Vec<T>
    where
        Self::Item<T>: Display;
}
```

The impl here appears to be legal: it has a where-clause that states that `Self::Item<T>` implements `Display`, so when we check whether the value is well-formed, we should be assuming that this is true. In fact `Vec<T>` doesn't implement `Display`, but that's ok -- since this is a where-clause, other code doesn't get to assume that `Item<T>: Display` but rather has to prove it (and it will be unable to do so). 

If however we blurred the lines for GATs, then we would have a problem where other code could treat this where-clause as an ensures clause (i.e., it can assume that `Display` holds), and impls can do the same! This is not good.

This problem doesn't arise for the where-clauses on traits because, in general, impls cannot assume that the where-clauses on traits hold. Rather, its the impls job to *prove* that the where-clauses on traits hold. And indeed if we lifted `Self::Item<T>: Display` from the associated type to a `for<T> Self::Item<T>: Display` bound on the trait, that could be ok, but there's no compelling reason to do that.

In general, nikomatsakis would prefer to rule out (perhaps with an edition) the special cases for associated types and trait where-clauses, but we definitely don't want to be adding more.


## Summary of concerns raised in PR thread

### GATs increase the complexity of the Rust language

* Limited new syntax
    * `Trait<Item<'a> = ...>`
* Existing syntax in new positions
    * Parameters on associated types
    * Where clauses on associated types
* Also more common uses of existing syntax (that currently isn't used much)
    * `for<'a>` in where bounds
    * `for<'a>` in places like `impl for<'a> Trait<Item<'a> = ...>`

### There are still implementation limitations

* Examples
    * HRTB implied bounds
    * Borrow-checker limitations (e.g. `filter` on `LendingIterator`)
* There is often no *clear* boundaries between "what works", "what doesn't work, but should", and "what doesn't work, and shouldn't"
* In the short term, this can be mitigated by creating tailored error diagnostics to point out GATs
* In the long term, there are plans to fix these in a backwards-compatible manner

### The rules for default bounds (e.g. `where Self: 'a`) are sometimes not immediately intuitive

* See examples above
* The rules are straightforward, but their interactions with e.g. implied bound elaboration isn't always clear
* Should we provide a way to opt-out?
    * Not entirely sure what that would look like. (Would it be stable or unstable?)
    * We want to know about those cases

### We might leave GATs in a less-than-ideal state for too long post-stabilization

* We have a not-so-great history of taking a while to fix some difficult bugs (e.g. normalization under binders, many type soundness issues)
* We do have the types team now that has more bandwidth as a whole to dig into and fix these issues

## Appendix A: Quotes from the thread

What follows are a long list of quotes from the thread, both for and against stabilization for various reasons. These were extracted by skimming over the thread and pulling out each case that either seemed "fresh" or was expressing experiences derived from using GATs in practice. 

### Some quotes from folks using GATs or favoring stabilization

There have been a large number of comments from people either using GATs or wishing to use GATs. The GAT initiative repo collected a lot of links [here](https://rust-lang.github.io/generic-associated-types-initiative/design_patterns.html#projects-using-gats), but there have been more comments posted in the meantime. Here are a few quotes.

> [type-exercise-in-rust](https://github.com/skyzh/type-exercise-in-rust) is an interesting project using GATs to implement the database's type and expression system. GATs help expressing abstraction over [arrays, scalars and reference to scalars](https://github.com/skyzh/type-exercise-in-rust/blob/db662f9367f16ef9bf981c5fb95afee97d988ba6/expr-common/src/array.rs#L42).
>
> There are also several databases using similar patterns to construct their type system:
> - [databend](https://github.com/datafuselabs/databend), a cloud data warehouse, has more sophisticated implementation:
>   - [Column](https://github.com/datafuselabs/databend/blob/a3f5327650/common/datavalues/src/scalars/column.rs), an abstraction over vector.
>   - [Scalar and ScalarRef](https://github.com/datafuselabs/databend/blob/a3f53276501d195853b6c7cf5b5354e5257b24b9/common/datavalues/src/scalars/type_.rs) , abstractions over scalar and reference to a scalar, allowing conversion between referenced type and owned type.
>   - Implements a [stateful aggregating state](https://github.com/datafuselabs/databend/blob/5648b5b5508c97adbb2b482d66ae889459167512/common/functions/src/aggregates/aggregate_scalar_state.rs) using the scalar abstraction, so we can define a generic function which takes reference of the scalar as input, and holding an owned scalar type as aggregation state.
> - [risingwave](https://github.com/singularity-data/risingwave), its [Array](https://github.com/singularity-data/risingwave/blob/36d4930ad060728562c96deb28acb5a186b0e982/src/common/src/array/mod.rs#L139) trait and generic [aggregator](https://github.com/singularity-data/risingwave/blob/36d4930ad060728562c96deb28acb5a186b0e982/src/expr/src/vector_op/agg/general_agg.rs#L25) use GATs
>
> I think [databend](https://github.com/datafuselabs/databend) and [rinsingwave](https://github.com/singularity-data/risingwave) would be some other user stories to show how GATs help building complicate but powerful applications. The main contributors @BohuTANG @sundy-li of databend, and @skyzh , the author of [type-exercise-in-rust](https://github.com/skyzh/type-exercise-in-rust) should have more experiments on using GATs and probably have more thoughts on that. ([link](https://github.com/rust-lang/rust/pull/96709#issuecomment-1170276610))

> Above, @nikomatsakis asked for stories, so I gave [writing one](https://gist.github.com/TimNN/e4516c1b8be51edc401a1d44d856bf01) a shot. I tried to write it from the perspective of a reasonably experienced programmer who is relatively new to Rust and is implementing "some algorithm", starting with hard-coded types and then generalizing it.
> 
> My main takeaways from that are:
>
> * It's really easy to stumble into situations where (IMO) GATs (in this case, lifetime GATs) are the obvious approach.
> * The (IMO most obvious) workaround the story went with (implementing the trait for `&'a T` instead of `T`) isn't nice to use and has its own stumbling blocks:
>     * I quickly ran into a wall that I couldn't immediately figure out a way around: When implementing a trait for `&'a mut T`, having a method that takes `&self` and returns `&'a ()` is apparently problematic (that's the `Data07` failure in the story).
> * The code with GATs is, IMO, definitely nicer than the workaround and mostly straightforward.
> 
> (I have some vague ideas about extending the story to also run into type GATs, though I'm not sure when I'll have the time to explore that). ([link](https://github.com/rust-lang/rust/pull/96709#issuecomment-1171645336))

> Since I've been told that GATs are suddenly controversial, I'd like to point to two situations where they would have been / have been extremely useful. One in wgpu, which has to work on stable, the other in Veloren, which works on nightly. \[..\] I'm bringing these up because I feel like abstract arguments about GATs as being a complex feature that language users are going to have to deal with, and be frustrated by, are missing that (1) a lot of their most interesting usecases are internal to libraries (and don't really need to be exposed to users of the library), and (2) when you need them, there is no good workaround--you just end up reinventing GATs, but badly, making the code far more complex than the GAT equivalent would be. ([link](https://github.com/rust-lang/rust/pull/96709#issuecomment-1150127168))

> I'm not currently using GATs, but I'm about to implement a feature that will need them in my unpublished WIP crate. \[..\] I don't know how clear my example was, but the gist is: I need a trait that says "here's how to build a rich pointer to me", which requires an associated type with a lifetime, which requires GATs. ([link](https://github.com/rust-lang/rust/pull/96709#issuecomment-1180200266))

> Agree with @skyzh, without GAT it's really hard for us to write vectorized expressions in simple way.
In the OLAP database area,  it's very common to have associate types among   `Type`, `Column`, `ColumnRef`,  `ColumnIterator` , `Scalar`, `ScalarRef` in compile stage,  such we can write generic execution codes much easier (with less enum match or type downcast),  refer to another repo [typed-type-exercise-in-rust](https://github.com/andylokandy/typed-type-exercise-in-rust/blob/master/src/types.rs#L33). ([link](https://github.com/rust-lang/rust/pull/96709#issuecomment-1170708143))

>  There are some solutions here that are actively being driven to be both *more unsafe* and *less generic*. One recent example I've run into is the opportunity to shrink a few generic types instead of using filler Options for missing state that never gets used at runtime. With GATs, this would be trivial to just extend the existing associated type to provide an alternative representation that is space and runtime efficient. Without GATs, [we've resorted to using const-generics to drive compile-time discriminated unions](https://github.com/bevyengine/bevy/blob/d9d4f9556e49f0b0537d35686fc6180d02912127/crates/bevy_ecs/src/query/fetch.rs#L1535=), which is more unsafe, harder to prove correct without lots of boilerplate or debug build checks, harder to extend and maintain long term, still has sharp and unexpected API edges, and still not an optimal solution. The alternative here to just leave it as is and eat the performance costs, or use even more unsafe. Both of which are antithetical to the developer experience Rust is known for. ([link](https://github.com/rust-lang/rust/pull/96709#issuecomment-1165016517))

> I've been using GATs on nightly for a little less than a year for various experimental proc-macro crates. I can only say that GATs simplify a lot of things for me! I'm not doing things like LendingIterator, my interest is more in "DSL"s. \[..\] I'm no "type theorist" at all, I have no deep insight into how the Rust typesystem works internally, or what is the deeper cause of some random compile error. With GATs I just tend to keep flicking on the code until the compiler is somehow happy. Just like classic non-GAT Rust code :) I have stumbled across some of the HRTB limitations, but have managed to work around them quite easily. My feeling is that for my use-cases GATs now feel like a natural part of the language. ([link](https://github.com/rust-lang/rust/pull/96709#issuecomment-1119760258))

> As a rust user for many years I would like to give my 0.02$ on the matter. I probably have GATs in every probject I do write. It all starts similarly: I have bunch of traits that return Vecs or something. Then I profile and see allocs in tight loops I cannot afford. \[..\] Every time you want to have zero-cost in traits GATs come into play. Iterators and futures are just some examples, there are more. And I bet that every program written in rust are using these concepts and people there either don't care and put boxes in traits or write their own super niche fragile implementations to fit their needs. ([link](https://github.com/rust-lang/rust/pull/96709#issuecomment-1120175346))

> For what it's worth I am a rust beginner coming from mostly Python / Typescript. There are MANY things in Rust that make my head hurt but GAT just seem natural after some introduction and I actually get stuck trying to go around the lack of GAT nearly every time I try to write Rust code (I have given up in every single of my rust projects so far and I would say lack of GAT and annoying lifetime handling were my 2 main problems). ([link](https://github.com/rust-lang/rust/pull/96709#issuecomment-1131885602))

> At my company, we started implementing the [mongodb](https://github.com/mongodb/mongo-rust-driver) interface behind a trait so that we could mock the db functionality out in unit tests. This has so far worked pretty well for us. For lone return-type generics, this is pretty straightforward with async_trait. However, there's a piece of functionality called [change streams](https://docs.rs/mongodb/2.2.2/mongodb/change_stream/struct.ChangeStream.html) which, tl;dr, looks like `ChangeStream<T>` 😄 and wraps the returned generic type. ([link](https://github.com/rust-lang/rust/pull/96709#issuecomment-1182645568))

> As a real production use case for GAT(and TAIT), we have open-sourced our internal RPC framework Volo: https://github.com/cloudwego/volo and its middleware abstraction layer Motore(which is greatly inspired by Tower): https://github.com/cloudwego/motore. Here's a feedback for that. \[..\] Though there may still be some bugs with GAT, GAT (together with TAIT) are super useful features and we are quite looking forward for their stabilizations. Thank you for all your great works! ([link](https://github.com/rust-lang/rust/pull/96709#issuecomment-1224313111))

> I am going to chime in here regarding the stabilization of GATs. Our use-case at [Skytable](https://github.com/skytable/skytable/) is very much similar to the example highlighted in RFC 1598, where we want to implement a single trait for a common group of data structures. GATs will enable us to do "hot swapping" of the underlying DS to test for performance impacts. ([link](https://github.com/rust-lang/rust/pull/96709#issuecomment-1228478376))

> Outsider viewpoint here, which might actually be valuable in the discussion (with the risk of all you type system pro's telling me there's a better way to go about this anyway). \[..\] I've been looking forward to GATs being stabilized for a while now. Coming from the real-time audio/DSP world, I would love to be able to express all my audio effects and generators in such a way: ([link](https://github.com/rust-lang/rust/pull/96709#issuecomment-1127323959))

> I am going to chime in here regarding the stabilization of GATs. Our use-case at [Skytable](https://github.com/skytable/skytable/) is very much similar to the example highlighted in RFC 1598, where we want to implement a single trait for a common group of data structures. GATs will enable us to do "hot swapping" of the underlying DS to test for performance impacts. ([link](https://github.com/rust-lang/rust/pull/96709#issuecomment-1228478376))

### Some quotes expressing confusion or pushing back on stabilization

> To summarize in a sentence: I think language complexity is a big problem, but I think GATs themselves are a great addition, but I think their interaction with WF is a leaky abstraction that needs to be plugged. ([link](https://github.com/rust-lang/rust/pull/96709#issuecomment-1148852889))

> As a solidly intermediate Rust user I can say that I have tried wrap my mind around GATs several times over the past 3-4 years, and I don't really "get" it. To be completely fair, I have never tried using them on nightly, so maybe this something best groked hands-on. ([link](https://github.com/rust-lang/rust/pull/96709#issuecomment-1120054138))

> My concern is that while people who are interested enough in language design to comment here, or who have experience with similar abstractions in other languages, might find these things intuitive, that group is a tiny minority of Rust users and potential Rust users. I've taught and TA'ed programming and types systems courses, and for the majority of programmers, even plain generics are not intuitive and take a lot of effort. HKT-like things are a constant stumbling block even for students opting in to types systems courses at a top university. We hear a lot of comments from Rust users who struggle with the existing system of generics/traits/lifetimes in Rust, and for those people I don't think there is evidence that GATs will simplify things at all. ([link](https://github.com/rust-lang/rust/pull/96709#issuecomment-1120856055))

> Firstly, I'd like to address what I think is a major hurdle in understandability for GATs: nonlocality. In addition, the syntax actually masks this non-locality in a way I think is confusing. What I mean non-locality is it's more or less the only place in Rust where you can't easily substitute and reason about types in-place. \[..\] Now, this is, obviously, the point. This is what makes them strong and why we want them. I've run into GAT-needed stuff multiple times. However, this chain of symbol substitution is, while very logical, kind of a large leap if you haven't been staring at GAT stuff for very long. It actually took me a minute to understand many of the examples. This isn't an argument against implementing GATs, but there are some concerns I have about both syntax and some... non-obvious uses. ([link](https://github.com/rust-lang/rust/pull/96709#issuecomment-1126690737))

> I think the dominant perspective in this thread is that GATs decrease complexity, and my guess as to why is because of aforementioned self selection. My suspicion is that most folks participating in this thread probably intuitively understand why that's true. GATs give you a new vocabulary with which to model problems. For a specific class of problems, that new vocabulary may greatly simplify the code, because, well, the alternative is something like a hack or unsafe or whatever. \[..\] But the other perspective, where GATs increase complexity, is also important to recognize. The complexity increase is interlocked with the idea that it gives more power to users of Rust to engage in more sophisticated abstractions. It lowers the barrier to entry to those abstractions precisely because you no longer have to pay some other cost that most aren't willing to pay (like crazy type-level hacks). \[..\] Also, as a disclosure and since the thread is so long: my position is generally in favor of GATs, but thinks that the UX should be improved first. That is, I am in favor of GATs while simultaneously recognizing what I think will be a categorical jump in complexity in Rust's abstraction power. ([link](https://github.com/rust-lang/rust/pull/96709#issuecomment-1168643277))

> I am mildly surprised that I am here writing a comment asking for gats to not be stabilised given how much I am looking forward to them but here I am 😅 I do not think gats should be stabilised until the bug around implied bounds are fixed.
>
> Most of the "big" issues with gats are not actually related to gats and are just other issues with the compiler that are pre-existing and observable on stable (so there is really no harm in shipping gats with them- everyone is already suffering gats isnt going to make it worse). But with the implied bounds bug this is specifically an issue with gats that does not exist with the stable workaround, this means that lifetime gats are less useful than the stable workaround which is not a good look. ([link](https://github.com/rust-lang/rust/pull/96709#issuecomment-1181992408))

> On the whole, I want GATs to become a reality, but am not convinced the rough edges have been sufficiently smoothed. I'm also concerned with forcing significant changes too quickly: I see a lot of NLL and scoped thread breakage and soundness issues in the queue right now, for example. I'd rather these issues be resolved before pushing the feature into stable, not after. I am in favor of keeping up the momentum to fix the problems and get these features stabilized. But this tendency to only do so in the narrow window between FPC and the next release is concerning and problematic.

# Questions

This section is for questions that arise during the meeting. We'll take notes on the question and answers.

## How should I format my questions?

> I'm confused! How do I format my questions?

Please add a new `##` section for each question so we get a nice linkable heading!

## Note for wider use of this document

The "Review of what is proposed for stabilization" seems better to read *before* "Major options"

nikomatsakis: Where would you put "known limitations"?

josh: Also before "Major options", to avoid forward-references.

## Time estimate until known bugs are fixed?

jackh726: Depends a lot, but it could easily be a year.

joshtriplett: Do any of the planned improvements come "for free" as a result of other work?

jackh726: as part of the mir-formality work, it's a very general piece of work.

joshtriplett: it sounds like you're saying there will now be 3 major projects over the horizon: chalk, polonius, implied bounds on gats?

nikomatsakis: it's tied in with the chalk work. The idea of chalk is to refactor the type/trait implementation so it can easily scale to new use cases and these would be under that,

## Mark: Illegal case 3

I seem to recall us generally wanting to move to a model where you can say "false" things and the function in question just isn't usable as a result (i.e., not callable). Is the rationale for not following a similar approach here that there's much less utility to a 'dead' associated type? Or is this something we expect to relax at a future point?

```
struct MyIterator3<T> { /* ... */ }
impl<T> LendingIterator for MyIterator3<T> {
    // Illegal because `T: 'static` is not required by the trait;
    // trait only requires `MyIterator<T>: 'a`:
    type Item<'a> = &'a T
    where
        T: 'static;
}
```

nikomatsakis: I do want to adopt that model, but it doesn't apply to traits/impls. The idea is that you write generic code against a trait, and you separately write the impl against the trait, and we need to be sure that the two can be combined (or else you would get post-monomorphization errors). So, while you can generally add "false where-clauses" at the impl level (preventing you from using the impl at all) or on free functions, you can't add them "extra where-clauses" to items within the impl that are not implied by the trait, since the callers are checked against the trait, and we assume that satisfying the trait where-clauses implies satisfying the impl where-clauses.

simulacrum: +1, makes sense. On the other hand, it seems like that assumption (satisfying trait where-clauses implies satisfying impl where-clauses) doesn't necessarily *have to* extend to individual items, right? Each item could be in its own trait conceptually and checked independently, though it's probably a needlessly complex model.


Josh: In theory, we could just require that the combination of (trait, type implementing trait, monomorphization of that type) is satisfied, without the ability to prove separately that the type satisfies the trait for *every* possible monomorphization. Why would that be a *bad* idea? That seems like it might give additional power. We do already allow, for instance, "this method only exists on some monomorphizations of this type, if the additional bounds on the method are satisfied". Could we do the same with associated types, such that the associated type only exists if its bounds are satisfied?

Example:

```rust
struct MyType<T> { }

impl MyType<u32> { }

impl<T: Debug