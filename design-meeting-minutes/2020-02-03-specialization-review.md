# Specialization Review, 2020-02-03

## Watch

[Recording available here](https://youtu.be/7_7gUxnbenU)

## Summary

I plan to review specialization with a focus on "the soundness problem".

## Links

The specialization design a has a long history. Here are some of the documents and blog posts I reviewed.


- [RFC 1210](https://github.com/rust-lang/rfcs/blob/master/text/1210-impl-specialization.md)
- [Intersection impls](http://smallcultfollowing.com/babysteps/blog/2016/09/24/intersection-impls/)
- [Lifetime dispatch](https://aturon.github.io/blog/2017/07/08/lifetime-dispatch/)
- [Maximally minimal specialization: always applicable impls](http://smallcultfollowing.com/babysteps/blog/2018/02/09/maximally-minimal-specialization-always-applicable-impls/)
- [Sound specialization](http://aturon.github.io/tech/2018/04/05/sound-specialization/)

Here are some more links I didn't get to review as thoroughly


- [Distinguishing reuse from override](http://smallcultfollowing.com/babysteps/blog/2016/09/29/distinguishing-reuse-from-override/)
- [Supporting blanket impls in specialization](http://smallcultfollowing.com/babysteps/blog/2016/10/24/supporting-blanket-impls-in-specialization/)
- [Specialize to reuse](http://aturon.github.io/blog/2015/09/18/reuse/)

More links:

- Associated type defaults, https://github.com/rust-lang/rfcs/pull/2532
    -  Implement RFC 2532 – Associated Type Defaults, #[61812](https://github.com/rust-lang/rust/pull/61812)
    -  Deny specializing items not in the parent impl #[64564](https://github.com/rust-lang/rust/pull/64564)
## Unknowns / future things / or something
- Simplistic tree specialization — this is considered a conservative start
- Soundness around lifetime-based dispatch
## Key ideas and use cases
- Permit *overlapping impls* (two impls applying to the same set of input types) so long as:
    - one of them is “more specific”
    - any items specified in the “less specific” impl are marked with `default`, signalling that they can be overridden
- Three main motivations:
    - Performance
    - Ergonomics and better defaults
    - Longer term: refining behavior (as in the [Specialize to reuse](http://aturon.github.io/blog/2015/09/18/reuse/) blog post)
- Examples
    - Specializing based on types
    impl<T> Extend for T where T: Iterator { ... }
    impl<U> Extend for [U]
    - Specialization based on traits
    unsafe trait TrustedLen { }
    impl<T> Extend for T where T: Iterator + TrustedLen
    - Ergonomics
    impl<T: Display> Debug for T { } // Can't add this backwards compatibly now, though
    - Better defaults
    trait Foo {
      fn foo() { .. }
    }
    
    // generalized to
    
    trait Foo { fn foo(); }
    default impl<T: ?Sized> Foo for T { fn foo() { .. } }
    
    impl Skip for T where T: Iterator {
      fn skip(&self, n: usize) {
        for i in 0..n { self.next(); }
      }
    }
    
    default impl Skip for T where T: Iterator + RandomAccess {
      fn skip(&self, n: usize) {
        self.goto(self.position() + n)
      }
    }
- How this works during compilation
    - During type-checking
    impl TheTrait for &'static u32 { }
    
    fn foo() {
      let x = 22;
      let y: &'0 u32 = &x; // ERROR triggered here because `x` does not live
      bar::<&'0 u32>(y);
      // To type-check this:
      //   Implemented(&'0 u32: TheTrait)
      //   this is true if `'0 == 'static`
      // we wind in up inferring that `'0 == 'static`
      // triggers the error above
    }
    
    fn bar<T: TheTrait>(t: T) { }
    - During “trans”, the `Reveal` mode
        - can’t figure out whether `TheTrait` applies to `&T` or not without knowing the lifetime
    impl<T> Foo for T {}
    impl<T> Foo for T where T: TheTrait {} // can't tell whether this applies or not
- Soundness hole: lifetime-based dispatch
    - See [Shipping specialization: a story of soundness](https://aturon.github.io/blog/2017/07/08/lifetime-dispatch/) blog post
    - Challenge:
        - During `Reveal` mode (and perhaps even during type-checking!), the compiler doesn’t know precise lifetime details
    impl<'a> TheTrait for &'a u32 { } // OK
    
    // Does this apply to `&'x u32`? Only if `'x = 'static`
    impl TheTrait for &'static u32 { }
    
    // Same as above
    impl<T> TheTrait for T where T: 'static { }
    
    // Does this apply to `(&'x u32, &'y u32)`? Only if `'x = 'y`
    impl<A> TheTrait for (A, A) { }
    impl<A> TheTrait1<A> for A { }
    
    // Note that the distinction between input and output types is important
    // e.g. this is OK
    trait TheTrait2 {
      type Output: Ord;
    } // where Self::Output: Ord (converted in compiler today)
    
    impl<A> TheTrait2 for A {
      type Output = A;
    }
    
    // Indirect
    impl<B> TheTrait for B where B: SomeOtherTrait { }
    impl SomeOtherTrait for &'static u32 { }


- Why is this a soundness problem?
    trait Something {
      type Foo;
    }
    
    impl<T> Something for T {
      default type Foo = u8;
    }
    
    impl Something for &'static u32 {
      type Foo = u16;
    }
    
    // What is the value of this?
    <&'? u32 as Something>::Foo
    
    fn foo<'a>(x: &'a u32) {
      // type checker would not know which impl is more specific
      // it would be conservative here
    
      let y: &'static u32 = ...;
      // but here it would pick u16
    }
    
    // Ambiguity where type checker has freedom of what lifetime
    // to pick, and it currently picks smaller.
    fn bar() {
      let x: &u32 = &22; // Using promotion => can be 'static.
      foo(x);
    }
- What if you wanted to specialize on static str?
    - you can’t but you can have some type like `struct StaticString { x: &``'``static str }` and use that for your constants (and you can specialize on that)
    - not great but plausible for some use cases
- Proposed solution
    - Derived from two blog posts:
        - [Maximally minimal specialization: always applicable impls](http://smallcultfollowing.com/babysteps/blog/2018/02/09/maximally-minimal-specialization-always-applicable-impls/)
        - [Sound specialization](http://aturon.github.io/tech/2018/04/05/sound-specialization/)
    - Causing bugs in practice
        - https://github.com/rust-lang/rust/issues/67194
    - Core concept: an *always applicable* impl
        - Impl that uses only constraints that can be derived from the types themselves
    impl<T>Extend for T where T: Iterator { ... }
    impl<U> Extend for [U] // OK
    
    impl<U> Extend for [U] where U: Copy // OK
    impl Copy for MyType<'static> { .. }
    
    // Not OK, because `T: TrustedLen` is not implied by WF on `T`
    impl<T> Extend for T where T: Iterator + TrustedLen { }
    - Various possible extensions
    - Next concept: a `specialize(T: Trait)` “mode”, which means
        - “`T` implements `Trait` with an always applicable impl” 
        - `Specialize(T: Trait) => Implements(T: Trait)` but not vice versa
    unsafe trait TrustedLen { }
    impl<T> Extend for T where T: Iterator + specialize(TrustedLen) { }
    - Way to think about this, I think
        - impls can be 
            - *lifetime dependent  (e.g.,* `*impl<T> Foo<T> for T*`)
            - *lifetime independent* (“always applicable”)
            - *neutral* (e.g., `impl<T> Bar for T where T: Foo`)
                - neutral means that, if the bounds are converted to `specialize()` bounds, then they would be applicable
    specialize(T: Bar) -- 
      specialize(T: Foo)
- When you have a specializing impl what are the set of things we accept
    - Permit bounds that are implied by traits (“case 1”, below)
    - Permit bounds implied by the base impl (“case 2”, below)
    - Simplest — all trait bounds have to be written `specialize(…)`


    // Case 1: follows from the type definition
    struct HashSet<K: Hash> { }
    impl<T> Foo for T { }
    impl<K> Foo for HashSet<K> where K: Hash { }
    impl<K> Foo for HashSet<K> where K: Hash + specialize(Ord) { }
    
    impl<'a> Ord for Evil<'a, 'a> { .. }
    // Implemented(Evil<'a, 'a> : Ord)
    // but specialize(Evil<'a, 'a> : Ord) does NOT hold
    
    // Case 2: follows from the impl we are specializing
    impl<T> Bar for T where T: 'static { }
    impl<K> Bar for HashSet<K> where K: 'static { }
    // HashSet<K>: 'static => K: 'static
- If you required it written on every bound, maybe it should move to the impl, for example?
    - `specialize impl …`
- Observation:
    -  we have the notion of always applicable already, in the dropck
    struct MyStruct<'a> { }
    impl Drop for MyStruct<'static> { } // illegal today, for precisely the same reason
- Minimal steps to move forward right now
    - add a lint or something we can use to enforce valid usage in the standard libary
        - add a moratorium on adding new specializations until we have that
            - would also permit crater runs to look for surprises
            - maybe add a note to forge about this, if there is a libs page
            - maybe ping libs team to discuss it
                - https://forge.rust-lang.org/libs/maintaining-std.html#is-specialization-involved
    - finish associated type defaults
    - need a reference or write-up that tries to document in one place the status

