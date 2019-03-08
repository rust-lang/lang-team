This file will contain summaries from the meetings where the
unsafe-code-guidelines WG synchronizes with the lang team and broader
community. That hasn't happened yet. =)

# First meeting agenda (Date TBD)

## Review of how UCG has operated thus far

- The UCG reference lives on the repository
- At any given time we have an "active area of discussion":
    - This is proposed in a PR which describes an initial set of discussion topics
      ([example](https://github.com/rust-lang/unsafe-code-guidelines/pull/54)).
    - When an issue seems to reach consensus, somebody writes up a
      summary in the form of a PR that edits the reference
      ([example](https://github.com/rust-lang/unsafe-code-guidelines/pull/57)).
    - Sometimes, the consensus contains further points of contention
      -- in this case, we open up new issues representing just those
      points (and reference them from the reference).
- Weekly meetings on Zulip; mostly try to look for issues that
  seem to have reached consensus and assign work, don't usually involve
  a lot of technical discussion.
  
Experience with this system? It seems to be working reasonably well,
though we've not had as much participation as I had hoped, and we've
been moving a bit slower than I might've thought. I think this is
largely because these discussions are nobody's top priority right
now. -nikomatsakis

## General roadmap

Our immediate goal is to produce an "RFC" covering two inter-related areas:

- [**Layout:**](https://github.com/rust-lang/unsafe-code-guidelines/blob/master/active_discussion/layout.md)
  - What are the layout rules that unsafe code authors can rely on?
  - When do we guarantee ABI compatibility between Rust types and C types?
  - When can you rely on "enum layout" optimizations?
- [**Validity invariants:**](https://github.com/rust-lang/unsafe-code-guidelines/blob/master/active_discussion/validity.md)
  - Which invariants derived from types are there that the compiler
    expects to be always maintained, and (equivalently) that unsafe
    code must always uphold (or else cause undefined behavior)?

## Current status

We've finished up the **layout** conversation. Some of the key points:

- [Struct layout](https://github.com/rust-lang/unsafe-code-guidelines/blob/master/reference/src/layout/structs-and-tuples.md)
  - The layout of tuples `(T1..Tn)` is defined "as if" there were a corresonding struct
    in libcore `struct TupleN<P1..Pn>(P1..Pn)`.
    - **cramertj notes:** sgrif and I have discussed lots of times
      that the easiest path towards variadic generics would preclude
      this, and I'd like to make sure we've fully evaluated that
      consequence.
  - The layout of (Rust) structs is generally undefined, but there are some open questions
    around possible exceptions:
    - [homogeneous structs](https://github.com/rust-lang/unsafe-code-guidelines/issues/36)
    - [single-field structs](https://github.com/rust-lang/unsafe-code-guidelines/issues/34)
    - [zero-sized structs](https://github.com/rust-lang/unsafe-code-guidelines/issues/37) 
  - repr(C) structs are [laid out in a C-compatible
    fashion](https://github.com/rust-lang/unsafe-code-guidelines/blob/master/reference/src/layout/structs-and-tuples.md#c-compatible-layout-repr-c),
    and we identified some subtle points around zero-sized types and
    C++ compatibility
- [Enum layout](https://github.com/rust-lang/unsafe-code-guidelines/blob/master/reference/src/layout/enums.md)
  - repr(C) enums were specified in [RFC 2195](https://rust-lang.github.io/rfcs/2195-really-tagged-unions.html)
  - We described a [conservative version of "discriminant
    elision"](https://github.com/rust-lang/unsafe-code-guidelines/blob/master/reference/src/layout/enums.md#discriminant-elision-on-option-like-enums)
    -- aka, "the option optimization" -- that people can rely on
  - Layout of other enums is not defined
- Unions tagged with `#[repr(C)]` behave like C unions. repr(rust)
  unions are not specified -- in particular, some of the fields may
  not be laid out at offset 0.
- [Function
  pointers](https://github.com/rust-lang/unsafe-code-guidelines/blob/master/reference/src/layout/function-pointers.md)
  are defined as being laid out the same way as their C counterparts,
  though they cannot be NULL.
- [Integers and
  booleans](https://github.com/rust-lang/unsafe-code-guidelines/blob/master/reference/src/layout/integers-floatingpoint.md)
  and [SIMD vector
  types](https://github.com/rust-lang/unsafe-code-guidelines/blob/master/reference/src/layout/vectors.md)
  are basically defined as C compatible; we did not revisit the "great bool debate".
- [References and raw
  pointers](https://github.com/rust-lang/unsafe-code-guidelines/blob/master/reference/src/layout/pointers.md)
  are largely specified as behaving as they do in rustc
  today. However, we do not specify the layout of `&dyn (Trait1 +
  Trait2)`, to leave room for multi-trait objects.

We are currently discussing **validity invariants**. Key points and
open questions so far:

- XXX

## Points to consider and places for feedback

- What form should an RFC take here?
- After validity invariants, what is a good next area for us to focus on?

