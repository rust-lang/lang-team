# Return Position Impl Trait in Dyn Trait, an overview

## Summary

Describes high-level summary of the "Return Position Impl Trait in Dyn Trait" (RPITIDT) approach that we propose:

[ToC]

## Problem statement

In a nutshell, we want to allow users to write traits and impls that include async functions:

```rust
trait AsyncIterator {
    type Item;
    async fn next(&mut self) -> Option<Self::Item>;
}

impl AsyncIterator for WidgetFactory {
    type Item = Widget;
    async fn next(&mut self) -> Option<Self::Item> { }
}
```

and then use the traits with `dyn` in a natural fashion:

```rust
fn count(f: &mut dyn AsyncIterator<Item = Widget>) -> usize {
    let mut c = 0;
    while let Some(_) = c.next().await {
        c += 1;
    }
    c
}
```

## Generalizing to impl Trait

We want the approach to generalize to `-> impl Trait` in a trait. In other words, the `async fn` example could be equivalently written like so:

```rust
trait AsyncIterator {
    type Item;
    fn(&mut self) -> impl Future<Output = Self::Item> + '_;
}

impl AsyncIterator for WidgetFactory {
    type Item = Widget;
    fn(&mut self) -> impl Future<Output = Self::Item> + '_ {
        async move { ... }
    }
}
```

and `count` should work the same way.

Without loss of generality, we shall now confine ourselves to talking about async fn.

## Why is this hard? Because of dyn trait.

In the case of static dispatch, adding `async fn` into traits and impls is not particularly hard. The idea is that the `AsyncIterator` trait would ultimately desugar to something like this, where the associated type `next` represents the future to be returned by `next()`:[^quibble]

[^quibble]: We can quibble about precise name of the associated type and what exactly it should represent, but that choice is orthogonal to the points raised in this doc.

```rust
trait AsyncIterator {
    type Item;
    
    type next<'me>: Future<Output = Self::Item> + '_
    where
        Self: 'me;
    
    fn next(&mut self) -> Self::next<'_>;
}
```

Now I can write a statically dispatched version of `count` easily enough:

```rust
fn count_static<A>(f: &mut A) -> usize 
where
    A: AsyncIterator<Item = Widget>,
{
    let mut c = 0;
    while let Some(_) = c.next().await {
        c += 1;
    }
    c
}
```

In this case, the call to `c.next()` will return `A::next`, which is some future type of some size, depending on the precise type `A`. When monomorphizing, the compiler will be able to allocate exactly the amount of stack space required to store that future `A::next` and await it.

**Unfortunately, when we try to have `f: &mut dyn AsyncIterator`, that strategy breaks down**. The whole point is that we don't know the precise type of iterator we have, so we also don't know the precise type of the `next` future that will be returned. Therefore we don't know how much stack space to allocate.

For this reason, the `#[async_trait]` adapter actually desugars `async fn` to return a `Pin<Box<dyn Future>>` and not an `impl Future`. This solves the problem, because now the returned future is always the same size, regardless of what type is being used. That is ok for `dyn` -- at least, presuming `Box` is ok -- but it's a pessimization for the static dispatch case. 

**The design we propose in this document does better than async-trait here.** For static dispatch cases, it has no allocation at all, and works exactly as you would expect. When using dynamic dispatch, it does require that the impl returns some kind of fat pointer, and it *defaults* to returning a `Pin<Box<dyn Future>>`, but it also gives users the ability to override that via adapters.

## Design goals

### Natural usage

Using traits with `async fn` should be just like using traits without it, at least most of the time.

### Static dispatch should not allocate; dynamic dispatch should be able to avoid allocation, but it takes some effort

We consider it ok if invoking an `async fn` via `dyn` allocates a `Box` most of the time, but there must be a way to invoke async functions (via `dyn`) without allocation to support no-std, high-performance loops, custom allocators, or other specialized cases.

### Separation of concerns

People should be able to write libraries that define traits, impls, and take `dyn` parameters in a natural way which can then readily be used by callers with different requirements (notably, by no-std callers who cannot allocate).

It is important that:

* a library L can write natural-looking traits, impls, and functions that take `dyn Trait` values;
* the library L can be used by a std project with zero annotation overhead;
* and the library L can then be used as a dependency by a no-std project with some effort (to specify the alternative memory allocation strategy they want).

Put another way, the code with the most information about its requirements should be making the decisions for how things work at runtime.

### Ship this year

We don't want a design that is blocked on "high-falutin'" features that may or may not ever see the light of day.

## Alternatives considered

We considered a number of approaches to resolving this problem.

### Design selected: "Return some wide pointer"

The design we describe here we call "return some pointer". The idea is that an async function, when invoked through `dyn`, always returns "some wide pointer" to a `dyn Future`. This wide pointer *may* be a `Box` (as in the common case) but could also be from some other source.

The "wide pointer" type carries a data pointer and a vtable, and the caller is able to invoke methods on it in the usual way. When the pointer is dropped, the vtable includes a drop function that will ensure the memory is deallocated if necessary.

The memory allocation strategy used to create this wide pointer is determined by the impl (and hence by the type implementing the trait). The default is to use `Box`, but impls can opt to create the "wide pointer" themselves instead via whatever means. Adapter types allow functions to "override" the choice to meet their local needs.

### Always return box

While simple, we rejected this alternative because of no-std.

### Box that is cached across calls

It has been observed that, for many async fn, the function takes a `&mut self` "lock" on the `self` type. In that case, you can only ever have one active future at a time. Even if you are happy to box, you might get a big speedup by caching that box across calls (this is not clear, we'd have to measure, the allocator might do a better job than you can).

We rejected this as the primary mechanism because it still requires Box. It also doesn't help with `&self` functions.

However, our proposed design can model this if code "opts in" to it with an adapter. Moreover, we believe it would be possible to 'upgrade' the ABI to perform this optimization without any change to user code: effectively the compiler would, when calling a function that returns `-> impl Trait` via `dyn`, allocate some "scratch space" on the stack and pass it to that function. The default generated shim, which today just boxes, would make use of this scratch space to stash the box in between calls (this would apply to any `&mut self` function, essentially).

### Return a fixed number of bytes ("small box")

We could make traits return some kind of `SmallBox` type which would permit stack allocation for small futures and fallback to box otherwise; this is vaguely similar to the design we have chosen, except that the result may not be a "wide pointer" but is more like "some data". We rejected this for a number of reasons:

* **What is the right threshold for the small box?** This value would have to be set at the trait level and this violates separation of concerns.
* **Futures tend to be quite large.** To avoid boxing an appreciable number of futures, we would have to make the "small box threshold" quite large, and that then becomes pure overhead for other cases.
* **It's just weird.** This "small box" type doesn't exist in libstd and just seems kind of non-canonical and arbitrary.

All that said, it wouldn't be hard to convert our existing design into a "small box"-like approach, if we had convincing data that it would pay off. (We should gather that data.)

### Unsized returns, aka, caller decides

We could allow functions to return unsized values:

```rust
fn foo() -> dyn Future<Output = Option<Item>> { 
}
```

The challenge then is that we must arrange for the caller to allocate the memory where the future will live without knowing up front how big the future is. This approach is appealing, but there are some major obstacles.

#### Supporting stack allocation

`async` contexts, and generators in general, do not support unsized allocation on the stack (also known as alloca). This is because their stack values that exist across await points are pre-allocated. Futures being awaited always exist across await points, so this approach would not support stack allocation.

#### "Caller decides" is not always desired

Intuitively, it may seem like letting the caller decide the allocation strategy is a correct approach, because the caller has more information than the callee.

This intuition breaks down, though, when working with libraries that take `dyn` or when using higher order functions. The code that has static information about the type being used is more likely to have information about what allocation strategy should be used.

Moreover, in our chosen approach, *any* code that has static information along the call stack can wrap that type in an adapter (as long as its callees are generic enough). So we think this approach gives far more flexibility than caller decides.

## How it feels to use

Defining a trait and impls look exactly as shown in the "problem statement". Importantly, it works the same way whether or not you are boxing:

```rust
trait AsyncIterator {
    type Item;
    async fn next(&mut self) -> Option<Self::Item>;
}

impl AsyncIterator for WidgetFactory {
    type Item = Widget;
    async fn next(&mut self) -> Option<Self::Item> { }
}
```

Invoking methods on a `dyn Trait` also works just as you would expect:

```rust
fn count(f: &mut dyn AsyncIterator) -> usize {
    let mut c = 0;
    while let Some(_) = c.next().await {
        c += 1;
    }
    c
}
```

### Creating dyn traits: for std users

If you are willing to allocate a future when `next` is invoked, then coercing a type into a `dyn Trait` works just as it does for any other trait:

```rust
fn invoke_count(mut x: impl AsyncIterator) {
    let c = count(&mut x);
    ...
}
```

### For the no-std user

Imagine we want to adapt `invoke_count` from the previous section for a no-std context; it should therefore avoid allocation. What we can do is to wrap our `impl AsyncIterator` in an "adapter". The `dyner` crate, to be published by rust-lang to cargo, will offer a number of convenient adapters, including one that allocates the space for each future on the stack:

```rust
fn invoke_count(mut x: impl AsyncIterator) {
    let c = count(&mut dyner::InlineAsyncIterator::new(x));
    //                 ^^^^^^^^^^^^^^^^^^^^^^^^^^ adapter type
    ...
}
```
### How you apply an existing "adapter" strategy to your own traits

The `InlineAsyncIterator` adapts an `AsyncIterator` to pre-allocate stack space for the returned futures, but what if you want to apply that inline stategy to one of your traits? You can do that by using the `#[inline_adapter]` attribute macro applied to your trait definition:

```rust
#[dyner::inline_adapter(InlineMyTrait)]
trait MyTrait {
    async fn some_function(&mut self);
}
```

This will create an adapter type called `InlineMyTrait` (the name is given as an argument to the attribute macro). You would then use it by invoking `new`:

```rust
fn foo(x: impl MyTrait) {
    let mut w = InlineMyTrait::new(x);
    bar(&mut w);
}

fn bar(x: &mut dyn MyTrait) {
    x.some_function();
}
```

If the trait is not defined in your crate, and hence you cannot use an attribute macro, you can use this alternate form, but it requires copying the trait definition:

```rust
dyner::inline::adapter_struct! {
    struct InlineAsyncIterator for trait MyTrait {
        async fn foo(&mut self);
    }
}
```

### Bounding return types

Sometimes you need to prove that the future returned by an async fn is `Send`. You'll be able to write something like this:

```rust
fn bar(x: &mut dyn MyTrait<foo: Send>) {
    tokio::spawn(|| async move {
        x.foo().await
    })
}
```

Here the return type of `foo` effectively becomes `impl Future<Output = ()> + Send`. The bound will be checked at the time of dyn coercion and the dyn bounds will follow normal coercion rules. There will be a similar way to bound the return types of trait methods of static type parameters.

## Design in more detail

### How to write an adapter

When you write an impl that contains an `async fn`, the default is that the resulting future will be boxed when used through dyn. But you can opt out of this by using the `#[dyn(identity)]` annotation. This annotation indicates that the function will return "some kind of pointer" -- we'll make that more precise in a sec, but for now it suffices to say that it must be a "thin pointer that implements `Future`". 

As an example, let's create an adapter `InAllocator<A, I>` that takes an async iterator and boxes the next future in the allocator `A`:

```rust
struct InAllocator<A: Allocator, I: AsyncIterator> {
    allocator: A,
    iterator: I,
}

impl<A: Allocator, I: AsyncIterator> InAllocator<A, I> {
    pub fn new(
        allocator: A,
        iterator: I,
    ) -> Self {
        Self { allocator, iterator }
    }
}
```

The impl for this would use `#[dyn(identity)]` and specify the return type as `Pin<Box<I::next, A>>`, which implements `Future` since `I::Next` implements `Future`:

```rust
impl<A, I> AsyncIterator for InAllocator<A, I>
where
    A: Allocator + Clone, 
    I: AsyncIterator,
{
    type Item = u32;

    #[dyn(identity)]
    fn next(&mut self) -> Pin<Box<I::next, A>> {
        let future = self.iterator.next();
        Box::pin_in(future, self.allocator.clone())
    }
}
```

When generating the vtable for `AsyncIterator`, the compiler would take the return value of this function (a thin pointer) and pair it with a custom vtable to return the "some pointer" type that the caller expects. To keep this document focused, we're purposefully eliding the details here; they will be the subject of a follow-up meeting. In a nutshell, the compiler generates an anonymous struct that pairs up the thin pointer and a vtable and implements `Future`, if you really want to know you can read more later at [the explainer](https://rust-lang.github.io/async-fundamentals-initiative/explainer/async_fn_in_dyn_trait/how_it_works.html).

## What we want from the lang team today

We are at the point in the design where we feel good about the direction and we were going to start sketching an implementation plan as well as going deeper into the details to work out some important "subquestions". We wanted to take the temperature of the team on this overall plan and in particular on the alternatives that we ruled out. Does anyone feel that one of those alternatives might be better? Have another idea to consider? Let's discuss!

## Notes from meeting

This section is where we take notes from meeting discussion. While you read the document, feel free to edit this section to add questions you would like to raise during the meeting. 

### How do I add a question?

Ferris: To add a question, create a section (`###`) like this one. That helps with the markdown formatting and gives something to link to. Then you can type your name (e.g., `ferris:`) and your question. During the meeting, others might write some quick responses, and when we actually discuss the question, we can take further minutes in this section.

### What are we sacrificing by having the "ship this year", "no highfalutin' features" requirement?

Josh: If we *did* wait, or rely on additional features, what better design might we be able to have? Do you have specific alternatives in mind that you ruled out on that basis? Or, even the broad strokes of alternatives?

nikomatsakis/tmandry: not really, *maybe* unsized returns

### "Unsized returns, aka, caller decides"

Josh: Could you talk more about what this alternative *could* look like? Could we make it straightforward for the caller to do the boxing in that case, if that's what they prefer? (That would also allow, in the future, using alloca or equivalent if we make that work.)
Josh: This would mean that defining *and* passing around `dyn Trait` would look straightforward, and calling functions that return unsized values would require stating what you want to do. That seems like the right level of visible difference for code that has a difference in behavior. And then we wouldn't need adapters, at all; it feels like a solution with more orthogonality.

cramertj: I've poked at this a lot, definitely not an easy case to make. Most of the realistic things I've wound up looking at look more like the smallbox approach, where you say "I want to return a dyn future with a size bounded a certain amount". Unsized return doesn't work very well. The case above (allocating stack space in a generator can't have an allocator, the stack is something you need to allocate up front). 

pnkfelix: An alternative I could imagine would be to allocate space on the stack with cow?

josh: giving the caller the option-- if we think that what callers will commonly do is box, they should be able to allocate a pointer on the stack and box. We can provide a syntax for boxing an unsized return value. Seems generally useful for more than this particular case.

nikomatsakis: we did have a design for caching across calls that seemed pretty straightforward.

josh: not what I wanted to do, but it would be easy to do with this approach.

nikomatsakis: caller writes something like `foo.bar(box)(...)`

cramertj: ...or `box foo.bar(...)` ....

joshtriplett: don't want to bikeshed, but in terms of the underlying calling convention, really not hard to imagine callee supplies a thing that caller has to catch either in a box...

nikomatsakis: ...it actually *is* really hard, if you want to do anything besides put it on a box.

cramertj: exactly I was going to give is the async iterator one, as given here, with next, you know there's only one at a time, so you can pre-reserve stack space. An example tyler linked in the comments shows how you can do it. https://rust-lang.github.io/async-fundamentals-initiative/explainer/inline_async_iter_adapter.html

cramertj: having callee decide the placement of the future is really helpful.

nikomatsakis: it's not exactly callee not caller, because of adapters

joshtriplett: kind of callee decides, but you can change who the callee is

tmandry: you might think the caller has more info, but really it's the point where the type is coerced to dyn that has the most

joshtriplett: if there is a reasonably straightforward design that does involve a "caller decides model" where we have some syntax where that makes sense, would that potentially be something to consider? obviously tradeoffs the completely transparent model. But not being transparent is "exactly as untransparent as it needs to be".

nikomatsakis: there are two problems I see--- it's actually really hard to figure out the ABI, since callee has to allocate in the caller's stack frame. 

pnkfelix: unless you pre-allocate...?

nikomatsakis: but to do that, the caller isn't the one, it's the one who coerces 

**tmandry: you could put the size in the vtable...**

joshtriplett: that would work sometimes? It would work anytime you know the size if you know the trait; it would only not work if the size is truly runtime-dependent.

nikomatsakis: it doesn't work in async, right?

joshtriplett: but I care about things outside of async, in other contexts, you might use alloca. (and it's not entirely obvious we *can't* make alloca work in async in the future)

cramertj: isn't a big motivation having it work without allocations in the async case

tmandry: could imagine a future where we have both

nikomatsakis: I don't think it's really a 1-way door, I think there's room to grow the ABI, the only real constraint is that we're forced to have some solution that boxes

joshtriplett: that is the 1-way door, that there has to be a default without an explicit syntax, you can't require an opt-in

nikomatsakis: yes, I just think that's not a lot

...back and forth...

### Stack-allocated return adapter

cramertj: "The InlineAsyncIterator adapts an AsyncIterator to pre-allocate stack space for the returned futures" -- how does this work exactly? For `AsyncIterator` specifically we know there can only ever be one `Next` future at a time and so can preallocate space for that, but it isn't clear how this would work for traits with `async` methods which take `&self`-- you could have any number of those futures at once. I suppose you could preallocate a fixed slab of stack space and error if you run out? But then you need the allocation mechanism there to be fallible (preferably not with just panic / abort).

nikomatsakis: the adapter we had permitted `&self` but panicked if it is used concurrently with itself (or recursed)

### Not a SmallBox

cramertj: The design says it isn't a `SmallBox` because it requires a pointer return value. It isn't clear to me why this is the case-- is there a fundamental limitation here that prevents returning pointer-sized futures directly? (e.g. `AsyncIterator::next`, which today is implemented via just a pointer to the `AsyncIterator`).

nikomatsakis: agree

### Pinning

cramertj: Are callers allowed to assume that a returned `dyn` Future pointer is `Unpin`? Presumably so, since it's just a pointer, but then there's a weird mismatch where switching to `dyn` actually causes users to stop having to manually pin things, and switching to static dispatch would force users to pin things they weren't pinning before.

all seems correct

nikomatsakis: we should like to look at the pinning stuff in more detail with you anyhow!

### What about Send + Sync

cramertj: just that-- it's fine to leave out of this focused design, but I am curious about the plan here.

nikomatsakis: let's talk about later :)

### What is natural result of composition

pnkfelix: I'm still wrapping my head around the picture here. Concrete question: If I end up having an `async fn` that returns the result of calling another `async fn` (with no intervening await), is the object code going to end up with nested boxing (i.e. `Pin<Box<Pin<Box<...>>>`? Or is the compiler going to somehow figure out that the innermost call is already returning a boxed thing and thus it can be elided? (Is this a place where one might use a manual `#[dyn(identity)]` annotation?)

...conclusion: probably you would get double indirection, but it's plausible we could add a special purpose optimization




