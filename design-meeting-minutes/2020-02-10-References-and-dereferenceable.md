# Lang Team Design Meeting: & and Deferenceable

* [Watch recording](https://youtu.be/mnFcQuxeUGI)

# Topic

This design meeting will cover problems around `&` and the dereferenceable attribute, such as [https://github.com/rust-lang/rust/issues/55005](https://github.com/rust-lang/rust/issues/55005) and MMIO.

# Links
-  [Arc::drop has a (potentially) dangling shared ref #55005](https://github.com/rust-lang/rust/issues/55005) 
- Ref and RefCell lifetimes are a “lie” https://github.com/rust-lang/unsafe-code-guidelines/issues/125
- Japaric’s embedded RFC https://github.com/rust-embedded/wg/pull/387
- MMIO and volatile memory conflicts https://github.com/japaric/volatile-register/issues/10
- Zulip conversation about dereferenceable attributes and LLVM ([link](https://rust-lang.zulipchat.com/#narrow/stream/187780-t-compiler.2Fwg-llvm/topic/deferencable.20attribute.20.20.2366600))
    - LLVM “introduce `dereference_globally`" https://reviews.llvm.org/D61652
    - LLVM “add/infer a `nofree` function attribute” https://reviews.llvm.org/D49165
# Agenda
- Current behavior:
    - The meaning of the “dereferenceable” attribute in LLVM
        - we have this attribute, using it to model C++ references, but realized that it is not what people need
        - the intended semantics was vaguely “this thing will be dereferenceable through the function call”
        - but the C++ spec doesn’t say that, the function can deallocate the memory backing a `T&` during course of the function, it just has to be “dereferenceable initially”
            - they intend to fix this
        - there has been discussion of how best to fix, so that you can express that something is dereferenceable *initially* 
    - What changes are being considered in the future from LLVM’s perspective?
        - a “nofree” function indicates that it doesn’t free *anything* 
            - so maintains dereferenceability (if you ignore concurrency)
            - with concurrency, you have more problems (c.f. Arc)
        - a “nosync” function doesn’t “interact with other threads” (no atomic ops, no loads/stores, no synchronizes-with edges introduced)
        - “nofree” can also be put on parameters, perhaps, to indicate that data behind a particular pointer is not freed
        - dereferenceable will change to have the *weaker* semantics to (ret-con) compatibility
        - `dereference_globally` means it will never be derefered ever? maybe useful, but not for Rust
    - How do we justify “derefenceable” in stacked borrows?
        - Context: We intend to justify emitting derefenceable because of UB caused under stacked borrows. If we were going to try to **stop** emitting dereferenceable, we’d presumably also have to adjust stacked borrows to make the pattern not UB. Right?
        - Answer: we justify by introducing these “protectors” on entry/exit from function
            - when you push something onto the stack, there’s an optional flag to say “this is protected by that function item”
            - when you pop an item that is “protected by” a fn that is ongoing, that is UB
            - this is “per item” — i.e., per element on the “stacked borrows stack”
            - a side effect of protectors is that having two `&mut` arguments that wind up being given the same value by caller, that is UB due to adding the protectors
                - Stacked Borrows - An Aliasing Model for Rust, https://www.youtube.com/watch?v=h9Fh4jRDGLo
- What kinds of bugs do we run into
    - `[Arc::drop](https://github.com/rust-lang/rust/issues/55005)` [#55005](https://github.com/rust-lang/rust/issues/55005)
        - thread A has an `AtomicUsize`, it does a decrement (and ref count reaches 1)
        - thread B does a decrement (and ref count reaches 0) and 
        - Stacked Borrows issue: https://github.com/rust-lang/unsafe-code-guidelines/issues/88
    - MMIO and volatile memory conflicts https://github.com/japaric/volatile-register/issues/10

Quoting Ralf: “They want to define structs like…


    struct MyRegisters {
      reg1: ReadOnly,
      reg2: WriteOnly,
      reg3: ReadWrite
    }

…and then hand around references to such structs as their encoding of MMIO registers.  But *any* reference currently permits spurious accesses, which obviously they cannot have.”

However, in this case, `ReadOnly` etc are presumably wrappers around a `VolatileCell` which in turn raises the question of how `VolatileCell` is to be implemented.

The RFC proposes various alternatives:

- “ZST all the way down” — use a unique type for every hardware set of register
    - each such type implements `read() → u8` or whatever, which is implemented by  something like ptr::read(0xXXXXX as *const u8)` 
- “VolAddress” — create a `VolAddress` type whose *value* is the address to read from; the `read` op is then `ptr::read(self.address as *const u8)`
    - [not zero cost](https://github.com/rust-embedded/wg/blob/deferenceable-volatile/rfcs/0000-deferenceable-volatile.md#non-zero-cost)


- Proposal 1 (addresses MMIO, not Arc):
    - can we make raw pointers sufficiently ergonomic?
    - so you are passing around `*MyRegisters`
    - ergonomic hits from raw pointers:
        - integration with methods and traits (`*const self`)
        - some form of postfix deref (or even auto-deref?), so that you can do 
        - first-class `&x` form that is nicer than `&raw const x`? autoref?
    - concern: “viral”
        - you can’t have a “safe struct” that contains 


- Proposal: remove dereferenceable for `&UnsafeCell` ([comment](https://github.com/rust-lang/unsafe-code-guidelines/issues/33#issuecomment-429112051))
    - What does this mean in terms of stacked borrows?
        - make adding protectors dependent on the type?
        - how infectious is it? if you have a `(u8, UnsafeCell<T>)`, does it affect the `u8`?
            - the effect of an unsafe cell is already “expanded” to cover enum variants
            - but not presently other struct fields
        - what about generic contexts (it interacts with optimizations potentially)
        - is this just about references? what if you newtype a reference and pass it to a struct, or pass in a `struct BunchOfRefs<``'``a> { .. }`?
            - the drain iterator of a vector is a struct that contains a `&``[T]`, and it (potentially) deallocates memory that is referenced by that `&` 
                - https://github.com/rust-lang/rust/issues/60076
            - two more instances: `Ref` and `RefMut`, the RefCell handles
            - https://github.com/rust-lang/unsafe-code-guidelines/issues/125
    - What does this help, and what *doesn’t* it help?
        - Ralf mentioned “this helps *some* cases…”, which cases doesn’t it help?


    // a pattern like this wouldn't necessarily be helped
    fn foo(x: &AtomicUsize, y: &Bar) {
      // in particular if the `AtomicUSize` is not in the `y`
      //
      // imagine say that all `Bar` are handles to some global resource
      // with a single atomic counter
      if x.decrement() == 0 {
        free(y);
      }
    }
        - doesn’t help with [“the drain problem”](https://github.com/rust-lang/rust/issues/60076) — where drain internally has an `&` without `UnsafeCell` to “the things still to be drained”, but there is some function you can call that will cause the
        - `RefMut` has a `&mut` references, so it does not help there either
    fn foo(x: RefMut<'a>, y: &RefCell<..>){
      // the `&mut` inside `x` gets a *protector* on entry here
      drop(x); // the "lock" is released
    
      y.borrow_mut(); // this will pop the protector, leading to UB
    }
        - problem is not confined to `&UnsafeCell<T>` --
            - the “disconnected” Arc example
            - drain is `&T` without `UnsafeCell`
            - `RefMut` has `&mut` 
                - interestingly, the `RefCell` test suite never hits the bad pattern here :(


    - One observation is that we would [lose some amount of optimization](https://github.com/rust-lang/unsafe-code-guidelines/issues/33#issuecomment-429114003),  do we have any idea how much?
        - We might enable a “opt back in” for types like `Cell`
    - Related to, but distinct from, the decision to forbid niches within `UnsafeCell<T>`
        - Like with niches: part of this is that `UnsafeCell` presently implies multiple threads potentially accessing the data
        - Unlike with niches: the violations of dereferenceable are not *inherently* caused by the presence of multiple threads, but more the things that threads commonly want to do (e.g., maintain ref counts)
        - Also: there are other patterns unrelated to `&UnsafeCell`, as noted above


- One actionable thing is to try to make raw pointers more ergonomic, so that the rule of thumb
- One thing these have in common is that you are “lieing” about a lifetime somewhere
    - they’re all cases where you want to have a pointer that is potentially dangling
    - we have `NonNull<T>` today but very unergonomic
- boats has been pondering the idea of a pointer in between raw and ref
    - `&unsafe` is working title, would also be an operator `let x = &unsafe y`
    - “can dangle” but can’t be null, may not be aligned, not safe to read
    - `Option<&unsafe T>` would get the niche optimization


    &unsafe T // non-null, non-aligned, but no lifetime (unsafe to deref) coerces to *const T etc
    &unsafe mut T


    - impact “virality” for safe abstractions:
        - it feels weird that `&unsafe RegisterBlock` is the abstraction
        - you would make `struct RegisterBlock``Ref` `{ &unsafe RegisterBlock }`
            - this has limitations, but is analogous to `Ref` (and hence has things like `Ref::map`)
            - maybe we could do something that makes that more natural, some kind of “safe projection” mechanism
            - unsafe references makes it easier to build, 
    - one thing we could do:
        - emit ‘dereferenceable-on-entry’, which solves most of the above problems
            - would still mean that `fn helper(&self)` (which never uses)
        - but not MMIO (which are never dereferenceable)
        - also gives up most optimization potential
    - connection to editions:
        - we might want to reserve some syntactic space here?
        - but `&unsafe` already dodges most of the problems, `&unsafe {` is the only “overlap” today


    &unsafe { 0 }
            - You can use 1 token look-ahead to parse `&unsafe { … }` in the way it is parsed today.
        - and would we maybe want to reserve `raw`
            - stdlib uses it, as do a number of libraries (e.g., C API wrappers may put the extern API in a raw module)


- Niko felt like they came in wanting to remove `dereferenceable` from `&UnsafeCell` but now are less sure that is a good choice
    - but if we “just” added `&unsafe` would that help to address the `Arc` problems? The APIs on `AtomicUsize` etc exist, and they take `&self`…?
        - presumably, if we made no other changes, either `fetch_sub` etc have to be modified to take `&unsafe self` (breaking change? maybe propagates to things like `AtomicCell` from crossbeam too..?)
        - or else we add new methods that take `&unsafe self`
            - and we have to propagate that back to things invoked from within the destructor


- two main ways to address existing `&self` APIs:
    - “remove derefenceable attribute entirely from `&UnsafeCell<T>`"
        - open design question: maybe still mark the parts outside `UnsafeCell` “dereferencable”? Or make it fully infectious?
        - pros
            - addresses MMIO and existing Arc APIs
                - a partially infectious solution would still break in a simple variant of `Arc` that has a helper method `fn decrement(&ArcInner)`
            - maintains a lot of optimization potential: the no-longer-deref pointers are anyway escaped and do not have `noalias`, so they do not get strong optimizations
        - cons
            - `RefMut` and `Drain` are unfixed (would have to check for `Ref`)
                - would need to use raw pointers
            - similarly trying to separate ref count from memory to be freed
            - more complex, type-dependent behavior
                - may interact with trying to do optimization pre-monomorphization (though not more than what we already do with `noalias`)
            - have to figure out “how infectious” `UnsafeCell` is
                - but, this is already somewhat true, since `UnsafeCell` has an impact on `&T`
    - “downgrade all derereference attributes to derefenceable-on-entry”
        - pros
            - simple and uniform
            - fixes everything except for MMIO
        - cons
            - we lose out on the more advanced, protector-based optimizations described in the stacked borrows paper (the ones that *extend* the lifetime of a pointer, moving an access down across call where previously the pointer was not used after the call)
            - doesn’t address MMIO
                - would need to use newtyped-wrapped `&unsafe` ptr or so
    - “downgrade attributes on `&UnsafeCell` only” — a mix of the two above
    

