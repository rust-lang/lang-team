# Design Meeting: Shipping const generics MVP

* Meeting proposal: [rust-lang/lang-team#37](https://github.com/rust-lang/lang-team/issues/37)
* [Watch the recording](https://youtu.be/e3cWvpEPWrA)
* [boats blog post](https://without.boats/blog/shipping-const-generics/)

## Idea: define a const generic MVP

* Origin: [PR to remove the "limiting trait"](https://github.com/rust-lang/rust/pull/74060) that prevents us from printing etc for arrays of arbitrary lengths
    * been using the const generic impls for >1 year
* Can we offer these same capabilities beyond std?
* Stabilizable subset that covers a lot of functionality. Mostly what is needed to make arrays a "first class citizen".
* Restrictions:
    * Only integral primitive types (u8...u128, i8...i128, char, bool, usize, isize)
    * Only certain expressions as the "value" for a const generic
        * Either things that could define a constant right now (and hence can be evaluated), not including any other generic parameter
        * Or a generic const parameter (by itself, not as part of any const expression)
* Exclusions:
    * Floating points, so we don't have to deal with NaN
    * `&str` (even `&'static str`)

## Ordering restrictions

* Currently require constants to come last
* Currently we permit you to have a defaulted type parameter before constants
* But in the future we probably want a free form order, but prohibit defaulted parameters from coming before non-defaulted parameters
* Current system can lead to an ambiguity because you might have `Foo<T>` where `T` is both a type and a constant, and the definition might be `Foo<A = u32, const B>` -- so is `T` meant to be a value for `A` or `B`?

## How to avoid dependencies on parameters?

* Can we just check for free variables or something?
* Not quite:

```rust
trait Trait<T> { const CONST: usize; }

impl Trait<()> for u8 { const CONST: usize = 0; }

type Foo<T, U> where u8: Trait<T> + Trait<U> = [(T, U); u8::CONST];
```

* compiles today but shouldn't, because it currently doesn't consider the `u8: Trait<T>` where clauses in scope, which would trigger an ambiguity

* you could use named constants, just disallow references to type parameters, which we detect lexically, apart from cases like the above where they get filled in by the resolution logic 

* Josh: Can you reference `T::SOME_CONST` (as long as you don't have an expression), or is `::` disallowed?
    * Answer: disallowed because it is a non-trivial expression that references `T` (i.e., the entire expression is not just `T`)

## Aggregate data types

* What about tuples of integral types and the like?
* That would require [integer trees](https://github.com/rust-lang/compiler-team/issues/323).
    * but do we really need to block on this?
    * if we permit aggregate types, this would allow us to resolve padding in a more elegant fashion
* But if we're going to block on that, maybe we can allow everything that permits "structural equality", though we'd have to come to a consensus on what that means
    * We can perhaps still avoid that by disallowing things that derive PartialEq, Eq and only including builtin types
* Main motivation:
    * Not that hard to implement
    * Would make feature richer for users
* But is it richer in a way that people actually *care* about?
    * Gives tuples but not custom types or strings
    * Does anyone care about this?

## Unsupported use case

* One big use case is cryptographic hashing
    * that library uses an interface abstract over all hashers
    * each impl chooses a length
    * this isn't supported by the MVP because you can't create `H::LENGTH` constants

## Arithmetic

* Josh: Does the subset we want to enable prevent inventing type-level arithmetic (e.g. Peano numbers)? We don't want to disable arithmetic but give people a complex workaround.
    * In other words, you can't do `Foo<{A + B}>` for some generic constants `A` + `B`, will people try to workaround this with horrible things like Peano numbers?
    * Those libraries already exist (`typenum`) and we don't fully replace those use cases
* Is there a concern that offering this subset will wind up pushing *more* people to use complex workarounds?
    * No.

## Equality

* RalfJ opened https://github.com/rust-lang/rust/issues/74446 for this
* Three notions of equality
    * using the `==` operation
    * matching on a constant
    * are two types `Foo<C1>` and `Foo<C2>`
        * `impl<T> TypeTrue for TypeEq<T, T>`
* Going to need a longer discussion, not an issue for the MVP

### Match expressions and const generics today

```rust
fn foo<const C: usize>(x: usize) {
  match x {
    C => ...,
    //~^ ERROR const parameters cannot be referenced in patterns
    _ => ...
  }
}
```

* Presently match patterns cannot reference const generic parameters like `C`
* Can match on associated constants [like this](https://play.rust-lang.org/?version=nightly&mode=debug&edition=2018&gist=045d654c0940f3436895c91ce9fa781f) or [even this](https://play.rust-lang.org/?version=nightly&mode=debug&edition=2018&gist=78f3c3e7c840a7024fdba34830428609).
    * What about if the normalization resolves to a constant?

## Stabilization steps

* Maybe blocked on integer trees
    * Compiler team needs to do an audit
* Compiler team MCP to carve out the feature
* Testing:
    * Would be nice to have systematic confidence that we are trying to use constants in all the places one *could* use constants and have defined behavior
    * We have a lot of tests, but a lot of them are strange border cases that arose
    * We do at least have the std traits using them

## Overlap between const generics and const evaluation?

* Yes but everybody is involved
* Connection is that anything you can "const eval" can become a parameter to a type, but that was always true for array lengths

## Some issues to consider before stabilizing

* Papercut and diagnostic issues
* Implementations sometimes can't distinguish between a type and a constant with the same name, even if the position is unambiguous
* Probably we should be prepared to put in some "polish" effort post stabilization

## More generally

* This is a great tactic, we should use it more often
* Maybe named impl Trait could be a good candidate
* Specialization might be, but we'd have to review, and it would take some "education"
    * would want to talk out a bit where we're going since where clauses in particular might want a different syntax
