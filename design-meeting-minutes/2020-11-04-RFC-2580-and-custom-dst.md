# Design meeting on RFC 2580

* [Lang-team issue #55](https://github.com/rust-lang/lang-team/issues/55)
* [Watch the recording](https://youtu.be/wYmJK62SSOM)

## Goal of the meeting

To evaluate [RFC 2580] and to generally review the state of Custom DST.

## Agenda

* Give an overview of RFC 2580 (15 min)
* Walk through various proposals to assess their impact

## What is contained in RFC 2580?

[RFC 2580] proposed a set of APIs that are meant to permit vtable manipulation while retaining forwards compatibility with various ideas.

Specifically proposed by the RFC:

* A `Pointee` trait, implemented automatically for all types (similar to how `Sized` and `Unsize` are implemented automatically) that defines a `Metadata` associated type.
    * `Metadata: Copy + Send + Sync + Ord + Hash + Unpin + 'static`
* A `Thin` trait alias, defined as `trait Thin = Pointee<Metadata=()>`. (If this RFC is implemented before type aliases are, uses of `Thin` should be replaced with its definition.)
* A `metadata` free function `fn<T: Pointee>(t: *const T) -> T::Metadata`
* A `DynMetadata` struct that represents the vtable from some `dyn Trait`
    * You can query the size/alignment and `alloc::Layout` of the underlying, erased type `T: Trait`
* A `from_raw_parts` constructor for each of `*const T`, `*mut T`, and `NonNull<T>`.
    * `pub fn from_raw_parts(data: *const (), meta: <T as Pointee>::Metadata) -> Self`
        * where `Self` is one of `*const T`, `*mut T`, `NonNull<T>`
* Replace the bounds on `null()` from `T: Sized` to `T: ?Sized + Thin` (also `null_mut()`, `NonNull::cast`)
    * This permits these functions to be used with extern types as well.
* Permit pointer casts from `&T` to `&dyn Foo` (etc) if `T: Thin + Foo`, not `T: Sized`
    * Unclear if this is actually meant to work by the RFC
    * Can't really work as we don't know the alignment and size

## Assumptions

* `Metadata` must be `Copy`, `Hash`, etc
    * It would be hard for this not to be the case, given that `&T` is copy and may be a fat pointer carrying `T::Metadata`
    * Hash and so forth are useful for implementing `Hash` for `*const T` where `T: Pointee`
        * [Existing impl](https://doc.rust-lang.org/src/core/hash/mod.rs.html#689-706)
    * Note that `DynMetadata` is not necessarily the *actual* vtable that we have in the compiler, but something that is wraps it
    * Interesting scenario to investigate:
        * dynamic loading, which would invalidate 'static bound potentially
        * [wide pointer equality is not guaranteed for the same object/trait combination](https://github.com/rust-lang/rust/issues/46139)
* We won't want further bounds on `Metadata` at some later point
    * `fn foo<T, U>() where T: Pointee<Metadata = U>` requires us to know the bounds on `U`
    * RFC [#2984] tried to sidestep this via an ad-hoc rule, but it's not clear how that would work
    * It may make sense for `Pointee` to remain unstable so that we can change pins and other things. Note that we added `Unpin` much later than the other traits in the list. 
    * Until you  have custom impls, this isn't as big a concern, except for well-formedness rules (and implied bounds might change those)
    * Observation: if we add to add `Unpin` later, we could've
        * the problem is `impl<T: ?Sized> Unpin for Box<T>` is nic
        * but we could've written `impl<T: Pointee<Metadata: Unpin>> Unpin for Box<T>`
  
## Possiblity extension

Make `DynMetadata` parameterized:

```rust
struct DynMetadata<T: ?Sized> {

}
```

So that you have `DynMetadata<dyn Foo>` instead of just `DynMetadata`.  This would permit `DynMetadata<dyn Foo + Bar>` to be multiple words.

An alternative would be to say that `DynMetadata` may change to a larger size or something.

Another alternative is to use const generics:

```rust
struct DynMetadata<const N: usize> { pointers: [*(); N] }
```

Another alternative is that `dyn Foo + Bar: Pointee<Metadata=DynMetadata<2>>`.

Also true: representation of `DynMetadata` doesn't have to match the actual pointer representation.

`metadata` could take a reference to a wide pointer and return a reference to `DynMetadata` (thus we don't need to guarantee the size and don't need `Copy`)

## DynMetadata

> Which of these types work in std vs core? And what traits do we guarantee `DynMetadata` will have? We talk all about the traits of `Metadata`, but what about `DynMetadata`?

`DynMetadata` can grow more traits, like other types.

## Minor comments on the RFC's design

* Josh:
    * My biggest open concern on *this* RFC is the 'static bound; is committing to Metadata (and DynMetadata) being 'static too strict?
    * Most of the other trait bounds don't concern me too much.
    * Also, is `* const ()` the way we want to spell "void pointer" here? That's incompatible with std::ffi::c_void, for instance. (which really should be core::ffi::c_void)
* Felix:
    * (I share Josh's `'static` concern, as I registered early; I'd also prefer a `Clone` bound over `Copy`, but I can understand skepticism about that.)
    * Niko: I think we can remove the `'static` bound because if we know that `T: 'static`, then we know that `T::Metadata: 'static`, and hence `*const T: 'static`
    * Josh: One case I could imagine: a structure that stores its own size, so figuring out the size requires looking at the structure.

```rust
impl Iterator for T {
   type Item = ;
}

(T: 'static) => (*const T: 'static)
```

## Proposals and compatibility notes

Exposing wide pointer internals "restricts" what we can do to those internals in the future. In order to even judge how future changes can affect these internals, we first need to know what future changes are even considered. The following list tries to be exhaustive of everything ever suggested via an RFC, please extend it if it is missing things.

* trait object upcasting `dyn Foo` -> `dyn Bar` where `Foo: Bar`
    * no real impact here
    * something I hope we will pursue soon
    * most likely way to implement this is to have vtable for `Foo` embed vtables for all supertraits
* multi-trait objects (`dyn Foo + Bar`), where `Foo` and `Bar` are entirely 
    * impacts if we opted for a "wide vtable" layout
    * but that is probably unwise
* custom wide pointer layouts
    * (e.g swapping the order of the pointer/len fields for a user-defined trait that has a similar API to slices or even creating zero sized pointers for embedded)
    * e.g. adding metadata in the known-zero-due-to-alignment bits
* custom vtable layouts
    * totally fine, this RFC does not prescribe anything about vtables except that they need a size, align and drop pointer
* extern types (wide pointers with zero sized metadata)
    * does not resolve the problem of `size_of_val` and `align_of_val` being applied to extern types
    * does permit `null` to be used with extern types
* custom metadata types (storing arbitrary custom data, not just a single pointer)
    * the proposal in its current form exposes the size of the metadata via the `DynMetadata` struct, but unstably since you need to transmute a `repr(Rust)` struct.
        * Niko: but only for dyn types, right?
    * japaric's [custom unsized type RFC](https://github.com/japaric/rfcs/blob/unsized2/text/0000-unsized-types.md) exposes the ability to change the metadata type, but is inherently incompatible with "custom wide pointer layouts".
* combining multi-trait objects and custom metadata, by allowing users to specify how the multi-trait object metadata is built.

Some of the proposals are discussed in more depth further down

### `dyn Foo + Bar`

* As defined, `DynMetadata` is the same for all `dyn` types. It carries a single pointer (just like fat pointers right now), which is fine for `dyn Foo + AutoTraits`, but may or may not work for `dyn Foo + Bar`
* Representing `dyn Foo + Bar` as a 3-word pointer would permit arbitrary subsetting, but might mean a lot more memory use (there is already a desire to avoid fat pointers in some cases).
* Otherwise, we are forced to "pre-allocate" vtables for all possible subsets and embed them within the original vtable (e.g., `dyn A + B + C` would need vtables for `A + B`, `B + C`, and `A + C` pre-computed).
* But see the unresolved question "Should `DynMetadata` have a type parameter for what trait object type it is a metadata of?" Such that `<dyn SomeTrait as Pointee>::Metadata` is `DynMetadata<dyn SomTrait>`. This would allow the memory layout to change based on the trait object type (potentially super-wide pointers with multiple vtable pointers for multi-trait objects?) or to have different methods.

## Extern types

[Tracking issue](https://github.com/rust-lang/rust/issues/43467)

This proposal enables `null` and so forth to work with extern types. The `Pointee` trait does not permit introspection of questions like size or alignment of the pointee type, so it can be implemented by extern types for which this information is unknown.

It does not attempt to address questions like `size_of_val`, which is still defined for all `T: ?Sized` and which hence still has to panic for extern types.

Side note: I (oli-obk) consider extern types to be obsolete if custom DSTs exist, as we can just create a custom DST that has zero sized metadata.

## Custom DST [RFC 1524] and [RFC 2594]

[RFC 2594], which superceded [RFC 1524], proposed Custom DST. It defined a similar `Pointee` type with metadata, along with an additional trait `Contiguous: Pointee` that defined size/alignment information.

It then permitted users to write custom `Pointee` impls for particular types, thus defining the "fat pointer" information that is found on a `&Foo` type. The APIs in RFC 2580 could then be used to extract that fat pointer from a `&self` reference, for example.

It also added additional APIs like `size_of_header` that were targeted at "flexible array size members

```rust
struct Matrix { }

impl Pointee for Matrix {
    type Metadata = [usize; 2];
}

impl Matrix {
    fn sum(&self) {
        let [width, height] = metadata(self);
    }
}
```

## Procedural vtables

[RFC 2967] (now closed) proposed "procedural vtables".  The idea was to give the ability to control how the vtable for a given trait is laid out, so that one could do `impl CustomUnsized for dyn MyTrait`.

Note that the RFC was closed due to its other major change: custom wide pointers which make unsizing and similar possibly expensive ops. Just the procedural vtable part is something I (oli-obk) think we should still pursue. At that point it just becomes a custom DST implementation detail, and we should thus not focus on it.

## Pointee and DynSized RFC #2984

[#2984]: https://github.com/rust-lang/rfcs/pull/2984

Introduces a trait hierarchy:

```rust
trait Pointee { type Meta: ...; } // can be target of a pointer, impl'd by all types
trait DynSized: Pointee { } // has a size that can be dynamically computed
trait Sized: DynSized<Meta = ()> { } // has a statically known size
```

and `DynSized` becomes a default bound unless `T: ?DynSized` is used.

## Other related work

I've not had time to dig into all of this, but this list is taken from [RFC 2594]:

- Existing Rust which could use this feature:
  - [CStr](https://doc.rust-lang.org/stable/std/ffi/struct.CStr.html)
  - [Pascal String](https://github.com/ubsan/epsilon/blob/master/src/string.rs#L11)
  - [Bit Vector](https://github.com/skiwi2/bit-vector/blob/master/src/bit_slice.rs)
- Other RFCs
  - [mzabaluev's Version](https://github.com/rust-lang/rfcs/pull/709)
  - [My Old Version](https://github.com/rust-lang/rfcs/pull/1524)
  - [japaric's Pre-RFC](https://github.com/japaric/rfcs/blob/unsized2/text/0000-unsized-types.md)
  - [mikeyhew's Pre-RFC](https://internals.rust-lang.org/t/pre-erfc-lets-fix-dsts/6663)
  - [MicahChalmer's RFC](https://github.com/rust-lang/rfcs/pull/9)
  - [nrc's Virtual Structs](https://github.com/rust-lang/rfcs/pull/5)
  - [Pointer Metadata and VTable](https://github.com/rust-lang/rfcs/pull/2580)
  - [Syntax of ?Sized](https://github.com/rust-lang/rfcs/pull/490)




[RFC 1524]: https://github.com/rust-lang/rfcs/pull/1524
[RFC 2967]: https://github.com/rust-lang/rfcs/pull/2967
[RFC 2594]: https://github.com/rust-lang/rfcs/pull/2594
[RFC 2580]: https://github.com/rust-lang/rfcs/pull/2580
