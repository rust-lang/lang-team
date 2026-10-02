# Frequently Requested Changes

Some ideas for language proposals come up quite often. They're attractive ideas
for one reason or another, but ones that we're unlikely to add.

This page documents some of those ideas, along with the concerns that argue
against them.

If something appears on this page, that doesn't mean Rust would *never*
consider making any similar change. It does mean that any hypothetical
successful proposal to do so would need to address *at a minimum* all of these
known concerns, whether by proposing a *new and previously unseen* approach
that avoids all the concerns, or by making an *extraordinary* case for why
their proposal outweighs those concerns.

Hopeful proposers of any of these ideas should document their extensive
research into the many past discussions on these topics and ensure they have
something new to offer.

## An operator for `unwrap`

People writing code that makes extensive use of `.unwrap()` often ask for a
shorthand operator for it, typically something postfix involving `!`.

Rust already provides the `?` operator for propagating errors. `.unwrap()`
exists largely for quick-and-dirty code. We don't want to make it substantially
easier than it already is to write code using `.unwrap()`, and we definitely
don't want to add dedicated syntax for it.

## An option to disable the borrow checker, or bypass it in `unsafe` code

People learning Rust, especially those arriving from other languages, often
spend time "fighting the borrow checker". And experienced developers sometimes
want to write "clever" code that the borrow checker doesn't understand.

In the course of doing so, some developers request a compiler option to
"disable the borrow checker", or a way to bypass the borrow checker in `unsafe`
code blocks.

Rust already provides a means of bypassing the borrow checker: you can write
`unsafe` code that uses "raw pointers" (`*const T` or `*mut T`, rather than
`&T` or `&mut T`). Using raw pointers, you can manipulate memory in any way you
see fit, and Rust's borrow checker will do nothing to stop you; if you misuse
raw pointers, you'll get crashes or incorrect behavior at runtime. Many safe
Rust data structures and libraries are safe wrappers designed to encapsulate
some carefully written unsafe code.

You can also defer borrow checking to runtime, with types like `RefCell`, `Rc`,
and `Arc`. These types also allow "interior mutability": modifying a value
whose type doesn't look modifiable, when the value isn't "semantically" being
modified. (For instance, managing mutable internal bookkeeping for an otherwise
immutable value.)

However, even in an `unsafe` block, Rust's normal borrowed types (`&T` and
`&mut T`) still follow the same rules they do everywhere else. You can't have
two mutable references to the same object at once; you can't have a mutable
reference and an immutable reference at the same time; you can't use an object
after giving away ownership of it. Having distinct types for safe borrows and
unsafe raw pointers provides the flexibility of writing unsafe code while still
getting support from the compiler in places where your code can benefit from
such support.

## Fundamental changes to Rust syntax

This includes proposals such as changing the generic syntax to not use `<`/`>`,
changing block constructs to not require braces, changing function calls to not
require parentheses, and many other similar proposals. These also include
proposals to add "alternative" syntaxes, in addition to those that replace the
existing syntax. Many of these proposals come from people who also write other
languages. Arguments range from the ergonomic ("I don't want to type this") to
the aesthetic ("I don't like how this looks").

Changes that would break existing Rust code are non-starters. Even in an
edition, changes this fundamental remain extremely unlikely. The established
Rust community with knowledge of existing Rust syntax has a great deal of
value, and to be considered, a syntax change proposal would have to be not just
*better*, but *so wildly better* as to overcome the massive downside of
switching.

In addition, such changes often go against one or more other aspects of Rust's
design philosophy. For instance, we don't want to make changes that make code
easier to write but harder to read, or changes that make code more error-prone
to modify and maintain.

That said, *we are open to proposals that involve new syntax*, especially for
new features, or to improve an existing fundamental feature. The bar for new
syntax (e.g. new operators) is high, but not insurmountable. But the bar for
*changes to existing syntax* is even higher.

## Changes to avoid writing `self.method()` when calling a method from a method

In Rust, within a method of an object, calling another method requires writing
`self.othermethod()`, and accessing a field requires `self.field`. In some
other languages, such accesses can omit the equivalent of `self.`, and just
write `othermethod` or `field`, implicitly referencing the "current object".
People writing code with such method calls or field accesses sometimes ask for
such shorthand in Rust.

Rust prefers the explicitness of writing `self.` for method calls and field
accesses, and in most ways treating `self` as a normal object of the type. This
avoids unexpected calls to the wrong method, makes it easier to distinguish
methods and free functions, and makes code easier to read. We don't want to
provide a shorthand that makes this syntax briefer at the expense of this
clarity and unambiguity.

## Arbitrary custom operator syntax

Rust allows overloading existing operators; for instance, you can implement the
`Add` trait to overload the `+` operator. However, Rust does not allow creating
*new* operators.

Some would argue this can make code more readable, if you know what the
operators mean. If you don't, it makes the code inscrutable.

In general, such a change would substantially raise barriers to entry for Rust
developers, making Rust code less approachable and less universally
understandable. We're unlikely to add support for this.

Note that Rust's existing operator overloading uses semantic trait names (`impl
Add`), rather than symbols or names of symbols (`Plus` or `+`), which tends to
encourage using overloaded operators for the same semantic purposes, rather
than for building arbitrary domain-specific languages.

Rust developers seeking to build arbitrary domain-specific languages (DSLs)
should consider the macro system.

## Numeric overflow checking should be on by default even in release mode

Whenever possible, Rust tries to do the safe thing by default.

Numeric overflow checking (e.g. `1000u16 * 1000u16`) is one case where Rust
compromised on this: on many targets, numeric overflow checking has high enough
overhead to hurt performance too much for a wide variety of code. As a result,
Rust defaults to having overflow checking only for debug builds, while release
builds have overflow checking off by default. (In release builds, numeric
overflow wraps, but code cannot count on overflow checking being disabled even
in release builds, as projects can turn on overflow checking in release builds.
In addition, library code cannot make any assumptions about overflow checking,
as the top-level compilation decides whether to enable or disable it.)

We've thought about this choice many times, and we're open to considering
changes to this default based on benchmarks. If, on some Rust targets, overflow
checking adds fairly little overhead on the vast majority of crates, we'd
consider enabling it by default for those targets.

It would also help to have ways to detect excessive overhead caused by overflow
checking (e.g. detecting numeric-heavy code) and suggesting the use of
explicitly non-overflowing numeric types such as `Wrapping`.

## Cross-function type inference

Rust's type inference generally stops at function boundaries; Rust requires
specifying explicit types for function parameters, rather than allowing
inference to work across functions.

This is an intentional design choice: by making functions an inference
boundary, type errors become easier to debug and compartmentalize, and Rust
developers can reason about code using local reasoning within a function.

## Built-in / mandatory garbage collection

Adding any form of mandatory garbage collection built into the language would
mandate that all targets support it, which would require some kind of
"runtime". Rust gets great benefit from having no required "runtime". Rust can
go anywhere, including in systems contexts where relatively few languages can.

That said, we're happy to add language features to *support* an *optional*
garbage collector, where needed.

## Suffix modifiers (`if` after `return`/`break`/`continue`, or after arbitrary statements)

We often get proposals for syntax like `return expr if condition;` or
`break if condition;`. We don't plan to make such a change to Rust.

Such a change would prioritize *concise* code over *readable* code. We don't
want people to start out thinking they're reading an unconditional `return`
statement, and only later see the `if` and realize it's a conditional return.

Such a change would also have non-obvious evaluation order (evaluating the
condition before the return expression).

## Size != Stride

Rust assumes that the size of an object is equivalent to the stride of an object -
this means that the size of `[T; N]` is `N * std::mem::size_of::<T>`. Allowing
size to not equal stride  may allow objects that take up less space in arrays due
to the reuse of tail padding, and allow interop with other languages with this behavior.

One downside of this assumption is that types with alignment greater than their size can
waste large amounts of space due to padding. An overaligned struct such as the following:
```
#[repr(C, align(512))] 
struct Overaligned(u8);
```
will store only 1 byte of data, but will have 511 bytes of tail padding for a total size of
512 bytes. This tail padding will not be reusable, and adding `Overaligned` as a struct field
may exacerbate this waste as additional trailing padding be included after any other members.

Rust makes several guarantees that make supporting size != stride difficult in the general case.
The combination of `std::array::from_ref` and array indexing is a stable guarantee that a pointer
(or reference) to a type is convertible to a pointer to a 1-array of that type, and vice versa.

Such a change could also pose problems for existing unsafe code, which may assume that pointers
can be manually offset by the size of the type to access the next array element. Unsafe
code may also assume that overwriting trailing padding is allowed, which would conflict with
the repurposing of such padding for data storage.

While changing the fundamental layout guarantees seems unlikely, it may be reasonable to add additional
inspection APIs for code that wishes to opt into the possibility of copying smaller parts of an object
-- an API to find out that copying only bytes `0..1` of `Overaligned` is sufficient might still be
reasonable, or something `size_of_val`-like that could be variant-aware to say which bytes are sufficient
for copying a particular instance. Similarly, move-only fields may allow users to mitigate the effects
of tail or internal padding, as they can be reused due to the lack of a possible reference or pointer.

Cross-referencing to other discussions:

* https://github.com/rust-lang/rfcs/issues/1397
* https://github.com/rust-lang/rust/issues/17027
* https://github.com/rust-lang/unsafe-code-guidelines/issues/176

## Implicit Widening

Often there are requests to no longer need to manually perform lossless numeric conversions,
such as from `f32` to `f64`, from `u8` to `i32`, etc.

While that's convenient in trivial cases, it's not necessarily good in more complex cases.
If code was perfect then yes it'd be convenient, but Rust would rather ask which thing you
want between different options when the types don't match.

Take something like this:
```rust
let x: u16 = …;
let y: u16 = …;
takes_u32(x + y);
```

It would certainly be *possible* to have that just compile, as though you'd written
```rust
takes_u32((x+y).into())
```

But having an error there is a good opportunity to ask whether perhaps you wanted
```rust
takes_u32(u32::from(x) + u32::from(y))
```
so that it can't overflow in the addition.
