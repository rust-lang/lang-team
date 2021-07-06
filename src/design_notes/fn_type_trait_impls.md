# Extending the capabilities of compiler-generated function types

## Background

Both standalone functions and closures have unique compiler-generated types.
The rest of this document will refer to both categories as simply "function
types", and will use the phrase "function types without upvars" to refer to
standalone functions _and_ closures without upvars.

Today, these function types have a small set of capabilities, which are
exposed via trait implementations and implicit conversions.

- The `Fn`, `FnMut` and `FnOnce` traits are implemented based on the way
  in which upvars are used.

- `Copy` and `Clone` traits are implemented when all upvars implement the
  same trait (trivially true for function types without upvars).

- `auto` traits are implemented when all upvars implement the same trait.

- Function types without upvars have an implicit conversion to the
  corresponding _function pointer_ type.

## Motivation

There are several cases where it is necessary to write a [trampoline]. A trampoline
is a (usually short) generic function that is used to adapt another function
in some way.

Trampolines have the caveat that they must be standalone functions. They cannot
capture any environment, as it is often necessary to convert them into a
function pointer.

Trampolines are most commonly used by compilers themselves. For example, when a
`dyn Trait` method is called, the corresponding vtable pointer might refer
to a trampoline rather than the original method in order to first down-cast
the `self` type to a concrete type.

However, trampolines can also be useful in low-level code that needs to interface
with C libraries, or even in higher level libraries that can use trampolines in
order to simplify their public-facing API without incurring a performance
penalty.

By expanding the capabilities of compiler-generated function types it would
be possible to write trampolines using only safe code.

[trampoline]: https://en.wikipedia.org/wiki/Trampoline_(computing)

## Purpose

The goal of this design note is describe a range of techniques for implementing
_trampolines_ (defined below) and some of the feedback regarding those solutions.
This design note does not intend to favor any specific solutions, just reflect past
discussions. The presence or absence of any particular feedback in this document
does not necessarily serve to favor or disfavor any particular solution.

## History

Several mechanisms have been proposed to allow trampolines to be written in safe
code. These have been discussed at length in the following places.

PR adding `Default` implementation to function types:

- https://github.com/rust-lang/rust/pull/77688

Lang team triage meeting discussions:

- https://youtu.be/NDeAH3woda8?t=2224
- https://youtu.be/64_cy5BayLo?t=2028
- https://youtu.be/t3-tF6cRZWw?t=1186

## Example

### An adaptor which prevents unwinding into C code

In this example, we are building a crate which provies a safe wrapper around
an underlying C API. The C API contains at least one function which accepts
a function pointer to be used as a callback:

```rust
mod c_api {
    extern {
        pub fn call_me_back(f: extern "C" fn());
    }
}
```

We would like to allow users of our crate to safely use their own callbacks.
The problem is that if the callback panics, we would unwind into C code and this
would be undefined behaviour.

To avoid this, we would like to interpose between the user-provided callback and
the C API, by wrapping it in a call to `catch_unwind`. Unfortunately, the C API
offers no way to pass an additional "custom data" field that we could use to
store the original function pointer.

Instead, we could write a generic function like this:

```rust
use std::{panic, process};

pub fn call_me_back_safely<F: Fn() + Default>(_f: F) {
    extern "C" fn catch_unwind_wrapper<F: Fn() + Default>() {
        if panic::catch_unwind(|| {
            let f = F::default();
            f()
        }).is_err() {
            process::abort();
        }
    }
    unsafe {
        c_api::call_me_back(catch_unwind_wrapper::<F>);
    }
}
```

This compiles, and is intended to be used like so:

```rust
fn my_callback() {
    println!("I was called!")
}

fn main() {
    call_me_back_safely(my_callback);
}
```

However, this will fail to compile with the following error:

> error[E0277]: the trait bound `fn() {my_callback}: Default` is not satisfied

## Implementing the `Default` trait

The solution initially proposed was to implement `Default` for function types
without upvars. Safe trampolines would be written like so:

```rust
fn add_one_adapter<F: Fn(i32) + Default>(arg: i32) {
    let f = F::default();
    f(arg + 1);
}
```

Discussions of this design had a few central themes.

### When should `Default` be implemented?

Unlike `Clone`, it intuitively does not make sense for a closure to implement
`Default` just because its upvars are themselves `Default`. A closure like
the following might not expect to ever observe an ID of zero:

```rust
fn do_thing() -> impl FnOnce() {
    let id: i32 = generate_id();
    || {
      do_something_with_id(id)
    }
}
```

The closure may have certain pre-conditions on its upvars that are violated
by code using the `Default` implementation. That said, if a function type has
no upvars, then there are no pre-conditions to be violated.

The general consensus was that if function types are to implement `Default`,
it should only be for those without upvars.

However, this point was also used as an argument against implementing
`Default`: traits like `Clone` are implemented structurally based on the
upvars, whereas this would be a deviation from that norm.

### Leaking details / weakening privacy concerns

Anyone who can observe a function type, and can also make use of the `Default`
bound, would be able to safely call that function. The concern is that this
may go against the intention of the function author, who did not explicitly
opt-in to the `Default` trait implementation for their function type.

Points against this argument:

- We already leak this kind of capability with the `Clone` trait implementation.
  A function author may write a `FnOnce` closure and rely on it only being callable once. However, if the upvars are all `Clone` then the function itself can be
  cloned and called multiple times.

- It is difficult to construct practical examples of this happening. The leakage
  happens in the wrong direction (upstream) to be easily exploited whereas we
  usually care about what is public to downstream crates.

  Without specialization, the `Default` bound would have to be explicitly listed
  which would then be readily visible to consumers of the upstream code.

- Features like `impl Trait` make it relatively easy to avoid leaking this
  capability when it's not wanted.

Points for this argument:

- The `Clone` trait requires an existing instance of the function in order to be
  exploited. The fact that the `Default` trait gives this capability to types
  directly makes it sufficiently different from `Clone` to warrant a different
  decision.

These discussions also raise the question of whether the `Clone` trait itself
should be implemented automatically. It is convenient, but it leaves a very
grey area concerning which traits ought to be implemented for compiler-generated
types, and the most conservative option would be to require an opt-in for all
traits beyond the basic `Fn` traits (in the case of function types).

### Unnatural-ness of using `Default` trait

Several people objected on the grounds that `Default` was the wrong trait,
or that the resulting code seemed unnatural or confusing. This lead to
proposals involving other traits which will be described in their own
sections.

- Some people do not see `Default` as being equivalent to the
  default-constructible concept from C++, and instead see it as something
  more specialized.

  To avoid putting words in people's mouths I'll quote @Mark-Simulacrum
  directly:

  > I think the main reason I'm not a fan of adding a Default impl here is
  > because you (probably) would never actually use it really as a "default";
  > e.g. Vec::resize'ing with it is super unlikely. It's also not really a
  > Default but more just "the only value." Certainly the error message telling
  > me that Default is not implemented for &fn() {foo} is likely to be pretty
  > confusing since that does have a natural default too, like any pointer to
  > ZST). That's in some sense just more broadly true though.

- There were objections on the grounds that `Default` is not sufficient to
  guarantee _uniqueness_ of the function value. Code could be written today that
  exposes a public API with a `Default + Fn()` bound, expecting all types
  meeting that bound to have a single unique value.

  If we expanded the set of types which could implement `Default + Fn()` (such
  as by stabilizing `Fn` trait implementations or by making more function
  types implement `Default`) then the assumptions of such code would be
  broken.

  On the other hand, we really can't stop people from writing faulty code and
  this does not seem like a footgun people are going to accidentally use, in
  part because it's so obscure.

### New lang-item

This was a relatively minor consideration, but it is worth noting that this
solution would require making `Default` a lang item.

## Safe transmute

This proposal was to materialize the closure using the machinery being
added with the "safe transmute" RFC to transmute from the unit `()` type.

The details of how this would work in practice were not discussed in detail,
but there were some salient points:

- This solves the "uniqueness" problem, in that ZSTs are by definition unique.
- It does not help with the "privacy leakage" concerns.
- It opens up a new can of worms relating to the fact that ZST closure types
  may still have upvars.
- Several people expressed something along the lines of:

  > if we were going to have a trait that allows this, it might as well be
  > Default, because telling people "no, you need the special default" doesn't
  > really help anything.

  Or, that if it's possible to do this one way with safe code, it should be
  possible to do it in every way that makes sense.

## `Singleton` or `ZST` trait

New traits were proposed to avoid using `Default` to materialize the function
values. The considerations here are mostly the same as for the "safe
transmute" propsal. One note is that if we _were_ to add a `Singleton` trait,
it would probably make sense for that trait to inherit from the `Default`
trait anyway, and so a `Default` implementation now would be
backwards-compatible.

## `FnStatic` trait

This would be a new addition to the set of `Fn` traits which would allow
calling the function without any `self` argument at all. As the most
restrictive (for the callee) and least restrictive (for the caller) it
would sit at the bottom of the `Fn` trait hierarchy and inherit from `Fn`.

- Would be easy to understand for users already familiar with the `Fn` trait hierarchy.
- More unambiguously describes a closure with no upvars rather than one which is a ZST.
- Doesn't solve the problem of accidentally leaking capabilities.
- Does not force a decision on whether closures should implement `Default`.

This approach would also generalize the existing closure -> function pointer
conversion for closures which have no upvars. Instead of being special-cased
in the compiler, the conversion can apply to all types implementing `FnStatic`.
Furthermore, the conversion could be implemented by simply returning a pointer
to the `FnStatic::call_static` function, which makes this very elegant.

### Example

With this trait, we can implement `call_me_back_safely` from the prior example
like this:

```rust
use std::{panic, process};

pub fn call_me_back_safely<F: FnStatic()>(_f: F) {
    extern "C" fn catch_unwind_wrapper<F: FnStatic()>() {
        if panic::catch_unwind(F::call_static).is_err() {
            process::abort();
        }
    }
    unsafe {
        c_api::call_me_back(catch_unwind_wrapper::<F>);
    }
}
```

## Const-eval

Initially proposed by @scalexm, this solution uses the existing implicit
conversion from function types to function pointers, but in a const-eval
context:

```rust
fn add_one_adapter<const F: fn(i32)>(arg: i32) {
    F(arg + 1);
}

fn get_adapted_function_ptr<const F: fn(i32)>() -> fn(i32) {
    add_one_adapter::<F>
}
```

- Avoids many of the pitfalls with implementing `Default`.
- Requires giving up the original function type. There could be cases where
  you still need the original type but the conversion to function pointer
  is irreversible.
- It's not yet clear if const-evaluation will be extended to support this
  use-case.
- Const evaluation has its own complexities, and given that we already have
  unique function types, it seems like the original problem should be solvable
  using the tools we already have available.

## Opt-in trait implementations

This was barely touched on during the discussions, but one option would be to
have traits be opted-in via a `#[derive(...)]`-like attribute on functions and
closures.

- Gives a lot of power to the user.
- Quite verbose.
- Solves the problem of leaking capabilities.
