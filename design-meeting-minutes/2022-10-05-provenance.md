# Do we need provenance?

###  What is provenance?

Provenance is the umbrella term for any state that a pointer holds in addition to the address it points to. In other words, a memory model has a concept of provenance if it is possible for two pointers to be numerically equal, but different in some other dimension. I will briefly give two examples of what provenance can look like:

 - C and LLVM both have "allocation level provenance." This means that every pointer "remembers" which allocation it belongs to, and using a pointer that originated in one allocation to access memory in another is undefined behavior.

 - [Stacked Borrows](https://plv.mpi-sws.org/rustbelt/stacked-borrows/paper.pdf) is a proposed aliasing model for Rust. The provenance it suggests consists of two parts: An **allocation id**, used to implement the same "allocation level provenance" as described above, and a **tag**, which describes what part of the borrow stack a pointer belongs to.

To emphasize: It is not the goal of this meeting to discuss which kind of provenance we should have, only to discuss whether we should have *any*.

### Reasons not to have provenance

The biggest downside of provenance is complexity. The existence of provenance means that authors of unsafe code must always not only be concerned with whether the pointer they have points to the right place, but also whether it has the right provenance (in practice, this means "was obtained the right way"). Not having provenance ensures that this is never a problem - all pointers that point to the right address are equally valid to use.

### Reasons to have provenance

#### Optimizations

Many (most?) optimizations done by compilers require some form of *alias analysis*. This is an analysis that reports when two memory operations might alias each other. Alias analysis benefits greatly from notions of provenance since this generally means there is more UB and more information with which to justify optimizations. For example, without provenance, the following program must be DB as the two marked lines are entirely equivalent. This means it is unsound to optimize to `print(0)`. However, with allocation level provenance it is possible to call this program UB (i.e., it is possible to declare that `q` and `p+1` do not alias) without the commented line being UB, and so the optimization can be permitted.

```c=
char p[1], q[1] = {0};
uintptr_t ip = (uintptr_t)(p+1);
uintptr_t iq = (uintptr_t)q;
if (iq == ip) {
  *(p+1) = 10; // <-- This line
  // *q = 10; // <-- And this line
  print(q[0]); // can be optimized only with provenance
}
```

Similarly, it has long been desirable for it to be sound to optimize code like this:

```rust=
fn foo(x: &mut i32) -> i32 {
    *x = 10;
    bar();
    *x
}
```

It's very difficult to see how to make this optimization sound without provenance. Ralf had [attempted](https://www.ralfj.de/blog/2017/07/17/types-as-contracts.html) such a model in the past, but it was unsuccessful in a number of ways. 

#### LLVM

LLVM IR (despite its lack of a clear spec) recognizes a notion of allocation level provenance. Compiling Rust to LLVM IR if Rust does not recognize provenance is likely to be impossible. We'd probably have to insert a `black_box` after every allocation and every memory access, and it's not clear that that is enough. As far as I know there is no option to turn this off, and the assumptions are sufficiently widespread that it is unlikely that we could convince upstream to add one.

---

### Appendix: Stabilizing parts of the strict provenance APIs

This topic initially came up because Ralf suggested [on Zulip](https://rust-lang.zulipchat.com/#narrow/stream/213817-t-lang/topic/Stabilizing.20strict.20provenance.20APIs.3F) that parts of the strict provenance APIs be stabilized. This lead to some confusion, and so I want to go over here exactly what is being proposed, and the reasoning for it.

The proposal consists of two parts:

 - Stably acknowledge the existence of provenance, ie acknowledge that the answer to the question in the title is "yes"
 - Stabilize the following methods: `pointer::addr`, `pointer::with_addr`, `ptr::map_addr`, `core::ptr::invalid`, `core::ptr::invalid_mut`

:::spoiler Brief description of what they do
 - `addr`: Yields the address of the pointer as a `usize`. There is no expectation that the address can be turned back into a usable pointer via an `as` cast.
 - `with_addr`: `p.with_addr(a)` yields a pointer that has the same provenance as `p` but the address `a`
 - `map_addr`: Literally
    ```rust=
    fn map_addr(self, f: impl FnOnce(usize) -> usize) -> Self {
        self.with_addr(f(self.addr()))
    }
    ```
 - `invalid`, `invalid_mut`: Yield a (possibly `mut`) pointer that has the given address, but some kind of null/None provenance and so is illegal to dereference.
:::

That is the entirety of the proposal. We are not suggesting any change to the memory model that would make these APIs required to write correct code. There are also other methods under `#![feature(strict_provenance)]`. We are not suggesting stabilizing those at this time. Some of those methods imply a PNVI-like treatment of ptr2int and int2ptr casts. (PNVI being an explicit and detailed provenance proposal for C.) We are also not suggesting stabilizing those concepts at this time.

The "main goal" of the these APIs is to allow users to operate on the provenance and address of a pointer separately. This in turn allows users to be precise about how they handle provenance, which makes the meaning of their code clearer both to humans and to the compiler, which will have more opportunities to optimize.

It is Jakob's (and others?) opinion that these APIs represent functionality that is fundamentally necessary in a language with provenance. The `expose*` methods and the associated semantics are not an alternative, but rather a possible addition. (Of course, that does not necessarily mean that this particular combination of functions is the right way to expose this functionality, we are still free to bikeshed that :) )

---
## Discussion 

### invalid/invalid_mut take a parameter

Yes. It just constructs a pointer with the given address that you *cannot use*

Use cases:
- Making pointer to zero-sized types (which you cannot use) without making null serve that purpose (because you want to preserve its usage as a niche)
- If you're interacting with some API (via FFI or otherwise) that accepts a pointer but that pointer does not need to actually access memory
    - Q from Felix: How would you know if the FFI requires provenance-tagged pointers? (I guess API of foreign function will promise to not dereference it)
        - Jakob: Yes.
    - Gary: Can we define a ZST deref to pointers with `invalid` provenance as UB and have a special provenance for ZSTs that allow dangling use of it in `Vec`? 
        - Ralf says defer this for future meetings.
- Q: Can this be used with embedded when I know there is some memory at a fixed address?
    - Jakob: Probably not. We don't have a good story for these kinds of allocations yet, but I would be surprised if this was the case
    - Gary: I don't think so, you would have to use `from_exposed_addr` for that.
    - Felix: maybe would need some notion of `volatile` in order to have the original question's proposal make sense, in terms of stopping compiler from optimizing things in way that interfere with such accesses? (Or just use `from_exposed_addr` as suggested.)
     
     
     
## Questions

### LLVM workarounds

Scott: Black box "every allocation" would need to do this for `alloca`s too, right?  Not just `__rust_alloc`?

Ralf: Yes. Heap and stack allocations are not fundamentally different.

These next two are Scott attempting to summarize after the fact:

> Scott: ok, so even if we had a way to do this in LLVM, it would be completely unacceptable because it would mean nothing goes into registers ever and performance would be terrible
>
> Ralf: Adapting LLVM is almost certainly not going to work, yes, so it would need a compiler build to not need provenance

Josh: Wouldn't completely rule out use of registers, but it'd require compiler backend support that doesn't exist and nobody is working on or has plans to work on.


### LLVM without provenance

Felix: are there languages (with references) with LLVM backends that do not support provenance? (I'm imagining gymnastics one could do, e.g. massive-array for memory + every pointer is an integer, but there's not much point in going down that road unless there's some existing evidence that it can ever work out well.)

Jakob: Hard to answer in the negative, but I don't know one. ~~As a reference example, Go uses LLVM as well, but (based on my limited understanding) its rules are "what LLVM does."~~ (Edit: wrong)

### Can we use aliasing rules instead of provenance

Gary: Can we use aliasing rules to cover many optimisations without having provenance?

Discussion: our goal is to develop a model that would justify code that is leveraging Rust's current rules e.g. `&mut`; but those syntactic rules alone are not sufficient to be able to e.g. justify the code that exists in the Rust stdlib itself.

See also Ralf's earlier model linked above (https://www.ralfj.de/blog/2017/07/17/types-as-contracts.html)

### Does comparison between pointers take provenance into account?

Ralf: No, it doesn't; comparison ignores provenance.

### (Defer until after "should we have provenance" is answered): Should we have `expose` or not?

Josh: Yes, we should. Seems necessary for widespread adoption of provenance.

Gary: We still have `as` which needs to be using the `expose` for compatibility reasons, right?

Scott: Yes, I think we need it, because <https://github.com/rust-lang/rust/pull/95583> says to me that if we don't have the method to compare against, people will start using `.addr()` in a place where they needed to expose.

### How useful is `invalid` and `invalid_mut`?

Gary: Specifically, what optimisation would we gain from them compared to just use `from_exposed_addr()`?

Connor: Because the compiler knows the pointer cannot be dereferenced, it can assume the pointer does not alias any other pointer.

Ralf: Also the entire point of these APIs is not to use `exposed`/`as`.


### What other arguments are there against provenance?

y86-dev: I have the feeling that the arguments are highly in favor of provenance. Are there any other arguments other than increased complexity?

Josh: "doesn't match the mental model many programmers have", but that still doesn't change how compilers work. :(

### What are we telling people?

nikomatsakis: I'm catching up here but I'm trying to understand what we would be telling folks if we were to stabilize these APIs. The text above explicitly says they're not needed to write correct code. But there is *something* we're deciding here, I'd like to see it written down. Is it...

> The final model is not determined, but if you can use these APIs as described, your code is free of UB (due to aliasing restrictions)

?

Jakob: I think we are telling people 1) provenance exists and your code might be wrong if it pretends it doesn't, and 2) these APIs exist and they do what they say they do

points raised during call:

* these APIs protect you w/r/t pointer-to-integer conversions, but it'd still be possible to violate `&mut` or other aliasing restrictions

One thing I would make sure is clear about for if and when the wider audience sees this announcment is that this affects unsafe code, and won't affect safe.

Josh: Is expose well-defined at this point, can people use that without UB?

ralfjung: it has a spec, but it's more questionable, uses angelic non-determinism. If you use integer-to-pointer casts to access memory that doesn't alias Rust data, would definitely be ok (e.g., a magic number outside domain of Rust you'd like to use a a pointer).

### should we have expose at same time?

scottmcm: people will try to do this anyway, using addr if they have it, as you can see from the /from-bits/to-bits/ discussion

jakob: as casts are probably a better way for people to write this, we have to deal with those anyway (--interpred, ed.)

joshtriplett: I think we should not steer people towards new uses of as-casts, if we're going to give guidance, let's steer them to something like expose

ralf: can't replace all the code that uses `as` with these APIs, we should encourage folks to use those subset when they can

joshtriplett: saying let's not have people switch *to* `as` casts

jakob: but stabilizing expose feels premature, still concerned about possible effects of angelic nondeterminism


















