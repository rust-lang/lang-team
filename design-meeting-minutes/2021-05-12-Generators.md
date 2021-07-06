# Sync and Async Iterator Items, AKA Minimal Generators


## Draft RFCs

* [Syntax](https://github.com/estebank/rfcs/blob/stable-generators/text/0000-stable-generator-syntax.md)
* [Semantics](https://github.com/estebank/rfcs/blob/stable-generators/text/0000-generators.md)

## What I am proposing

I want to introduce a way to express sync and async iterator combinators through new syntax.

I explicitly reject the implementation of generalized coroutines for this feature to ensure being able to land this subset of that feature as quickly as possible.

### Why?

The main impetus for this is that these kind of algorithms are common enough in both library and end user code, writing it by hand can be verbose to express, writing a `Stream` requires interaction with `Pin`, and transforming an `Iterator` implementation into a `Stream` is non-trivial. The desire is to reduce the pain of implementing new async iterators, but if the feature is introduced it makes sense to also support sync iterators.

Iterators (both sync and async) are very common for a *subset* of applications.

Implementating `Stream`s also has [subtleties that people can make mistakes on](https://nikomatsakis.github.io/wg-async-foundations/vision/status_quo/aws_engineer/missed_waker_leads_to_lost_performance.html), and even when everything is done correctly, [the experience isn't great](https://nikomatsakis.github.io/wg-async-foundations/vision/status_quo/alan_hates_writing_a_stream.html).

### Status-quo

The nightly compiler currently has a `generator` feature, used by `async`/`await`, but its surface syntax isn't ideal. It takes over the closure syntax when the user uses `yield` inside of them, with no externally visible flagpost on the closure's head, and the way to interact with them is through the `Generator` `trait` and `GeneratorState` `enum`.

Some crates have explored exposing similar syntax, with varying levels of success on their ease of use.

#### gen-iter

[gen-iter] is a nightly-only crate that exposes a macro-by-example that allows the user to write:

```rust
#![feature(generators)]
#![feature(conservative_impl_trait)]

use gen_iter::gen_iter;

fn my_range(start: usize, end: usize) -> impl Iterator<Item = u64> {
    gen_iter!({
        let mut current = start;
        loop {
            let prev = current;
            current += 1;
            yield prev;
            if prev >= end {
                break;
            }
        }
    })
}

for i in my_range(0, 3) {
    println!("{}", i);
}
```

[gen-iter]: https://docs.rs/gen-iter/0.2.0/gen_iter/

#### Propane

[Propane] is a nightly-only crate that exposes proc-macros that allows the user to write `Iterator`s and `Stream`s by annotating a function or inline in an expression:

```rust
#[propane::generator]
fn foo(start: usize, end: usize) -> usize {
    for n in start..end {
        yield n;
    }
}

fn main() {
    let mut iter = propane::gen! {
        for x in 0..3 {
            yield x;
        }
    };
    let mut iter = foo(0, 3);
    for _ in iter {}
}
```

[Propane]: https://github.com/withoutboats/propane

#### async-stream

[async-stream] is similar to propane, providing a proc-macro to write `Stream`s exclusively, ignoring `Iterator`s:

```rust
async fn foo() {
    let s = stream! {
        for i in 0..3 {
            yield i;
        }
    };

    pin_mut!(s); // needed for iteration

    while let Some(value) = s.next().await {
        println!("got {}", value);
    }
}
```

[async-stream](https://github.com/tokio-rs/async-stream)


### Syntax
I do not find the conversation about the surface syntax of the feature to be where the most problems might arise. I want to focus on the semantics of the feature. Having said that, because we need *some* syntax to talk about the feature, I'll be using a strawman syntax for the examples:

```rust
fn* ident(arg: ty) yield ty {}
async fn* ident(arg: ty) yield ty {}
```

which would roughly desugar to:

```rust
fn ident(arg: ty) -> impl IntoIterator<Item = yield_ty> {}
fn ident(arg: ty) -> impl IntoAsyncIterator<Item = yield_ty> {}
```

We can bikeshed the actual syntax at a later time, but the crux of my proposal is to introduce a new type of Item that desugars to a function returning an iterable.

Changing from a sync to async generator should be straighforward to the end user.

I'm explicitly ignoring an expression position generator feature for now to focus on the broader strokes of the design. There's no one-way-door that wouldn't let us add it to the grammar in some way.  Open to discuss this today. A rough syntax for it could be, if we don't mind adding keywords:

```rust
let iter = async move iterator {
    for await x in stream {
        yield x;
    }
};
for await x in iter {
    println!("{:?}", x);
}
```

------

### Borrowing through a `yield` point, or "If you like it better put a `Pin` on it"

The same considerations around `Pin`ning that people must have when writing `async fn`s are present when writing generators. For example, writing a generator that results in a self-borrowing type, requires that the borrow is `Pin`ned:

```rust
fn* foo() yields i32 {
    let v = vec![1, 2, 3];
    for i in &v { // <-- can't do this without pin
        yield *i;
    }
}
```

There are a few options here:

 - auto-`Pin` inside of `fn*`s
 - reject it and force the user to be clearer about their ownership requirements
 - `Pin` the whole generator immediately to avoid compilation issues, but this is overly restrictive

Note that making things "just work" would likely require to _also_ implement some kind of automatic "pin projection" and including some "auto-pinning before iteration".

Writing the following today

```rust
use core::pin::Pin;
async fn bar() -> i32 {
    1
}

async fn foo() {
    let v = vec![bar(), bar(), bar()];
    for i in Pin::new(&mut v) {
        i.await;
    }
}
```

results in

```
error[E0277]: `from_generator::GenFuture<[static generator@src/main.rs:2:23: 4:2 {}]>` cannot be unpinned
 --> src/main.rs:8:14
  |
8 |     for i in Pin::new(&mut v) {
  |              ^^^^^^^^ within `Vec<impl Future>`, the trait `Unpin` is not implemented for `from_generator::GenFuture<[static generator@src/main.rs:2:23: 4:2 {}]>`
  |
  = note: required because it appears within the type `impl Future`
  = note: required because it appears within the type `impl Future`
  = note: required because it appears within the type `PhantomData<impl Future>`
  = note: required because it appears within the type `Unique<impl Future>`
  = note: required because it appears within the type `alloc::raw_vec::RawVec<impl Future>`
  = note: required because it appears within the type `Vec<impl Future>`
  = note: required by `Pin::<P>::new`
```

The distance between `Iterator` (which doesn't have a `self: Pin<&mut Self>`) and `Stream` (which does) needs to be accounted for.

------

## Example: Merging Intervals

Here, you can see an example excercise to compare and contrast the current experience in Python, Rust and the proposed experience.

Given K iterables of sorted "intervals" with two values (start and end), we want to produce a single iterable that merges overlapping intervals.

### Status quo: Python

One possible implementation of a solution to this problem is to split concerns and write two combinators, one to merge overlapping intervals in a sorted iterable, and another to merge a list of sorted iterables that returns a single sorted iterable. In Python such an implementation could look like the following:

```python
# Use tuples as the type for the intervals
intervals = [(1, 4), (3, 6), (8, 10), (9, 11)]

def merge_intervals(input):
    """
    Takes a sorted interval iterable (list or generator) and
    returns a generator of non-overlapping intervals.
    """

    prev = None
    for interval in input:
        if prev is None:
            prev = interval

        if prev[1] >= interval[0]:
            prev = (prev[0], max(prev[1], interval[1]))
        else:
            yield prev
            prev = interval
    if prev is not None:
        yield prev

def join_n_intervals(inputs):
    """
    Takes a list of sorted interval iterables and returns
    a single generator of sorted intervals.
    """

    latest = {}
    for k, input in enumerate(inputs):
        n = next(input, None)
        if n is not None:
            latest[k] = n

    while len(latest.keys()) > 0:
        min_interval, pos = None, None
        for current_pos, interval in latest.items():
            if min_interval is None:
                min_interval = interval
                pos = current_pos
            elif min_interval[0] > interval[0]:
                min_interval = interval
                pos = current_pos
        yield min_interval
        interval = next(inputs[pos], None)
        if interval is None:
            del latest[pos]
        else:
            latest[pos] = interval

intervals = [
  (i for i in [(1, 4), (8, 10), (13, 29)]),
  (i for i in [(3, 6), (8, 10), (9, 11), (25, 50)]),
  (i for i in [(3, 9), ]),
]

print(list(merge_intervals(sort_n_intervals(intervals))))
```

### Status quo: Rust

The same algorithm written in stable Rust today, would look the
following way:

```rust
use std::cmp::max;

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
struct Interval { start: usize, end: usize, }

fn i(start: usize, end: usize) -> Interval { Interval { start, end, } }

struct MergeIntervals<I: Iterator<Item = Interval>> {
    input: I,
    prev: Option<Interval>,
}

impl<I: Iterator<Item = Interval>> Iterator for MergeIntervals<I> {
    type Item = Interval;

    fn next(&mut self) -> Option<Interval> {
        loop {
            let next = match self.input.next() {
                Some(next) => next,
                None => {
                    let prev = self.prev;
                    self.prev = None;
                    return prev;
                }
            };
            let prev = self.prev.unwrap_or(next);
            if prev.end >= next.start {
                self.prev = Some(Interval {
                    start: prev.start,
                    end: max(prev.end, next.end),
                });
            } else {
                self.prev = Some(next);
                return Some(prev);
            }
        }
    }

}

fn merge_intervals(mut input: impl Iterator<Item = Interval>) -> impl Iterator<Item = Interval> {
    let prev = input.next();
    MergeIntervals {
        input,
        prev,
    }
}

struct JoinIter<I: Iterator<Item = Interval>> {
    inputs: Vec<I>,
    latest: Vec<Option<Interval>>,
    current_min: Option<(usize, Interval)>,
}

impl<I: Iterator<Item = Interval>> Iterator for JoinIter<I> {
    type Item = Interval;

    fn next(&mut self) -> Option<Interval> {
        for (k, input) in self.latest.iter().enumerate() {
            match (input, self.current_min) {
                (Some(interval), Some((_, min))) => {
                    if min.start > interval.start {
                        self.current_min = Some((k, *interval));
                    }
                }
                (Some(interval), None) => {
                    self.current_min = Some((k, *interval));
                }
                (None, _) => {}
            }
        }

        match self.current_min {
            Some((k, current)) => {
                self.latest[k] = self.inputs[k].next();
                self.current_min = None;
                Some(current)
            }
            None => None,
        }
    }
}

fn join_n_intervals<I: Iterator<Item = Interval>>(
    mut inputs: Vec<I>,
) -> impl Iterator<Item = Interval> {
    let latest = inputs.iter_mut().map(|i| i.next()).collect();
    JoinIter {
        inputs,
        latest,
        current_min: None,
    }
}

fn main() {
    let intervals = vec![
        vec![i(1, 4), i(8, 10), i(13, 29)].into_iter(),
        vec![i(3, 6), i(8, 10), i(9, 11), i(25, 50)].into_iter(),
        vec![i(3, 9), ].into_iter(),
    ];

    println!("{:?}", merge_intervals(join_n_intervals(intervals)).collect::<Vec<_>>());
}
```

Besides the verbosity difference, the approach is roughly the same.  The main difference lies in having to define a new `struct` for each iterator that we desired to construct, as well as an appropriate `Iterator` implementation. The requirement of having to explicitly define what data will continue living from one iteration to the next isn't insurmountable, but it is there.

### Proposed future

Under (one possible) end state of the current iterator item feature, the same algorithm would look like this:

```rust
use std::cmp::max;

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
struct Interval { start: usize, end: usize, }

fn i(start: usize, end: usize) -> Interval { Interval { start, end, } }

fn* merge_intervals(mut input: impl Iterator<Item = Interval>) yield Interval {
    let mut prev = input.next()?;
    while let Some(interval) = input.next() {
        if prev.end >= interval.start {
            prev.end = max(prev.end, interval.end);
        } else {
            yield prev;
            prev = interval;
        }
    }
    yield prev;
}

fn* join_n_intervals<I: Iterator<Item = Interval>>(mut inputs: Vec<I>) yield Interval {
    let mut latest = inputs.iter_mut().map(|i| i.next()).collect();
    let mut current_min = latest.iter().enumerate().filter_map(|(pos, i)| i.map(|i| (pos, i))).next()?;

    loop {
        let mut any = false;
        while let Some((k, input)) in latest.iter().enumerate().next() {
            if let Some(interval) = input {
                any = true;
                if current_min.1.start > interval.start {
                    current_min = Some((k, *interval));
                }
            }
        }
        if !any {
            break;
        }
        yield current_min.1;
        latest[current_min.0] = inputs[current_min.0].next();
    }
}

fn main() {
    let intervals = vec![
        vec![i(1, 4), i(8, 10), i(13, 29)].into_iter(),
        vec![i(3, 6), i(8, 10), i(9, 11), i(25, 50)].into_iter(),
        vec![i(3, 9), ].into_iter(),
    ];

    println!("{:?}", merge_intervals(join_n_intervals(intervals)).collect::<Vec<_>>());
}
```

The most obvious change is that the iterator backing `struct`s are implicit. This is an analogous situation to closures and `async fn`s: you can always represent the equivalent of a closure or `async fn` by hand, with a backing `struct` that holds the closed over values or the values retained accross yield points.

### Status-quo: `feature(generators)`

Under the current nightly-only generators feature, the same algorithm can be expressed, but can't be used as nicely, principaly due to the lack of a `[generator yield X]: IntoIterator<Item = X>` implementation.

```rust
#![feature(generators, generator_trait)]

use std::cmp::max;
use std::ops::{Generator, GeneratorState};
use std::pin::Pin;

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
struct Interval { start: usize, end: usize, }

fn i(start: usize, end: usize) -> Interval { Interval { start, end, } }

fn merge_intervals(mut input: impl Iterator<Item = Interval>) -> impl Generator<Yield = Interval> {
    move || {
        if let Some(mut prev) = input.next() {
            while let Some(interval) = input.next() {
                if prev.end >= interval.start {
                    prev.end = max(prev.end, interval.end);
                } else {
                    yield prev;
                    prev = interval;
                }
            }
            yield prev;
        }
    }
}

fn join_n_intervals<I: Iterator<Item = Interval>>(mut inputs: Vec<I>) -> impl Generator<Yield = Interval> {
    move || {
        let mut latest: Vec<Option<Interval>> = inputs.iter_mut().map(|i| i.next()).collect();
        let mut current_min = latest.iter().enumerate().filter_map(|(pos, i)| i.map(|i| (pos, i))).next().unwrap();

        loop {
            let mut any = false;
            while let Some((k, input)) = latest.iter().enumerate().next() {
                if let Some(interval) = input {
                    any = true;
                    if current_min.1.start > interval.start {
                        current_min = (k, *interval);
                    }
                }
            }
            if !any {
                break;
            }
            yield current_min.1;
            latest[current_min.0] = inputs[current_min.0].next();
        }
    }
}

fn main() {
    let intervals = vec![
        vec![i(1, 4), i(8, 10), i(13, 29)].into_iter(),
        vec![i(3, 6), i(8, 10), i(9, 11), i(25, 50)].into_iter(),
        vec![i(3, 9), ].into_iter(),
    ];

    let mut gen = merge_intervals(vec![i(3, 6), i(8, 10), i(9, 11), i(25, 50)].into_iter());
    loop {
        match Pin::new(&mut gen).resume(()) {
            GeneratorState::Yielded(interval) => println!("{:?}", interval),
            GeneratorState::Complete(_) => break,
        }
    }
}
```

That lack of easy conversion of a `[generator]` into an `Iterator` also makes it harder to compose these functions neatly, not only how to consume them.

If we continued with the current surface for generalized generators, using them in `for` and `while let` loops would require either extra syntactic sugar or extra method calls to explicitly convert them to somethng iterable.



----

## Example: Zip to streams

### Status quo

The following implementation is taken from async_std

```rust
use core::fmt;
use core::pin::Pin;

use pin_project_lite::pin_project;

use crate::stream::Stream;
use crate::task::{Context, Poll};

pin_project! {
    /// A stream that takes items from two other streams simultaneously.
    ///
    /// This `struct` is created by the [`zip`] method on [`Stream`]. See its
    /// documentation for more.
    ///
    /// [`zip`]: trait.Stream.html#method.zip
    /// [`Stream`]: trait.Stream.html
    pub struct Zip<A: Stream, B> {
        item_slot: Option<A::Item>,
        #[pin]
        first: A,
        #[pin]
        second: B,
    }
}

impl<A: Stream + fmt::Debug, B: fmt::Debug> fmt::Debug for Zip<A, B> {
    fn fmt(&self, fmt: &mut fmt::Formatter<'_>) -> fmt::Result {
        fmt.debug_struct("Zip")
            .field("first", &self.first)
            .field("second", &self.second)
            .finish()
    }
}

impl<A: Stream, B> Zip<A, B> {
    pub(crate) fn new(first: A, second: B) -> Self {
        Self {
            item_slot: None,
            first,
            second,
        }
    }
}

impl<A: Stream, B: Stream> Stream for Zip<A, B> {
    type Item = (A::Item, B::Item);

    fn poll_next(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Option<Self::Item>> {
        let this = self.project();
        if this.item_slot.is_none() {
            match this.first.poll_next(cx) {
                Poll::Pending => return Poll::Pending,
                Poll::Ready(None) => return Poll::Ready(None),
                Poll::Ready(Some(item)) => *this.item_slot = Some(item),
            }
        }
        let second_item = futures_core::ready!(this.second.poll_next(cx));
        let first_item = this.item_slot.take().unwrap();
        Poll::Ready(second_item.map(|second_item| (first_item, second_item)))
    }
}
```

### Proposed future

Under the proposed final behavior, the current will 
```rust
async fn* zip<A: Stream, B: Stream>(a: A, b: B) yield (A::Item, B::Item) {
    let mut a_item = a.next();
    let mut b_item = b.next();
    loop {
        match join!(a_item, b_item).await {
            (Some(a), Some(b)) => {
                yield (a, b);
                a_item = a.next();
                b_item = b.next();
            }
            _ => break,
        }
    }
}
```

And this is how it can be used:

```rust
async fn main() {
    let mut s = zip(foo(), bar());
    pin_mut!(s); // let mut s = Pin::new_unchecked(&mut s);
    while let Some((a, b)) = s.next().await {
        println!("{:?} {:?}", a, b);
    }
}
```

## Example: miniredis

[miniredis uses `async-stream`](https://github.com/tokio-rs/mini-redis/blob/c3bc304ac9f4b784f24b7f7012ed5a320594eb69/src/cmd/subscribe.rs#L172-L200) instead of hand-crafting a `Stream`:

```rust=
// Subscribe to the channel.
let rx = Box::pin(async_stream::stream! {
    loop {
        match rx.recv().await {
            Ok(msg) => yield msg,
            // If we lagged in consuming messages, just resume.
            Err(broadcast::error::RecvError::Lagged(_)) => {}
            Err(_) => break,
        }
    }
});
```

----


## Appendix A: Consuming async iterators

As you can see in the previous examples, iterating over async iterators today requires explicit pinning:

```rust
async fn main() {
    let mut s = zip(foo(), bar());
    pin_mut!(s); // let mut s = Pin::new_unchecked(&mut s);
    while let Some((a, b)) = s.next().await {
        println!("{:?} {:?}", a, b);
    }
}
```

For futures, `f.await` evades the need to pin explicitly by making pinning part of the await desugaring:

```rust=
// .await: `fn lower_expr_await`

unsafe {
     ::std::future::Future::poll(
         ::std::pin::Pin::new_unchecked(&mut pinned),
         ::std::future::get_context(task_context),
     )
}
```

Although not the subject of this design meeting, conceivably we could add an await loop in the future that would incorporate pinning:

```rust
async fn main() {
    for await (a, b) in zip(foo(), bar()) { // <--- the stream is pinned here!
        println!("{:?} {:?}", a, b);
    }
    // or for (a, b).wait in zip(..)...
}
```

one could also imagine a comparable `while await` syntax:
    
```rust
while await let Some((a, b)) in zip(foo(), bar()) { // <--- the stream is pinned here!
    println!("{:?} {:?}", a, b);
}
```

## Appendix B: `Pin`ning on iteration consumption

Async loop expressions could potentially auto-desugar to using `Pin::new_unchecked(&mut stream)`.


## Apendix C: what we *can't* do

- Not being able to influence a generator after it's being created limits the potential use cases. The distinction I've used when talking about this is "you can implement a lexer, but not a parser".
- The meaning of the "return value" of a generator is currently completely ignored/unused and represented as a `()`.

## Apendix C: how to do this today using channels

In [a pre-meeting discussion](https://rust-lang.zulipchat.com/#narrow/stream/213817-t-lang/topic/MVP.20Generators.20use-case), @cramertj showed [the following alternative that people can write today](https://rust-lang.zulipchat.com/#narrow/stream/213817-t-lang/topic/MVP.20Generators.20use-case/near/238392076):
```rust=
#[pin_project]
struct GeneratorStream<Fut: Future<Output = ()>, Item> {
    #[pin]
    fut: Fuse<Fut>,
    receiver: Receiver<Item>,
}

impl<Fut: Future<Output = ()>, Item> Stream for GeneratorStream<Fut, Item> {
    type Item = Item;
    fn poll_next(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Option<Self::Item>> {
        let this = self.project();
        if let Ok(item) = this.receiver.try_next() {
            return Poll::Ready(item);
        }
        let _ = this.fut.poll(cx);
        this.receiver.poll_next_unpin(cx)
    }
}

// Users can just write this:
fn my_stream() -> impl Stream<Item = u8> {
    let (mut sender, receiver) = channel(0);
    let fut = async move {
        for i in 0..100 {
            sender.send(i).await.unwrap();
        }
    };
    GeneratorStream { fut: fut.fuse(), receiver: receiver }
}

// Just for example
fn main() {
    let s = my_stream();
    futures::pin_mut!(s);
    for i in futures::executor::block_on_stream(s) {
        println!("{}", i);
    }
    println!("done")
}
```

# Appendix D: Notes from the meeting

## Question queue

- Don't want to get hung up on syntax bikeshed, but we should observe that there's a likely point of lively discussion about whether the return value should look like the item type (e.g. `u32`) or `impl Iterator<Item=u32>` (or `yields u32` (cramertj)). (josh)
    - esteban: Part of the reason of having `yield` or `yields` or some other way of expressing the `Item` in the syntax is to allow the potential future extension for return types/values:

```rust
    async fn* foo(_: i32) yield i32 -> Result<i32, ()> {...}
    // Some other syntaxes I toyed with.
    async gen foo(_: i32) yields i32 -> Result<i32, ()> {...}
    async iterator foo(_: i32) where yield = i32 -> Result<i32, ()> {...}
```

- josh: How do we distinguish between returning `impl Iterator<Item=Result<u32, SomeError>>` and returning `Result<impl Iterator<Item=u32>, SomeError>`? Or some other combination? That would be much easier if we spell out the return type. Related: how *do* we smoothly handle errors? Whatever first-class syntax we use for iteration should allow bubbling up `Result`, which means it should have a natural place for a `?` in it somewhere.
- Have you tried implementing the examples with `propane` or some other procedural macro? My biggest concern is generally that `Iterator` is not pinned but "async iterator" (stream) is, which means that one can borrows and one can't. I don't see what can be done about this, but I'm thinking about whether it's likely to come up. (I believe propane has this same disparity.) (nikomatsakis)
- Not a question, just noting that I'm explicitly opposed to `for await`--  iterating over Streams one element at a time is usually not what you want (cramertj)
    - Esteban had a convincing (to me) example of where this is useful --niko
        - Esteban: Composability is the main reason there: 
        ```rust
        for i await in foo() {
            if i < 3 {
                yield i * 2;
            }
        }
        // which could instead be
        foo().filter_map(|i| if i<3 {Some(i * 2)} else {None})
        // but then we are restricted in control flow and ownership,
        // the same situation why you would use a `for` loop today
        // instead of `filter_map` on an `Iterator`.
        ```
- This document seems to use "async iterator" and "stream" interchangeably; might be good to give a definition there near the top. (Would also be good to settle the naming question, but that's likely a meeting unto itself.)
    - Esteban: fair


## Notes and minutes

### Clarifying questions 

- Is *this* a one-way door? (Or at least, would we be less likely to consider generalized co-routines, if we already have provided a generator MVP?)
    - (regarding not using generalied coroutines)
    - what is missing in this proposal:
        - being able to return data that is not `()`
        - passing arguments through yield
    - nothing precludes us from adding those features after the fact
    - might have a more elaborate surface syntax to expose these other options
- "Itereators/streams are common for some applications (but not all)".
    - end-user applications use streams  and implement iterators
    - not only streams, also async-read and async-write
- Python code has a typo :) 
- Observation: `iter::from_fn` could be used to lighten the verbosity (somewhat)
    - Yes, but it just saves writing the stream, but you still have to manually encode the state machine
    - Also, async streams `from_fn` doesn't work as well
- (Discussed somewhat on Zulip) What are the motivating use-cases for priotizing this MVP functionality now? (cramertj)
    - estebank:
        - From the async vision doc, working with `Pin` is one of the biggest pain points that people have when learning async-await
            - the semantics are subtle and somewhat counterintuitive
        - Common cases where people have to interact with Pin:
            - when writing a new stream
        - Have seen cases of people mis-implementing streams in practice 
            - The code compiles but doesn't work "right"-- in some cases, just slow performance
        - Common enough thing which any microservice that has to deal with connections needs
            - Not just libraries that implement steams
        - Implementing streams from scratch is possible but has discoverability and complexity challenges
            - finding the procedural macros, multiple alternative implementations
            - need to read the write blog posts, etc
        - Worth noting that the consumer side also requires pinning, though not the subject of this document
            - having some way to express iteration would help us
    - cramertj: What is the specific user story where people are not being served today? I think a lot of the people writing streams today manually won't be served by this feature and will still have to write streams manually:
        - performance reasons
        - the stream in appendix C, for example, is complex; you can already write `impl Stream` today if you don't mind that the resulting stream has some restrictions
            - (not conveniently named, can't add methods)
        - what audience is not being served by the macro libraries?
            - what makes those solutons not good enough for those users?
    - estebank: discoverability. When you're told how to do it, easy enough to modify the example, but it's the same kind of ergonomic distance you would have if you had to implement every closure as a custom struct.
        - benefit: if we have a syntax for streams, we get iterators almost for free, which would change dramatically how some kinds of code are written
    - cramertj: most people writing iterators by hand would, I think, continue to do so
    - josh: I would love to never write an iterator by hand. If the syntax is good enough, that'd be great.
    - cramertj: when do you write iterators by hand today?
    - josh: When I'm trying to provide a data structure and I have to implement an iterator over it. (e.g. a custom collection-ish type `Xyz` for which I need an `XyzIter`)
    - cramertj: I usually want to offer a `Debug` impl, or make it `Clone`, things like that, make it nameable. Those are things I can't do if I don't write it by hand.
    - josh: I haven't hit that, apart perhaps from `Debug`, but it sounds like that may be a requirement for the syntax (e.g., being able to name the type that is being produced and then add methods, impls, etc)
    - scott: I feel like data structures often want to be double ended, etc
    - nikomatsakis: find myself writing small iterators, like DFS
    - cramertj: There are existing tools for writing streams
    - nikomatsakis: Many customer conversations bring up streams, and people aren't using those tools; partly discoverability, but I think it's also the matter of 'builtin'-ness
    - estebank: it's true that some folks won't make the switch if they're already using those tools; but folks who aren't writing streams etc but would like to may adopt it
    - cramertj: I feel like providing these tools via clear standard crates, which can't be done because of all the traits and things floating around, would be a good first step. The ecosystem is in a mess right now because of insufficient standardization. This isn't a primary step towards solving this problem. Other things with more payoff.
    - nikomatsakis: I think I agree that there are small first steps that have high impact. But we can plausibly do both, right?
    - cramertj: Arguing over syntax will take a lot of bandwidth, I think
    - scottmcm: being able to use yield return in C# for iterators means nobody ever writes manual iterators
        - But C# iterators are also less featured -- no DEI or ESI or similar -- and notoriously meh for performance, so a bunch of the goodness that could be done with a manual implementation of Rust's `Iterator` just don't exist in C#.
- Queued question: Why does the desugared `fn ident` return an `IntoIterator` rather than an `Iterator`? Is this for future-compatibility with more expansive Generator types that are not themselves iterators, but which can be converted into iterators? (cramertj)
    - Flexibility. If we were to return a specific `impl Iterator` we have less freedom to change things.
    - Would be useful to be able to extend and potentially name that type: if we return something nameable, could actually extend it and have not only iteration, but extra methods.
    - josh: Returning `impl Iterator` doesn't prevent naming it elsewhere. Or we could give a syntax for naming it.
    - Felix: would you be able to return one thing that could be used in two modes?
    - nikomatsakis: I think we have regretted not making ranges, but I'm not sure how relevant that is
    - scottmcm: otoh, the reason we stalled changing it is that it's nice to not have to do `into_iter` just to compose iterators (like zip)
    - scottmcm: Could lending iterators mean a different tradeoff here?  There are a bunch of things in `Itertools` that return `IntoIterator` that you need to store somewhere so that the `Iterator` can return borrows into it, but maybe `-> impl LendingIterator` would avoid all those problems.
- Queued question: How does this mix with other `Iterator` methods that are important for some situations, like `size_hint` or `fold`?  Providing a useful `size_hint` seems fundamentally hard?  Providing a `fold` might be particularly nice, though -- and maybe a `try_fold` could be made to work?  Interesting that one of the examples has the `next` implementation using a `loop`, which reminds me of how `next` is always `try_for_each(Err).err()`... (scott)
    - How *could* we handle things like `size_hint`?
    - nikomatsakis: you can certainly imagine writing impls on named impl traits
    - cramertj: how would you capture accessing captured variables
    - nikomatsakis: applicable to closures, it's come up before that e.g. not all closures want to be clone
    - mark: map could look a lot nicer than a generator; it'd be sad if you get worse performance because you lose your size-hint
        - I struggle to imagine how we could automatically generate a size hint (is this what you meant mark?)
    - cramertj: something like map probably wants to be hand-written so it can be micro-optimized
    - mark: I mean like "I have library code that is mapping some thing to some other thing"
    - cramertj: I think there's still a lot of early building block functions
    - nikomatsakis: i do want to see us preserve the tradition of "iterators are not only ergonomic but efficient"; we should at least be able to propagate some of this stuff
    - scottmcm: we do have some problem with that today, if you return `-> impl Iterator`, you lose `ExactSize` etc
    - mark: that affects methods, but not necessarily perf if it is achieved through specialization
    - related questions Niko thinks are answered
        - A little worried about addressing 'too little': Many of std's iterators have other 'extra' methods, not on Iterator. I think this is pretty common in other cases too (I've written "application" level iterators which add a few helper methods for construction etc). Those cases are not too well served by this feature, right? Are we convinced that custom-but-not-that-custom streams/iterators are sufficiently generally *wanted* that this particular definition is correct? (simulacrum)
        - How does this mix with other `Iterator` methods that are important for some situations, like `size_hint` or `fold`?  Providing a useful `size_hint` seems fundamentally hard?  Providing a `fold` might be particularly nice, though -- and maybe a `try_fold` could be made to work?  Interesting that one of the examples has the `next` implementation using a `loop`, which reminds me of how `next` is always `try_for_each(Err).err()`... (scott)
            - Could we have a way of naming the type created by the generator, so that you can expand that type further via an `impl` block?
            - It worries me that libraries would be discouraged from using this, since it wouldn't be `DoubleEndedIterator` or `ExactSizeIterator` or ...
    - josh: here is a hypothetical proposal:


A generator function could return an iterator type, which may be either an anonymous iterator or a named iterator type. In the latter case, it'd also be an option to implement further functions (e.g. `size_hint`) on the named type.
```
gen fn xyz(...) -> impl Iterator<Item=u32> { ... }
gen fn abc(...) -> MyIterator { ... }
```
 - Esteban: this could be done if generators are blocks: fn abc(...) -> impl Iterator { gen { yield 1; } }

- cramertj: but how do you access captured values?
- pnkfelix: something something zen ;) doesn't seem like an unsolvable problem to give access to state
- josh: we could "invert" the flow of syntax; have a way to make an impl of iterator for a type where you don't have to handwrite the next function

- Queued question (retroactively inserted): Could it be easier to do a version of this that's an expression instead of the function-level?  Like how we're having an easier time deciding `try{}` expressions instead of `try fn`?  There may be some pinning complexity with a block, though...
- scott: would an expression form avoid some of the questions?
- josh: I find only having a block syntax (and not a function syntax) appealing.
- niko: it doesn't solve the size-hint problem and you don't get to use
- cramertj: sidesteps bikeshed
- josh: gives you just enough syntax support to let you write the iterator block, then I can use a proc macro that results in defining the myiterator type and lets you define size-hint
- niko: you can do that with a fn item
- josh: having "just enough" syntax support does allow for more experimentation in the ecosystem with the top-level syntax
- cramertj: I agree with that, and it could help avoid some syntax questions
- josh: a block syntax solves some fraction of the "painful to write an iterator or stream" and anything beyond it is additional ergonomic and expressiveness improvement
- cramertj: not totally convinced, still need some way to annotate the return type
- scottmcm: not sure how I would store the generator etc 


- estebank: is size-hint about "base" iterators or propagating size hints from other iterators? the latter might be handleable..
    - cramertj: I don't know we would...

- josh: for the sake of completeness, it's *possible* in some cases for the compiler to infer and write a `size_hint` function, if the iteration is "obvious" (e.g. `for x in other_iter { ... yield ... }` with no `break` we could infer and propagate.)

- Queued question: I'm not sure I understand the bit about "auto-`Pin`-ing inside `gen fn`/`fn*`." I would expect the resulting generator to need to be pinned, not any value inside the generator itself. (cramertj)

- `yield`ing borrows into the stream state is out of scope, right? (cramertj)
    - not lending iterators, right?
    - niko: yes but we should leave syntactic space to not have to infer whether it's a lending iterator
