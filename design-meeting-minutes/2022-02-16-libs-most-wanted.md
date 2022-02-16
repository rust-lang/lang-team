# Language features the library team would like to have.

## Ways to generalize an API without breaking things

Unlike crates on crates.io, we can't bump our major version and do anything backwards incompatible. :( We need more flexibility for making improvements and fixing small mistakes.

Examples:

1. `PartialEq` now only works between `Option<T>` of the same `T`. This resulted in [a request for `Option::contains`](https://github.com/rust-lang/rust/issues/62358), even though `Some(a) == Some(b)` should ideally just work if `a == b` works.
   - Adding that impl breaks lots of things, including `Some(1) == None`.
   - Solution? Default to `Option<!>`? Default to `T == U`? .. ?
2. Similarly, adding `Index<u8> to [T]` breaks stuff. (We're not sure if we want to do that yet, but right now we *can't* do that.)
3. Changing `str::repeat` to not return `String` but instead return `impl StringLike` (or something) to allow e.g. `OsString` would break.
   - Again: Solution might be some kind of default/hint?
   - Type parameter defaults might help here?
   - Inference failures are "allowed breakage", but they still have too much community impact to do lightly, especially for commonly used functions.
4. ..

## Never type

Time to get rid of `std::convert::Infallible` :)

## Ways to add things from popular crates to std without breakage

Every time we add something popular from for example `itertools` to std, stuff breaks. Also for macros like `matches` and `assert_matches`.

Possible solutions:
- Different mechanism for selecting from available methods / resolving ambiguity.
- `cfg(not(accessible(..)))` to allow crates like `itertools` to hide their versions when std adds them.
    - Note that we don't need a full `cfg(accessible(...))` implementation on arbitrary paths here, just support for absolute paths to external crates (`cfg(accessible(::...))`), or even just std/alloc/core (`cfg(accessible(::std::...))`).
- .. ?

## RFC 2492 "Existential types with external definitions"

We already have specific versions of this for the panic handler and global allocator, where we just assume it exists, but the user (or any of the dependencies pulled in) can define it.

If we can generalize that mechanism, then instead of "i assume there's a panic handler that looks like this" and "i assume there's a memory allocator that looks like this", we could allow ourselves and users to specify "I assume there's an X that looks like Y", which can be a very powerful tool for keeping platform-specific code maintainable. Especially in embedded systems this is very useful and now often hacked in with some linker tricks. But also in std we could use this to allow defining a 'platform' in a separate crate, so we don't have to maintain them all in-tree in the same crate.

wg-async folks are interested in this feature as well, and have it on their roadmaps.

- Combining this with some sort of 'specialization' would be even more intersting. Rather than a ton of `#[cfg]` with 7 different platforms, I'd like to pick e.g. the Mutex implementation based on whether a platform provides a futex-like api. (Specialize on `where Platform: FutexAPI`? Though if we can move the platform part to a different crate, `#[cfg(accessible(::std_sys::futex))]` would also work.)

## RFC 1868 "Portability lint"

This RFC is already approved but not implemented. It removes the need for our platform-specific extension traits (e.g. `std::os::unix::fs::FileExt`). We'd just use `#[cfg]` on inherent functions instead, and have a lint to prevent people from depending on platform-specific functions unless they make it explicit that that's what they want.

However, it might make sense to take a look at the problem set again from the perspective of RFC 2492 (the one above here). The lint in this RFC is based around `cfg`, which often results in hard-to-maintain code as the code is simply not compiled/checked in many situations. RFC 2492 makes me wonder if we could instead do these things with types and traits with a `where` clause. Basically `fn ... where Platform: Posix;` or something like that, where the user must specify that their crates assumes that `Platform` implements `Posix` (or something more specific, e.g. that it is `Linux`) to use those functions.

## Sealed traits

'Sealed traits' is a pretty common hack to prevent others from implementing your trait. But it's hacky and slightly annoying to implement. (In std we now use a single 'sealed' trait that's implemented for a ton of very unrelated types.)

We'd like to have a "native" syntax to implement sealed traits.

It's often used for 'extension traits' that are only implemented for one single type. Maybe this case needs its own special syntax and handling. E.g. `impl extension trait Bla for Type { .. }` to avoid having to duplicate the signatures in the `trait` and `impl`.

## Inherent traits

We'd like to declare a trait as "inherent" to a type, such that if you have the type in scope, you can use the trait's methods on the type without having to import the trait.

We should just update RFC 2375 (https://github.com/rust-lang/rfcs/pull/2375) to resolve feedback on that thread (minor unresolved questions) and then FCP-merge it.

## Multiple `#[unstable]` tags.

This one is simple. We sometimes have things that should be gated by multiple features. E.g. `ScopedJoinHandle::is_running` is part of `scoped_threads`, but also of `thread_is_running`. We now only use one of them, and sometimes accidentally stabilize something when only one of the features gets stabilized.

## Make `#[stable]` and `#[unstable]` available for third-party crates

Not something we need for `std`, but this would be useful for other crates. Right now there's no good way for a crate to specify that something is experimental and not part of its stable API yet. Feature flags work, but don't have the strong "this is exempt from semver" convention. (Maybe that's okay, as it's easier to have different versions of third-party crates than of `std`.)

Ideally, this should have some kind of top-level control, to prevent a random dependency from opting into unstable features of a sub-dependency without the top-level crate's knowledge and approval.

## Less dangerous specialization

A clear future path towards a specialization mechanism that doesn’t feel dangerous to use.

We do a lot of things with specialization that third-party crates cannot do, which makes the standard library too special. We've also had quite a few soundness bugs due to the specialization in the standard library.

Right now one of the things is that we'd like to specialize some `T: Debug` things on `T: Error`. E.g. `Termination` and `Result::unwrap`.

## Avoiding the `TypeTag` hack in RFC 3192

See https://github.com/rust-lang/rfcs/pull/3192#issuecomment-1016839273.

Not sure what feature this would need, but it looks like a hack that works around some missing language feature.

## Some way to update/split the Range types without breaking stuff

Ideally we'd split the `Range` types into two types each: The trivial type that's Copy and just contains the start and(/or) end, and the iterator type that might contain some extra flag (for `RangeInclusive`) and implements `Iterator`.

There's tons of ways this would break, and we'd need to change what `..` syntax expands to, and we'd need to change the return type of `[T]::as_ptr_range`, and everywhere people use these types in their API things would break or get really confusing, and, and .. It's all terrible.

No clue what the solution could be. Ideas welcome.

## Target-feature-optimized versions of core/alloc/std functions

We'd like to optimize some functions in the standard library for specific target features, either compiler-optimized (by enabling a target feature) or hand-optimized (with intrinsics or `asm!`).

We'd like to support both compile-time optimization (based on features chosen by the user) and runtime selection without a detection conditional on every call (perhaps via IFUNC or support for emitting hwcaps libraries?)

This may require some combination of lang and compiler machinery.


# Attendees

Lang: Niko, Josh Triplett, Felix, Taylor, Scott
Libs: Mara, Jane, David, Amanieu, Josh Triplett, Josh Stone
Other attendees: Mark, Michael Goulet, the8472

# Discussion / Question queue

## Question: Where do I put my questions?

nikomatsakis: Here!

## Observation: `"foo"` having type string would require some similar kind of inference work

nikomatsakis: The document talks about having `str::repeat` return a `impl StringLike`, but the same kind of inference would also presumably permit `let x: String = "foo"`, which is a common "toe-stub". In the past when we talked about this we sometimes got stuck on "finer points" of inference breakage (I don't believe it can be done *totally* compatibly) but editions might give us a useful lever there.

cramertj: don't we have some open issue and/or accepted RFC on inference fallback?

nikomatsakis: yes. but it is stalled because of lots of interactions. would need someone to own it, maybe carve out a safer path.

## Q: role of editions as way to inject "controlled breakage"

pnkfelix: re: "Ways to generalize an API without breaking things", how much do (or could) editions be leveraged here?

josh: Historically, Libs can't really use editions, because we need every crate to be using the same standard library (e.g. types). In theory we could *partially* use editions, and re-export types/traits, but not straightforward, and doesn't help with evolving functions on those types/traits.

mara: Totally possible, impl for edition-dependent types here: https://github.com/rust-lang/rust/pull/82489  But probably not a great idea. Hard to document. Doesn't help much with cases where people use our types in their public API. (E.g. if only one of the caller/callee gets updated to a new edition.)

mara: doing this for macros is painless because they are not the public API. Having edition-dependent versions of *types* makes the public APIs of other crates get real complicated. Return types could work but the docs are hard.

pnkfelix: I can see how upgrading editions could be a real roadblock.

joshtriplett: There are tools that make it easy to evolve lang, but not as easy that make it easy to evolve libs. Asking about ways to make libs easier to develop.

mara: Especially because the language usually evolves in bigger steps, but the library evolves in lots of smaller steps; mistakes can add up over time in the form of papercuts.

joshtriplett: No one of them is worth an edition but any 20 might be.

pnkfelix: What about having different versions of modules in std, and having people say which version they want...?

mara: same as before, wouldn't work for types, works if things are not part of public API

nikomatsakis: is that just "new function with new name and deprecate the old one"?

pnkfelix: It's just thought I thought specifying which version I want might be a lighter weight migration path. 

joshtriplett: especially in cases where code written for the old name will work 99% of the time. if you generalize something that accept *more types* as a parameter, or tweak the return type so that it has one as opposed to not, almost every caller will still work. Only two issues will be inference breakage.

cramertj: this whole section just feels like it's asking for type inference defaults to me.

joshtriplett: agreed that's the most obvious solution.

scottmcm: the range part doesn't feel like type inference defaults, if we wanted to change there.

cramertj: the indexing...if we had automatic upcasting of integers, we just wouldn't need to be able to index by u8. Those feel like one-offs. The hardest thing here is the `PartialEq` impl. That can't be solved by introducing a new trait, or at least not easily, but even there you would need a type inference default (e.g., `Some(1)` and `None`).

mara: that's interesting because either default needs to be that Left=Right or it should just default to `!` and if you could compare anything to a `None::<!>` that would also work.

joshtriplett: solves the `x == None` problem but not `Some(a) == Some(b)`

mara: depends if you want inference on `a` and `b` but we currently have that

cramertj: but the reason we don't have `Some(a) == Some(b)` is *because* of the `None` case

scottmcm: it also breaks `Some(22) = Some(iter().sum())` or `Some(22) == Some(Default::default())`

cramertj: in any case the solution seems to involve a type inference default, right?

mara: not sure if it's possible to have defaults in such a way that it doesn't break any existing case while still adding flexibility

scottmcm: I think if it handles 99% of them, inference breakage is allowed

nikomatsakis: that seems to be tied to editions or maybe other kinds of improved tooling, so that we can handle inference breakage better

mara; e.g. in older editions it still uses older signature or something?

cramertj: yes

## Observation: Portability has some 'negative' space (e.g., floats in kernel)

Mark: Is there need for special casing here? Maybe something that could also enable basic/minimal portability lint? In particular, thinking along the lines of what is written above with the `Self: Posix` style 'specialization'.

simulacrum: I was thinking about `#[cfg(no_alloc)]` or whatever it's name is -- is there a way to have configs that are less painful to work with and generalize -- but unlike portability lint where it's more of a lint, in this case I think the kernel people want to completely remove the types.

pnkfelix: what is "negative space"?

mara: features *add* things, but kernels want to remove features. We have 3 levels right now, core, alloc, std. Even if you have core, you already assume there are floating points. But kernel people don't want to assume that. Would be nice to split the categories into more specific features (atomics, floats, etc). And labels of core/std just represent groups of those.

joshtriplett: this one I expect we have ways of solving this without lang features, e.g. getting std + flags onto crates.io. Portability lint "in the positive"

mara: even with the features, it's maintenance hell if you have 20 different features you can enable. Checking all the combinations would be very difficult. Doing stuff with configure is really hard for maintenance. It happens now that you do something broken for windows and CI passes because it doesn't check windows. Would be really cool if we could use bounds on where clauses instead, so that you can still check the rest of the function based on that assumption.

joshtriplett: does feel like a really clever approach. I would expect that you can't compile the function body because some things don't exist, but I guess that if you use it pervasively all the way down.

nikomatsakis: obviously connected to scoped capabilities stuff we've been kicking around.

mara: might say "requires Platform: Posix" and you can then use it, just as you can have a trait with zero impls and write code against it.

nikomatsakis: right now we sometimes give you errors if there are where clauses that are not satisfiable, but I think we should give you warnings at most anyway.

joshtriplett: at the very least we can flag these types to avoid that. Could almost do that today with a struct, right?

cramertj: we have an accepted RFC for this ("trivially false where clauses"). If I remember correctly inside the body we assume things exist (e.g., `where String: Copy` would be ok). This is useful for all kinds of things, most notably having better automatic generation of impls for proc macros, so they can generate an impl where a field's type implements a trait without having to know whether it does or not.

nikomatsakis: I think we'll get there this year. I like Josh's suggestion also for prototyping it.

## Observation: sealed traits

nikomatsakis: Sealed traits came up in the async land in a way. As tmandry and I have been talking through what it means to extend the definition of `dyn`, especially so it can support argument-position impl trait, one question that arises is "who implements a given trait". We'd like to introduce synthetic types that implement traits in some cases, which is equivalent to "some struct that anybody could have added", but if the trait is not name-able, then those people could not have added them. We can add some kind of thing that checks "is this trait nameable from here" but that feels grody and I would prefer to have a "sealed" annotation for sure.

nikomatsakis: and of course *some versions* of the "sealed traits" pattern are anti-abstraction and annoying for users. e.g. given a `fn foo(x: &impl MyTrait)` where `MyTrait` is sealed, I cannot wrap `foo` in my own function because I can't name `MyTrait`.

Correction: "unnameable" trait is used to keep number of type parameters and other details private; "sealed trait" pattern is usd to prevent implementation. Current approach uses an unnameable supertrait to prevent implementation. Native implementation of "sealed trait" wouldn't need the unnameable supertrait.

nikomatsakis: strongly in favor of this, I think people want this pattern a lot, I think with some impl improvements we could also support crate-local impls or feature-gated impls, too.

amanieu: a common use for sealed trait is wanting to add methods, which we can't do if there are outside implementations.

scottmcm: kind of non-exhaustive for traits?

mara: non-exhaustive or exhaustive depends on how you look at it... the impls are exhaustive...

scottmcm: feels like how non-exhaustive works.

nikomatskais: +1

mara: similar because people used similar hacks for non-exhaustive enums (doc-hidden variants and things). Works but it's nicer to have a language thing. 

scottmcm: yeah, in stdlib we used stability hacks too, right?

mara: could work.

## Sealed traits and coherence

scottmcm: language-understood sealed traits seem like they could loosen some of the rules about blankets, since they're in a closed world.  Is that something libs might care about?  (I'm thinking of it from all the URLO threads about "why can't I `impl<T: MyTrait> Debug for T`" kinds of things.)

Josh: You can still name the trait from outside the crate, so someone could use it from outside the crate to make such an implementation. If you had a trait nobody could depend on like that, it might as well be private. (Scoped traits would be nice.)

mgoulet: I was actually thinking along scott's question as well. Josh, can you explain how that would break blanket rules? If I have a sealed trait and a set of types that implement that trait, I would like to be able to do `impl<T> OtherTrait for T where T: Sealed` in a downstream crate. (At least this might work if the sealed trait is only implemented for local types..)

scottmcm: The point about it only working for local types is a good point.

nikomatsakis: kind of like `#[fundamental]`.

scottmcm: gives another reason on top of "people want it" for a language feature.

nikomatsakis: something you can't do with current hacks, in other words.

scottmcm: right.

## 'extension traits'

pnkfelix: is the core idea that you want to be able to say something like:
```rust=
extend Trait for ConcreteType {
	fn new_method() { ... }
}
```

and now, in other code, *just* doing `use Trait;` will allow you to call `concrete_instance.new_method()`? (I.e. the key advantage is that one doesn't have to know yet another name of some random extension trait, and instead it something that is conceptually associated with the original parent `Trait`?)

mara: No, just a way to define a trait and impl it (once) at the same time.

nikomatsakis: there is currently `#[extension_trait::extension_trait]` ...

scottmcm: a very old brainstorming thread: https://internals.rust-lang.org/t/idea-simpler-method-syntax-private-helpers/7460?u=scottmcm

dtolnay: We often duplicate 3 places: the extension trait definition, the 1 implementation of the extension trait, and a set of inherent methods so that you get to call them without importing a trait. [`extension_trait`](https://crates.io/crates/extension-trait) deduplicates the first two. [`inherent`](https://crates.io/crates/inherent) deduplicates the last two.

mara: there's a feature close to this that I didn't write down in the document, "inherent trait impls". i.e. so you can say "if you impl the trait for this type, you can use the trait on that type without importing it". Say you make a `Range` that implements iterator and we don't want to put it in the prelude.

pnkfelix: I imagine the pattern today is making an inherent method and forwarding it?

mara: yes

scottmcm: my favorite example, feels silly whenever I have to import `BufRead` to use those methods on `BufReader`

joshtriplett: annoying you have to import the `Write` trait on stdout or `File`. If you could just open a file and write it...

nikomatsakis: there was an RFC, what happened to it? I think the proposal was to write `#[inherent]` on the impl or something like that?

scottmcm: https://github.com/rust-lang/rfcs/pull/2375

cramertj: ...last few comments... it looks like there were a few unresolved questions. How to resolve some trivial ambiguities. Somebody needs to spend an hour to fix this RFC up. Original author hasn't done that. We should "just do this". I think the questions had obvious answers that we all agreed on but which were not in the text of the RFC. 

joshtriplett: sounds like we should make the obvious changes and start an FCP

## Re: "Make `#[stable]` and `#[unstable]` available for third-party crates"

pnkfelix: My memory is that historically it was available, back in ancient times, and a deliberate decision was made to limit it solely to libstd. It would be good to review the reasoning that led to that choice at that time.

(got a link for that discussion?)
pnkfelix will look after they finish reading the doc itself
References found:
* https://github.com/rust-lang/rust/issues/22830
* https://github.com/rust-lang/rfcs/issues/1491
	* specifically: https://github.com/rust-lang/rfcs/issues/1491#issuecomment-222686126

mara: How were std's features and other crate's features separated? Or did you need nightly rust for other crate's unstable features as well?

## Observation: Some of these items feel like T-compiler

nikomatsakis: These two items don't really feel *lang* to me...

* Multiple `#[unstable]` tags.
* Target-feature-optimized versions of core/alloc/std functions

The first one definitely not; the second one problem is though it feels like it is pretty tied to low-level impl details. That kind of task often is outside the knowledge of folks on team and waits for someone more knowledgeable to show up and drive it. Motivation seems good though.

Josh: The first one would need lang review for extending the semantics, even if the subsequent impl is compiler. The second *may* require lang semantics in addition to compiler changes, though it does seem like the most compiler-centric item in this doc.

## "the `TypeTag` hack"

pnkfelix will need separate time to look at that section of that RFC. :)
https://github.com/nrc/rfcs/blob/dyno/text/0000-dyno.md#guide-level-explanation

## Question: coherence

nikomatsakis: I expected to see some discussion about coherence. I am thinking here about the "overall ecosystem effects". It seems to me that the coherence rules make it much harder to form standards in the crates.io land, and it is often kind of unclear *who* is responsible to (e.g.) implement whatever trait. The most obvious example is around async runtimes: there are a plethora of one-off compatibility traits but no obvious way to incrementally make progress on defining them outside of moving them to std (because there is a kind of chicken-and-egg problem).

Josh: +1 for coherence being an issue. Hard to handle the problem where crate A implements a type, crate B implements a trait, and anyone wanting to provide glue has to do it in either A or B, not in an AB glue crate. That makes it harder to scale the ecosystem.

## Question: about `..` syntax

pnkfelix: Would making the meaning of `..` syntax more context-dependent help at all here?

mara: It'd be very hard to tell the difference between `(1..5).sum()` vs `(1..5).into_iter().sum()`

## Question: Adding impls after the fact

nikomatsakis: another thing I expected to see was breakage about e.g. adding `From` impls


## Discusison: what comes next

Mara: should we have another meeting?

joshtriplett: maybe?

nikomatsakis: maybe focus on specific things? is there anything anyone wants to pick up and run with?

scottmcm: some things had clear next steps, like sealed and inherent traits

joshtriplett: feels like there are 3 categories: we know the answer, we kind of know the answer, and we don't have a clue

mara: I'm especially interested in the last category. I'm very interested in that portability lint direction and I don't have a concerete proposal to discuss. Don't know what the best way forward is.

pnkfelix: can I double check, you said you tried it and got stuck? did you actually try to implement something?

mara: no, I just don't know which direction here

nikomatsakis: I'd like to brainstorm that with you in a smaller setting. I'd probably loop Yosh into that.

yaahc: I'd be interested as well 

josh: me 3, I'd like to see if we can prototype it

nikomatsakis: I'm interested in talking more about coherence, it feels like a persistent pain point. Probably wg-traits.

mara: another libs related thing that might be interesting to discuss is simply the list of all the features we enable in libstd. There's a lot and it's prob a good list of things that might be nice to stabilize some time soon, or stop using.

josh: we've been talking about how close can we get libstd to compile on stable, it'd be nice to push closer to that line.

mara: yes would be nice if libstd were less special

scottmcm: at least alloc/std

mara: yes, core will always be special

nikomatsakis: someone game to go through and pick out the "obvious" items? was it only inherent trait?

scottmcm: I think sealed 

pnkfelix: maybe type inference defaults?

nikomatsakis: I think that's the middle categoryof "we kinda know what we want but it's got some big unknowns"

scottmcm: I think a sealed attribute would be easy enough to add, type inference defaults seem much harder

nikomatsakis: agreed, similarly with inherent trait

