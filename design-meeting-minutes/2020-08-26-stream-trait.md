# Stream trait design meeting

* [Watch the recording](https://youtu.be/rIQHws2_lQg)
* [Stream trait draft](https://github.com/rust-lang/wg-async-foundations/blob/158cce517090a5c7648d1ca7576f3c6d0741ca22/rfc-drafts/stream.md)

## Summarize the design of stream

"Async" iterator:

```rust
trait Stream {
    type Item;

    fn poll_next(
        self: Pin<&mut Self>, 
        cx: &mut Context<'_>,
    ) -> Poll<Option<Self::Item>>;

    fn next(
        &mut self,
    ) -> impl Future<Output = Option<Self::Item>> where Self: Unpin;
    
    # Include try_next? vv helpful for while let Some(x) = s.try_next().await?
}
```

Async means:

* pinned receiver
* takes `cx` context
* returns `Poll`

The `next` helper lets you do things like:

```rust
let x = stream.next().await;
```

and in particular:

```rust
while let Some(x) = stream.next().await {

}
```

## Lang team considerations

### Adding methods later

* Adding new methods from the futures crate creates ambiguity
* A number of the "canonical" versions of stream methods are blocked by the missing AsyncFn trait
* This is already a problem, but its priority may increase as a side-effect of stream/future
* Some proposed solutions:
    * some kind of "low priority" annotation
        * but this is weird because you would want to resolve to the method from the crate, which is good for older crates, but not what newer crates want (which is stdlib)
        * could be tied to editions perhaps, but that would be a complex thing to manage
    * or a "feature detection" approach where e.g. itertools removes methods when they exist in the stdlib
        * would still break people with a lockfile on the old version, so suboptimal experience
            * Would reduce breakage if we get a version with feature detection out well in advance of the methods becoming part of the standard library.
    * or potentially "not solve it" and try to deal  with it
* Would we have changed design of the trait for Iterator?
    * Maybe, maybe not, more of a "libs concern"
    * interacts perhaps with specialization (being able to override a specific method)
    * extension traits per method would be the main alternative today but that's not so great
* In the case of the stream trait in futures, the ambiguity exists only when you forward the trait
* other places this problem exists:
    * Future trait -- but these are used less because of await syntax, or things like `join!(...)` macros
    * Iterator trait and itertools -- but Iterator has a "reasonably full"

### next method

* if futures-core redirects to stdlib's Stream
* people (at least today) may still use an older version of futures-util and hence they would still have the `next` method and get the ambiguity
* can be done as a non-breaking change
    * but it would require everyone to upgrade rustc
    * we may want to pick the timing, maybe wait a cycle or something
    * most folks are using "latest" Rust
    * should discuss the transition plan and what it means for users

### Lending traits

[Lending traits](https://github.com/rust-lang/wg-async-foundations/blob/158cce517090a5c7648d1ca7576f3c6d0741ca22/rfc-drafts/stream.md#lending-streams)

```rust
trait LendingIterator {
  type Item<'a>;
  
  fn next<'a>(&'a mut self) -> Self::Item<'a>;
}
```

* aka "attached" or "streaming" traits
* impl A: `impl<T> LendingStream for T where T: Stream`
* combines poorly with
    * impl B: `impl<T> Stream for Box<T> where T: Stream`
    * impl C: `impl<T> LendingStream for Box<T> where T: LendingStream`
* `Box<impl Stream>: LendingStream` could be satisfied in multiple ways:
    * would need a wrapper step or some way to declare that it doesn't matter which order the impls are applied in
    * possible extension would be some way to declare "commutative" impls such that we don't care which order things are applied in
    * intersection impls and specialization perhaps is another way to say 
        * `impl<T> LendingStream for Box<T> where T: Stream` -- the intersection impl
* maybe we want a supertrait relationship eventually (`trait Stream: LendingStream`)
    * and leverage `default impl`

* where clauses to say that "this lending-stream actually gives ownership of its items":

```rust
fn foo<T>(x: impl LendingStream<for<'a> Item<'a> = T>)
```

#### Generator syntax

* [Generator syntax](https://github.com/rust-lang/wg-async-foundations/blob/158cce517090a5c7648d1ca7576f3c6d0741ca22/rfc-drafts/stream.md#generator-syntax

```rust
gen fn foo() -> X { /* Iterator */ }
async gen fn foo() -> X { /* Stream */ }
```

* for Iterators:
    * because they don't have a pinned self, either need an explicit pinning step or can't permit borowing over yields
        * maybe some syntactic opt-in to "can't borrow over yield"
        * but maybe it's less important for iterators to be able to borrow over yields
        * in particular, no iterator today permits a borrow over a yield, but we've gotten quite far with the set of combinators (much farther than we got with manually written futures, which immediately hit problems)
    * example:
        * yielding from inside a for loop
        * but that's ok so long as the for loop is over a borrowed input, and not something owned by the stack frame
        * but clearly it would be useful sometimes
    * in spirit of experimentation:
        * boats has written the [`propane`](https://github.com/withoutboats/propane) crate
        * `#[propane] fn` that changes function signature to return `impl Iterator` and lets you `yield` -- the non-async version uses "static generator" that we have in nightly only
        * in part


### for-await syntax

You can use `while let Some(x) = ` to iterate today, but we would probably eventually want custom syntax:

* boats wrote a blog post about this but there may be newer developments
    * in particular maybe some issue with the proposed solution
    * we would probably want some special syntax
* one complication to while let is the need to pin:
    * but a `for` loop syntax that takes ownership of the stream would be able to do the pinning for you
* we may not want to make sequential processing "too easy" without also enabling parallel/concurrent processing, which people frequently want
    * one challenge is that parallel processing wouldn't naively permit early returns and other complex control flow
    * We could easily have a `par_stream()` similar to Rayon's `par_iter`
