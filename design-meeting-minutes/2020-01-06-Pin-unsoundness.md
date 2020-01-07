# Pin soundness design meeting 2020.01.06

# Links
- [Internals thread](https://internals.rust-lang.org/t/unsoundness-in-pin/11311/)
- [Video recording](https://youtu.be/MX_GRNLhlY8)

# Meeting notes
- Were 3 unsoundnesses, but one got fixed
    - One, for `PartialEq`, which we resolved in [PR #67039](https://github.com/rust-lang/rust/pull/67039). 
- Background:
    - `Pin<P>` is a contract between 3 parties
        - the one who defines `P::Target` ? (the `Deref` impl)
        - the person who calls a method that takes a `Pin<P>` argument
        - the one who defines `P` (typically standard library thus far)
    - stability note:
        - it is technically possible to define new `P` pointer type in your own code
        - but it typically requires unsafe code — `new_unchecked` 
            - but contract here is rather ill-defined, we need to define it
- Previous versions of API had many types
    - `PinBox`
    - Pin(Mut)Ref
    - but decided to abstract over `Pin<P>` without realizing that the constructor soundness
- Unsoundness can occur in two directions
    - Construct a `Pin<&T>` to address `A`
    - Somehow in your code you get a `Pin<&mut T>` to address `B` which is not actually pinned
        - This “somehow” is through an “evil” `DerefMut` impl (or `Clone`)
            - previously you could do this with `PartialEq` and friends but closed that in PR #67039 (above), so that `Pin<P>` always invokes `<P::Target as PartialEq>::partial_eq` etc
    - How sure are we that it is limited to `Deref/DerefMut/Clone`? Pretty sure because `Pin` doesn’t have methods that use other traits
- 


- [Boats](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=980b86aeb02c923e25937297e5018ef6)’[s reproduction](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=980b86aeb02c923e25937297e5018ef6) is by far the most minimal I saw

      // A !Unpin type to reference
      struct MyType<'a>(Cell<Option<&'a mut MyType<'a>>>, PhantomPinned);
    
      impl<'a> DerefMut for &'a MyType<'a> {
          fn deref_mut(self: &&mut MyType<'a>) -> &mut MyType<'a> {
              self.0.replace(None).unwrap()
          }
      }
    
      fn main() {
          let mut unpinned = Box::new(MyType(Cell::new(None), PhantomPinned));
          let pinned: Pin<Box<MyType<'_>>> = Box::pin(MyType(Cell::new(Some(&mut unpinned)),
                                PhantomPinned);
          let mut pinned_ref: Pin<&MyType<'_>> = pinned.as_ref();
    
          // call the unsound DerefMut impl
          let pinned_mut: Pin<&mut MyType<'_>> = pinned_ref.as_mut();
        
          // pinned_mut points to the address at `unpinned`, which
          // can be moved in the future. UNSOUND!
          let unpinned1 = *unpinned;
      }


- Two routes to fix, essentially, not incompatible
    - make the `impl` of `DerefMut` for `&LocalType` illegal (somehow)
        - long term: `impl<T> !DerefMut for &T` 
            - declares that this impl can never be added
    - add another (unsafe) trait `PinDerefMut` that is required to invoke `as_mut`
        - implemented for `Box`, but not for `&T`
        - could be implemented for `&MyType`
- Right now how it is meant to work:
    - If you define a pointer type `P` that can be pinned to a `Pin<P>`, that is unsafe (unless `P::Target: Unpin`)
        - you are responsible for proving stuff that makes `as_ref` etc safe
- Impact of the two options:
    - First disallows the `DerefMut` impls for `&T`
    - Second disallows custom pointer types (but we know of none that exist)
        - investigate https://docs.rs/pin-cell/0.1.1/pin_cell/
    - Second also might impact generic code?
        - `fn foo<T: DerefMut>(p: Pin<…>)`, right?
- What about `Unpin`?
    - If we want to get control over which pointer types are used, we can’t fully do that, because any pointer type whose target is Unpin...
    - …is that sufficient?
    - in other words, we can get control over “all pointer types whose targets are not unpin”
- Key question for a formal proof:
    - Can we describe the invariant for `P`?
    - We have a formal model of pinning, but it’s the “old style” with custom types for each pointer type
        - Each has distinct invariants
    - But nothing like the generic `Pin<P>` for any `P`



    // Unsafe trait contract:
    // *define* the pinned invariant of this pointer
    unsafe trait DerefPin: Deref {
    }
    
    struct Pin<P: DerefPin> {
      // whatever DerefPin says
      inner: P
    }
    
    // Define the invariant for `&T` here
    unsafe impl DerefPin for &T { }


- can we define the invariant on `new_unchecked`?
- what does the invariant look like?
    - invariant is ultimately up to the target type, but the key is that the *pinned* invariant of the type can refer to the location of the type instance in memory
    - for a self-referential generation: “oh when I’m pinned, I know that my field address won’t change, I can rely that it is still the same in between invocations of poll" (or something..?)
- pointer type has to prove:
    - when you call `DerefMut` on a pinned mutable reference:
        - “that is ok”
        - the return type is also a `Pin<&mut>`
- what do the “docs” look like?
    - If we rely on negative impls:
    - If we do not:
        - When you unsafe impl `PinDeref`
- key distinction (for Ralf) is that the `unsafe impl` is where the “pinned reference invariant” is defined
- ..discussion around `Pin::new` that is hard to summarize..
    - in Ralf’s proposal, you would not be able to do `Pin::new` without implementing `DerefPin`
- argument for a coherence-based solution
    - most unsafe APIs that people use will not be formally proven
    - informally, people are able to generally reason that a “type doesn’t implement a trait”
        - generally works because of orphan rules
        - but doesn’t work for fundamental types (&T, Box) — oops
    - people will generally write APIs that rely on orphan rules and we should accept it 
- Ralf tried to sketch an invariant for Pin based on that principle
    - some value `v` is valid `Pin<P>`
        - if `P: Deref<Target = T>`, then: 
            - calling deref on `&``v` must be “ok” and its return value is a valid `Pin``Shr``<T>`
                - “PinShr” is sort of a newtype’d `Pin<&T>` with specific invariants that we have defined
                - “this is a pointer to some memory that satisfies the PinShr invariant of T”
        - if `P: DerefMut`, then
            - calling `deref_mut` must be ok and its return value is a valid `PinMut<T>`
        - if `P: Clone`, then
            - must be ok to call clone and must return a … 
    - `Pin::new` is now justified:
        - this is true because `PinShr<T> = &T` in terms of invariant when `T: Unpin`
    - incorrectness of `&T` impl is now located at definition of  `as_ref`, since it could not have proven the second clause
- when you invoke `new_unchecked`, you have to prove that for that specific value, the conditions above hold
    - a shortcut would be to say that there is no impl, but we would really want a ‘positive statement’ of the fact that there is no impl
        - this also helps with extension of orphan rules
- centril: do you get to assume in coherence that the impl would never exist?
    - boats: point of the negative impl would be to assert that it doesn’t exist
    - does orphan check get to rely on that?
        - would be useful but we don’t *have* to allow that
- negative reasoning is something to be careful of
    - e.g. `where T: !Foo` could only work if we required an explicit negative impl, and even then we’d want to be cautious
- we’d want the invariant to require an explicit negative impl
    - ie, you must have the impl and talk about it
    - or have a negative impl so that you don’t have to 
- we could then take the formal steps to make proofs without fully modeling the trait system
    - we could “modularize” the proof to say “we assume that `&T: Deref` is implemented and has these properties” in isolation
- plausible to tell people:
    - if you are going to rely on orphan rules, you need a negative impl
        - they may overlook it but oh well
- who would need negative impls?
    - specifically for &T, &mut T, Box, Rc, and Arc — and any pointer types that use `new_unchecked`
    - all need a negative impl of DerefMut (except for Box)
    - and `&mut T` neegs a negative impl of Clone
- what does fundamental have to do with this then?
    - as far as we know, orphan rules guarantee the above for all cases but `&T: DerefMut`
- to make a `Rc<T>` you can do `Rc::pin`, only possible because it must be newly created
- to fully correctly and stably use `Pin<P>` you would need to use a negative impl
    - but “for now” folks *could* rely on coherence
    - but the risk is if we “improve” our orphan impls…
        - `OwningRef` or `Rental` were broken by the same exploit...
- action steps
    - explore adding the explicit negative impls
        - implementation (relatively easy, Niko asserts)
        - longer term plans
    - `CoerceUnsized`.. make it unsafe?? (need more discussion?)
        - “just” another way to construct a `Pin`
        - problem is that user could define a `CoerceUnsized` that doesn’t meet the requirements
            - unsafe impls of `CoerceUnsized` will have to verify the condition, boats claims “same as `new_unchecked`”
        - we could add a marker trait `PinCoerceUnsized` here
    - write down the proof obligations for `new_unchecked` in some form
        - good to compare relative complexity of the various proposals




