
# Variadic Generics

A draft for Variadic Generics has existed since ~2013, and there have been multiple postponed RFCs surrounding
the topic. Despite this, the difficulty of designing such a system as well as lack of singular best choice
has led to no resolution. This document has been written as an attempt to 'sum up' proposals and discussion in
this space.

# Proposals so far

Following will be an overview of each of the proposals so far, followed by their individual pros and cons

## EddyB's Draft

This is a simple proposal, and the oldest one. It proposes the basic idea that all variadic types can
be seen as just a tuple with N elements. The syntax is thus built around allowing expansions of tuple
types into their components, and taking multiple types that will be condensed into a tuple.

For using these types, it is proposed that one destructure the tuple, generally into a single head, and the remaining
tail. This matches the recursive style used in C++.

### Pros

- Simple. Doesn't add much syntax while allowing the driving use case, the `Fn` traits and similar designs.

- Allows intuitive function call ergonomics

### Cons

- Mentioned by eddyb, a subtuple type could have different padding than its parent.
  EG: `&(A, B, C, D) != &A, &(B, C, D)` 

- The `..` syntax is already used in ranges, so some other syntax would be needed

### Syntax

```rust
type Tuple<..T> = T;

(..(true, false), ..(1, "foo")) == (true, false, 1, "foo")

fn bar(..x: (A, B, C)) {}
```

## Cramertj's Proposal

Proposes both a syntax similar to C++ as well as a Tuple trait that will be implemented by all tuples.
The trait would contain helpful types and methods for working with variadic tuples:
- AsRefs type, `(A, B, C) -> (&A, &B, &C)`
- AsMuts type, `(A, B, C) -> (&mut A, &mut B, &mut C)`
- elements_as_refs fn, with signature `(&'a self) -> Self::AsRefs<'a>`
- elements_as_mut fn, with signature `(&'a mut self) -> Self::AsMuts<'a>`

- Not provided, but proposed as future extensions:
  - Allowing unpacking tuple types in an argument position, preventing the need to call variadic functions like
  `foo((1, 2.0, "3"))`
  - Allowing the `...` syntax in generic type position, preventing the need to write traits such as `Fn` like
  `Fn<(T1, T2, T3)>`

### Pros

- Proposes simple syntax, similar to eddyb proposal

- Includes Tuple trait, which is helpful for working with variadic arguments

### Cons

- Overlaps with inclusive range syntax, though it shouldn't actually break any backwards compatibility

### Syntax

```rust
trait MyTrait: Tuple {}

impl MyTrait for () {}

impl<Head, Tail> MyTrait for (Head, ...Tail) where Tail: Tuple {}

fn foo<T: MyTrait>(args: T) {}
```

## Fredpointzero's Proposal

Takes a similar path to previous proposals in terms of syntax, but adds on a lot of ergonomics syntax.
This additional syntax is proposed to allow easier usage of the variadic types, allowing the user to generate
more imperative loops over the variadic type. A lot of the debate on this proposal surrounded the proposed for
loop syntax, as many found it hard to read/comprehend at a glance compared to the rest of the proposed syntax.

### Pros

- Ergonomics, this proposal allows more linear usage of variadic types, which the other proposals lack.

- Power, this proposal allows almost full control over the bounds on variadics, and provides many example
  implementations.

### Cons

- Complexity, much more new syntax is needed than in the other proposals

- No way to terminate recursive functions currently, they just won't compile

### Syntax

```
struct Foo<(..T)>
where
    ..(T: Debug)
{
    items: (..T)
}

fn append<(..L), (..R)>(l: (..L), r: (..R)) -> (..L, ..R) {
    todo!()
}

fn foo<(..T)>(args: (..T)) {
    let (head, tail @ ..) = args;

    (for arg <ARG> @in args <T> {
        Vec::<Arg>::new();
    })
}
```

## Common notes

All the proposals start with a similar syntax, using two or three dots to represent packing/unpacking types.
They all act on tuples, extending language syntax to allow tuples of varrying instead of constant arity, and building
all new functionality on top of that idea.

The overlap with range is a common topic of discussion, but all full proposals seem to still use that syntax.
Overall, the basic syntax seems to be agreed on, but usability and 'extra' ergonomic functionality still requires
a lot of work.
