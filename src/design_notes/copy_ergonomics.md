# Copy type ergonomics

## Background

There are a number of pain points with `Copy` types that the lang team is
interested in exploring, though active experimentation is not currently ongoing.

Some key problems are:

## `Copy` cannot be implemented with non-`Copy` members

There are standard library types where the lack of a `Copy` impl is an
active pain point, e.g., [`MaybeUninit`](https://github.com/rust-lang/rust/issues/62835)
and [`UnsafeCell`](https://github.com/rust-lang/rust/issues/25053), when the
contained type is actually `Copy`.

### History

 * `unsafe impl Copy for T` which avoids the requirement that T is recursively
   Copy, but is obviously unsafe.
    * https://github.com/rust-lang/rust/issues/25053#issuecomment-218610508
 * `Copy` is dangerous on types like `UnsafeCell` where `&UnsafeCell<T>`
   otherwise would not permit access to `T` in [safe
   code](https://github.com/rust-lang/rust/issues/25053#issuecomment-98447164).

## `Copy` types can be (unintentionally) copied

Even if a type is Copy (e.g., `[u8; 1024]`) it may not be a good idea to make
use of that in practice, since copying large amounts of data is slow. This is
primarily a performance concern, so the problem is usually that these copies are
easy to miss. However, depending on the size of the buffer, it can also be a
correctness concern as it may cause an unintended stack overflow with too many
accidental copies.

Should we want to lint on this code, deciding on a size threshold may be
difficult. It's not generally possible for the compiler to know whether a
particular copy operation is likely to lead to stack overflow or undesirable
performance. We don't have examples yet of cases where there's desirable large
copies (that should not be linted against) or concrete cases where the copies
are accidental; collecting this information would be worthwhile.

Implementations of `Copy` on closures and arrays are the prime example of Rust
currently being overeager with the defaults in some contexts.

This also comes up with `Copy` impls on `Range`, which would generally be
desirable but is error-prone given the `Iterator/IntoIterator` impls on ranges.

The example here does not compile today (since Range is not Copy), but would be
unintuitive if it did.

```rust,compile_fail
let mut x = 0..10;
let mut c = move || x.next();
println!("{:?}", x.next()); // prints 0
println!("{:?}", c()); // prints 0, because the captured x is implicitly copied.
```

This example illustrates the range being copied into the closure, while the user
may have expected the name "x" to refer to the same range in both cases.

The move keyword here likely disambiguates this particular case for users, but
in closures with more captures it may be not as obvious that the range type in
particular was copied in.

A lint has been [proposed](https://github.com/rust-lang/rust/issues/45683) to
permit Copy impls on types where Copy is likely not desirable with particular
conditions (e.g., Copy of IntoIterator-implementing types after iteration).

Note that "large copies" comes up with moves as well (which are copies, just
taking ownership as well), so a size-based lint is plausibly desirable for both.

### History

* Proposed lint: [#45683](https://github.com/rust-lang/rust/issues/45683)

## References to `Copy` types

Frequently when dealing with code generic over T you end up needing things like
`[u8]::contains(&5)` which is ugly and annoying. Iterators of copy types also
produce `&&u64` and similar constructs which can produce unexpected type errors.

```rust
for x in &vec![1, 2, 3, 4, 5, 6, 7] {
    process(*x); // <-- annoying that we need `*x`
}

fn process(x: i32) { }
```

```rust
fn sum_even(v: &[u32]) -> u32 {
    // **v is annoying
    v.iter().filter(|v| **v % 2 == 0).sum()
}
```

Note that this means that you in most cases want to "boil down" to the inner
type when dealing with references, i.e., `&&u32` you actually want `u32`, not
`&u32`. Notably, though, this may *not* be true if the Copy type is something
more complex (e.g., a future Copy Cell), since then `&Cell` is quite different
from a `Cell`, the latter being likely useless for modification at least.

There is also plausibly performance left on the table with types like `&&u64`.

Note that this interacts with the unintentional copies (especially of large
structures).

This could plausibly be done with moved values as well, so long as the
semantics match the syntax (e.g. `wants_ref(foo)` acts like `wants_ref(&{foo})`)
similar to how one can pass `&mut` to something that only wants `&`.
This would be a tradeoff: in some cases people may want the type-checker to flag such cases and require explicitly taking a reference, while in other cases people may want the compiler to automatically make such code work. We would want to consider and evaluate this tradeoff, and whether we can usefully separate such cases.

### History

* [RFC 2111 (not merged)](https://github.com/rust-lang/rfcs/pull/2111)
* [Rust tracking issue (closed)](https://github.com/rust-lang/rust/issues/44763)
* "Allow owned values where references are expected" in [rust-roadmap-2017#17](https://github.com/rust-lang/rust-roadmap-2017/issues/17)
