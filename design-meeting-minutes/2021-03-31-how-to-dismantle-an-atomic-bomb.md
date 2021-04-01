# 2021-03-31 How to dismantle an &Atomic bomb

* [Watch the recording](https://youtu.be/4iNVRZ34Km8)


(This is a summary/refinement of my [blog post on this topic](dismantle-blog-post))

[dismantle-blog-post]: http://blog.pnkfx.org/blog/2021/03/25/how-to-dismantle-an-atomic-bomb/

# The problem

Driving deallocation of state via ref-counting under our current atomic API means that you cannot avoid having a `&AtomicUsize` variable at the same time that you are deallocating the block of memory that contains the referenced atomic usize state.

To make things concrete, here's a couple types:

```rust=
type Data = [u8; 128];
struct LabelledPayload { label: char, data: Data }
struct Inner { ref_count: AtomicUsize, payload: LabelledPayload }
```

Consider this code snippet:


```rust=
{
    let inner: &Inner = [...];
    // ASSUME: just a normal deref+field project; no `Deref` trait involved.
    let label: char = inner.payload.label;
    [...]
    println!("finished processing {}", label); // ASSUME: sole use of `label`
}
```

Can you, as a compiler developer or an unsafe code author, expect to be able to safely transform the above into the *operational equivalent* of:

```rust=
{
    let inner: &Inner = [...];
    [...]
    println!("finished processing {}", inner.payload.label);
}
```

Lets call this the "Label Code Motion" transformation, or LCM.

Here's my argument for why you might think you can LCM: "The `inner` local variable is lexically scoped to the end of the block. The projection to `label` has no interior mutability, so it must be unchanged at the end of the block. So we should be able to delay the extraction of the label."

Here's the argument for why you cannot: Big surprise, our `Inner` holds the shared payload for a hand-crafted atomic ref-counted value (see [blog post][dismantle-blog-post] for full example):

```rust=
struct Handle {
    ptr: *const Inner,
}

impl Drop for Handle {
    fn drop(&mut self) {
        let inner: &Inner = unsafe { &(*self.ptr) };
        let label: char = inner.payload.label;

        // reminder: concurrent actors may also be running drop,
        // decrementing the ref-count...
        let pre_decr = inner.ref_count.fetch_sub(1, Ordering::Release);

        // ... so if `pre_decr > 1`, then someone else might be *freeing*
        // `*inner` (or may have *already* freed it). At this point in
        // control flow, the only time it is safe to access `*inner` is
        // if we can prove we are sole remaining owner (which is implied
        // by `pre_decr == 1`).

        if pre_decr == 1 {
            // missing: acquire-fence()
            unsafe { dealloc(inner as *const Inner); }
        }

        println!("finished processing {}", label);
    }
}
```

An alternative, more abstract argument might go along the lines of "non-lexical lifetimes means you cannot assume the lifetime attached to `inner: &Inner` actually lives to the end of `inner`'s' lexical scope." (Notably: I wrote "operational equivalent" above because we are not literally rewriting the source code itself, and we aren't talking about injecting safe code that would inform Rust's static-analyses that `inner` has to live that long.)

So: For Rust to support `Arc`, there needs to be at least *some* scenarios where the described Label Code Motion transformaion does not apply.

(Supporting `Arc` exactly as it is currently implemented is somewhat desirable; but we do have the option of tweaking its implementation in order to make the semantic model more expressive and/or easier to explain or implement.)

There are some variants on this example that we may want to keep in mind. Examples follow.

### version 2: "inner-ref handled out-of-line"

```rust=
unsafe fn may_drop_inner<'a>(inner: &'a Inner, label: char) {
    let pre_decr = inner.ref_count.fetch_sub(1, Ordering::Release);
    // (at this point, `pre_decr > 1` ==> someone else
    //  might be freeing `*inner`)
    if pre_decr == 1 {
        unsafe { dealloc(inner as *const Inner); }
    }
    println!("finished processing {}", label);
}

impl Drop for Handle {
    fn drop(&mut self) {
        unsafe {
            let inner: &Inner = &*self.ptr;
            let label: char = inner.payload.label;
            may_drop_inner(inner, label);
        }
    }
}
```

This version is notable since it moves the problematic code into a helper, and thus may get special treatment based on different assumptions made about function  parameters as opposed to local variables.

(This differs from the [blog post][dismantle-blog-post]; the blog post left the use out of the refactor, but I am no longer sure that illustrates what I thought it did.)

### version 3: "atomic-ref handled inline"

```rust=
impl Drop for Handle {
    fn drop(&mut self) {
        let ref_count: &AtomicUsize = unsafe { &(*self.ptr).ref_count };
        let label: char = unsafe { (*self.ptr).payload.label };
        let pre_decr = ref_count.fetch_sub(1, Ordering::Release);
        // (at this point, `pre_decr > 1` ==> someone else
        //  might be freeing `*self.ptr`)
        if pre_decr == 1 {
            unsafe { dealloc(self.ptr); }
        }
        println!("finished processing {}", label);
    }
}
```

This version is notable since it only keeps a reference to the `AtomiUsize`, not a whole `&Inner`. (Thus Label Code Motion cannot even be applied as described.) This kind of narrowing of focus may be a way to justify the correctness of the deallocation from other actors even in the face of transformations like Label Code Motion.

(Versions 2 and 3 together yield "atomic-ref handled out-of-line", where the refactored method is just a trivial wrapper around `AtomicUsize::fetch_sub`

# The Design Landscape

## Option 1. Outlaw them all

The heart of this option says: "You should never have a `&Struct` as a parameter or local (on current or ancestor stack frame) where `Struct` holds an atomic-ref-count and simulataneously decrement that ref-count."

And thus, it says that the Label Code Motion optimization is applicable to all the versions written above. They are all illegal code that should never have been written, should be explicitly documented as undefined behavior, *et cetera*.

Instead, in order to design an atomic-ref-counted type, you need to use unsafe pointers (i.e. `*const Inner` or `*AtomicUsize`) at the points where you are manipulating the reference-count.

There are sub-variants of this option.

### 1a. Outlaw them all and stick with current atomic API

This variant says:

> work with `*const T`, but cast the resulting `*AtomicUsize` to `&AtomicUsize` whenever you need to call operations like `fetch_sub`.

This raises questions like can" the developer similarly cast the `*const Inner` to a ``&Inner` and trust that the lifetimes will work out in their favor?" (I assert that doing so puts us into the same morass again.)

pnkfelix does not think this option is reasonable.

### 1b. Outlaw them all and put in alternative atomic API

This variant says:

> work with `*const T`, and the Rust stdlib will be extended `*const AtomicUsize` variants of all the current methods on atomics.

There are [drawbacks](http://blog.pnkfx.org/blog/2021/03/25/how-to-dismantle-an-atomic-bomb/#L1b..Outlaw.them.all.and.put.in.alternative.atomic.API) to this.

## Option 2. Allow them all

[Option 2 blog discussion](http://blog.pnkfx.org/blog/2021/03/25/how-to-dismantle-an-atomic-bomb/#Option.2..Allow.them.all)

## Option 3. Function boundaries are special

[Option 3 blog discussion](http://blog.pnkfx.org/blog/2021/03/25/how-to-dismantle-an-atomic-bomb/#Option.3..Function.boundaries.are.special)

## Option 4. Put deallocation on same footing as mutation

[Option 4 blog discussion](http://blog.pnkfx.org/blog/2021/03/25/how-to-dismantle-an-atomic-bomb/#Option.4..Put.deallocation.on.same.footing.as.mutation)

### Option 4a. Focus on Atomics (or any repr(transparent) UnsafeCell wrapper) alone

### Option 4b. Blast radius includes Buddy fields

## Option 5. Add marker to indicate that struct may be deallocated in a volatile manner.

[Option 5 blog discussion](http://blog.pnkfx.org/blog/2021/03/25/how-to-dismantle-an-atomic-bomb/#Option.5..Add.marker.to.indicate.that.struct.may.be.deallocated.in.a.volatile.manner.)

# Notes from meeting

## Q: Are you focused primarily on immediate-deallocation strategies, or do you also want this to support deferred-reclamation strategies (e.g. RCU or hazard pointers)?

Both but most immediate problem is Arc.

But also deallocating later is probably not "more UB" -- if we are allowed to deallocate now, we are also allowed to deallocate in 5 minutes.

But want to be sure we don't prohibit RCU-like strategies.

Conclusion: Let's defer this and focus on Arc.

## Design space exploraton

Axes considered:

* Considering just `&AtomicUsize` vs `&StructThatContainsUsize`?
* Is the reference just a local variable or a function parameter?
    * Because of the stack protector logic in stacked borrows, which is there to model LLVM dereferenceable
    * Also this enables key optimizations
* cramertj: where and when do we currently provide dereferenceable?
    * Ralf: I think we do it for all references, even `&UnsafeCell`.
    * Ralf: Including newtype'd references, which conflicts with `RefCell`, but that's a separate discussion.
* Proposed rule that prompted some discussion
    * "if you can mutate, you can deallocate"
    * has the advantage that a lot of existing code works
* Question: "if you can mutate, you can deallocate", how does that apply to structs?
    * Deallocation would apply solely to the area covered by the unsafe cell
    * This is enough for Arc as currently written
    * But the point of the blog post is to explore other things
* Ralf: This optimization is not actually correct, right?
    * Because of NLL?
    * Felix: true
* Josh: This first block of code is meant to be part of the drop, right?
    * Felix: yes
    * Josh: You may need a stronger barrier on the atomic operation to prevent the *hardware* from moving the load across the decrement.
    * Josh: The compiler reordering things past atomic operations seems like it will make many things break.
        * Ralf: This is a shared reference to something immutable, so I'm not sure that this applies.
* Ralf: actual optimization:
    * `fetch_sub` could add another read after the `fetch_sub` because of the protector
* Ralf: stacked borrows general rule is
    * references are valid until last use
    * except for function parameters, which last until the end of the fn (stack protectors)
        * this is to model dereferenceable and support optimizations
* Felix:
    * worth pointing out that some folks might *think* the optimization is not broken

To confirm, this code is UB under most variants, because:

* we deallocate memory (`inner.label`) to which we have shared, read-only access
    * and, specifically, because `inner` is a function parameter, such access lasts until the end of the function

```rust=
struct Inner {
    ref_count: AtomicCell,
    label: char,
}
unsafe fn may_drop_inner<'a>(inner: &'a Inner) {
    let pre_decr = inner.ref_count.fetch_sub(1, Ordering::Release);
    // (at this point, `pre_decr > 1` ==> someone else
    //  might be freeing `*inner`)
    if pre_decr == 1 {
        unsafe { dealloc(inner as *const Inner); }
    }
    
    // compiler could e.g. insert a synthetic read of `inner.label` here if it wanted
}
```

Related example:


```rust=
struct Inner {
    ref_count: AtomicCell,
    label: char,
}
unsafe fn may_drop_inner<'a>(inner: &'a AtomicCell, label: &char) {
    let pre_decr = inner.ref_count.fetch_sub(1, Ordering::Release);
    // (at this point, `pre_decr > 1` ==> someone else
    //  might be freeing `*inner`)
    if pre_decr == 1 {
        unsafe { dealloc(inner as *const Inner); }
    }
    
    // compiler could e.g. insert a synthetic read of `inner.label` here if it wanted
}
```


* to make the code correct, you modify `&Inner` to `*const Inner`
* if we want the code with `&Inner` to be correct, need "bubbling":
    * what does bubbling mean? bubbling means that the effects of `UnsafeCell` propagate out to adjacent fields that are covered by a single reference
    * so e.g. because `&Inner` contains an `UnsafeCell`:
        * then it has effects on the other fields of `Inner`, even those not contained within the `UnsafeCell`

* Can we put dereferenceable on `&Inner` if we take the 'can free the bytes you can mutate' rule?
    * Maybe. 
    * If there is at least one (non-padding) byte not inside an unsafe cell, 
        * and because we know that you cannot free just some bytes,
            * we know that the unsafecell is also not being freed,
                * so maybe dereferenceable is ok.
* Restated Rule (nicknamed 'mutation is freedom' by Niko, who will accept no other name):
    * In order to free something, you must be permitted to write to every byte of it
        * where "permitted to write to every byte of it" includes all of `(UnsafeCell<u16>, UnsafeCell<u8>)`? (includes writing to padding)
* Implications under "mutation is freedom":
    * the code with `&Inner` is not correct
* One alternative
    * could add flags to the struct;
    * but then the struct can be embedded elsewhere?
* Compared to bubbling out from unsafe cell?
    * Advantage to unsafe cell bubbling: simplifies the model
    * Disadvantage: no `dereferanceable` on non-`Freeze` types
        * Freeing is *slightly* different from mutations because it's a *final* mutation
            * seeing a further read means we know that it is still live
* Should Drop take `*mut`?
    * no -- compiler will go on to touch other fields, `&mut` is better for that reason (though it has some flaws)
* Does "allow them all" ("bubble") make `&` more subtle?
    * One could argue the opposite, and say that with bubbling `&T` means two things:
        * if it is `T: Freeze`, it cannot be freed or be mutated
        * otherwise, it cannot be mutated outside of an unsafecell (but it can be freed)
* Downsides of "mutation is freedom"
    * No dereferenceable on `&UnsafeCell` or newtypes thereof
        * We believe we can have dereferenceable on `T: !Freeze` if there are extra fields (e.g. `&(i32, Cell<i32>)`), but this is not entirely obvious
    * Moves us quite a bit closer to "mutation is volatile"
* "mutation is volatile freedom"
    * Means: "anything you can write, you have volatile reads"
    * No dereferenceable on `!Freeze` types (like bubbling)
* Downsides of "bubbling mutation"
    * No dereferenceable on Freeze types
    * You can expect that 
* Downsides of non-bubbling "volatility" (`NonBubblingVolatileCell<T>`)
    * No dereferenceable on 
* Downsides of bubble "volatility" (`BubblingVolatileCell<T>`)
    * No dereferenceable on 
* Downsides of "unprotect the atomic methods"
    * No new API surface is required, but atomic cells are now very special
    * You can't pass around `&AtomicUsize` but the docs suggest you can
* Downsides of "raw-everywhere"
    * We need a whole new API surface for `fetch_sub` and friends
    * Arc is not correct as written