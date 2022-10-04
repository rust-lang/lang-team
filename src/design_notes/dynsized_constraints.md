# Exotically sized types (`DynSized` and `extern type`)

## Overview

In current Rust, there's two kinds of types with respect to sizing:
if a type is `Sized`, its layout (size and alignment) is known statically,
and if a type is `?Sized`, its layout may not be known until runtime (e.g. via a vtable).

However, more exotically sized types exist; the most common example is opaque `extern type`.
`extern type`s have an *unknown* layout to Rust, and as such can only be used behind a pointer type.
Since the most unsized a type can currently be is `?Sized`, though,
the compiler has to make up a size and alignment to return from `mem::size_of_val`/`align_of_val`.
Currently the compiler returns a size of 0 and an alignment of 1.
Lying in this fashion is considered undesirable \[2].

Additionally, some C-header-interface libraries expose an opaque (incomplete) type
but also provide a function returning the size of the type and expect the caller to allocate space.
This is useful to allow the library to change the size of the type,
but still allow the caller to control allocation (e.g. using a custom arena allocator).
When bridging to Rust, these types should ideally have access to dynamic size/align.

## Proposed Solution

The most obvious and independently reinvented solution is a "`DynSized`" trait that provides dynamic size/align information.
`extern type` would not implement `DynSized`, and generic code could opt into `?DynSized` types to support such.

At the time of writing, there is weak approval from T-lang to proceed with an internal-only version of `DynSized`
which is used to prohibit the use of `extern type` in standard `<T: ?Sized>` generic arguments \[2].

This design document is about the restrictions on what `T: ?Sized + DynSized` actually needs to imply.

## Design Constraints

### `Arc` and `Weak`

`Arc` supports "zombie" references, where all strong `Arc` and the pointee have been dropped,
but `Weak` handles still exist and so the allocation still exists.
This means that `Weak` needs to be able to determine the layout of the allocation from a dropped pointee,
as the `T` is dropped with the last `Arc` but the allocation freed with the last `Weak`.

In addition, `Weak` are pointers to the *reference count* part of the `ArcInner` allocation,
and thus need to *statically* know the alignment of the pointee type to determine the offset
(it cannot call `align_of_val_raw` without first knowing the offset).

There are three potential resolutions that handle both size and alignment uniformly:

- Store layout information in the `ArcInner` header, or
- Require that layout be determined solely from pointee metadata, or
- Require that layout be determinable from a dropped pointee.[^why]

[^why]: This is trivially the case if determining the layout does not read the pointee (i.e. is derivable by just the potentially wide pointer);
    alternatively, the pointee could ensure that layout information (e.g. vtable pointer) remains valid to read even after it's been dropped.]

Dealing with alignment can be simplified by changing `Arc<T>` from storing `*mut ArcInner<T>` to
storing `*mut T` and storing the refcount metadata at a fixed negative offset independent of `T`.

T-lang commented on this in \[3] (w.r.t. const `Weak<T>::[into|from]_raw` and `Weak::new`):

> Consensus from meeting:
> - We approve the option to make `align_of_val_raw` require a once-valid-but-dropped value, in order to better support thin objects
>   - we believe the sentinel design \[of `Weak::new`] means that `align_of_val_raw` is only ever invoked on once-valid-but-dropped values
> - We do not want `align_of_val_raw` to be forced to work for metadata + thin pointer
> - Implement `Weak::from_raw` to check for sentinel and take some special action if it is observed
>   - potential cost: for unsized types (only), there is an extra branch (but if custom dst doesn’t require \[dynamic] alignment, we can change this later)
> - It is not really lang team’s call, but we are -1 on adding more fields to `Rc`/`Arc`
> - For custom dst, the design will have to accommodate getting the size and alignment from “once-valid-but-dropped” values (values that were once valid but have been dropped); this is a non-issue for known use cases like c-string and thin-objects (which store a vtable)
>   - (but could be relevant for dynamically allocated vtables)

### `Mutex` (and more generally, `UnsafeCell`)

The problem statement here is the combination of `&Mutex<T>` and `&mut T` both being usable concurrently,
plus the following presumably sound function:

```rust
fn noop_write<T: ?Sized>(it: &mut T) {
    let len = std::mem::size_of_val(it);
    let ptr = it as *mut T as *mut u8;
    unsafe { std::ptr::copy(ptr, ptr, len); }
}
```

To make the conflict abundantly clear, consider the following:

```rust
let mutex: &Mutex<ThinCStr> = /* elided */;

join(
    || {
        let mut lock = mutex.lock();
        let it: &mut ThinCStr = &mut *lock;
        noop_write(it);
    },
    || {
        std::mem::size_of_val(mutex);
    },
);
```

In order to determine the size of `Mutex<ThinCStr>`, you have to know the size of `ThinCStr`, which is inline to the `Mutex`.
To determine the size of `ThinCStr`, you have to read every byte to find the terminating nul byte (equiv. call `strlen`).
However, in the other fork, we lock the mutex and use the `&mut ThinCStr` to read and write-back every byte of the `ThinCStr`.
Because the `&mut` side of the operation is surely nonatomic (and `strlen` likely isn't), this is an unsafe data race, thus UB.

This constraint is more difficult to resolve than the previous one coming from `Arc`/`Weak`.
Fundamentally, types like `ThinCStr` which require reading the pointee to determine layout information break a core property of `UnsafeCell`
that `&UnsafeCell<T>` cannot (safely) read (or write) any of `T`'s bytes, if `std::mem::size_of_val` works without locking.

Thus (at the time of writing) there are three known potential resolutions to this constraint:

- Require layout to be calculated solely from thin pointer and pointee metadata,
- Require `size_of_val` to acquire a read lock (for `Mutex`-like types),
- Declare `noop_write` is only sound for types which determine layout without reading the pointee, or 
- Prohibit the use of pointee-determined-layout types in `Mutex`-like types.

## Potential Conclusions

This heading is the notes' author's (@CAD97's) opinion only:

From the above, there result *four* classes of sizedness that Rust *could* care about \[1]:

- "`T:  Sized +  MetaSized +  DynSized`", where the size and alignment are known statically;
- "`T: ?Sized +  MetaSized +  DynSized`", where the size and alignment are known from the data pointer and metadata;
- "`T: ?Sized + ?MetaSized +  DynSized`", where the size and alignment require reading the pointee; and
- "`T: ?Sized + ?MetaSized + ?DynSized`", where the size and alignment cannot be determined by (generic) code.

Examples of these are respectively `u8`, `dyn Trait`, `ThinCStr`, and `extern type`.

@CAD97 posits that in the majority of cases,
`OwningPointer<T>`-like types want "`?Sized + ?MetaSized + DynSized`",
`Ref<T>`-like types want "`?Sized + ?MetaSized + ?DynSized`", and
`UnsafeCell<T>`-like types want "`?Sized + MetaSized + DynSized`".

Additionally, it could be useful to restrict `MetaSized` to only know the pointee metadata and not the data pointer;
this would allow things like `[T] where T: ?Sized + MetaSized` using both slice and `T` metadata for an extra-fat pointer
(e.g. `[[T]]` for 2D slices doing the obvious thing (without stride)).

## References

- \[1] https://internals.rust-lang.org/t/erfc-minimal-custom-dsts-via-extern-type-dynsized/16591?u=cad97
- \[2] https://github.com/rust-lang/rust/issues/49708
- \[3] https://hackmd.io/7r3_is6uTz-163fsOV8Vfg
