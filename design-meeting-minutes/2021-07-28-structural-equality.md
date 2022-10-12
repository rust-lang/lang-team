# Structural Equality

## What is this

A write-up concerning the design space around structural equality.

## Motivation

Pattern matching and const generics require a stricter notion of equality than `PartialEq + Eq` can promise.

Sometimes we need to know that the `PartialEq` impl is equivalent to running the `PartialEq` impl of each field and using `&&` to combine the results. This is true for automatically derived `PartialEq` impls.

When using constants in patterns we can use this knowledge to "destructure" the constant and consider it during exhaustiveness checking. In case this property does not apply to the field of whatever we are matching on we can always fall back to treating the constant as unknown wrt exhaustiveness and call `PartialEq::eq` as if the user would have used a pattern guard instead of a pattern. An example of such fallback (more details later) would be

```rust
struct NotStructEq(i32, i32);
impl PartialEq for NotStructEq {
    // not comparing second field
    fn eq(&self, other: &Self) -> bool { self.0 == other.0 }
}
const FOO: (i32, NotStructEq) = (42, NotStructEq(1, 2));
match something {
    // This constant is taken apart to be the same internally
    // as the next pattern.
    FOO => {},
    // Same pattern, and unreachable, but rustc doesn't know
    (42, NotStructEq(1, 2)) => {}
    // Different pattern to a human, but unreachable and rustc doesn't know.
    (42, NotStructEq(1, 3)) => {}
}
```

For const generics it must be a recursive property, as we must be able to process the entire constant to all of its leaves.

## Possible changes from current behaviour

Before the next section digs into the details of how everything behaves, this section will give a summary of the possible and suggested (by @oli-obk) changes.

### Pattern matching on constants

1. do nothing, lazy fallback (requires allocation-based constant -> pattern lowering)
2. eager fallback
    * if `PartialEq + Eq` is derived (recursively), deconstruct the constant.
        Note that this is checked for a given value, not the type itself.
        Enums can have some variants that are recursively `PartialEq + Eq`
        while other variants are not.
    * if `PartialEq` or `Eq` is manually implemented anywhere (for fields' types or fields' fields' types...) fall back to invoking `PartialEq` at the top level.
    * affects unreachable patterns warnings (will stop detecting overlap between constants and patterns if `PartialEq` is not recursive), but not a breaking change
    * A possible variant could be to let the user explicitly opt-in to treating constants of some type as a valtree, via `[unsafe] impl StructuralEq` or similar.
3. hard error on using types that don't have a recursively derived `PartialEq + Eq` impl in patterns

@oli-obk would prefer option 3, but that is a breaking change. Thus prefers option 2, because it allows completely getting rid of deconstructing `mir::ConstValue` (allocation based) and allows working with `ty::ValTree`. We will need to keep maintaining two systems for deconstructing constants.

RalfJ wants option 2 (but mediated via an `unsafe trait StructuralEq`).

@lcnr wants option 2



#### Floats

Things we could do:

1. ban floats from patterns, using them already produces [a future incompatibility lint](https://github.com/rust-lang/rust/issues/41620)
2. make them structural-eq, even if they aren't `Eq`
    * this means subnormal floats are not equal to their normal variant
    * this means `f32::NAN` matches exactly `f32::NAN` [Ralfj: what about different NAN bit patterns? It is not clear what the semantics of comparison would even be here.]
    * probably confusing that it isn't behaving like the `PartialEq` impl
    * seems more in line with the meaning of pattern matching than the other alternatives
3. keep them not structural-eq, just always lower to invoking `PartialEq`
    * current behaviour
4. forbid float constants, only permit literals
    * weird, unergonomic
5. forbid problematic constants
    * weird, unergonomic

@oli-obk would like floats to be `StructuralEq` (option 2)

RalfJ thinks we can make *all floats except for NaN* be `StructuralEq` (so the type wouldn't be `StructuralEq` but most of its values would). They don't think we can have `f32: StructuralEq` while still having meaningful guarantees associated with `StructuralEq`... if we don't want to go that route, RalfJ prefers 3.


### Const generics

1. do nothing, adding complex types will introduce post monomorphization errors
2. make `StructuralEq` recursive, unsafe + stabilize,
    * so users can write `fn foo<T: StructuralEq, const C: T>`,
    * users can then also manually `unsafe impl StructuralEq for TheirType {}`, which allows divergence of `PartialEq` from structural equality, as long as it still matches some rules for symbol generation
        * lcnr argues that `unsafe` is irrelevant, as there are no soundness concerns, just weird behaviour and with post monomorphization errors being preventable as `StructuralEq` impls can be verified by the compiler (Similar to `Copy` impls).
3. make `StructuralEq` recursive and autogenerate `T: StructuralEq` bounds for all `T` that are used in const generic parameters' types like `fn foo<T, const C: T>`. (@lcnr: autogenerating `T: StructuralEq` bounds seems [difficult](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=f328da90726f670fbf3eec058086224a) to get right)

@oli-obk proposes to go for option 3, as the effects of allowing users to write `StructuralEq` impls are similar to option 1, but explicitly opted in to.

@lcnr strongly favors 2 here, with `StructuralEq` being safe to implement

[RalfJ: The difference between 2 and 3 is just whether we allow the user to `unsafe impl StructuralEq` or keep that as a compiler priviledge, right? So we can always start with 3 and remain future compatible with 2?] [oli-obk: yes]

## Requirements and considerations

The term structural equality is used in two very different situations in the Rust language:

* Pattern matching against a named constant, and in particular exhaustiveness checking
* Const generics: when are `T<C1>` and `T<C2>` equal?

While the actual details are very similar, the motivation for requiring structural equality is not the same.

### Pattern matching

Constants used in patterns participate in exhaustiveness checking. So

```rust=
const FOO: usize = 42;
match foo {
    FOO => {},
    42 => {},
    _ => {}
}
```

complains about an unreachable arm in the second arm.

For aggregated constants, rustc deconstructs the constant into individual patterns, so


```rust=
struct Foo { a: usize, b: usize };
const FOO: Foo = Foo { a: 42, b: 99 };
match foo {
    FOO => {},
    Foo { a: 42, b: 99 } => {},
    _ => {}
}
```

could also participate in exhaustiveness checking.

The problem here is that it is unclear whether `Foo` has a `PartialEq` impl, and whether it behaves exactly the same way as a derived `PartialEq` impl. While we can obtain this information during const->pat deconstruction, the question is what we do with it.

* if `PartialEq` is derived, deconstructing the constant does not change behaviour when compared to just lowering the constant in the pattern to a PartialEq call
* if `PartialEq` is manually implemented, deconstructing the constant *may* change the behaviour, and be very suprising
    * if `PartialEq` is not implemented, adding an implementation later may thus change the behaviour of matches 
* values of some types cannot be deconstructed into patterns, irrespective of the status of their `PartialEq` impl
    * raw pointers
    * floats
    * unions
    * function pointers

So what we do right now is

* if `PartialEq` is derived, deconstruct the constant
* if `PartialEq` is manually implemented, emit a compile-time error
    * for back compat, we accept it in some situations and fall back to invoking `PartialEq`
* if `PartialEq` is not implemented, emit a compile-time error

#### Floats

Floats are not structural-eq (they aren't `Eq` after all) [RalfJ: aren't fn ptrs structural-eq even though some of them are not even `PartialEq`?]. Among other things, this is because a `NaN` is not equal to itself. 

```rust=
match some_float {
    f32::NAN => {}
    f32::NAN => {}
    _ => {}
}
```

Even ignoring different representation of `NaN`, the first match arm will never match anything, because matching is done via `PartialEq`. Amusingly, we do report an "unreachable pattern" lint on the second pattern, because the bits of the two constants are equal.

### Const generics

From [ralfs github comment](https://github.com/rust-lang/rust/issues/74446#issuecomment-841637790) 
> In particular here we need a notion of equality on consts that acts more like PartialEq than memcmp, i.e., comparing references by comparing their pointees and skipping padding bytes.

The problem is that we can't use `PartialEq`, as that is inherently a runtime operation, and const generics needs to compare constants at compile time, or in the case of symbols for functions with const generic args, we need to make sure that two *equal* constants generate the *exact same* symbol.

If we just encoded the [`Allocation`](#Representing-data-structures-via-Allocation) of a constant into the mangled name of a const generic function, depending on how the constant was created, two equal constants could end up with different mangled names (e.g. because padding bytes could differ).

This is avoided in the current implementation by carefully deconstructing the `Allocation` into its components. The confidence in this code is not very high, as it is not very maintainable or extendable. It is also a fallible operation. If we encounter something that cannot be represented, we have no option but to bail out with an error. If we didn't have lots of restrictions on what types are allowed in const generics, we would get post-monomorphization errors whenever a constant has an unrepresentable value.

On the implementation side, the proposal is to move to [valtrees](#Representing-data-structures-via-ValTree). For the implementation these have the advantage that they can just be serialized, deserialized and compared directly and require no type-based special handling at any level.
[RalfJ: I view valtrees as also being a nice intermediate representation for const → pattern conversion. So I wouldn't burry them in the const generic section.]

#### Generic const parameter types

In order to know what further types we can allow for the types of generic constants, we need two things:

* some way to know whether all values of the type are ok
* a way to deconstruct values of the type into something that we can mangle.

When/if we get generic const parameter types (think `fn foo<T, const X: T>()`), we will need a trait bound on `T` that makes sure only types are used that satisfy the two rules above.

#### Unifying generic constants

In the future we want to be able to check for equality between generic constants without evaluating them, i.e. `N + 1` and `N + 1` should be equal. For this to be sound, structurally equivalent values have to be considered equal by the compiler. While this means that we need *some* notion of structural equivalence here, this should not influence its design.

[RalfJ: What we need for this is a notion of functions respecting structural equality. I can't quite make sense of this subsection.]

## Links to other documents

* (ancient) [RFC 1445](https://github.com/rust-lang/rfcs/blob/master/text/1445-restrict-constants-in-patterns.md)
* [Valtree master plan](https://hackmd.io/Qvrj_eOFTkCHZrhJ7f1ItA#Patterns)
* [structural match proposal](https://hackmd.io/-IB2BH1hSWWMQVSfbvVViA) by @**lcnr**
* [Ralf's github comment](https://github.com/rust-lang/rust/issues/74446#issuecomment-841637790)
* [Zulip discussion thread](https://rust-lang.zulipchat.com/#narrow/stream/260443-project-const-generics/topic/structural.20equality) and a [key comment](https://rust-lang.zulipchat.com/#narrow/stream/260443-project-const-generics/topic/structural.20equality/near/239274444) by Ralf

## Background concepts

### Representing data structures via `Allocation`

An [`Allocation`] is an untyped, opaque "blob of bytes" representation (with explicit undef markings and pointer/relocation markings) for a constant. This is what miri/CTFE uses. Allocations can represent a large range of values, including unions and other types that don't have a structured representation.

[`Allocation`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/mir/interpret/struct.Allocation.html

### Representing data structures via `ValTree`

A `ValTree` is an "up and coming" alternative representation for a constant value that is more structured. You can think of it as a tree with "scalar values" of various kinds at the leaves:

```
ValTree     = (ValTree, .., ValTree) | ValTreeLeaf
ValTreeLeaf = Integer
```

This set of leaf types is carefully chosen to represent only values where a deep notion of equality is possible.

* Some types, like `i32`, `(i32, i32)`, `struct Foo(i32)` or even references give rise to values that can always be represented with a val-tree.
* Other types give rise to values that simply cannot be represented as a val-tree. For example, union types do not have a val-tree representation, and neither do raw pointers.
* Finally, Still other types, like `Option<*const i32>`, have some values which can be represented with a valtree (`None`) and some which cannot (`Some(p)`).

Valtrees could be used to formalize the exact requirements of `StructuralEq`, if we end up having it as an unsafe trait.

## Questions

### Mark: Is there a description of exactly what the stricter requirements are?

Mark's hypothesis:

* exhaustiveness checking const patterns in `match`
	* needs some way to check if a pattern has been considered, StructuralEq provides the ability to deconstruct and compare each field individually, otherwise cannot do field-level exhaustiveness (only top level?)
	* needs comp-time == operator
* const generics
	* symbolic comparison, to check if `[T; N + 1] == [T; N + 1]` need 'reliable' comp-time PartialEq
	* also need to write out const generics into symbols, in a way that doesn't depend on how the constant was created -- for example, padding bytes cannot influence. Equality at compile time is equality in symbols.

Ralf: When it comes to exhaustivness checking, the question is really about "decomposition" of a constant into patterns. If we are able to decompose constants into patterns, the impl doesn't matter. So here the concerns are:

* Post-monomorphization errors: we have constants sometimes (e.g., with associated constants) whose value we don't know
    * this could fail to decompose
* Weird "non-linearities": 
    * if there are cases where we accept PartialEq impls: 
        * adding PartialEq impls should not change behavior
    * if you already have a PartialEq impl:
        * decomposition may behave differently if that impl
    * you add a PartialEq impl, and decomposition stops working, but now your behavior changes or code fails to compile
    * xxx this isn't quite right
    * Also relevant for reasoning: do we want to guarantee that *if* a constant has a PartialEq impl, then match will behave like that (or be rejected)? If yes, then we need some relationship between PartialEq and pattern construction (maybe via StructuralEq).
    * Also: we want a constant in pattern to behave the same no matter whether we know its value at MIR building time or not

* Ralf: This has to do with a part of the compiler that is converting a constant into a Pattern, where a Pattern has
    * matching on patterns etc
    * in some cases, "opaque" constants that invoke PArtialEq
* some of the proposal for how we conert constants into Patterns have included "fallback", and that is prone to non-linearities

### Mark: What does it mean for a value to be unrepresentable?

Ralf: Rust values like raw pointers can't be represented in some well-defined set for pattern matching, or things like uninitialized memory. 

### Does it make sense that `StructuralEq` inherit from `Eq`?

If floats were `StructuralEq`, that would imply that it's not `unsafe trait StructuralEq : Eq {}`.  Would that be weird?

- Ralf: I don't think so; StructuralEq is all about the behavior of PartialEq. I don't think Eq has to have much to do with it. But it might be more intuitive if it does.
- Ralf: note that we already allow matching on floats and have to deal with that
    - so we have some concept of matching
- Oli: not clear that there is a connection between partial-eq and structural-eq exactly, right?
- Ralf: two kinds of reasons you might want structural eq
    - being able to construct a val-tree (for this, partial eq is irrelevant)
    - non-linearities in pattern matching, where partial eq is relevant, because we sometimes have opaque patterns that invoke partial eq
        - in those cases, we might want to ensure that "if we do use structural eq, behavior is the same as if we used partial eq"
        - so that if you see a match, you can know for sure that behavior of the match is *like* partial eq
            - if you want that property, we need some connection between structural and partial eq
- pnkfelix: my memory of why we even have this dispatch into partial eq was more about saving on code size for match generation
    - if we're adding a new trait with its own method
- ralf: no method, it's a marker trait
- pnkfelix: ...still, if we choose to divorce partial-eq and structural-eq, there will be some code attached to structural-eq, because you need the code to do the traversal of the constant, right?
- ralf: I am not talking about code size, just observable behavior
- pnkfelix: even talking about observable semantics, are you ever intending someone to override the behavior to use a structural eq that doesn't correspond like structural traversal?
    - ralf: what
    - pnkfelix: do you ever intend to allow a match that doesn't correspond to structural traversal
- nikomatsakis: I'm a bit confused what we're talking about, do we mean only constants whose types implement structural eq? e.g. floats
- pnkfelix: my interpretation, in terms of matching, if we add structural eq, and we start talk about semantics of match in terms of structural eq...
- ralf: semantics of matches is not defined in terms in terms of structural eq, it's defined in terms of the **language of patterns**
    - the patterns you can syntactically write
    - and opaque constants (invoke partial-eq)
    - semantics of *those patterns* depends on PartialEq but not structural eq
- pnkfelix: why does that have to be the case?
    - ralf: that's what we currently do!
- pnkfelix: my memory was that the cases where we currently allow structural equality are pretty subtle
    - you tend to hit the existing checks that are meant to only permit you to include types that derive partial eq...
    - ralf: ...there's a loophole with references
    - loophope: https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=1e9d24b82e47959aad50617918716e45
- ralf: rough structure is that we start with a pattern in surface syntax which may contain constants
    - this is compiled to the internal pattern language of THIR
    - that's where the exhaustiveness checking is done, that language includes:
        - syntactic patterns
        - opaque constants (compared with partial-eq)
    - the semantics of that are pretty reasonable, there isn't much else you could do opaque constants
    - question is: how do we turn a constant in the syntactic space into this pattern language?
        - that might be affected by structural eq
- pnkfelix: edition boundary might ... permit us to change rules for opaque patterns ...
- ralf: to what?
- pnkfelix: to structural eq!
- ralf: but then it doesn't have code... that's a corner I hadn't explored, not sure what that code would do
- pnkfelix: to me the float situation "boils my bottom"
- ralf: floats are primitive patterns, right?
- pnkfelix: I don't see the motivation for saying PartialEq is the thing


- A possible resolution here would be that this is more like `IntoValTree`, but I think maybe be should move off this specific question, as the other questions might go to the more-interesting parts more helpfully.


### Q
What does `StructuralEq` "only for some of the values" mean?  Does that make it not a normal trait?
- Ralf: We can already have things like "`Copy` only for some values": `None::<Vec<i32>>` is, for all intents and purposes, `Copy`. This is similar, only the property is more complicated.

### Q

Ralf: Do we agree that the rules around pattern matching should ensure that adding a `PartialEq` impl, and (if that exists) adding a `StructuralEq` (potentially unsafe, then we assume it follows the contract) does not change the behavior of existing matches?

### Q
Mark: The document states that we can't rely on PartialEq as a "inherently a runtime operation", but we are moving towards const impls. Is `const PartialEq` not a compile-time property?
- That still does not help exhaustiveness checking or help us deconstruct the constant. So you're right, the "inherent runtime op" comment by itself is not a reason to go for `StructuralEq`. We could try to do something like "deconstruct, then run `const PartialEq` on all elements and `&&` the results and see if that matches the `const PartialEq` of the parent", but that seems awful.

### Q
lcnr: What exactly are the safety requirements of an `unsafe StructuralEq` impl and where would we be able to rely on them.
- fwiw oli-obk agrees, with the additional checking of the impl, we should go with option 2 + safe `StructuralEq`
- fwiw Ralf disagrees with oli. ;)  The additional checking has to ensure *some* property; we should make that property itself precise (and not just the check, which will probably approximate the property). Once we do, we might as well let people unsafely promise that their type satisfies the property.

### Q
Josh: Do any of your recommendations change if you have an edition boundary to work with? Or, even more strongly, if there were some way to make a difficult transition and *not* worry about the older semantics? (I'd like to know what we would do if not for compatibility, separate from what we would do as an evolutionary step.)

### Q
- pnkfelix: can you remind me what cases stable Rust currently exposes the fallback to `PartiaEq::eq` in match patterns? My attempt to translate the example to the playground hit: "error: to use a constant of type `NotStructEq` in a pattern, `NotStructEq` must be annotated with `#[derive(PartialEq, Eq)]`"

### Q

nikomatsakis: The doc talks a lot about dead patterns, but I think there are soundness concerns involving exhaustiveness, right? For example:

```rust
#[derive(PartialEq, Eq)]
struct Foo(bool);
const TRUE: Foo = Foo(true);
const FALSE: Foo = Foo(false);

match foo {
    TRUE => { }
    FALSE => { }
}
```

This code, I think, compiles today [yes](https://play.rust-lang.org/?version=nightly&mode=debug&edition=2018&gist=1020c1d75031b632f54d4fc831454c27)? If we had a `PartialEq` impl that were buggy or what have you, then it might wind up with "no path" to execute and hence an unsound result, correct? (This also applies to running user-defined code mid-match, as that could potentially alter shared data...?)

- lcnr: That would happen if we were to destructure the constant for exhaustiveness checking but still use the `PartialEq` impl. This seems both unlikely to happen as a bug and undesirable to be implemented. So I don't think we would ever do this even if `StructuralEq` was an `unsafe` trait. [Ralf: agreed.]
- User-defined code mid-match is interesting... but it should have no way to mutate the scrutinee, does it? This might rely on `UnsafeCell` not permitting matching on its field.

### Q

nikomatsakis: I have to admit I had a hard time understanding exactly what scheme was being proposed here. Maybe it's because of the range of options. I'm trying to wrap my head around what this "feels like as a user" and not quite there. It seems like:

- there is a notion of "structural equality" that means "equality = recursively compares all fields for equality"
    - something is only "structural eq" if all of its contents are also structural eq, right?
- if you derive PartialEq, Eq, do you get "structural eq" for free?
    - but surely we want the ability for people to manually specify it, is this where an impl of structural eq comes into play?
- ...

### Q

Example 1:
```rust
struct Foo(bool);

const TRUE: Foo = Foo(true);

fn foo() {
    match foo {
        // Error: no traits implemented
        TRUE => ...
        _ => ...
    }
}
```

Example 2:
```rust
#[derive(PartialEq)]
struct Foo(bool);

const TRUE: Foo = Foo(true);
const FALSE: Foo = Foo(false);

fn foo() {
    match foo {
        // Compiles
        TRUE => ...
        FALSE => ...
    }
}
```

```rust
struct Foo(bool);

impl PartialEq for Foo {
    fn eq(&self, other: &Self) -> bool {
        self.0 != other.0
    }
}

const TRUE: Foo = Foo(true);
const FALSE: Foo = Foo(false);

fn foo() {
    match foo {
        // Error: structural eq is not implemented
        TRUE => ...
        FALSE => ...
    }
}
```

Example 3:

```rust
struct Foo(bool);


impl PartialEq for Foo {
    fn eq(&self, other:&Self) {
        self.0 != other.0
    }
}

unsafe impl StructuralEq for Foo { }

const TRUE: Foo = Foo(true);
const FALSE: Foo = Foo(false);

fn foo() {
    match foo {
        // Compiles uses pattern destructuring
        TRUE => ...
        FALSE => ...
    }
}
```

Ralf's compiler from syntactic patterns to semantic patterns:

* given a constant of type T:
    * check that T: StructuralEq (must be recursively derived)
        * if so: recursively deconstruct it
            * this can't always be done pre-monomorphization
        * if not: treat it opaquely
    * generic constants?
        * always treat as opaquely
* properties we wish to maintain:
    * want semantics of pre- and post-monomorphization to be equal
        * if you compiled a syntactic pattern to a semantic pattern pre-monomorphization vs
            * compiling post-monomorphization
            * should get the same runtime semantics
        * may get more warnings for redundant pattern arms
    * if we know a constant has PartialEq, we might want to be sure that as a pattern it behaves like that PartialEq
    * want `match x { C => true, _ => false }` and `x == C` to be equivalent

## Oli's example of non-StructuralEq-match

https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=dc3b8eb7eb67cfaa0c0a38c0526a9ed9

```rust=
#[derive(PartialEq, Eq)]
// => impl <X> ::core::marker::StructuralPartialEq for Wrap<X> { }
struct Wrap<X>(X);

enum Foo {
    A, B
}

impl Eq for Foo {}
impl PartialEq for Foo {
    fn eq(&self, _other: &Self) -> bool {
        true
    }
}

const CFN: &Wrap<Foo> = &Wrap(Foo::A);
const BFN: &Wrap<Foo> = &Wrap(Foo::B);
fn main() {
    match CFN {
        CFN => {}
        BFN => {}
        _ => {}
    }
}
```

desugars to
```
impl <X> ::core::marker::StructuralPartialEq for Wrap<X> { }
```
and will call `<Wrap<Foo> as PartialEq>::eq` to pattern-match.

because we unroll to the constant: https://github.com/rust-lang/rust/blob/master/compiler/rustc_mir_build/src/thir/pattern/const_to_pat.rs#L337-L340 and catch that unroll here: https://github.com/rust-lang/rust/blob/29d61427ac47dc16c83e1c66b929b1198a3ccc35/compiler/rustc_mir_build/src/thir/pattern/const_to_pat.rs#L508

## Q

Josh: What are the *goals* of this effort? What problems are we trying to solve? 
Josh: I felt quite a bit confused about the context. Feels like a conversation-in-progress came to a meeting that includes several people who weren't in that conversation, and continued from where it left off without any context.
Ralf: Current structural-eq semantics might not be what we want (see Oli's example, which AFAIK works by accident). The extact contract of the StructuralEq trait is unclear. Now we want const generics so we need to extend our structural eq story, for which we first need to clean up that story around matches (unless we want a totally independent story for match and const generics, which I doubt).


