# Generalized coroutines

Since even before Rust 1.0, users have desired the ability to `yield` like in
other languages. The compiler infrastructure to achieve this, along with an
unstable syntax, have existed for a while now. But despite *a lot* of debate,
we've failed to polish the feature up enough to stabilize it. I've tried to
write up a summary of the different design considerations and the past debate
around them below:

## Coroutines vs Generators

- The distinction between a "coroutine" and a "generator" can be a bit vague,
  varying from one discussion to the next.
- In these notes a generator is anything which *directly* implements `Iterator`
  or `Stream` while a coroutine is anything which can take arbitrary input,
  yield arbitrary output, and later resume execution at the previous yield.
- Thus, the "generator" syntax proposed in [eRFC-2033][1] and currently
  implemented behind the "generator" feature is actually a coroutine syntax for
  the sake of these notes, *not a true generator*.
- Generators are a specialization of coroutines.
- I will continue to group the [original eRFC text][1] and the later [generator
  resume arguments](https://github.com/rust-lang/rust/pull/68524) extension
  togther as "[eRFC-2033][1]". That way I only have 3 big proposals to deal
  with.

```rust
// This is a coroutine
|stuff| {
  yield stuff + 1;
  yield stuff + 2;
  return stuff + 3;
}

// This is a generator
stream! {
  yield 1;
  yield 2;
  yield 3;
}
```

## Coroutine trait

- The coroutine syntax must produce implementations of some trait.
- [RFC-2781][2] and [eRFC-2033][1] propose the `Generator` trait.
- Note that Rust's coroutines and subroutines look the same from the outside:
  take input, mutate state, produce output.
- Thus, [MCP-49][3] proposes using the `Fn*` traits instead, including a new
  `FnPin` for immovable coroutines.
  - Hierarchy: `Fn` is `FnMut + Unpin` is `FnPin` is `FnOnce`.

## Coroutine syntax

- The closure syntax is reused for coroutines by [eRFC-2033][1], [RFC-2781][2],
  and [MCP-49][3].
- Commentators have suggested that the differences between coroutines and
  closures under [eRFC-2033][1] and [RFC-2781][2] justify an entirely distinct
  syntax to reduce confusion.
- [MCP-49][3] fully reuses the *semantics* of closures, greatly simplifying the
  design space and making the shared syntax obvious.

## Taking input

- The major disagreement between past proposals is whether to use "yield
  expressions" or "magic mutation".
  - Yield expression: `let after = yield output;`
    - Used by [eRFC-2033][1]
  - Magic mutation: `let before = arg; yield output; let after = arg;`
    - Used by [RFC-2781][2] and [MCP-49][3]
- Many people have a strong gut preference for yield expressions but they tend
  to fall apart, grow in complexity, and become ever more surprising when you
  look into them further.
  - I think it might be a bit like the `.await` bikeshed? People didn't *feel*
    like `.await` was correct but it turned out to be well-justified.
- "Magic mutation" is a bit of a misnomer. The resume arguments are *not being
  mutated.* The argument bindings are simply being reassigned across yields.
  - In a sense, argument bindings are reassigned in the exact same way across
  returns.
- "Yield expression" causes problems with first-resume input.
  - [eRFC-2033][1] passes the first resume argument via a closure parameters
    while later arguments are produced by `yield` expressions.
  - This does not play nicely with alternate coroutine syntaxes like `coroutine
    { }`.
- "Yield expression" moves resume arguments into the generator state.
  - `!Copy` arguments bloat the witness size.
  - By comparison, "magic mutation" passes arguments in the same way as any
    normal function. The arguments can then be moved into the state manually
    when needed.
  - In practice, coroutines rarely care about old inputs anyway. Futures are
    strictly forbidden from using old contexts, sinks tend to fully process each
    item after moving onto the next, parsers process each token as much as possible before continuing, etc. Usually people yield because they are
    finished with whatever they already have and need something new.
- "Magic mutation" allows coroutines (like subroutines) to receive multiple
  resume arguments naturally rather than requiring users to use tuples.

## Enum-wrapping

- [RFC-2781][2] and [eRFC-2033][1] propose that `yield x` should produce
  `GeneratorState::Yielded(x)` or equivalent as an output, in order to
  discriminate between yielded and returned values.
- [MCP-49][3] instead gives `yield x` and `return x` nearly identical semantics
  and output `x` directly, so the two must return the same type.
- Note these two approaches are "isomorphic": a coroutine that returns
  `GeneratorState<T, T>` could be wrapped to return `T` by some sort of
  combinator and a coroutine that only returns `T` can have `yield` and `return`
  values manually wrapped in `GeneratorState`.
- Enum-wrapping here is analogous to Ok-wrapping elsewhere. Similar debates
  result.
- When using enum-wrapping, the syntax to specify distinct return/yield types is
  hotly debated.
- The difference is strictly one of ergonomics.
  - Keep in mind any un-ergonomic cases can potentially be tackled with more
    specialized, macro-powered syntaxes.
  - Auto-enum-wrapping can slightly improve type safety in some cases where
    `return` should be treated specially to avoid bugs.
  - No-enum-wrapping when combined with the `impl Fn*` choice of trait, allow
    the coroutine syntax to be used directly with existing higher-order methods
    on iterator, stream, collection types, async traits, etc.


## Movability

- All proposals want movability/`impl Unpin` to be inferred.
  - Exact inference rules may only be revealed by an attempt at implementation.
- Unpin inference rules have an impact on the soundness of `pin_mut!`
  expansions:
  - Stack-pinning (really witness-pinning) may be able to force coroutine
    `!Unpin` by creating a mutable reference to closure state, depending on
    rules.
    - It must often force `!Unpin` even in cases where the mutable reference
      is short-lived: the stack-pinning must last until drop.
  - Could instead place a `PhantomPinned` onto the pseudo-stack, to be included
    in the coroutine witness and force `!Unpin`.
- Until inference is solved, the `static` keyword can be used as a modifier.

```rust
|| {
  pin_mut!(stuff);
  yield;
} // stuff drops here. It better not have moved!
```

## "Once" coroutines

- A lot of coroutines consume captured data.
- These coroutines (notably futures) can be resumed many times but can only be restarted "once".
- In contrast to non-yield `FnOnce` closures, this can not be solved at the type
  level because a coroutine can run out after an arbitrary, runtime-dependent
  number of resumptions.
  - Attempts to discriminate with enums tend to run up against `Pin`.
- Coroutines must have the ability to block restart with a `panic!`.
  - Following `return`.
  - Following `panic!` and recovery.
  - The term "poison state" technically refers to only the later case. But here
    I will use it to mean any state from which the closure can not be resumed.
- [RFC-2781][2] and [eRFC-2033][1] propose that all coroutines become poisoned
  after returning. Always.
- [MCP-49][3] optionally proposes instead that closures can only be poisoned if
  explicitly annotated, and otherwise loop around after return, yield-or-no.
  - The looping semantics can be very handy in a few situations.
  - But the behavior of the `mut` modifier may be too obscure and require too
    much explanation vs "closures poison if they contain yield".

## Async coroutines

- *Async generators must totally be a thing.* But any specialized generator
  syntax is a distinct discussion which needs to solve totally different
  problems.
- At this point async coroutines are a bit nonsensical. I am aware of no strong
  proposal for combining the two syntaxes although a fair amount of discussion
  has taken place.
  - In the context of [MCP-49][3], how should `async || { ... yield ...}` be
    handled in the very long-term? *Error right now.*
- Coroutines can be async most easily by taking an explicit `&mut Context`
  resume argument and explicitly producing some `Poll` state on yield and
  return.
  - Such implementations regularly want a sort of `await_with!` abstraction that
    can be used directly with poll functions like `AsyncRead::poll_read` or
    `Stream::poll_next` without a wrapping `Future`.
  - In the very long-term this could be a first-class syntax like
    `.await(context, buffer)`. *But not right now.*

## Try

- All proposals work with the `?` operator fine without even trying (haha).
- `Poll<Result<_, _>>` and `Poll<Option<Result<_, _>>>` already implement `Try`!
- Generators usually want a totally different `?` desugar that does `yield Some(Err(...)); return None;` instead of `return Err(...)`.
  - This comes up a lot in discussions of general coroutine syntaxes but just
    muddies things up because (say it with me) generators ≠ coroutines.
  - Sugar-free implementation is easy: `yield Some(try { ... }); None`

## Language similarity

- Rust's version of coroutines can be a bit unusual compared to other languages.
  But the reason for this is simple: you don't need arguments to *construct* a Rusty coroutine, only to resume it.
  - This is similar to async functions in other languages vs Rust's async
    blocks.

```python
# Coroutine takes an argument for construction
def square_numbers(n):
    while True:
        yield n * n
        n += 1
```

```rust
// Ordinary function takes an argument for construction. Coroutine is below.
fn square_numbers(mut n: i32) -> impl FnMut() -> i32 {
  // Capture construction args. No need to call anything!
  || loop {
    yield n * n;
    n += 1;
  }
}

```

## Language complexity

- The main selling point of [MCP-49][3] is that it avoids adding a whole new language feature with associated design questions. Instead, the common answer to questions regarding MCP-49 is that yield-closures simply do whatever closures do.
  - The syntax is the same.
  - Captures work the same way.
  - Arguments are passed the same way.
  - Return and yield both drop the latest arguments and then pop the call stack.
  - The only big difference is that once `yield` is involved, some variables get
    stored in a witness struct rather than in the stack frame. Plus the need for
    a poison state.

## Past discussions

There are a *lot* of these. Dozens of internals threads, reddit posts, blog
posts, draft RFCs, pre RFCs, actual RFCs, who knows what in Zulip, and so on.
So this isn't remotely exhaustive:
- https://github.com/CAD97/rust-rfcs/pull/1
- https://github.com/rust-lang/lang-team/issues/49
- https://github.com/rust-lang/rfcs/pull/2033
- https://github.com/rust-lang/rfcs/pull/2781
- https://github.com/rust-lang/rust/issues/43122
- https://github.com/rust-lang/rust/pull/68524
- https://internals.rust-lang.org/t/crazy-idea-coroutine-closures/1576
- https://internals.rust-lang.org/t/no-return-for-generators/11138
- https://internals.rust-lang.org/t/syntax-for-generators-with-resume-arguments/11456
- https://internals.rust-lang.org/t/trait-generator-vs-trait-fnpin/10411
- https://reddit.com/r/rust/comments/dvd3az/generalizing_coroutines/
- https://samsartor.com/coroutines-2
- https://smallcultfollowing.com/babysteps/blog/2020/03/10/async-interview-7-withoutboats/#async-fn-are-implemented-using-a-more-general-generator-mechanism
- https://users.rust-lang.org/t/coroutines-and-rust/9058

[1]: https://github.com/rust-lang/rfcs/pull/2033
[2]: https://github.com/rust-lang/rfcs/pull/2781
[3]: https://github.com/rust-lang/lang-team/issues/49
