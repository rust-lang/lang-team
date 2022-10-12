# Design meeting 2021-10-20: Dyn upcasting coercion safety

## Source document

https://rust-lang.github.io/dyn-upcasting-coercion-initiative/design-discussions/upcast-safety.html

## Scenario

* Casting `*dyn Foo` to `*dyn Bar` requires adjusting the vtable from a vtable for `Foo` to one for `Bar`
* For raw pointers, the metadata can be supplied by the user with `from_raw_parts`
    * If that metadata is incorrect, this could cause UB:
        * our current vtable format requires loads, and the pointers may not be valid
        * a flat vtable layout could give rise to out-of-bounds loads

Example:

```rust
trait Foo: Bar {
}

trait Bar {

}

fn test(x: *const dyn Foo) {
    // Performing this upcast requires adjusting the vtable
    // from a `Foo` vtable to a `Bar` vtable. Given the current
    // layout, this may involve loading from memory.
    //
    // But `x` is a raw pointer: do we have any reason to assume
    // that the metadata for `x` is valid?
    let y: *const dyn Bar = x;
}

fn bad_actor() {
    let x: *const dyn Foo = from_raw_parts(ptr::null(), /*garbage*/);
    test(x); // crash
}
```

So, clearly unsafety is needed, but where? Whose fault is the crash above?

Note that, today, [from_raw_parts](https://doc.rust-lang.org/std/ptr/fn.from_raw_parts.html) is safe. It does however require a valid `Metadata` argument, which implies a correct vtable.

## Option 1: `*const dyn Foo` metadata must always be valid

* Implies:
    * It is unsafe to create a `*const dyn Foo`.
        * Interestingly, from_raw_parts requires a valid metadata argument, so it can still be safe. The unsafety moves to transmuting and synthesizing the metadata somehow, which already requires unsafe.
        * The other source of unsafety is `transmute`. It would be UB to transmute something to `*const dyn Foo` without valid metadata.
* Also implies:
    * There is no obvious way to have a "null pointer" for `*const dyn Foo`
        * Without a data pointer, you don't know what its metadata should be
        * We could plausibly have a "sentinel" vtable for null, or have some way to synthesize a vtable for a given trait that is "structurally valid" but has null pointers for every method.
        * Note that currently `ptr::null<T>` requires `T: Sized`

## Option 2: Raw pointer upcast is only allowed with valid metadata

* Implies:
    * It is unsafe to perform an "unsizing upcast" to `*const dyn Bar`
* It is also a rather special rule, since ordinarily upcasts are managed by the [`CoerceUnsized`](https://doc.rust-lang.org/std/ops/trait.CoerceUnsized.html) trait, and we don't have a mechanism to make trait matching unsafe depending on which *impls are used* (i.e., [this impl](https://doc.rust-lang.org/core/ops/trait.CoerceUnsized.html#impl-CoerceUnsized%3C*const%20U%3E-2) would be unsafe to use, which is not a concept the trait system has right now).
    * But then upcasting is already a builtin operation. So we can totally have a rule that says "when doing an unsize coercion, look at source/target types and apply this ad-hoc safety rule".

The following code would no longer compile (is there a version of this code that doesn't require a feature gate?):

```rust
#![feature(unsize)]

use std::marker::Unsize;

fn unsize<A, B>(v: *mut A) -> *mut B
where
    B: ?Sized,
    A: Unsize<B>,
{
    v
}
```

## Other options?

We could potentially alter the vtable format to avoid needing a load but that is an undesirable limitation.

## Recommendation

Option 1: Require metadata for `*const` wide pointers to be valid

### Just have unsafe upcast?

* cramertj: I think there's another option: make raw pointer upcasting unsafe. Isn't that the place where the issue actually occurs?

### Special-case NULL?

* Josh Triplett: I don't know if this would be an excessive performance hit, but could we special-case the NULL pointer, return another NULL pointer in that case, and *otherwise* look at the metadata? This would effectively treat NULL as None for a raw pointer, and the upcast operation as a `.map`.
    * Related: this assumes it's OK for a NULL pointer to not have valid metadata.

### Why is it unsafe to create *const dyn Foo if it requires valid metadata?

* Mark: In particular, it seems like if you're creating *from* e.g. `&dyn Foo`, this should be safe, right?
* cramertj: did this perhaps just mean "from an integer or cast from other raw pointer"? I agree with Mark that `&dyn Foo -> *const dyn Foo` seems like it should be safe no matter what choice we make.

### Q: Why is `fn from_raw_parts` not an unsafe fn? 

* pnkfelix: is there any chance we could justify the breaking change of making it an `unsafe fn`?
 
### Q: Do we need raw pointer upcasting?

* Mark: Can we get away with just supporting upcasting for non-raw pointers, and then we "skip" this question? This is similar to Taylor's unsafe upcast, but is in some sense even simpler (and maybe in practice equivalent, presuming that the guarantees on upcast are equivalent to "needs to be safe to convert into &/&mut dyn temporarily"?).
 









