# Autoref/Autoderef in operators

Rust permits overriding most of the operators (e.g., `+`, `-`, `+=`). In part
due to `Copy` types not auto-dereferencing, it is common to have `T + &T`
or `&T + &T`, with potentially many levels of indirection on either side of the
operator.

There is desire in general to avoid needing to both add impls for referenced
versions of types because they:

- bloat documentation
- are never quite sufficient (always more references are possible)
- can cause inference regressions, as the compiler cannot in general know that
  `&T + &T` is essentially equivalent at runtime to `T + T`.

The inference regressions are the primary target of historical discussions.

The tradeoff to some feature like this may either mean that the exact impl
executed at runtime is harder to determine, or that the compiler is
synthetically generating new implementations for some subset of types,
potentially adding confusion around which impls are actually present.

However, generic code may want the impls on references because the generic code
may not want to require `T: Copy`. One version of this could involve
something like `default impl<T:Copy> Add<&T> for &T` so that people don't *need*
to write special impls themselves. It's worth noting that this still would need
some special support in the compiler to avoid the two possible impls leading to
inference regressions.

## History

The standard library initially had just the basic impls of the operator traits
(e.g., `impl Add<u64> for u64`) but has since gained `&u64 + &u64`, `u64 + &u64`
and `&u64 + u64`. These impls usually cause some amount of inference breakage in
practice.

Especially with non-Copy types (for example bigints), forcing users to add references can be
increasingly verbose: `&u * &(&(&u.square() + &(&a * &u)) + &one)`, for example.

There have also been a number of discussions on RFCs and issues, including:
- [Rust tracking issue #44762](https://github.com/rust-lang/rust/issues/44762)
    - This includes some implementation/mentoring notes.
- [RFC 2147](https://github.com/rust-lang/rfcs/pull/2147)
- [Trying to add T op= &T](https://github.com/rust-lang/rust/pull/41336)
    - Showcases dealing with inference breakage regressions when adding new
      reference-taking impls.
