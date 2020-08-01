# align(ptr)

## Summary and problem statement

The `align` attribute can be insufficiently expressive for some types, and we aim to improve some use cases.

## Prioritization

This group calls under the "Targeted ergonomic wins and extensions" priority.

## Motivation, use-cases, and solution sketches

It's sometimes the case that a type must be aligned based on the platform's pointer width. For example, the `AtomicPtr` type is defined as follows:

```rust
#[cfg_attr(target_pointer_width = "16", repr(C, align(2)))]
#[cfg_attr(target_pointer_width = "32", repr(C, align(4)))]
#[cfg_attr(target_pointer_width = "64", repr(C, align(8)))]
pub struct AtomicPtr<T> {
    p: UnsafeCell<*mut T>,
}
```

That's three `cfg` lines without being able to say quite what we really want to say, "this is aligned to the size of a pointer".

So far, the `align` attribute only allows for an integer literal as the input. This project group aims to add a new alignment input, `ptr`, which is always the value of the local pointer size.

It has also been suggested that `size` be allowed as an input value, to request that a type have alignment equal to its own size.

This charter considers it **out of scope** to persue the general concept of const expressions within an `align` attribute. It's not impossible to design, but that's a much bigger project than this project group is setting out to do.

## Links and related work

[MCP Issue](https://github.com/rust-lang/lang-team/issues/35)

## Initial people involved

Initial People: Lokathor, ScottMCM

