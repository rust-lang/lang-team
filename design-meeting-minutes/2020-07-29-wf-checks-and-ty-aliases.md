# Discuss WF checks and type aliases

* Meeting issue: https://github.com/rust-lang/lang-team/issues/25
* [Watch the recording](https://youtu.be/tIBZYQSA_eM)

## Context

The way we implement type aliases goes way back. We basically eagerly expand them when 'convering' from the syntax tree into the compiler's internal representation of types. When you have e.g. `type MyAlias = u32; fn foo(x: MyAlias)`, the type-checker just sees `fn foo(x: u32)`, in effect, and we ignore the `type MyAlias = u32`.

We also permit types that are not well-formed. These only report an error if they are used:

```rust
struct MyType<T: Debug> { }
type MyAlias = MyType<dyn Display>; // dyn is not Sized, Display is not Debug

fn foo(x: MyAlias) { }
// seen as `fn foo(x: MyType<dyn Display>)`
```

One consequence is that we allow type aliases with both too many and too few where clauses:

```rust
struct MyType<T: Debug> { }

type JustRightAlias<T: Debug> = MyType<T>; // JustRightAlias<X> will error if X: Debug is not true
type TooFewAlias<T> = MyType<T>;
type TooManyAlias<T: Debug + Display> = MyType<T>; // TooManyAlias<Vec<u32>> is not an error
```

We do not report errors it `TooManyAlias<Vec<u32>>` is used, even though `Vec<u32>: Display` is not true.

One way we do look at those where clauses:

```rust
type IteratorItem<T> = T::Item; // error
type IteratorItem<T> = <T as Iterator>::Item;
type IteratorItem<T: Iterator> = T::Item;
```

We also give warnings if you do use bounds.

## What behavior do we ultimately expect?

One important thing for us to decide is what behavior we ultimately expect here, and perhaps the interaction with implied bounds.

### Naive behavior

I think what I would most naively expect is that `type Foo<T> where WC = U` is legal if

```rust
struct Foo<T> where WC {
  f: U
}
```

is legal (not including any struct-specific requirements around `Sized`).

### Implied

But Niko has usability concerns related to implied bounds -- will this result in a bunch of copying of where clauses around?

```rust
struct Foo<T: Debug> { }
impl Foo<T> {
  // can assume T: Debug is true in here
}
```

```rust
struct SomeStruct<T: Debug> = ...;

// feels like annoying boilerplate for `T`:
type VecSomeStruct<T> = Vec<SomeStruct<T>>
```

Another implied bounds like question:

```rust
fn foo<T>(x: VecSomeStruct<T>) {
// can I assume T: Debug, just like I would if `Vec<SomeStruct<T>>`? Probably yes?
}
```

### Rationalizing the current behavior

I've considered that we might "rationalize" the current behavior if we had a notion of implied bounds where a type-alias *assumes* the RHS is well-formed. For example, `type TooFewAlias<T> = MyType<T>` would be sort of "short for" 

```rust
type TooFewAlias<T> where WellFormed(MyType<T>) = MyType<T>
```

(this is not legal syntax, of course).

This is probably similar to what we would wind up doing for WF in `fn`, since we have some similar backwards compatibility constraints there (e.g., `for<'a> fn(Foo<'a>)` is always considered WF today, even if you have `struct Foo<'a: 'static> { }`).

We can also change the default for type parameters to be `?Sized` here, which is key for examples like `type Foo<T> = Box<T>`.

**Catch.** However, this does not handle the fact that `TooManyAlias` does not enforce its "extra bounds".

## Experiments conducted

eddyb made a number of PRs around the time of the Rust 2018 edition doing various experiments. They were looking at ways to enforce bounds at the definition site. If we could show that there were "exactly the right set of bounds" for type aliases, and that the RHS is well-formed, then we have a reasonably complete check.

Some interesting examples:

* `type Foo<T> = Box<T>` -- too many constraints, but common
    * `type Foo<T: ?Sized>`
    * if we had the implied bounds approach, we could also make `T: ?Sized` the default in type aliases

## estebank's PR

Error at the declaration site if:

```
type Foo = Vec<dyn Debug>;
```

## Observation: 

If we wanted the "implied bounds" solution, then checking for "too many bounds" at the declaration site would be a conservative version of this (ignoring Sized bounds). eddyb ran this test in https://github.com/rust-lang/rust/pull/69741 and found 60-odd regressions, including some impact on Diesel.

## Questions

* What direction do we want?
    * "Naive approach" or "implied bounds" approach
    * Note: Implied bounds says that impls/fns get to assume their "arguments" are valid, this is a bit different potentially
* How much do we care about pushing on this right now, versus later? (e.g., when the chalk work has landed and implied bounds machinery can be better evaluated)

Josh: 

* I see the appeal of the "implied bounds" approach, but I think that if you write *any* bounds, you should write both "the exact correct" set of bounds, and not too many / too few
    * Suggests a possible "migration strategy" we might enforce rules only in cases where people wrote *some* explicit bound (and not e.g. `type Foo<T>`).
* We should not warn if you specify no bounds at all; that'd be consistent with implied bounds.
    * We should do the same for functions: you shouldn't have to restate all the bounds of a type in order to use that type.
* We should warn if you specify bounds but not enough bounds.
* If you specify *too many* bounds, I think it is reasonable to either:
    * warn against using too many bounds 
    * or enforce them
        * Another argument to enforce them: you might put a type alias in your public API, and want those additional bounds for forward-compatibility

Felix:

* I feel similarly but I think you should be allowed to specify extra bounds (and they should be enforced)
    * i.e., you might want to have a type alias that accepts a smaller set than the full type could accept

Niko:

* I feel similarly that I think you should be able to write too many bounds and they should be enforced, but that implied bounds is probably more ergonomic
* One hesitation around implied bounds:
    * Before the rule for "where you had to write all the bounds" was "in types you do, in fns/impls you don't", but this makes it a bit more subtle
        * basically in structs/enums/unions/traits you do :)
        * "nominal type declarations" or "definitions" (as opposed to aliases)

### Related case

Functions:

```rust
for<'a> fn(Foo<&'a T>, Foo<&'static T>)

for<'a> where WellFormed(Foo<&'a T>), WellFormed(Foo<&'static T>) fn(
    Foo<&'a T>, // <-- accepted today
    Foo<&'static T>, // <-- error today 
)
```

One compromise might be linting (warning, at least) against types that are certainly going to be errors (like https://github.com/rust-lang/rust/pull/69741).

```
fn foo() where Vec<u8>: Display { }
```

## Immediate step:

Issue warning for cases where we know it must be illegal to use the type alias.

Could be future-compatibility but it's not clear this will ever be a hard error.
