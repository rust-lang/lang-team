---
tags: design-meeting
---

# 2022-04-13: Implied bounds, perfect derive

(Text is from a Niko blog post)

There are two ergonomic features that have been discussed for quite some time in Rust land: *perfect derive* and *expanded implied bounds*. Until recently, we were a bit stuck on the best way to implement them. Recently though I’ve been working on a new formulation of the Rust trait checker that gives us a bunch of new capabilities — among them, it resolved a soundness formulation that would have prevented these two features from being combined. I’m not going to describe my fix in detail in this post, though; instead, I want to ask a different question. Now that we *can* implement these features, should we?

Both of these features fit nicely into the *less rigamarole* part of the [lang team Rust 2024 roadmap][2024]. That is, they allow the compiler to be smarter and require less annotation from you to figure out what code should be legal. Interestingly, as a direct result of that, they both *also* carry the same downside: semver hazards.

[2024]: https://blog.rust-lang.org/inside-rust/2022/04/04/lang-roadmap-2024.html

## What is a semver hazard?

A **semver hazard** occurs when you have a change which *feels* innocuous but which, in fact, can break clients of your library. Whenever you try to automatically figure out some part of a crate’s public interface, you risk some kind of semver hazard. This doesn’t necessarily mean that you shouldn’t do the auto-detection: the convenience may be worth it. But it’s usually worth asking yourself if there is some way to lessen the semver hazard while still getting similar or the same benefits.

Rust has a number of semver hazards today.[^hazards] The most common example is around thread-safety. In Rust, a struct `MyStruct` is automatically deemed to implement the trait `Send` so long as all the fields of `MyStruct` are `Send` (this is why we call `Send` an [auto trait]: it is *automatically* implemented). This is very convenient, but an implication of it is that adding a private field to your struct whose type is not thread-safe (e.g., a `Rc<T>`) is potentially a breaking change: if someone was using your library and sending `MyStruct` to run in another thread, they would no longer be able to do so.

[^hazards]: Rules regarding semver are documented [here](https://doc.rust-lang.org/cargo/reference/semver.html), by the way.

[auto trait]: https://doc.rust-lang.org/reference/special-types-and-traits.html#auto-traits

## What is “perfect derive”?

So what is the *perfect derive* feature? Currently, when you derive a trait (e.g., `Clone`) on a generic type, the derive just assumes that *all* the generic parameters must be `Clone`. This is sometimes necessary, but not always; the idea of *perfect derive* is to change how derive works so that it instead figures out *exactly* the bounds that are needed. 

Let’s see an example. Consider this `List<T>` type, which creates a linked list of `T` elements. Suppose that `List<T>` can be deref’d to yield its `&T` value. However, lists are immutable once created, and we also want them to be cheaply cloneable, so we use `Rc<T>` to store the data itself:

```rust
#[derive(Clone)]
struct List<T> {
    data: Rc<T>,
    next: Option<Rc<List<T>>>,
}

impl<T> Deref for List<T> {
    type Target = T;

    fn deref(&self) -> &T { &self.data }
}
```

Currently, derive is going to generate an impl that requires `T: Clone`, like this…

```rust
impl<T> Clone for List<T> 
where
    T: Clone,
{
    fn clone(&self) {
        List {
            value: self.value.clone(),
            next: self.next.clone(),
        }
    }
}
```

If you look closely at this impl, though, you will see that the `T: Clone` requirement is not actually necessary. This is because the only `T` in this struct is inside of an `Rc`, and hence is reference counted. Cloning the `Rc` only increments the reference count, it doesn’t actually create a new `T`.

With *perfect derive*, we would change the derive to generate an impl with one where clause per field, instead. The idea is that what we *really* need to know is that every field is cloneable (which may in turn require that `T` be cloneable):

```rust
impl<T> Clone for List<T> 
where
    Rc<T>: Clone, // type of the `value` field
    Option<Rc<List<T>>: Clone, // type of the `next` field
{
    fn clone(&self) { /* as before */ }
}
```

## Making perfect derive sound was tricky, but we can do it now

This idea is quite old, but there were a few problems that have blocked us from doing it. First, it requires changing all trait matching to permit cycles (currently, cycles are only permitted for auto traits like `Send`). This is because checking whether `List<T>` is `Send` would not require checking whether `Option<Rc<List<T>>>` is `Send`. If you work that through, you’ll find that a cycle arises. I’m not going to talk much about this in this post, but it is not a trivial thing to do: if we are not careful, it would make Rust quite unsound indeed. For now, though, let’s just assume we can do it soundly.

## The semver hazard with perfect derive

The other problem is that it introduces a new semver hazard: just as Rust currently commits you to being `Send` so long as you don’t have any non-`Send` types, `derive` would now commit `List<T>` to being cloneable even when `T: Clone` does not hold. 

For example, perhaps we decide that storing a `Rc<T>` for each list wasn’t really necessary. Therefore, we might refactor `List<T>` to store `T` directly, like so:

```rust
#[derive(Clone)]
struct List<T> {
    data: T,
    next: Option<Rc<List<T>>>,
}
```

We might expect that, since we are only changing the type of a private field, this change could not cause any clients of the library to stop compiling. **With perfect derive, we would be wrong.**[^other] This change means that we now own a `T` directly, and so `List<T>: Clone` is only true if `T: Clone`. 

[^other]: Actually, you were wrong before: changing the types of private fields in Rust can already be a breaking change, as we discussed earlier (e.g., by introducing a `Rc`, which makes the type no longer implement `Send`).

## Expanded implied bounds

An *implied bound* is a where clause that you don’t have to write explicitly. For example, if you have a struct that declares `T: Ord`, like this one…

```rust
struct RedBlackTree<T: Ord> { … }

impl<T: Ord> RedBlackTree<T> {
    fn insert(&mut self, value: T) { … }
}
```

…it would be nice if functions that worked with a red-black tree didn’t have to redeclare those same bounds:

```rust
fn insert_smaller<T>(red_black_tree: &mut RedBlackTree<T>, item1: T, item2: T) {
    // Today, this function would require `where T: Ord`:
    if item1 < item2 {
        red_black_tree.insert(item);
    } else {
        red_black_tree.insert(item2);
    }   
}\
```

I am saying *expanded* implied bounds because Rust already has two notions of implied bounds: expanding supertraits (`T: Ord` implies `T: PartialOrd`, for example, which is why the fn above can contain `item1 < item2`) and outlives relations (an argument of type `&’a T`, for example, implies that `T: ‘a`). The most maximal version of this proposal would expand those implied bounds from supertraits and lifetimes to **any where-clause at all**.

## Implied bounds and semver

Expanding the set of implied bounds will also introduce a new semver hazard — or perhaps it would be better to say that is expands an existing semver hazard. It’s already the case that removing a supertrait from a trait is a breaking change: if the stdlib were to change `trait Ord` so that it no longer extended `Eq`, then Rust programs that just wrote `T: Ord` would no longer be able to assume that `T: Eq`, for example.

Similarly, at least with a maximal version of expanded implied bounds, removing the `T: Ord` from `BinaryTree<T>` would potentially stop client code from compiling. Making changes like that is not that uncommon. For example, we might want to introduce new methods on `BinaryTree` that work even without ordering. To do that, we would remove the `T: Ord` bound from the struct and just keep it on the impl:

```rust
struct RedBlackTree<T> { … }

impl<T> RedBlackTree<T> {
    fn len(&self) -> usize { /* doesn’t need to compare `T` values, so no bound */ }
}

impl<T: Ord> RedBlackTree<T> {
    fn insert(&mut self, value: T) { … }
}
```

But, if we had a maximal expansion of implied bounds, this could cause crates that depend on your library to stop compiling, because they would no longer be able to assume that `RedBlackTree<X>` being valid implies `X: Ord`. As a general rule, I think we want it to be clear what parts of your interface you are committing to and which you are not.

## PSA: Removing bounds not always semver compliant

Interestingly, while it is true that you can remove bounds from a struct (today, at least) and be at semver complaint[^maybe], this is not the case for impls. For example if I have

```rust
impl<T: Copy> MyTrait for Vec<T> { }
```

and I change it to `impl<T> MyTrait for Vec<T>`, this is effectively introducing a new blanket impl, and that is not a semver compliant change (see [RFC 2451] for more details).

[^maybe]: Uh, no promises — there may be some edge cases, particularly involving regions, where this is not true today. I should experiment.

[RFC 2451]: https://rust-lang.github.io/rfcs/2451-re-rebalancing-coherence.html

## Summarize

So, to summarize:

* Perfect derive is great, but it reveals details about your fields—- sure, you can clone your `List<T>` for any type `T` now, but maybe you want the right to require `T: Clone` in the future?
* Expanded implied bounds are great, but they prevent you from “relaxing” your requirements in the future— sure, you only ever have a `RedBlackTree<T>` for `T: Ord` now, but maybe you want to support more types in the future?
* But also: the rules around semver compliance are rather subtle and quick to anger.

## How can we fix these features?

I see a few options. The most obvious of course is to just accept the semver hazards. It’s not clear to me whether they will be a problem in practice, and Rust already has a number of similar hazards (e.g., adding a `Box<dyn Write>` makes your type no longer `Send`).

## Another extreme alternative: crate-local implied bounds

Another option for implied bounds would be to expand implied bounds, but only on a *crate-local* basis. Imagine that the `RedBlackTree` type is declared in some crate `rbtree`, like so…

```rust
// The crate rbtree
struct RedBlackTree<T: Ord> { .. }
…
impl<T> RedBlackTree<T> {
    fn insert(&mut self, value: T) {
        …
    }
}
```

This impl, because it lives in the same crate as `RedBlackTree`, would be able to benefit from expanded implied bounds. Therefore, code inside the impl could assume that `T: Ord`. That’s nice. If I later remove the `T: Ord` bound from `RedBlackTree`, I can move it to the impl, and that’s fine.

But if I’m in some downstream crate, then I don’t benefit from implied bounds. If I were going to, say, implement some trait for `RedBlackTree`, I’d have to repeat `T: Ord`…

```rust
trait MyTrait { }

impl<T> MyTrait for rbtrait::RedBlackTree<T>
where
    T: Ord, // required
{ }
```

## A middle ground: declaring “how public” your bounds are

Another variation would be to add a *visibility* to your bounds. The default would be that where clauses on structs are “private”, i.e., implied only within your module. But you could declare where clauses as “public”, in which case you would be committing to them as part of your semver guarantee:

```rust
struct RedBlackTree<T: pub Ord> { .. }
```

In principle, we could also support `pub(crate)` and other visibility modifiers. 

## Explicit perfect derive

I’ve been focused on implied bounds, but the same questions apply to perfect derive. In that case, I think the question is mildly simpler— we likely want some way to expand the perfect derive syntax to “opt in” to the perfect version (or “opt out” from it).

There have been some proposals that would allow you to be explicit about which parameters require which bounds. I’ve been a fan of those, but now that I’ve realized we can do perfect derive, I’m less sure. Maybe we should just want some way to say “add the bounds all the time” (the default today) or “use perfect derive” (the new option), and that’s good enough. We could even make there be a new attribute, e.g. `#[perfect_derive(…)]` or `#[semver_derive]`. Not sure.

## Conclusion

In the past, we were blocked for technical reasons from expanding implied bounds and supporting perfect derive, but I believe we have resolved those issues. So now we have to think a bit about semver and decide how much explicit we want to be.

Side not that, no matter what we pick, I think it would be great to have easy tooling to help authors determine if something is a semver breaking change. This is a bit tricky because it requires reasoning about two versions of your code. I know there is [rust-semverer]  but I’m not sure how well maintained it is. It’d be great to have a simple github action one could deploy that would warn you when reviewing PRs.

[rust-semverer]: https://github.com/rust-lang/rust-semverver

## Notes from meeting

This section is where we take notes from meeting discussion. While you read the document, feel free to edit this section to add questions you would like to raise during the meeting. 

### How do I add a question?

Ferris: To add a question, create a section (`###`) like this one. That helps with the markdown formatting and gives something to link to. Then you can type your name (e.g., `ferris:`) and your question. During the meeting, others might write some quick responses, and when we actually discuss the question, we can take further minutes in this section.

### What's the takeaway we want from the meeting?

Scott: Are we trying to pick a design direction?  Agree to experiment somewhere?  Or _____?

nikomatsakis: I want to move this feature forward and I wanted to (a) find out if there are other problems and (b) see if there is preliminary consensus around how to resolve any of the problems.

### Can the derives be used across crates?

nikomatsakis: any crate that changes its derive to use the new strategy would be "perfect"

barsky: would be good to call out how you could use it as an implementor

scottmcm: I like that point, David, because it pushes me very strongly towards "it's a new attribute". If all the Derives are supposed to be perfect now... or change their default based on an edition... coordinating the ecosystem derives to do that would be super awkward. But if it's a new thing, if the generated code has to change in all 3rd party derives, it being a new thing they op in to if they wish to provide that makes a lot of sense.

nikomatsakis: +1, I hadn't thought about this either

barsky: I feel like an edition boundary would be reasonable, since derive is so handy it'd be a shame to lose it

pnkfelix: do you mean an alternative to the word derive, or some kind of attribute you attach to the types...?

scottmcm: I mean something that is not identical to the current syntax, no strong feelings of what that would be. `perfect_derive` was a strawman.

nikomatsakis: I don't think I would actually want that :) 

nikomatsakis: `#[derive(@Clone)]` or something like that -- one question is whether we'd ever want it to be the default (in new edition)

joshtriplett: I would love for this to become the default in an edition, and have a way of opting in for a previous edition, but I don't think it can become the default in the previous edition due to breakage. But we will need some sort of mechanism for declaring what you want out of the derive. If we had that mechanism, maybe that same mechanism can serve as the opt-in 

nikomatsakis: opt-in to stricter boundars?

joshtriplett: either way, you can write the weaker bound, and if that turns out to be sufficient

nikomatsakis: the whole point of the feature is that you don't have to know which bounds, though :)

joshtriplett: you're saying having some opt-in to "figure out the fields for me" would make sense?

nikomatsakis: yeah, I think you probably want both, it's basically "make the bounds be the field types"

scottmcm: I certainly agree a lot of people want it but don't have a good instinct to how valuable it would be to change the default

nikomatsakis: the other thing that people want (or at least I want) is `PartialEq, Eq`, which is orthogonal, but maybe not?

scottmcm: I'd love to solve that not in derive, but with some kind of trait feature, where if you've implemented Ord, you shouldn't have to implement PartialOrd...?

nikomatsakis: because one can be derived from the other

scottmcm: right

nikomatsakis: specialization I guess?

scottmcm: similarly, if you could say `#[derive(Copy)]` and there was a provided `Clone` impl somehow...and this is somehow not specialization...

nikomatsakis: my thought was kind of "if there is a sigil opt-in, maybe it also expands out the supertraits"

jlusby42: I'd prefer to have the default be the more semver-conservative thing but have a hint that "hey you can opt-in here"

scottmcm: Is it always a backwards compatible change to switch from derive to perfect-derive?

nikomatsakis: it can never increase bounds, at least

scottmcm: if you go through a metafunction to look at an associated type? maybe?

nikomatsakis: should be fine, that wouldn't compile in the first place -- we must have concluded from the original derive that all field types are cloneable, so saying exactly that 

nikomatsakis: I hadn't thought about jane's suggestions of a lint saying "hey you might want to opt-in to this"

simulacrum: I think for stuff like serde it's possible to have attributes or something that create a more complex story that does not as easily be forwards compatible when switching 

nikomatsakis: if you for example skipped fields, but then again, your perfect derive would presumably take those into account

simulacrum: unconvinced you can always switch, it seems correct

jlusby42: would this apply to proc-macros define

nikomatsakis: they'd have to change the code they generated, so if it's opt-in, they'd have to be able to know that you opted in and adjust their strategy accordingly

scottmcm: to my previous point, even if we don't suggest perfect derive, somebody can show up with a PR switching to perfect derive and it wouldn't be a breaking change, so...

nikomatsakis: it seems like there is consensus that (a) there should be some sort of opt-in and (b) it should be as discoverable as possible, but no change to the default

pnkfelix: trying to get my finger on exactly what is being committed to... a clean syntax for saying that there's a public commitment being made ...

### At what boundary should we worry about semver?

(Followup from previous discussion.)

Josh: Declaring how public your bounds are seems like the right solution. But when not declared, I don't think we should default to module-private; I think we should default to crate-private. Modules aren't a semver boundary, so they don't have a semver hazard.

nikomatsakis: I agree that crate boundaries are the most important; the main reason I pointed out that there are other options is that "if you write `pub` you might wonder what happens if you write `pub(super)`"...

joshtriplett: we can accept `pub(crate)` as redundant but for the rest...

nikomatsakis: I think I would just reject for now anything but `pub`

### Is semver the only problem?

Mark: It seems to me that even if we solve the semver problems (with tooling, privacy declarations, etc.) you can still end up with somewhat confusing code. For example, today I believe we allow the following. If the T there was actually wrapped in something, then it might be really confusing where the `fmt` method comes from if the Debug bound is implied. (This is already sometimes confusing today).

In general it seems like implied bounds are rarely *super* helpful with "simple" bounds (where writing a little is not hard), which maybe is something to consider. (Tradeoff there).

```rust=
fn foo<T: std::fmt::Debug>(b: T, f: &mut std::fmt::Formatter) {
    b.fmt(f).unwrap();
}
```

Josh: I think there are two cases for implied bounds, with different requirements. In private, within the crate, it's helpful to have all the traits and everything they imply. In public, it's mostly helpful to not have to *duplicate* the trait bounds, but it's not actually especially helpful to be able to rely on them (e.g. you don't necessarily want to rely on the trait methods themselves, just the ability to create the type without declaring all the traits it needs).

nikomatsakis: we don't necessarily have to consider methods from implied bounds to be "inherent methods" on type parameters

simulacrum: I'd be concerned, since our story is already not so great if you don't have rust-analyzer, it seems like this makes it worse

nikomatsakis: I think I would just not allow it

pnkfelix: your point, niko, is that you'd accept code that's passing along data according to what traits it implements, but what you get to the point where code is doing something with it, you have to be explicit

```rust
// within the rbtree crate:

fn insert<T>(rb: RedBlackTree<T>, value: T) 
where
    WellFormed(RedBlackTree<T>),
    WellFormed(T),
{
    // WellFormed(RedBlackTree<T>) => T: Ord
    
    rb.insert(value); // OK 
    
    value < value // not ok because we didn't explicitly say that `T: Ord`
}

impl<T> RedBlackTree<T> {
    // maybe you get more assumptions in an impl than you do from fn parameters?
}
```


joshtriplett: distinguishing these feels really complicated?

nikomatsakis: maybe, I could see a case for "the impl feels lke part of the type but random methods in other modules do not"

joshtriplett: I've run into this since early days and many of those cases are places where I really *do* want to rely on the properties of the type, and not just that I have a valid instance. The ability to at least get implied bounds within the crate would be a major improvement and a big part of why I'd want to have it. I could understand wanting not to rely on them outside the trait, but it still seems like a tradeoff between complexity and semver

```rust
fn foo<T>(rb: RedBlackTree<T>, value: T) {
    value.cmp(); // maybe this is an error, because trait is not imported
    Ord::cmp(value); // but this is maybe ok?
}
```

simulacrum: the idea of `value < value` not working kind of appeals to me, one of my concerns is that implied bounds are kind of "infectious", and just because you use `T` here or there, but haven't written things everywhere-- if you delete one of your parameters, then stuff will stop compiling, figuring out what happens is complicated, compiler has no idea that you *used* to have an implied bound

nikomatsakis: you don't want to use this logic across crates, because you don't want to accept `rb.insert(value)` without knowing that the `RedBlackTree<T>` meets the `T: Ord` requirement -- danger is that if you moved the bound from the struct to the impl, that would be a breaking change.

scottmcm: I was thinking that it would be nice to allow very limited operations without having satisfied all the bounds -- e.g., you can match on `Cow::Borrowed`, but not invoke methods

nikomatsakis: I think you could do that

joshtriplett: Borderline case: even if we said that outside the crate you can't use the trait's methods, what about the case where you *pass* a `T` to something other than the type constructor itself, a function that wants the trait:

```rust
fn func<T>(rb: RedBlackTree<T>, value: T) {
    poke_at(value);
}

fn poke_at(value: impl Ord) {
    /// ...
}
```

### Are there sub-parts of this that work outside the crate?

Scott: As a sketch, maybe there could be something like "you don't need to prove the bounds on the type to mention it in a signature" -- but you still need to write the bounds to *use* those bounds, either in a struct initializer or to call something that needs those bounds.  (Maybe combined with something more powerful inside the crate, too.)

(addressed above)

### Are there other features that can help?

Scott: Inspired by Mark's question earlier, if we have trait aliases, say, would the bound being easy to write obviate the need for this somewhat?

Mark: Or could generate the impl "body" for traits supporting derives (with a supplied header).

```rust=
#[derive(@Clone)]
struct F<T> { ...} 

impl<Bar<T>: Clone> Clone for F<T> { /* magical body insert */ }
```

nikomatsakis: for implied bounds, you could imagine adding fn into the body to get the bounds without repeating...

```rust
struct RedBlackTree<T: Ord> {
    ;
    
    fn insert(&mut self) { ... }
}
```

joshtriplett: would we get any value from an impl-only derive mechanism that didn't supply any bounds for you at all?

nikomatsakis: my thought is that 

- most of the time, I actually want *no* bounds, i.e., I don't want `T` to be clone
- but there is this "mental jump" that I get stuck, "wait, why isn't this cloneable?" and I feel like that's the part I most want to avoid, but if it's opt-in that's less effective, at least as simple opt-in plus lint would help

nikomatsakis: implied bounds is also helpful in another case, I just remembered: Send and Sync, `derive(Send)` with a `&T` field needs `T: Sync`; this currently doesn't compile.

### Is there a potential ambiguity if a crate makes an existing bound public?

Josh: Example: crate A has trait Trait that privately has requirement for Debug; crate B depends on A and uses Trait, and declares its own bound for Display; crate A makes Debug requirement `pub`, does crate B now have ambiguity in resolution of `fmt` which exists in both?

nikomatsakis: potentially yes, but this can arise in other cases too; if we avoid `foo.bar()` notation, it becomes less likely (though still possible)


### Tooling/usability improvement

Mark: The doc sort of mentions this, but I will say that as someone who regularly-ish reviews std code, it is a real challenge today to know what is affecting the public interface. What can we do to enable semverver and similar tooling? I think some of the questions in this doc become much simpler if that's something that you could declare on your types succinctly in some way, for example:

```rust=
struct Foo {
    ...
}

#[interface_desc]
type Foo = {
    impl Send, Sync, ...;
    ...
    
    fn bar(&self)
};
```

Mostly a musing than a real question, but it feels like we're not doing enough to support people here, and it might ease discussions in general about what is and isn't actually public API. (e.g., with non_exhaustive we have lots of levels of "what ifs" too).


### Does this apply to *all* derives?

scottmcm: Are there derives that don't care about the difference?



### Implied bounds for well-formedness only

Jakob: Is there a possible version of implied bounds that allows the user to write code like this example:
```rust=
struct RedBlackTree<T: Ord> { … }

impl<T> RedBlackTree<T> {
    fn insert(&mut self, value: T) { … }
}
```

but not actually make use of the `T: Ord` except to allow the impl at all? (This got discussed above)



### Possible interaction: inherent traits

Scott: `pub` bounds might be inherent traits, sorta?  (Actually, the more I think about this the less I'm confident this makes sense, but I'll leave it here in case it's a thing.)















