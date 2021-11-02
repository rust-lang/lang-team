---
title: "Design meeting 2021-10-13: Where the where"
tags: "design meeting"
---

# Design meeting 2021-10-13: Where the where

Issue: https://github.com/rust-lang/rust/issues/89122
[Source](https://rust-lang.github.io/generic-associated-types-initiative/design-discussions/where-the-where.html)

First brought up on zulip: https://rust-lang.zulipchat.com/#narrow/stream/144729-wg-traits/topic/GAT.20syntax.20whining

## Summary

Proposed: to alter the syntax of where clauses on type aliases so that they appear *after* the value:

```
type StringMap<K> = BTreeMap<K, String>
where
    K: PartialOrd
```

This applies both in top-level modules and in traits (associated types, generic or otherwise).

## Background

The current syntax for where to place the "where clause" of a generic associated types is awkward. Consider this example ([playground](https://play.rust-lang.org/?version=nightly&mode=debug&edition=2018&gist=bdb55a5d5cb17e20d73e22a3f2db0e57)):

```rust
trait Iterable {
    type Iter<'a> where Self: 'a;

    fn iter(&self) -> Self::Iter<'_>;
}

impl<T> Iterable for Vec<T> {
    type Iter<'a>
    where 
        Self: 'a = <&'a [T] as IntoIterator>::IntoIter;

    fn iter(&self) -> Self::Iter<'_> {
        self.iter()
    }
}
```

Note the impl. Most people expect the impl to be written as follows (indeed, the author wrote it this way in the first draft):

```rust
impl Iterable for Vec<T> {
    type Iter<'a>  = <&'a [T] as Iterator>::Iter
    where 
        Self: 'a;

    fn iter(&self) -> Self::Iter<'_> {
        self.iter()
    }
}
```

However, this placement of the where clause is in fact rather inconsistent, since the `= <&'a [T] as Iterator>::Iter` is in some sense the "body" of the item.

The same current syntax is used for where clauses on type aliases ([playground](https://play.rust-lang.org/?version=nightly&mode=debug&edition=2018&gist=74eeed1795b693f238150f825a0e8438)):

```rust
type Foo<T> where T: Eq = Vec<T>;

fn main() { }
```

## Top-level type aliases

Currently, we accept where clauses in top-level type aliases, but they are deprecated (warning) and semi-ignored:

```
type StringMap<K> where
    K: PartialOrd
= BTreeMap<K, String>
```

Under this proposal, this syntax remains, but is deprecated. The newer syntax for type aliases (with `where` coming after the type) would remain feature gated until such time as we enforce the expected semantics.

## Interaction with trait aliases

One thing discussed in the thread was the interaction with trait alias syntax. With trait aliases, the where clauses can appear in different positions relative to `=` with different meanings:

```rust
// To use this alias, `T: Bar` must hold
trait Foo<T> where T: Bar = ...

// `Foo<T>` is an alias for `T: Bar`
trait Foo<T> = ... where T: Bar
```


However, this is related to another point about GAT syntax:

```rust
trait Foo {
    // To use Bar, T: Ord must hold
    type Bar<T: Ord>;
    
    // Impl must prove that `Self::Bar<T>: Ord`,
    // and other users can rely on that
    type Bar<T>: Ord;
}
```

There is no syntax with GATs today to add "impl must prove" constraints like `T: Ord`. Perhaps if there were, it might apply to trait aliases, too?

## Alternatives

### Keep the current syntax.

In this case, we must settle the question of how we expect it to be formatted (surely not as I have shown it above).

```rust
impl<T> Iterable for Vec<T> {
    type Iter<'a> where Self: 'a 
        = <&'a [T] as IntoIterator>::IntoIter;

    fn iter(&self) -> Self::Iter<'_> {
        self.iter()
    }
}
```

### Accept either

What do we do if both are supplied?

### Questions / Discussion

### Q

> Mark: I'm not sure I understand the distinction between the following two (from the trait alias section). Isn't it true in both that the `T: Bar` bound must hold?
> * is there a concise explanation somewhere for this? It feels like a critical element but also not something obvious (to me, anyway).


```rust
trait SendSync: Send + Sync
```

What about something more complex? [RFC #1733](https://rust-lang.github.io/rfcs/1733-trait-alias.html) gave this syntax:

```rust
trait RevPartialEq<T> = where T: PartialEq<Self>;
```

To illustrate difference:

```rust
// OK
trait RevPartialEq<T> = where T: PartialEq<Self>;
fn test<X: RevPartialEq<u32>>(x: X) {
    22_u32 == x
}
```

```rust
// Error: `u32: PartialEq<T>` does not hold
//
// Related to implied bounds, though.
trait Test<T> where T: PartialEq<Self> = Debug;
fn test<X: Test<u32>>(x: u32) {
    22_u32 == x
}
```

just as if I had written:

```rust
// just as if I had written:
//
// trait Test<T>
// where T: PartialEq<Self>
// { }
```

```rust
trait Test<T>
where Self: PartialEq<T>
{ }

// No error: `Self: PartialEq<u32>` is considerd a "super trait"
// and hence part of "implied bounds".
fn test<X: Test<u32>>(x: u32) {
    22_u32 == x
}
```

No way to make an implied bound that doesn't begin with Self.

```rust
trait DebugIterator {
    type Item: Debug;
    // Knowing that X: DebugIterator implies X::Item: Debug
}
```

No way to make an implied bound that doesn't begin with Self.

```rust
trait DebugIterator {
    type Item<T>: Debug;
    // What if I wanted `T: Debug<U>` as an implied bound?
    // Does that even make sense??
}
```

Niko: I feel the trait alias syntax is just confusing

Mark: Is there a reason to have both kinds of where clauses in traits? What if you only had implied bounds with trait aliases? (Or only the reverse...)

Felix: Does it have any impact on quality of dev ex?
    * Maybe you get an error that points to a better line of code?
    
We don't know.

Jack: As a data point, there was at least one issue with GATs where they were using where clauses but shouldn't have been (or maybe it was the other way around...) but I don't know what the right syntax would be.

(Update: I think this was the issue: https://github.com/rust-lang/rust/issues/87831)

Niko: I remember this, I think they were putting the where clause on the trait maybe?

### Q

> Felix: is part of expectation here that `cargo fix` or `rustfmt` will auto-convert the ugly syntax to the nice one? (This ties into my Q above regarding the increase in severity for parametric type aliases.) ((or maybe type aliases would be excluded from auto-suggested change))

niko: cargo fix for sure. For rustfmt, if we said that it was an error in GATs, then it would be new ground, right? Accepting a superset of rust and emitting rust? But I don't see any fundamental reason why not.

pnkfelix: if you convert type aliases, you are going from a where clause that gets ignored to one that gets enforced.

niko: yeah we'd need some opt-in of some kind.

### Q

> * Felix: In the "accept either" option: I assume we will always have to accept the old syntax, at least in old editions.
>	* so it makes some sense to continue accepting either.
>	* but that does *not* mean we have to accept both in tandem on the same item.

That makes sense, at least for top-level type aliases.

### Q

> * Scott: Random thought: `WHERE` and `HAVING` being different in SQL is also confusing to people, so I don't know if different keywords would necessarily be better.  Probably still better than it being position-dependent, though.

Niko: egads I forgot what those mean

Scott: HAVING comes after the GROUP BY, and you can reference difference things; it applies to the results of your projection.

Niko: I kind of agree that unless the keywords are *very well chosen* it won't help.

Scott: Is there some way to leverage the same thing twice somehow to get the semantics, vs two different keywords?

Niko: Do people even want that distinction? When discussing implied bounds, there was discussion about how removing where clauses becomes semver significant. Almost a private vs a public bound. Do you get to rely on it, or does everyone else get to rely on it too?

### Q

> * Scott: we have `struct Foo<T>(T) where T: Copy;` but `struct Foo<T> where T: Copy { x: T }`.  Does anyone remember why those are different?  Is there a general rule there that we can divine from there that would help us pick here?  Something about braces?

Niko: I don't recall it being discussed a lot, but the general rule was that the first one looked like a function. [RFC 0135](https://rust-lang.github.io/rfcs/0135-where.html)

Scott: Seems like it has to do with braces enclosing a lot of content, but the parens were short... maybe that says that the type is "one thing", not a multi-line whatever, so it is more like the tuple struct case.

Niko: I feel that way, it feels more like the fn or tuple struct case to me.

Scott: It feels like it would be weird to have the where clause list before the parameter list or `->` in a function, although you could have it there if you wanted.

Niko: The original RFC argued that function parameters and return types were more important than the where clauses, and that the bounds were a kind of footnote. I think you can make an analogous case here.

Niko: Does anyone want to argue *against* the proposed where clause syntax?

Scott: I'm in favor, but I'm trying to figure out *why*.

Jack: It's very subjective. Everybody seems to agree that the where clauses should go at the end because of formatting, but it's very subjective.

Arguments for the current syntax (where before =):

* Consistency of 'copying and pasting' an item from trait and appending 'the definition' (which in this is the value of the type alias).
* Trait alias subtleties.

Arguments against (iow, for where after =):

* Consistency with function and tuple struct placement
    * Note that we initially had tuple structs put the where before the `()` ([link](https://github.com/rust-lang/rust/issues/17904#issuecomment-58603749)) but it was "obviously wrong" somehow
* The thing after the where clauses (if any) should start on its own line, and we don't have precedent for `=` starting on its own line
    * where clauses are a "big long list" of things and you want it to be easy to find the end
* Calls attention to the value of the type alias vs the bounds
    * can put them in the angle brackets to emphasize them (most of the time)
    * ["secondary notation"](https://en.wikipedia.org/wiki/Cognitive_dimensions_of_notations) works better, more able to draw a distinction

There's a reason that rustfmt behaves like so:

```rust
let x = long_expression;

// Becomes
let x =
    long_expression;

// Not
let x
    = long_expression;
```


Example of issue with "long" where clauses: https://github.com/rust-lang/rust/issues/86787

```rust
    type T = Either<Left::T, Right::T>;
    type TRef<'a>
    where 
        <Left as HasChildrenOf>::T: 'a,
        <Right as HasChildrenOf>::T: 'a
    = Either<&'a Left::T, &'a Right::T>;

    type T = Either<Left::T, Right::T>;
    type TRef<'a>
    where 
        <Left as HasChildrenOf>::T: 'a,
        <Right as HasChildrenOf>::T: 'a
        = Either<&'a Left::T, &'a Right::T>;

    type T = Either<Left::T, Right::T>;
    type TRef<'a> = Either<&'a Left::T, &'a Right::T>
    where 
        <Left as HasChildrenOf>::T: 'a,
        <Right as HasChildrenOf>::T: 'a;
    
    // Scott: but ending a multiline list with commas?
    // Niko: how is it different from tuple structs?
    
    struct Foo<T>(u32)
    where
        T: Ord;
        
    // Yup, checked, and rustfmt does
    struct Foo<T, U>(T, U)
    where
        T: Ord,
        U: Ord;
```

### Q

Would implied bounds even make sense on GATs?

```rust
trait MyTrait {
    type MyType<T> = where T: PartialEq<Self::MyType<T>>;
}

impl MyTrait for .. {
    type MyType<T> = ValueType where T: PartialEq<Self::MyType<T>>;
}

impl<A> PartialEq<ValueType> for A { }

```

It is true that

* if we do want a syntax for this
* and it is the trait alias syntax

we just messed it up.
- Comment from Jack - But actually the "implied bound" where clause is in front of the equals, and this is not that

### Q

> Scott: Do we need people to copy them over? Is it a breaking change to remove them? Does the impl need to repeat them?

Today they are implied, but we've also discussed this proposed change:

```rust
trait Foo {
    unsafe fn foo();
}

impl Foo for () {
    fn foo();
}

fn safe_fn() {
    <() as Foo>::foo(); // No unsafe needed
}
```

"If the compiler can identify your impl, it can use the signature for your impl" -- should that work just for unsafe, but not for where clauses? Seems weird.

Scott: What about lifetimes?

Niko: Currently everything is typed against the trait signature. I would expect that to be consistent with unsafe but it occurs to be that there is more of a circularity implementation wise.

* Today:
    * Consumers of the associated type would ignore the bounds on the impl, so they could be defaulted to the ones that appear in the trait.
* Tomorrow:
    * But if we made it significant that you use fewer, that would be a change, and in particular having an empty list would make code stop compiling if those predicates were required for the value to be well-typed.

But of course we could do this over an edition easily enough.

