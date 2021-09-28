---
tags: "design-meeting"
title: "2021-09-22: GAT Defaults"
---

# Outlives defaults

## Links

* Discussion issue: [#87479](https://github.com/rust-lang/rust/issues/87479)
* [Source](https://rust-lang.github.io/generic-associated-types-initiative/design-discussions/outlives-defaults.html)

## Summary

We are moving towards stabilizing GATs (tracking issue: [#44265]) but there is one major ergonomic hurdle that we should decide how to manage before we go forward. In particular, a great many GAT use cases require a surprising where clause to be well-typed; this typically has the form `where Self: 'a`. In "English", this states that the GAT can only be used with some lifetime `'a` that could've been used to borrow the `Self` type. This is because GATs are frequently used to return data owned by the `Self` type. It might be useful if we were to create some rules to add this rule by default. Once we stabilize, changing defaults will be more difficult, and could require an edition, therefore it's better to evaluate the rules now.

[#44265]: https://github.com/rust-lang/rust/issues/44265

## Background: what where clause now?

Consider the typical "lending iterator" example. The idea here is to have an iterator that produces values that may have references into the **iterator itself** (as opposed to references into the collection being iterated over). In other words, given a `next` method like `fn next<'a>(&'a mut self)`, the returned items have to be able to reference `'a`. The typical `Iterator` trait cannot express that, but GATs can:

```rust
trait LendingIterator {
    type Item<'a>;

    fn next<'b>(&'b mut self) -> Self::Item<'b>;
}
```

Unfortunately, this trait definition turns out to be not quite right in practice. Consider an example like this, an iterator that yields a reference to the same item over and over again (note that it owns the item it is referencing):

```rust
struct RefOnce<T> {
    my_data: T    
}

impl<T> LendingIterator for RefOnce<T> {
    type Item<'a> where Self: 'a = &'a T;

    fn next<'b>(&'b mut self) -> Self::Item<'b> {
        &self.my_data
    }
}
```

Here, the type `type Item<'a> = &'a T` declaration is actually illegal. Why is that? The assumption when authoring the trait was that `'a` would always be the lifetime of the `self` reference in the `next` function, of course, but that is not in fact *required*. People can reference `Item` with any lifetime they want. For example, what if somebody wrote the type `<SomeType<T> as LendingIterator>::Item<'static>`? In this case, `T: 'static` would have to be true, but `T` may in fact contain borrowed references. This is why the compiler gives you a "T may not outlive `'a`" error ([playground](https://play.rust-lang.org/?version=nightly&mode=debug&edition=2018&gist=821e30ee635326a22fc19cd940bbaf62)). 

We can encode the constraint that "`'a` is meant to be the lifetime of the `self` reference" by adding a `where Self: 'a` clause to the `type Item` declaration. This is saying "you can only use a `'a` that could be a reference to `Self`". If you make this change, you'll find that the code compiles ([playground](https://play.rust-lang.org/?version=nightly&mode=debug&edition=2018&gist=87cb2430ee76ece77499d3c6605874df)): 

```rust
trait LendingIterator {
    type Item<'a> where Self: 'a;

    fn next<'b>(&'b mut self) -> Self::Item<'b>;
}
```

## When would you NOT want the where clause `Self: 'a`?

If the associated type cannot refer to data that comes from the `Self` type, then the `where Self: 'a` is unnecessary, and is in fact somewhat constraining.

### Example: Output doesn't borrow from Self

In the `Parser` trait, the `Output` does not ultimately contain data borrowed from `self`:

```rust
trait Parser {
    type Output<'a>;
    fn parse<'a>(&mut self, data: &'a [u8]) -> Self::Output<'a>;
}
```

If you were to implement `Parser` for some reference type (in this case, `&'b Dummy`) you can now set `Output` to something that has no relation to `'b`:

```rust
impl<'b> Parser for &'b Dummy {
    type Output<'a> = &'a [u8];

    fn parse<'a>(&mut self, data: &'a [u8]) -> Self::Output<'a> {
        data 
    }
}
```

Note that you would need a similar `where` clause if you were going to have a setup like:

```rust
trait Transform<Input> {
    type Output<'a>
    where
        Input: 'a;

    fn transform<'i>(&mut self: &'i Input) -> Self::Output<'i>;
}
```

### Example: Iter static

In the previous example, the lifetime parameter for `Output` was not related to the `self` parameter. Are there (realistic) examples where the associated type is applied to the lifetime parameter from `self` *but* the `where Self: 'a` is not desired?

There are some, but they rely on having "special knowledge" of the types that will be used in the impl, and they don't seem especially realistic. The reason is that, if you have a GAT with a lifetime parameter, it is likely that the GAT contains some data borrowed for that lifetime! But if you use the lifetime of `self`, that implies we are borrowing some data from `self` -- however, it doesn't *necessarily* imply that we are borrowing data of any particular type. Consider this example:

```rust
trait Message {
    type Data<'a>: Display;

    fn data<'b>(&'b mut self) -> Self::Data<'b>;

    fn default() -> Self::Data<'static>;
}

struct MyMessage<T> {
    text: String,
    payload: T,
}

impl<T> Message for MyMessage<T> {
    type Data<'a>: Display = &'a str;
    // No requirement that `T: 'a`!

    fn data<'b>(&'b mut self) -> Self::Data<'b> {
        // In here, we know that `T: 'b`
    }

    fn default() -> Self::Data<'static> {
        "Hello, world"        
    }
}
```

Here the `where T: 'a` requirement is not necessary, and may in fact be annoying when invoking `<MyMessage<T> as Message>::default()` (as it would then require that `T: 'static`).

Another possibility is that the usage of `<MyMessage<T> as Message>::Data<'static>` doesn't appear inside the trait definition, although it is hard to imagine exactly how one writes a useful function like that in practice.

## Alternatives

### Status quo

We ship with no default. This makes it hard to add a default later, as that would potentially be a breaking change. In practice, though, a sufficiently smart default (such as the one described next) may be something we could add later because it has no known "counterexamples".
- Another point here: this wouldn't be a terrible option if there was a nice/easy way to give useful diagnostics, but that's rather difficult

### Smart default: add `where Self: 'a` if the GAT is used with the lifetime from `&self` (and extend to other type parameters)

Analyze the types of methods within the trait definition. If a GAT is applied to a lifetime `'x`, examine the implied bounds of the method for bounds of the form `T: 'x`, where `T` is an input parameter to the trait. If we find such bounds on all methods for every use of the GAT, then add the corresponding default.

Consider the `LendingIterator` trait:

```rust
trait LendingIterator {
    type Item<'a>;

    fn next<'b>(&'b mut self) -> Self::Item<'b>;
}
```

Analyzing the closure body, we see that it contains `Self::Item<'b>` where `'b` is the lifetime of the `self` reference (e.g., `self: &'b Self` or `self: &'b mut Self`). The implied bounds of this method contain `Self: 'b`. Since there is only one use of `Self::Item<'b>`, and the implied bound `Self: 'b` applies in that case, then we add the default `where Self: 'a` to the GAT. 

This check is a fairly simple syntactic check, though not necessarily easy to explain. It would accept all the examples that appear in this document, including the example with `fn default() -> Self::Data<'static>` (in that case, the default is not triggered, because we found a use of `Data` that is applied to a lifetime for which no implied bound applies). The only case where this default behaves *incorrectly* is the case where all uses of `Self::Data` that appear within the trait need the default, but there are uses outside the trait that do not (I couldn't come up with a realistic example of how to do this usefully).

#### Extending to other type parameters

The inference can be extended naturally beyond `self` to other type parameters. Therefore this example:

```rust
trait Parser<Input> {
    type Output<'i>;

    fn get<'input>(&mut self, i: &'input Input) -> Self::Output<'input>;
}
```

would infer a `where Input: 'i` bound on `type Output<'i>`.

Similarly:

```rust
trait Parser<Input> {
    type Output<'i>;

    fn get(&mut self, i: &'input Input) -> Self::Output<'input>;
}
```

would infer a `where Input: 'i` bound on `type Output<'i>`.

#### Avoiding the default

If this default is truly not desired, there is a workaround: one can declare a supertrait that contains just the associated type. For example:

```rust
trait IterType {
    type Iter<'b>;
}

trait LendingIterator: IterType {
    fn next(&mut self) -> Self::Iter<'_>;
}
```

This workaround is not especially obvious, however.

#### Related precedent

We used to require `T: 'a` bounds in structs:

```rust
struct Foo<'a, T> {
    x: &'a T
}
```

but as of [RFC 2093] we infer such bounds from the fields in the struct body. In this case, if we do come up with a default rule, we are essentially inferring the presence of such bounds by usages of the associated type within the trait definition.

[RFC 2093]: https://rust-lang.github.io/rfcs/2093-infer-outlives.html

## Recommendation

Niko's recommendation is to use the "smart defaults". Why? They basically always do the right thing, thus contributing to [supportive](https://rustacean-principles.netlify.app/how_rust_empowers/supportive.html), at the cost of (theoretical) [versatility](https://rustacean-principles.netlify.app/how_rust_empowers/versatile.html). This seems like the right trade-off to me.

The counterargument would be: the rules are sufficiently complex, we can potentially add this later, and people are going to be surprised by this default when it "goes wrong" for them. It would be hard, but not impossible, to add a tailored error message for cases where the `where T: 'b` check fails.

Not sure about Jack's opinion. =)
- Jack's opinion is to use the "smart default"; I haven't seen an example where this would fail, but think it would be rare enough that moving to a supertrait is a good alternative.

## Appendices

* [Appendix A: Ruled out alternatives](https://rust-lang.github.io/generic-associated-types-initiative/design-discussions/outlives-defaults.html#appendix-a-ruled-out-alternatives)
* [Appendix B: Considerations](https://rust-lang.github.io/generic-associated-types-initiative/design-discussions/outlives-defaults.html#appendix-b-considerations)
* [Appendix C: Other examples](https://rust-lang.github.io/generic-associated-types-initiative/design-discussions/outlives-defaults.html#appendix-c-other-examples)


# Questions / Queue

## Q

Josh: Not directly about GATs themselves, but shouldn't `LendingIterator::next` return an `Option`, so that it can stop?

Yes

## Q

Mark: Are there any downsides to the "smart default avoidance" of separating out a `trait Foo { type Item<'a>; }`? (in particular, I seem to recall some possible difficulty with trait objects but can't quite remember it.)

* Niko: I don't thnk so but there may have been some bugs around this general area
* Casting?
    * Niko: if you do this workaround, there wouldn't be much reason to upcast to `Foo`

## Q

Mara: Smart diagnostics/suggestions can also be *supportive*, while keeping the language itself 'simple'. If the smart defaults are implementable, then no defaults with good diagnostics+suggestions are possible too.

* Niko: 
    * detect the problem in the impl
    * but the fix is in the trait, that's a bit awkward
    * where clauses appear in the rustdoc and so forth, so people have to think about it
* Josh:
    * Couldn't you detect it in the trait? Instead of inferring the default, ask people to write it
* Niko:
    * Something about that is bothering me; perhaps because I don't have a realistic example where you don't want the default
* Josh:
    * If we think it's rare, but possible, then a lint may make sense
    * And a way to say explicitly "not this"
* Felix:
    * If it's rare but possible, there's the workaround of supertrait
* Josh:
    * I find that to be too obtuse, I'd rather have a syntax
    * My inclination is that there probably aren't many "false positives"
* Niko:
    * the syntax might be `#[allow(lint)]`
* Scott:
    * by the time we have explicit syntax for both sides, shouldn't it just be a compiler error, not a lint
* Josh:
    * One advantage of the lint is that we can choose to drop it, in favor of doing the inference
    * If we decided that there are really no false position
* Niko:
    * Wouldn't that require an error? Someone could have allowed the lint
* Josh:
    * Right, if we enforce some explicit syntax, we could always switch it later, once we've had time for people to (potentially) find cases where you don't want the default
* Niko:
    * I am not saying there are no examples, but I'm saying that those examples have to fit a very particular pattern
        * they have to be borrowing from a type that includes a type parameter
        * and it has to be known that the impls will borrow from that parameter but not from any generics
            * (but then why is it a GAT?)
* Mara:
    * How often would it happen that you add a method that changes the meaning of a GAT?

### Q

Josh: `fn next<'b>(&'b mut self) -> Self::Item<'b>;` has an explicit lifetime. Will this work just as well with lifetime inference?
(Scott: I'd guess it'd need `-> Self::Item<'_>` under the new lints, but seems like it would?)

Yes, I was just using explicit names.

### Q

Josh: Is there a way to raise the declaration of `'a` to the level of `trait LendingIterator` so that the same `'a` can be used in both declarations, rather than introducing a `'b` in the function declaration?

```rust
trait LendingIterator {
    fn foo<'a>(&'a self) -> impl Trait<'a>;
}

// means something quite different:
trait LendingIterator<'a> {
}
```

(Withdrawn, any kind of syntax for that could imply a runtime equivalence that doesn't exist.)

### Q

Josh: Third possibility: what about running the inference to detect if the smart default may be appropriate, then emitting a warning with suggestion (and having a syntax to say "no, really, I don't want that")? Closely related: if we *do* use the smart default, I think we need a syntax for averting that default, rather than introducing an artificial supertrait.

had this conversation

### Q

Scott: Are these bounds relevant to the impls too, or just to the trait definition?  (Guess: the impl uses the concrete thing so can be less restrictive, like works with lifetimes in impls today.)

* Niko: relevant to the impls too, I would assume the impl inherits the defaults from the trait too
* Scott: could we remove the default? It'd be a breaking change to get rid of them, right?
    * Niko: well, without an edition
* Scott: I'm thinking about cases where we do require people to repeat things...
* Scott: Smart default looks at uses in the trait definition to figure out whether the where bound is there
    * does the impl look at the methods as defined in the impl, or does it copy from the trait?
* Niko: Good question, I assumed it would copy from the trait, but that might work too
* Scott: if it did neither, what would happen?
* Niko: the impl would just get an error
* Felix: I'm confused, don't the traits/impls have to stay in sync
* Niko: today the impl can be looser
* Scott: I'm imagining if you have an impl that has no lifetimes, then I wouldn't need a where clause there
* Niko: yes, and that was kind of my example above -- traits in general can't know that, but sometimes you might know that the impl is not going to use type/lifetime parameters from the self type, and hence the where clause wouldn't be needed
    * but this requires that you know the set of impls you will have

### Q

Felix: Can we ship with the status quo and add inference later without it being a breaking change? I'm guessing "no", but I wanted to confirm. (copied from inline comment above)

Niko: In theory no, but maybe in practice?

### Q

Mark: How dangerous is it that folks *adding* a method (perhaps with a default body) will potentially cause breaking changes to downstream consumers?

```rust
trait Something {
    type Foo<'a>;
    
    // added later:
    fn foo<'a>(&'a self) -> Option<Self::Foo<'a>> {
        None
    }
}

// previously existent
impl<T> Something for Foo<T> {
	type Foo<'a> = ();
}

// also previously existing
fn foo<T>(x: <T as Something>::Foo<'static>) // that was invoked with T = Foo<U>

```

```rust
trait Foo {
  type Bar<'a>;

  // added later
  fn foo<'a>(&'a self) -> Option<Self::Bar<'a>> {
    None
  }
}
trait Parser: Foo {
  fn parse<'a, 'b>(&'a self, input: &'b MyData<'b>) -> Self::Bar<'b>;
}
impl Foo for Baz {
  type Bar<'a> = ();
}
// After addition, requires that Self: 'input
fn bar<'a, input, T: Parser>(&self, input: &'input MyData<'input>) -> <T as Foo>::Bar<'input> {} 
```
Jack: not sure if this actually is a good example

### Q

Josh: Can we talk about the first alternative in appendix A, the `'self` lifetime? That would be really useful in general. That seems to be what appendix B is suggesting too. How difficult would it be to add?

```rust
trait Foo {
    type Bar<'self> = ...;
    // a lifetime parameter 'a and a where clause that says Self: 'a
}
```

```rust
fn foo(&self, x: &str) -> &str; // is already elided to have the output be the lifetime from `&self`.
```

```rust
struct S {
    field1: Vec<String>,
    field2: &'self str,
}
```

Niko: a 'self syntax would not cover cases like this, though the smart default would

```rust
trait Parser<Input> {
    type Output<'i> where Input: 'i;
    
    fn parse<'i>(&self, input: &'i Input) -> Self::Output<'i>;
}
```

### Q

Scott: Should we explore what a more direct opt-out would be, to see what it would be like if the default was less smart?

* Mark: you could opt out if you write any where clauses, right?

```rust
trait Parser {
    type Output<'a, T> where T: Debug; // you want the default here, too
    
    fn foo<T>(&self) -> Self::Output<'_, T>;
}
```

Josh proposal:

```rust
type Output<'a> where 'a; // (1)
type Output<'a> where 'a: Self; // (2)  --- not legal syntax today
type Output<'a> where T: Foo<'a>; // (3) what would happen here?
type Output<'a> where 'a:; // (4) this is legal today, right?
```

Felix proposal:

```rust
type Output<'a> where Self: ?'a
```

Definitely obscure, but maybe it's very unusual, and that's ok.  The analogy to `?Sized` holds up.

Could use a marker:

```rust
#[no_lifetime_default]
type Output<'a>;
```

Josh: I'd support the smart default iff we have an opt-out syntax that doesn't require an artificial supertrait or anything else that would require changes in users or implementers of the trait.

Jack: The lint is appealing because we could find out if anybody in practice is using this.

Niko: I'd vaguely prefer to force people to write `where Self: 'a` or an opt-out, and a plan to either add the default or something else in the future.

Quick poll:

* scott: interested in a naive default with some way to opt-out, kinda like lifetime elision, if that hits enough scenarios to make the explicit form rare enough.  But I could probably also be convinced that the analogy to structs holds up and that we should look at how they're used too.
