# Impl Trait Overview

This meeting is meant to give a brief overview of the "in flight" areas around impl Trait. While writing this doc, we realized that it makes sense to focus most of the discussion on the question of how to name the return type for functions that return impl trait, and that will be the main focus of the doc. But before we go there we give a *very* brief overview of the other in-flight developments.

There are two other major "in flight" things:

* Revised inference algorithm:
    * oli-obk is working on landing an overhauled impl trait inference algorithm. The intent is to better match the RFCs, but also to reconcile some differences between the "impl trait in let bindings" and the "type alias impl trait" RFCs.
    * [Details available here;](https://rust-lang.github.io/impl-trait-initiative/explainer/inference.html) there is also an "Appendix" to this document that covers a few key examples.
* [Return position impl trait in traits](#Return-position-impl-trait-in-trait-RPITIT) (RPITIT)
    * Basically: desugars to an anonymous associated type.
    * Basis for [static async fn in traits RFC #3185](https://github.com/rust-lang/rfcs/pull/3185).
    * Spastorino is currently implementing this.
    * RFC is open with no major problems identified; there is one "open question":
        * Should impls be forced to use `impl Trait`, or should they be able to be more precise, as they would with other associated types?
    * Current plan is to move this to an "unresolved question" and to have it be the focus for an entire design meeting.

## Named function types

The primary shortcoming of [return-position impl trait (RPIT)][rpit] is that, when you use it, there is no way to name the return type of your function from elsewhere. This limits the use of RPIT in a number of situations and it leads to coding guidelines that recommend against using RPIT and instead introducing named types. Some cases where it is useful to name the return type from a RPIT function:

* Storing return value into a struct, esp. when building a compound future or iterator
* Bounding the return type, such as to say that the future returned by an `async fn` is `Send`
* Debugging type inference errors

A number of approaches have been proposed to permit naming the return type from an RPIT function, but at this point we have narrowed things down to two main competitors:

* *Approach 1:* Give a name `N` to fndef types (the zero-sized types that represent a specific function) and introduce a shorthand `N()` that accesses `<N as FnOnce>::Output`
* *Approach 2:* Introduce `typeof` to permit computing the type of expressions without evaluating them.

It's worth pointing out that the two approaches are not contradictory; `typeof` for example has some other use cases that might make it a valuable addition regardless.

## Running examples

To make this more specific, here are two examples that we will use to illustrate the options and how they might work. We'll use `async fn` as they represent a kind of 'worst case' that captures all lifetimes by default.

### Example A: Compound struct

```rust
impl Widget {
    async fn frob(&self) {}
}
```

We would like to construct a `Thunk<'a>` that stores the future resulting from invoking `Widget::frob` with a `&'a Widget`.

### Example B: Send bound

```rust
trait Log {
    async fn log(&self, s: &str);
}

fn spawn_logger<L: Log + Send>(logger: L) {
    tokio::spawn(async move {
        for event in load_events() {
            logger.log(event).await;
        } 
    });
}
```

For this function to work, we need to know that the future returned by `logger.log(event)` is `Send`.

## Approach 1: Give names to function types

The core of the design is to expose a way to name the "fndef type" (zero-sized function type) for a function, wherever it appears. For free functions, this is done via a new name `fn#foo`:

```rust
fn foo() {
}

fn main() {
    let f: fn#foo = foo;
    //     ^^^^^^ this type could not have been written before
}
```

In traits and inherent impls, each associated function `foo` is exposed as an associated type named `fn#foo`, as well as a type named `foo`, presuming there is no other associated type named `foo`:

```rust
trait SomeTrait {
    fn foo(&self) { }
}

impl SomeTrait for u32 { }

fn main() {
    let f: <u32 as SomeTrait>::foo = <u32 as SomeTrait>::foo;
    //     ^^^^^^^^^^^^^^^^^^^^^^^ not a type that could've been written before
}
```

The primary use for these types is to reference their `Output` associated type, but giving names to these types has some advantages:

* Explicit syntax helps to explain how things work by allowing one to give concrete type annotations and by giving a name for the concepts required.
* When printing diagnostics, there is an actual syntax to use (instead of the compiler's current approach of writing `fn() {foo}`).

### Shorthands

Given fn types, one can access the return type from a particular call via the `Output` associated type:

```rust
<fn#foo as FnOnce<(A1, A2, A3)>>::Output
```

However, this construction has two shortcomings. First, it is not stable, because we wish to keep room to adopt variadic generics later, so we have not hard-coded the tuple in `FnOnce<>`. (Using the `FnOnce(A1, A2, A3)` shorthand means specifying the value of the `Output` associated type, and if we could do that we wouldn't need this feature!)

Second, it is long and cumbersome. To make things worse, as we will see below, one often wishes to write higher-ranked constraints, and that can make this notation even harder to follow.

We therefore introduce a shorthand: the type `T(A1...An)` means `<T as FnOnce<(A1...An)>>::Output`:

```rust
fn foo() {
}

fn main() {
    let f: fn#foo() = foo();
    //     ^^^^^^^^ return type of the `foo` function
    //              Equivalent to: `<fn#foo() as FnOnce<()>>::Output`, which in turn is
    //              Equivalent to: `()`
}
```

Moreover, in the context of a where-clause (or other bound), one can use elided lifetimes like `T(&u32)` or even elide all argument types like `T(..)`. The result is a maximally higher-ranked where clause. Consider this trait and impl:

```rust
trait SomeTrait {
    fn foo(&self) -> impl Debug + '_;
}
```

One could write a generic function like so, which requires that `foo` return a `Send` value, no matter what lifetime is used for `self`: 

```rust
fn foo<T: SomeTrait>()
where
    T::foo(&T): Send
    // Equivalent to: `for<'a> <T::foo as FnOnce<(&'a T,)>>::Output: Send`
{
}    
```

Even more conveniently, one could use `..` to elide all argument types:

```rust
fn foo<T: SomeTrait>()
where
    T::foo(..): Send
    // Equivalent to: `for<'a> <T::foo as FnOnce<(&'a T,)>>::Output: Send`
{
}    
```

These shorthand forms are not currently permitted outside of where clauses. See the FAQ for more discussion.

### Examples using Approach 1

#### Example A: Compound struct.

```rust
impl Widget {
    async fn frob(&self) {}
}

struct Thunk<'a> {
    future: Widget::frob(&'a Widget),
}
```

Here, the `Widget::frob(&'a Widget)` is shorthand for:

```rust
<Widget::frob as FnOnce(&'a Widget)>::Output,
```

#### Example B: Send bound.

```rust
trait Log {
    async fn log(&self, s: &str);
}

fn spawn_logger<L: Log + Send>(logger: L)
where
    L::log(..): Send, // <-- added this!
{
    tokio::spawn(async move {
        for event in load_events() {
            logger.log(event).await;
        } 
    });
}
```

Here, `L::log(..): Send` is shorthand for:

```rust
for<'a, 'b> <<L as Log>::log as FnOnce<(&'a L, &'b str)>>::Output: Send
```

## Approach 2: `typeof`

Another idea that has been proposed is to introduce a `typeof` operator. This would take an expression, compute its return value, and use that as the type. Using `typeof`, one could write the following to "name" (indirectly) the `foo` type:

```rust
fn foo() {
}

fn main() {
    let f: typeof(foo) = foo;
}
```

In principle, this can offer some of the benefits for diagnostics (the compiler could print `typeof(foo)`, for example), but it's quite a bit more complex to figure out how to generate such a type.

### Examples

In order to use `typeof` to generate return values for functions that have late-bound parameters, we need to be able to generate a call to that function. This sometimes requires synthesizing values of arbitrary types. One can do this via something like this `value` function, which could be in the stdlib:

```rust
fn value<T>() -> T {
    unimplemented!()
}
```

#### Example A: Compound struct.

```rust
impl Widget {
    async fn frob(&self) {}
}

struct Thunk<'a> {
    future: typeof(Widget::frob(value::<&'a Widget>()))
}
```

#### Example B: Send bound.

```rust
trait Log {
    async fn log(&self, s: &str);
}

fn spawn_logger<L: Log + Send>(logger: L)
where
    for<'a, 'b> typeof(L::log(value::<&'a Widget>(), value::<&'b str>())): Send
{
    tokio::spawn(async move {
        for event in load_events() {
            logger.log(event).await;
        } 
    });
}
```

## Frequently asked questions

### Why not permit the `T(..)` shorthand outside of where clauses?

One imagine permitting the `T(..)` shorthand outside of a where clause, such as in this example:

```rust
fn parse(&str) -> impl Display {}

fn main() {
    let t: fn#parse(..) = parse("hi");
}
```

The challenge here is that one cannot easily determine what to do with the late-bound lifetimes involved. One option is that, in this context, `fn#parse(..)` requires that the return type be independent of all late-bound lifetimes or other arguments -- or even that there aren't any. But it is not clear whether, in the future, we might have other useful meanings. In short, it's easy to add this later, and hard to take it away, so we opted to hold off.

### Wasn't there something about making APIT late-bound in the RFC draft?

Yes, there was! The [RFC draft] also attempts to address the long-standing question of how turbofish and [argument position impl trait][apit] interact. We have opted to remove that from this document and we plan to separate that question into its own RFC. At first, it seemed inextricably linked, but we now believe that is not the case, and that we can keep the two questions separated by imposing the same restrictions on providing explicit type parameters to named function types that we current impose for turbofish:

[RFC Draft]: https://rust-lang.github.io/impl-trait-initiative/RFCs/rpit-in-traits.html
[apit]: https://rust-lang.github.io/impl-trait-initiative/explainer/apit.html
[rpit]: https://rust-lang.github.io/impl-trait-initiative/explainer/rpit.html

* You cannot write explicit type arguments to a named fn type if it uses APIT.
* You cannot write explicit lifetime parameters if there is a mix of early- and late-bound regions.

The plan is definitely to revisit these questions in future impl trait follow-up work. In particular it might be useful to give serious thought to whether we can unify or cleanup the situation with early- and late-bound generics more generally, and also to wait for chalk-related work to progress so that we gain the ability to express higher-ranked where clauses that are generic over types, `for<U: Debug> { u32: Blah<U> }`.

### Why not offer any shorthands for the `typeof` syntax?

You might have noticed that the named fn type proposal got some sweet syntactic sugar that made it a lot more usable, but the named fn type proposal did not. It would certainly be possible to introduce some kind of shorthand that could be used in `typeof` expressions to generate a value of suitable type. For example, we might permit `typeof(Widget::frob(_ as &'a Widget))` or `typeof(L::Log(_, _)): Send`. But we weren't sure what syntax to use, and there is the challenge is that these shorthands become part of the grammar for expressions, which is a fairly dense space (and that same syntactic space might be useful for other things).

---

## Appendix A: Revised inference algorithm

[Description](https://rust-lang.github.io/impl-trait-initiative/explainer/inference.html)

oli-obk is working on landing an overhauled impl trait inference algorithm. The intent is to better match the RFCs, but also to reconcile some differences between the "impl trait in let bindings" and the "type alias impl trait" RFCs.


The description above describes the algorithm in a way that is meant to be "reasonably precise" without giving every detail by enumerating various key rules:

### Key changes, illustrated by way of examples

#### Defining uses can occur anywhere

The TAIT implementation before the refactoring only allows a use of a TAIT to be a "defining use" when it occurs in a return type or another rather limited set of positions. The following example does not compile without the refactoring, although the TAIT RFC suggested it should:

```rust
type Foo = impl Send;

fn constrain_in_let() {
    // OK, `f` has type `Foo`, and we require that `Foo = i32`
    let f: Foo = 22_i32;
}
```

It will compile after the refactoring lands.

#### Opaque types remain opaque, even within the defining scope

Within the defining scope for an opaque type, it is now significant if the user chose to write the opaque type or not. If the type of a value is an opaque type, then uses of that value are always limited to the bounds in the type, regardless of whether the use occurs within the defining scope. In other words, the following two functions are not equivalent:

```rust
type Foo = impl Send;

fn as_opaque() -> Foo {
    // OK, `f` has type `Foo`, and we require that `Foo = i32`
    let f: Foo = 22_i32;
    
    // ERROR: Cannot add `Foo` and `i32`
    f + 1;
    
    // OK: Can return `Foo` as type `Foo`
    f
}

fn as_revealed() -> Foo {
    // OK, `f` has type `i32`.
    let f: i32 = 22_i32;
    
    // OK: Can add `i32` and `i32`
    f + 1;
    
    // OK: return `i32` as type `Foo`, requires `Foo = i32`
    f
}
```

This behavior matches the behavior specified by the let bindings RFC. As a result, although the implementation of that RFC has been removed owing to bugs, it will be relatively easy to re-add and stabilize:

```rust
fn foo() {
    let mut f: impl Send = 22_i32;
    
    // ERROR
    x + 1; 
}
```


#### Can dispatch methods on opaque types even before they are known

As another positive side-effect of the above rule, the following examples from the TAIT RFC will now work as they were meant to:

```rust
type Foo = impl Clone;

// Does not introduce a constraint on `f`
fn no_constraint(f: Foo) -> Foo {
    let g = f.clone();
    g
}
```

### Future work

Some things don't work as expected but we don't believe there is a backwards compatibility hazard.

#### Auto-trait leakage within the defining scope

We currently do not support auto-trait leakage within the defining scope. We believe we can fix this up later. In the interim, it is possible to workaround by using the hidden type within a fn body.

```rust
type Foo = impl Clone;

fn is_send<T: Send>() {}

fn test() {
    let x: Foo = 22_u32;
    is_send::<Foo>();
}
```


# Questions and notes from the meeting

## How do I add a question?

Please add your questions using a `##` header (much like this sample). That way we can take notes on the discussion. Feel free to add a summary of the question and then the details below.

## Explain what the heck is happening here :) 

For example A, using the explicit syntax

```rust
impl Widget {
    async fn frob(&self) {}
}

struct Thunk<'a> {
    future: Widget::fn#frob(&'a Widget),
}
```

```rust
impl Widget {
    // async fn frob(&self) {}
    fn frob(&self) -> impl Future<Output = ()> + '_ {}
}

impl<'a> FnOnce<(&'a Widget)> for Widget::fn#frob {
    type Output = impl Future<Output = ()> + 'a;
}
```

## Mark - Q: Why not rely on `type Foo = impl Bar` and then `fn foo() -> Foo` as the definer?

* In some sense, why are named (but still inferred) types bad/limiting?
* Is it "because" we don't actually write the type `async fn` returns?

niko: short version, it's a much higher bar to add the bound, if I have to go back and rewrite the trait.

simulacrum: it feels like the problem being solved is how do we make it ergonomic to access the return type. To some extent, we are giving the low-level tools. 

## Mark - Is this a backwards compatible change to make?

```rust
impl Widget {
	async fn frob(&self) { ... }
	// ->
	type Frob<'a> = impl Future + 'a;
	fn frob(&self) -> Frob<'_> {
		// Somehow make it possible to use .await here, e.g., with a macro which uses a shim async fn inside.
	}
}
```

cramertj: right, my old design for this was adding assoc type inference. But I agree that the syntax you wind up with as a result of that is sort of cumbersome, but you want something to make it nicer, so that you don't have to name.

nikomatsakis: I see two problems, first, that you the set of parameters is really complex, but second you may not control the definition.

cramertj: my proposal was more like we desugar for you and you get the CamelCase name Frob or whatever.

mark: I'm not sure it's so bad to say "if I'm writing a library, I have to be explicit".

cramertj: My conclusion is that you will have a rash of people saying "never write async fn if you're writing a library", and I wasn't sure how I felt about that. People are going to constantly be receiving PRs making their code less ergonomic.

mark: you can also do this downstream, right? you can write a simple fn that calls it, right?

```rust
type YourFuture = impl Future + 'a
async fn _wrapper<T: Widget>() -> YourFuture {
    T::future()
}
```

cramertj: you'll be using in bounds for generic functions, right? You could list the bound as "my wrapper" with "your type as a type parameter" implements Send. Not saying you couldn't do it, but as a user of a library, it would always be better if they had not used the async fn syntax, right? I don't like that world.

mark: it feels like the leap from "async fn hides the return type" to "let's add a bunch of stuff for pulling the return type out of the function" feels too large. Maybe something like introducing an associated type is better.

niko: That's where I started, but I moved. Still, it wouldn't have to be camel-cased, and it's maybe better if it's not because that's problematic for other languages etc, but the thing I'm worried about is how many parameters there will be.

mark: The current proposal throws in some other syntax niceties to make that associated type "fancier". I guess my question is, do we only want that for async fn desugaring, or will we discover that we want it in all kinds of places?

cramertj: are you talking about other instances of RPIT?

mark: Not necessarily return position. If you can write an async fn that has this problem, you can probably get it in many other cases?

niko: this is not specific to async fn, but it is specific to return types of fns.

cramertj: we don't have a way to name things that aren't the top-level return type in the first syntax. e.g. if you return a tuple, you can't talk about the type of just one member.

tmandry: you should be able to use `..` more places.

nikomatsakis: I've been thinking about what `'_` should means in where clauses, I think a `for` bound is the only reasonable thing:

```rust
fn foo()
where
    T: PartialEq<&u32>

// this interpretation, adding a new input parameter,
// doesn't really make sense b/c there's nothing to infer
// it from
fn foo<'a>()
where
    T: PartialEq<&'a u32>

// this interpretation seems right to me -- nikomatsakis
fn foo()
where
    for<'a> T: PartialEq<&'a u32>
```

## cramertj: My old proposal for solving this, for comparison

I had previously suggested we solve this by introducing associated type inference rather than using new syntax, e.g.:

```rust
impl Iterator for Foo {
    fn next(&mut self) -> Option<u32> {
        ...
    }
}
```

For `async fn` in traits + RPIT in traits to be nameable, we'd also need a consistent scheme for desugaring those to associated types.





## pnkfelix: bikeshed `fn#` vs e.g. `type#`

I found `fn#` did not match my intuition; I would expect a notation like `fn#` to mean "we're looking in the `fn` namespace", while I *think* the semantics here is meant to mean "we are looking in th namespace of types". So maybe `type#foo` would be a better match for my intuition...

## Does it make sense to have `foo` as an associated type "if not shadowed" or is that risky, should we use `fn#foo`.

## Can the `..` syntax really work?

pnkfelix: Can't we have overloaded functions? Can we really do `Foo(..)`?

## Taylor: what about generic functions?

If you have `Foo(..)`

It could work here: `T: PartialEq<U(..)>`, `for<A> { T: PartialEq<U(A)> }`

Various cases:

```rust
fn foo<T>(t: T)
```

```rust
fn foo<T>() -> T
```

niko: I had hoped to add enough limitations to side-step this for now, similar to how we side-stepped it for APIT and impl Trait interaction. 

cramtertj: You could imagine `fn#foo<..>` as a kind of "for all the possible values for the generics" notation.

## Tyler: What about the missing step between `..` and specifying all types?
