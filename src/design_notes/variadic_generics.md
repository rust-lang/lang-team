# Variadic Generics

A draft for Variadic Generics has existed since ~2013, and there have been multiple postponed RFCs surrounding
the topic. Despite this, the difficulty of designing such a system as well as lack of singular best choice
has led to no resolution. This document has been written as an attempt to 'sum up' proposals and discussion in
this space.

## Background reading

- [Analysing variadics, and how to add them to Rust - Olivier Faure](https://poignardazur.github.io/2021/01/30/variadic-generics/)
- [rust-variadics-background - Alice Cecile](https://github.com/alice-i-cecile/rust-variadics-background/)
- [More enum types - Yoshua Wuyts](https://blog.yoshuawuyts.com/more-enum-types/)
- [\[Brainstorming\] Use cases for variadic generics - /r/Rust
](https://www.reddit.com/r/rust/comments/l8vaa6/brainstorming_use_cases_for_variadic_generics/)
- [A mirror for Rust - JeanHeyd Meneide](https://soasis.org/posts/a-mirror-for-rust-a-plan-for-generic-compile-time-introspection-in-rust/)

## Potential use cases

- Varargs library functions ([`iter::zip`](https://doc.rust-lang.org/std/iter/fn.zip.html), [`futures::join`](https://docs.rs/futures/latest/futures/future/fn.join.html)…)
- Fix the [`Fn` traits](https://doc.rust-lang.org/std/ops/trait.FnOnce.html), eliminate `"rust-call"` abi ([GitHub issue](https://github.com/rust-lang/rust/issues/41058))
- Implement traits for tuples ([stdlib](https://doc.rust-lang.org/std/primitive.tuple.html#trait-implementations), [Sourcegraph](https://sourcegraph.com/search?q=context:global+lang:Rust+impl%5B+%3C%5D+.*+for+%5C%28&patternType=regexp&sm=0&groupBy=repo))
- Implement traits for function pointers/`dyn Fn` ([stdlib](https://doc.rust-lang.org/std/primitive.fn.html#trait-implementations), [`wasm-bindgen`](https://github.com/rustwasm/wasm-bindgen/blob/f569fddb62cdeb2bb9ba230fe59d3fba143cf92c/src/convert/closures.rs))
- `#![feature(unsized_fn_params)]`
- Homogeneously-typed varargs (for example [`cmp::max`](https://doc.rust-lang.org/std/cmp/fn.max.html))
- Fully replace [`frunk::HList`](https://docs.rs/frunk/latest/frunk/hlist/index.html)
- Variadic enums (for example [`futures::select`](https://docs.rs/futures/latest/futures/future/fn.select.html))
- Other [`frunk`](https://docs.rs/frunk/latest/frunk/) use-cases?
- Compile-time reflection/macro-free `derive`?

## In other languages

- [C](https://en.cppreference.com/w/c/variadic)
- [Clojure](https://clojure.org/guides/learn/functions#_multi_arity_functions)
- [Common Lisp](http://clhs.lisp.se/Body/03_dac.htm)
- [Crystal](https://crystal-lang.org/reference/1.8/syntax_and_semantics/default_values_named_arguments_splats_tuples_and_overloading.html#components-of-a-method-definition)
- [C++](https://en.cppreference.com/w/cpp/language/parameter_pack)
  - [Circle](https://github.com/seanbaxter/circle/blob/master/new-circle/README.md#pack-traits)
  - [Reversing a tuple](https://stackoverflow.com/questions/17178075/how-do-i-reverse-the-order-of-element-types-in-a-tuple-type)
- [C#](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/params)
- [D](https://dlang.org/spec/function.html#variadic)
- [F#](https://learn.microsoft.com/en-us/dotnet/fsharp/language-reference/parameters-and-arguments#parameter-arrays)
- [Go](https://go.dev/ref/spec#Passing_arguments_to_..._parameters)
- [Haskell](https://wiki.haskell.org/Varargs)
- [Java](https://docs.oracle.com/javase/8/docs/technotes/guides/language/varargs.html)
- [JavaScript](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Spread_syntax)
- [Julia](https://docs.julialang.org/en/v1/manual/functions/#Varargs-Functions)
- [Kotlin](https://kotlinlang.org/docs/functions.html#variable-number-of-arguments-varargs)
- [Lua](https://www.lua.org/pil/5.2.html)
- [ML](http://mlton.org/Fold)
- [Nim](https://nim-lang.org/docs/manual.html#types-varargs)
- [PHP](https://www.php.net/manual/en/functions.arguments.php#functions.variable-arg-list)
- Python ([varargs](https://docs.python.org/3/reference/compound_stmts.html#function-definitions), [variadic generics](https://peps.python.org/pep-0646/))
- [Racket](https://docs.racket-lang.org/guide/lambda.html#%28part._rest-args%29)
  - [Typed Racket](https://docs.racket-lang.org/ts-guide/types.html#%28part._varargs%29)
- [R](https://cran.r-project.org/doc/manuals/r-release/R-intro.html#The-three-dots-argument)
- [Raku](https://docs.raku.org/language/signatures#Slurpy_parameters)
- [Ruby](https://docs.ruby-lang.org/en/3.2/syntax/calling_methods_rdoc.html#label-Array+to+Arguments+Conversion)
- [Scala](https://docs.scala-lang.org/scala3/reference/changed-features/vararg-splices.html)
- [Scheme](https://standards.scheme.org/corrected-r7rs/r7rs-Z-H-6.html#TAG:__tex2page_sec_4.1.4)
- [Swift](https://github.com/apple/swift-evolution/blob/main/proposals/0393-parameter-packs.md)
- [TCL](https://www.tcl.tk/man/tcl/TclCmd/proc.html)
- [TypeScript](https://www.typescriptlang.org/docs/handbook/2/objects.html#tuple-types)
- [VB.NET](https://learn.microsoft.com/en-us/dotnet/visual-basic/programming-guide/language-features/procedures/parameter-arrays)
- [Zig](https://ziglang.org/documentation/master/#comptime)

## Proposals so far

Following will be an overview of each of the proposals so far, followed by their individual pros and cons.

### [EddyB's draft](https://github.com/rust-lang/rfcs/issues/376)

This is a simple proposal, and the oldest one. It proposes the basic idea that all variadic types can
be seen as just a tuple with N elements. The syntax is thus built around allowing expansions of tuple
types into their components, and taking multiple types that will be condensed into a tuple.

For using these types, it is proposed that one destructure the tuple, generally into a single head, and the remaining
tail. This matches the recursive style used in C++. There is also some desire to
be able to iterate in both directions (from left or right) rather than fixing
the choice to a single direction.

#### Pros

- Simple. Doesn't add much syntax while allowing the driving use case, the `Fn` traits and similar designs.

- Allows intuitive function call ergonomics

#### Cons

- Mentioned by eddyb, a subtuple type could have different padding than its parent.
  EG: `&(A, B, C, D) != &A, &(B, C, D)`

- The `..` syntax is already used in ranges, so some other syntax would be needed
  - Note that `...` syntax may be available, as `..=` is now the inclusive
      range syntax.

#### Syntax

```rust
type Tuple<..T> = T;

(..(true, false), ..(1, "foo")) == (true, false, 1, "foo")

fn bar(..x: (A, B, C)) {}
```

### [Cramertj's proposal](https://github.com/rust-lang/rfcs/pull/1935)

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

#### Pros

- Proposes simple syntax, similar to eddyb proposal

- Includes Tuple trait, which is helpful for working with variadic arguments

#### Cons

- None yet.

#### Syntax

```rust
trait MyTrait: Tuple {}

impl MyTrait for () {}

impl<Head, Tail> MyTrait for (Head, ...Tail) where Tail: Tuple {}

fn foo<T: MyTrait>(args: T) {}
```

### [Fredpointzero's proposal](https://github.com/rust-lang/rfcs/pull/2775)

Takes a similar path to previous proposals in terms of syntax, but adds on a lot of ergonomics syntax.
This additional syntax is proposed to allow easier usage of the variadic types, allowing the user to generate
more imperative loops over the variadic type. A lot of the debate on this proposal surrounded the proposed for
loop syntax, as many found it hard to read/comprehend at a glance compared to the rest of the proposed syntax.

#### Pros

- Ergonomics, this proposal allows more linear usage of variadic types, which the other proposals lack.

- Power, this proposal allows almost full control over the bounds on variadics, and provides many example
  implementations.

#### Cons

- Complexity, much more new syntax is needed than in the other proposals

- No way to terminate recursive functions currently, they just won't compile

#### Syntax

```rust
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

### [Jules Bertholet's draft](https://hackmd.io/@x5qEAKM5Q36wKV_Wftw96g/HJFy6uzDh)

Most similar to Fredpointzero's proposal. Like all previous proposals, uses tuples to model variadics. Unlike all other proposals, variadic lifetime and const generics are fully supported.

#### Pros

- Iteration/expansion is explicit (`static for`)
- Most flexible proposal so far, including:
  - Lifetime and const variadics
  - Generalized MxN -> NxM transformation
  - Homogeneous varargs, using arrays

#### Cons

- Most complex of all proposals so far
  - Lots of new syntax, which will need bikeshed

#### Syntax

```rust
// `futures::join`

use  futures_util::future::{MaybeDone, maybe_done};

#[pin_project::project]
pub struct Join<...Futs: Future>(#[pin] ...for<F in Futs> MaybeDone<F>);

impl<...Futs> Join<...Futs> {
    fn new(...futures: ...Futs) {
        let wrapped_futs = static for future in futures {
            maybe_done(future)
        };

        Self(...wrapped_futs)
    }
}

impl<...Futs: Future> Future for Join<...Futs> {
    type Output = for<F in Futs> F::Output;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        let mut all_done = true;

        // TODO: what is the best API for pin projection?

        // Long, annoying type specified for example purposes.
        // In reality, you would infer it with `(..)`.
        let futs: for<F in Futs> Pin<&mut F> = self.project();

        static for fut in futs {
            all_done &= fut.poll(cx).is_ready();
        }

        if all_done {
            let ready = static for fut in futs {
                fut.take_output().unwrap()
            };

            Poll::Ready(ready)
        } else {
            Poll::Pending
        }
    }
}
```

```rust
// Implement non-symmetric `PartialEq` for tuples

impl<...Ts, ...Us> PartialEq<Us> for Ts
where
    for<T, U in Ts, Us> T: PartialEq<U>,
{
    fn eq(&self, other: &Us) {
        static for l, r in self, other {
            if l != r {
                return false;
            }
        }

        true
    }
}
```

### [soqb's design document](https://gist.github.com/soqb/9ce3d4502cc16957b80c388c390baafc)

This is a collection of ideas, not a complete proposal. Among the ideas:

- This is the only document so far to [discuss the trait solving algorithm in detail](https://gist.github.com/soqb/9ce3d4502cc16957b80c388c390baafc#trait-solving).
- Unlike other proposals which use an imperative for-loop style, this document leans heavily into recursion and a functional cons-list style.
  In doing so, it makes an important insight: many transformations of typelists [can be modeled via associated types](https://gist.github.com/soqb/9ce3d4502cc16957b80c388c390baafc#decomposition).

#### Pros

- Typelist recursion is extremely expressive, with a comparatively small cost in syntax
  - Can express transformations like reversing a list
- Recursive and imperative paradigms don't conflict, Rust could support both

#### Cons

- Many people find recursive code hard to read/write/understand, prefer an imperative style
  - for-loop style code can be more compact
- The chain of generic function calls, and successive layout shufflings, that recursive variadics code could generate
  may lead to a lot of extra work in codegen/LLVM, and to runtime slowness or even stack overflow
  if the calls aren't inlined and optimized properly

```rust
trait MyNumsSummed {
    type Output: (..);
}

impl MyNumsSummed for () {
    type Output = ();
}

impl<T: Add, U: (..Add)> for (T, ..U) {
    type Output = (<T as Add>::Output, ..(<U as MyNumsSummed>::Output))
    // `forall<U> { U: MyNumsSummed }` is provable given `U: (..Add)`,
    // using the rules defined in "trait solving#candidate selection".
}

fn sum_my_nums<T: (..Add)>(xyz: T, abc: T) -> <T as MyNumsSummed>::Output {
    match (xyz, abc) {
        // both `xyz` and `abc` are empty; we have no work left to do.
        ((), ()) => (),
        // both `xyz` and `abc` have > 1 element.
        // we add what we can and recurse using the remaining elements.
        (
            (x, y_etc @ ..),
             (a, b_etc @ ..),
        ) => {
            let x: impl Add = x;
            let y_etc: impl (..Add) = y_etc;

            compose!(x + a, ..sum_my_nums(y_etc, b_etc))
        },
        // the tuples aren't the same length.
        // perhaps the compiler will be able to detect that `xyz` and `abc`
        // resue `T` and so are gauranteed to be the same length
        // but this isn't necessary for an MVP.
        _ => panic!("you've been a naughty boy!"),
    }
}

fn main() {
    let abc = (1u8, 2u16, 3u32);
    let xyz = (2u8, 3u16, 5u32);
    let summed_up = sum_my_nums(abc, xyz);
    assert_eq!(summed_up, (3, 5, 8));
}
```

## Common notes

All the proposals start with a similar syntax, using two or three dots to represent packing/unpacking types.
They all act on tuples, extending language syntax to allow tuples of varrying instead of constant arity, and building
all new functionality on top of that idea.

The overlap with range is a common topic of discussion, but all full proposals seem to still use that syntax.
Overall, the basic syntax seems to be agreed on, but usability and 'extra' ergonomic functionality still requires
a lot of work.

## Tuple layout

Another important consideration for these proposals, particularly those resting
on tuples, is the layout of the tuple. We do not currently guarantee any
particular ordering of the fields in a tuple, which can limit our ability to
"subset" the tuple under a reference. See [this
comment](https://github.com/rust-lang/lang-team/pull/76#issuecomment-857206830)
for some discussion.
