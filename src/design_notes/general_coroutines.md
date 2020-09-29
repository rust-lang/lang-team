# Generalized coroutines

Since even before Rust 1.0, users have desired the ability to `yield` like in
other languages. The compiler infrastructure to achieve this, along with an
unstable syntax, have existed for a while now. But despite *a lot* of debate,
we've failed to polish the feature up enough to stabilize it. I've tried to
write up a summary of the different design considerations and the past debate
around them below:

## Terminology

- The distinction between a "coroutine" and a "generator" can be a bit vague,
  varying from one discussion to the next.
- In these notes a generator is anything which *directly* implements `Iterator`
  or `Stream` while a coroutine is anything which can take arbitrary input,
  yield arbitrary output, and later resume execution at the previous yield.
- Thus, the "generator" syntax proposed in [eRFC-2033][1] and currently
  implemented behind the "generator" feature is actually a coroutine syntax for
  the sake of these notes, *not a true generator*.
  - [RFC-2996][4] defines a true generator syntax in "future additions".
- Note also that "coroutines" here are really "semicoroutines" since they can
  only yield back to their caller.
- I will continue to group the [original eRFC text][1] and the later [generator
  resume arguments](https://github.com/rust-lang/rust/pull/68524) extension
  togther as "[eRFC-2033][1]". That way I only have 3 big proposals to deal
  with.
- In rustc, a coroutine's "witness" is the space where stack-allocated values
  are stored if needed across yields. I'm borrowing this terminology here. Any
  such cross-yield bindings are said to be "witnessed".

```rust
// This is an example coroutine which might assist a streaming base64 encoder
|sextet, octets| {
    let a = sextet; // witness a, b, and c sextets for later use
    yield;
    let b = sextet;
    octets.push(a << 2 | b >> 4); // aaaaaabb
    yield;
    let c = sextet;
    octets.push((b & 0b1111) << 4 | c >> 2); // bbbbcccc
    yield;
    octets.push((c & 0b11) << 6 | sextet) // ccdddddd
}

// This is an example generator which might be used in Iterator::flat_map.
gen {
  for item in inner {
    for mapped in func(item) {
      yield mapped;
    }
  }
}

// This is an example async generator which might be used in Stream::and_then
async gen {
  while let Some(item) = inner.next().await {
    yield func(item).await;
  }
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
    - May not be *required* at the trait level (someone may someday find a use
      to implementing `FnMut + !FnPin`) but all closures implement the traits in
      this order.

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
- Many people have a strong gut preference for yield expressions.
  - In simple cases, Rust generally prefers to produce values as output from
    expressions rather than by mutation of state. "Yield expressions *feel* more
    Rusty."
  - However, magic mutation is likely correct, even though at first glance it
    feels surprising. In addition to reasons below, holding references to past
    resume args is rare, often a logic error. Rust can use mutation checks to
    catch and give feedback.
- "Magic mutation" is a bit of a misnomer. The resume argument values are *not
  themselves being mutated.* The argument bindings are simply being reassigned
  across yields.
  - In a sense, argument bindings are reassigned in the exact same way across
  returns.
  - Previous arguments (if unmoved) are dropped prior to yielding and are
    reassigned after resuming.
  - People will get yelled at by the borrow checker if they try to hold borrows
    of arguments across yields. But the fix is generally easy: move the argument
    to a new binding before yielding.
```
=> |x| {
    let y = &x;
    yield;
    dbg!(y, x);
}

error[E0506]: cannot pass new `x` because it is borrowed
 --> src/lib.rs:3:4
  |
2 |     let y = &x;
  |             -- borrow of `x` occurs here
3 |     yield;
  |     ^^^^^ assignment to borrowed `x` occurs here
4 |     dbg!(y, x);
  |             - borrow later used here
  |
  = help: consider moving `x` into a new binding before borrowing

=> |x| {
    let a = x;
    let y = &a;
    yield;
    dbg!(y, x);
}
```
- Magic mutation could be replaced by "magic shadowing" where new arguments
  shadow old ones at yield in order to allow easy borrowing of past argument
  values. But this is a huge footgun. See if you can spot the issue with the
  following code if `ctx` shadows its past value rather than overwriting it:
```rust
std::future::from_fn(|ctx| {
  if is_blocked() {
    register_waker(ctx);
    yield Pending;
  }

  while let Pending = task.poll(ctx) { .. }
})
```
- "Yield expression" causes problems with first-resume input.
  - [eRFC-2033][1] passes the first resume argument via a closure parameters
    while later arguments are produced by `yield` expressions.
  - This part of why it is so hard to unify generalized coroutines with a
    generator syntax like `gen { }` or `gen fn`. Where does the first input go?
    Where do you annotate the argument type even?
- Users usually want to give resume arguments named bindings to increase
  clarity.
  - Magic mutation provides name-ability via pattern-matching in the closure
    parameters. Since unmoved arguments are dropped just prior to yielding, they
    do not need to be captured into closure state.
  - Yield expressions require users to repeatedly re-bind resume arguments to
    named bindings. Such bindings must be included in the closure state if they
    have any drop logic.

## Borrowed resume arguments

- What happens when a coroutine witnesses a borrow passed as a resume argument?
  For example:
```rust
let co = |x: &i32| {
  let mut values = Vec::new();
  loop {
    values.push(x);
    yield;
  }
};

// potentially ok:
let mut x = 0;
co(&x);

// must not be allowed:
x = 1;
co(&x);
```
- As of writing, [RFC-2781][2] leaves this as an unresolved question with a note
  to potentially restrict resume arguments to being `'static`.
- Since coroutines under [MCP-49][2] act as much like closures as possible, and
  treat the witness and capture data the same whenever possible, the example
  above would fail in a similar way to the example below, giving a "borrowed
  data escapes into closure state" error or similar even if `x` is not mutated.
```rust
let mut values = Vec::new();
|x: &i32| {
  loop {
    values.push(x);
//  ^^^^^^^^^^^^^^ `x` escapes the closure body here
    yield;
  }
}
```
- As of writing, [eRFC-2033][1] appears to take a similar approach (although the
  error message is not super descriptive).
- Ideally someday we'd do something nicer but any such solution would apply to both captured state and witnessed state in the same way.

## Lending

- Coroutines would eventually like to yield borrows of state to the caller. This
  is "lending" coroutine (sometimes also called an "attached" coroutine).
- Using [MCP-49][3], a lending coroutine might look like:
```rust
|| {
  let mut buffer = Vec::new();
  loop {
    let n = fill_buffer(&mut buffer);
    yield &buffer[..n];
  }
}
```
- None of the major proposals have made an effort to resolve this directly as
  far as I am aware.
  - [RFC-2996][4] gets the closest with a mention of `LendingStream` and
    `LendingIterator` traits in "future additions".
  - We should probably get some experience with lending traits at the lib level
    before attempting to add language level support.
- If lending closures were implemented, [MCP-49][3] could immediately be used to
  build lending streams, iterators, etc so long as the respective traits have
  the needed GAT-ification.

## Enum-wrapping

- [RFC-2781][2] and [eRFC-2033][1] propose that `yield x` should produce
  `GeneratorState::Yielded(x)` or equivalent as an output, in order to
  discriminate between yielded and returned values.
- [MCP-49][3] instead gives `yield x` and `return x` nearly identical semantics
  and output `x` directly, so the two must return the same type.
- Enum-wrapping here is analogous to Ok-wrapping elsewhere. Similar debates
  result.
- When using enum-wrapping, the syntax to specify distinct return/yield types is
  hotly debated.
- Generators always want return and yield to have different types (`()` vs `T`)
  but a generator syntax on top of coroutines could be used to auto-insert enum
  wrappers around yield vs return arguments.
- Auto-enum-wrapping can slightly improve type safety in some cases where
`return` should be treated specially to avoid bugs.
- No-enum-wrapping when combined with the `impl Fn*` choice of trait, allow
  the coroutine syntax to be used directly with existing higher-order methods
  on iterator, stream, collection types, async traits, etc.
- Note these two approaches are "isomorphic": a coroutine that returns
  `GeneratorState<T, T>` could be wrapped to return `T` by some sort of
  combinator and a coroutine that only returns `T` can have `yield` and `return`
  values manually wrapped in `GeneratorState`:
```rust
// Without enum wrapping:
std::iter::from_fn(|| {
  yield Some(1);
  yield Some(2);
  yield Some(3);
  None
}).map(|x| {
  yield -x;
  yield x;
});

// With enum wrapping:
std::iter::from_gen_fn(|| {
  yield 1;
  yield 2;
  yield 3;
}).map(unwrap_gen_state(|x| {
  yield -x;
  yield x;
}));

// Needed for un-enum-wrapping when not desired.
// Could be replaced by sufficiently fancy !-casting?
fn unwrap_gen_state<T>(f: impl FnMut() -> GeneratorState<T, !>) -> T { ... }
fn merge_gen_state<T>(f: impl FnMut() -> GeneratorState<T, T>) -> T { ... }

// With no wrapping + generators:
(gen {
  yield 1;
  yield 2;
  yield 3;
}).map(|x| {
  yield -x;
  yield x;
})
```

## Movability

- All proposals want movability/`impl Unpin` to be inferred.
  - Exact inference rules may only be revealed by an attempt at implementation.
- Soundness of `pin_mut!` is a little tricky but seems to be fine no matter what.
  - If the resulting mutable reference is live across a yield ⇒ coroutine is
    `!Unpin` because of inference rules
  - If the pinned data is `!Unpin` and dropped after a yield ⇒ coroutine is
    `!Unpin` because witness contains `!Unpin` data
  - Thus, if the coroutine can be moved after resume, any data stack-pinned
    (really witness-pinned) by `pin_mut!` is not referenced and is `Unpin`.
- Until inference is solved, the `static` keyword can be used as a modifier.

```rust
// movable via inference
|| {
  let x = 4;
  let y = &x;
  dbg!(y);
  yield;
}

// guaranteed movable (pending inference)
static || {
  ...
}

// immovable
|| {
  let x = 4;
  let y = &x;
  yield;
  dbg!(y);
}
```

## "Once" coroutines

- A lot of coroutines destroy captured data when run.
- These coroutines (notably futures) can be resumed many times but can only be
  run through "once".
- In contrast to non-yield `FnOnce` closures, this can not be solved at the type
  level because a coroutine can run out after an arbitrary, runtime-dependent
  number of resumptions.
  - Attempts to discriminate with enums tend to run up against `Pin`.
- Coroutines must have the ability to block restart with a `panic!`.
  - Following `return`.
  - Following `panic!` and recovery.
  - The term "poison state" technically refers to only the later case. But here
    I will use it to mean any state at which the closure panics if resumed.
- [RFC-2781][2] and [eRFC-2033][1] propose that all coroutines become poisoned
  after returning.
- [MCP-49][3] optionally proposes that capture-destroying closures should only
  implement `FnOnce` unless explicitly annotated, even if they should apparently
  be resumable several times.
  - `mut || { drop(capture); }` is recommended as the modifier, to hint that an
    `FnMut` impl is being requested when the closure in question would otherwise
    impl only `FnOnce`.
  -  But the behavior of this modifier is probably too obscure and requires
      too much explanation vs "closures always impl FnMut/FnPin if they contain
      yield".
- [MCP-49][3] also recommends that all non-capture-destroying coroutines resume
  at their initial state after returning.
  - This can be very handy in some situations. In fact, I use it several times
    in examples to increase readability. See anywhere I `iter.map(coroutine)` or
    the base64 encoder.
  - Similar question around generators: should they loop to save on a state or
    should they be fused-by-default?
  - If decided against: can be worked-around using `loop { .. }` as the
    coroutine body instead of just `{ .. }`.

## Async coroutines

- I am aware of no strong proposal for an async version of generalized
  coroutines although a fair amount of discussion has taken place.
  - In the context of [MCP-49][3], how should `async || { ... yield ...}` be
    handled in the very long-term? *Error right now.*
- Async coroutines don't make much sense because of resume arguments. Async
  functions are already coroutines which take an `&mut Context` as a resume
  argument. How should additional arguments should be specified?
  - Do the additional args need to be passed every single poll or are they only
    needed when resuming after `Ready`?
  - If they are stored between `Ready`s, how does that interact with the ban on
    witnessing external borrows? Badly.
  - On resume, the the coroutine might only take the additional arguments. It
    could then yield a future to take the async context and handle any `Pending`
    yields.
  - If so, how is the coroutine body broken up into distinct futures to be
    yielded?
  - What happens if a yielded future is destroyed early? Panic on resume?
- Generators and async are both sugars on top of coroutines and are orthogonal
  to each other. But neither is orthogonal to the underlying coroutine feature:
```rust
// an async block
async {
  "hello"
}

// an async generator
async gen {
  yield "hello";
}

// an "async coroutine"
|ctx: &mut Context| {
  yield Poll::Ready("hello");
}
```
- Taking the async context explicitly makes it cleaner to implement some complex
  async functions which take additional poll parameters.
  - An `await_with!` macro would be quite useful for implementing `await` loops
    on arbitrary `Poll`-returning functions.
    - Would be a good candidate for an `.await(args..)` syntax if very heavily
      used.
  - For example, an simple little checksumming async write wrapper might look
    like this:
```rust
|ctx: &mut Context, bytes: &[u8]| -> Poll<usize> {
  let mut checksum = 0;
  let mut count = 0;
  pin_mut!(writer);

  loop {
    let n = 4096 - count;
    if n == 0 {
      await_with!(writer.poll_write, ctx, &[checksum]);
    }

    let part = &bytes[..bytes.len().min(n)];
    checksum = part.fold(checksum, |x, &y| x ^ y);
    await_with!(writer.poll_write, ctx, part);

    count += part.len();
    yield Ready(part.len());
  }
}
```

## Try

- All proposals work fine with the `?` operator without even trying (haha).
- `Poll<Result<_, _>>` and `Poll<Option<Result<_, _>>>` already implement `Try`!
- Generators usually want a totally different `?` desugar that does `yield Some(Err(...)); return None;` instead of `return Err(...)`.
  - This comes up a lot in discussions of general coroutine syntaxes but just
    muddies things up because (say it with me) generators ≠ coroutines.
  - Sugar-free implementation is easy: `yield Some(try { ... }); None`
- Try blocks in general are super useful for handing errors by moving into
  specific error-handeling states.

## Language similarity

- Rust's version of coroutines can be a bit unusual compared to other languages.
  But the reason for this is simple: you need arguments to resume Rusty
  coroutines.
- Resume arguments in other languages can be passed just fine by sharing mutable
  data. So all they need to implement are generators, not true coroutines as
  defined here.

```python
# Generator function takes a word list and name on construction.
# The shared list is mutated to make room for new words.
def write_greeting(name, words):
    words.append('hello')
    if words.is_full:
        yield
    words.append(name)

```

```rust
// Function only needs name to construct coroutine.
// Coroutine gets mutable access to the word list each resume.
fn write_greeting(name: String) -> impl FnMut(&mut Vec<String>) {
  |words| {
    words.push("hello".to_string());
    if words.len() == words.capacity() {
      yield
    }
    words.push(name);
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
- In fact, `return` behaves exactly like a simultaneous `yield` + `break 'closure_body`.
  - In a sense, every closure already has a single yield point at which it
    resumes after `return`.
  - A `yield` adds a second resume point: hence the need for a discriminant.
- Under that proposal, anywhere a closure can be used, a coroutine can too. And
  vice versa.

## Generator unification

- So far in this proposal, I've been very careful to distinguish generators (as
  supported by the async_stream crate, the draft stream trait RFC, etc) from the
  coroutines discussed here. They are treated as two separate language features.
- Does Rust have "room" for both stream syntax and a generator syntax? Would it
  be better to find a single solution to both?
- A single solution is difficult for a few reasons:
  - Taking resume arguments muddies the syntax. For example, what would be the
    syntax for a generator function which takes an explicit resume argument?
  - The closure syntax works great for coroutines which implement `Fn*` a la
    [MCP-49][3]! But reusing that syntax to magically implement `Iterator` or
    `Stream` would cause confusion.
  - On that note, generators *definitely* want to implement different traits vs
    coroutines. `Iterator` and `Stream` rather than `Fn` or (ironically)
    `Generator`.
  - As stated above, async coroutines don't make much sense: async interacts
    poorly with resume arguments.
  - Async generators are super important, don't care about resume arguments.
  - As mentioned in the section on try, generators and coroutines generally want
    different error handling. Or at lest, some more complex `?` desugar is not
    so obvious for coroutines in general as it is for generators specifically.
- Once generalized coroutines are in place, a generator syntax like the one in
  [RFC-2996][4] is a trivial sugar on top:
```rust
gen {
  for item in inner {
    for mapped in func(item) {
      yield mapped;
    }
  }
}

// becomes

std::iter::from_fn(|| {
  for item in inner {
    for mapped in func(item) {
      yield Some(mapped);
    }
  }
  None
})
```
```rust
async gen {
  while let Some(item) = inner.next().await {
    yield func(item).await;
  }
}

// becomes

std::stream::from_fn(|ctx| {
  while let Some(item) = await_with!(inner.next(), ctx) {
    yield Ready(Some(await_with!(func(item), ctx)));
  }
  Ready(None)
})
```
- Proc-macro crates could provide very satisfactory `gen` and `gen_async` macros
  until we are sure of the need to support such a sugar directly in language as
  a keyword or in core as a first-party macro.

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
[4]: https://github.com/rust-lang/rfcs/pull/2996/
