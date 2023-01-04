# Rust Contracts RFC Draft

* Feature Name: contracts
* Start Date: 25th November 2022
* RFC PR: TBD
* Rust Issue: rust-lang/lang-team/issues/181

# Expected outcomes

This RFC is intended to solicit feedback on the best approach for adding contracts as a standard feature of Rust. We hope that language designers, tool developers and others will feel able to contribute their experience and efforts into such a design process.

# Agenda

We are considering investing effort in developing a specification language for Rust. Before we do so, we would appreciate feedback from the language team:

1. If adding specifications to rust is something the community and the language team wins interested in and will support.
1. If this is something the community supports, what's the best way to drive this effort?
1. Get a general sense of what the desired goals for specifications are, and their relative importance to the rust community.
1. If time allows, discuss some of the key questions about what the spec language should look like, and how we might start implementing it.

# Summary

Other languages have shown the utility of contracts in supporting various forms of verification, covering both static and dynamic styles. Currently, there are a number of research tools developing such capabilities, but each developing its own contract language which risks fragmentation, repetition of effort and long-term incompatibility. A single, standard, core contract language would offer users benefit in familiarity and productivity, and would enable a rational market to develop for competitive and complementary verification tools.

While Rust has an expressive design-by-contract system for generics units and their instantiation (i.e. trait bounds), the lack of the more "basic" contract forms, such as pre- and post-conditions remains a curious omission. The use of contracts and their verification has become an accepted practice in safety-critical systems, an area where interest in Rust is growing substantially. For those markets, the lack of contracts might be perceived as a weakness.

Several tools (notably Kani, Creusot and Prusti) have begun to address this problem, but are taking different, and incompatible approaches. This risks divergence, repetition of work for tool vendors, and (for customers) a risk of "lock in" to a particular single tool for the lifetime of their project.

The design of a contract language is not a simple matter. There are several subtle issues that must be addressed, especially if the goal is to support a reasonable case for the soundness of subsequent verification activities. Later sections of this RFC cover some of the less-obvious issues, mainly based on the authors' experience with SPARK (the Ada subset), Frama-C and contemporary Rust tools.


# Motivation

Imagine a future where Rust has a carefully designed contract language and a mature ecosystem of verification tools. Benefits to users might include:

* The language allows programmers to express key correctness properties about their code in an ergonomic and familiar syntax.  This enables developers to add testable and verifiable documentation 
* The language allows programs to combine both static (e.g. deductive proof) and dynamic (e.g. fuzzing) verification styles, with strong guarantees of correctness between module boundaries.
* A single set of contracts for a program can be read and processed by several tools, perhaps coming from distinct tool vendors, providing both dissimilar verification styles and redundancy of verification.
* Static proof of key program properties (e.g. panic-freedom) is possible, supported by a defensible soundness case, even where a full-blown proof of correctness is neither achievable nor desirable.
* Contracts provide strong guarantees at module boundaries. This enables verification tools to run in parallel, and so enables tools that scale to large programs.
* Where panic-freedom (and, more generally, "type safety") has been proven, units can be compiled in a manner that eliminates all run-time overhead, and so brings Rust close to matching the run-time performance of C or SPARK.
* Rust's "core" and parts of "std" libraries have contracts supplied as part of their specification, and so enables verification of units that depend on them. Assurance of the implementation of those libraries is also improved.
* For developers of real-time and embedded systems, the contracts work “as advertised” in Rust’s “no_std” profile.

# A Draft Roadmap

If contracts are to become a standard feature of Rust, we need to consider a roadmap for their design, implementation and adoption that makes technical sense, fits with Rust’s release schedule, and offers an approachable adoption path for users.

A risk is to be over-ambitious here in an attempt to support the union of all features currently supported by all existing tools.

An entirely hypothetical “batting order”:


0. Terminology. Define terms and their meanings, inheriting what we can from existing tools and other languages.
1. Decide: static, dynamic or both? (We presume that “no static checking at all” is off the table, given that Kani, Prusti, Creusot and Flux all emphasize static verification, so the only serious options remains “All static” and “Both”)
2. Function purity. Rules for panic-freedom and absence of UB in contracts.
3. Decide how to deal with static mutable state.
4. Simple pre- and post-conditions with existing expression language.
5. Determine subset of “core” libraries required to support majority of early-adopter use-cases. We may have to seek input from the safety-critical/embedded community on what features they really require.
6. Attempt to contractualize subset of “core” to guarantee safe and intended behaviour.
7. Language extension: Validity intrinsic, ranged types, extended expressive power of “expression” to support more contract forms, including quantifiers, logical operators, pledges, original and final values of mutable parameters and so on.
8. Loop invariants
9. Ghost entities.
10. Other items... all TBD.


# Guide-level explanation

The Rust type system is a powerful tool for ensuring the memory safety and correctness of your code, but it has its limits.  Type-safe code can still panic, and even panic-free code can return wrong or unexpected behaviour due to programming bugs.  We consider code “correct” when it follows its "specification".  Typically, specifications are documented in plain language (e.g. English), possibly along with executable assert! checks.  However, there is nothing that enforces that the checks match the documentation, and that the documentation matches the code. 

For example, consider a simple Cursor datatype:

```
struct Cursor<T> {
    data : &mut [T], //TODO should this be a *?
    read : usize,
    write: usize,
}
```

This struct has an implicit correctness criteria: the read cursor should lag the write cursor, and both cursors should remain within the allocated data.  If we add this as a “type invariant“, then Rust can now check this property whenever Cursor objects are modified.

```
// This is pseudocode. Bikeshed later. 
impl <T> TypeInvariant for Cursor<T> {
  pub fn is_valid(&self) -> bool {
    self.read <= self.write && self.write <= self.data.len
  }
}
```
    
Similarly, the methods of Cursor have both pre and postconditions.

```
// This is pseudocode. Bikeshed later. 
impl <T> Cursor<T> {

  pub fn write_bytes (&mut self, newdata &[u8])
  // TODO check for no integer overflow??
      precondition(self.is_valid() && self.data.len < self.write + newdata.len)
      postcondition(self.is_valid() && 
                    UNCHANGED(self.read) && 
                    self.write == old(self.write) + newdata.len)
  {
      // Code goes here
  }
}
```

These specifications provide value in multiple ways:

1. They form precise, machine readable documentation of the expected behaviour of your code.
2. They can be automatically converted into assert! checks, and then validated using your existing unit, property, fuzz, and integration tests. 
3. They form the specification for formal verification tools.  In cases where these tools can *prove* the absence of bugs, the checks can be elided entirely from the code, giving you safety with no runtime cost.  These formally verified checks can even form the basis of assume statements, enabling additional optimization within the compiler (e.g. allowing the compiler to remove bounds checks on accesses that can be formally proven to be safe).

# Reference-level explanation

This section expands on some of the issues that need to be considered in the design of a contract language.

## Semantics of violated contracts
It is clearly a **bug** when a contract is violated.  The question is what should happen when a contract violation occurs, and what can the compiler assume 

1. **Is it required that all contract violations be detected?**
We suspect that this will be too expensive for many users to enable at all times.  Some contracts may involve checks that are expensive to perform at runtime, e.g. a `forall` predicate over a large array.  Others may involve primitives, such as pointer validity, that would require Rust runtime extensions to support. As with regular assertions, we  propose that contracts be marked as either `contract` or `debug_contract`, allowing users to disable expensive checks.  This is parallel to how integer overflow behaviour differs in debug and production builds.
2. **Are contract violations UB**?
From a compilers prespective, it is tempting to say that contract violations constitute UB, because that opens the door to various optimizations, since the compiler can then `assume` the correctness of the contract. However, this would have the preverse effect that adding contracts to code could actually make it less safe.  We propose that, at least to start, contract violations should therefore NOT be considered UB. 


## Static, Dynamic, or Both?

The design of a contract language is influenced by the competing demands of expressiveness, simplicity, and ease of adoption into developer workflows.

At one extreme, languages and tools designed for interactive functional verification (e.g. early versions of SPARK, and Creusot) prefer contracts that exhibit an abstract and mathematical semantics, and are NOT designed to be compiled and checked at run-time.  Advantages of this approach include higher expressive power of the contracts (since the language is not limited to constructs that the compiler can understand and compile), and freedom from concerns for overflow of arithmetic and so on.  For example, Creusot supports “prophecy variables” which reflect the value that a variable will take at a point in the future.  Similarly, a mathematical contract for a perfect hashing function might be ∀ i, j | i ∋  domain(f) ^ j ∋ domain(f) ^ i != j => f(i) != f(j).  A drawbacks are that the contracts are seen as a "new" language by developers (which has to be learned), leading to more difficult adoption for users.  Since this style of contract often includes constructs that cannot be concretely evaluated by the compiler, it is difficult for users to adopt them into their existing workflow: how do you test or debug a contact that you can’t compile?

At the other extreme are contracts designed to be compiled and checked at run-time, as notably offered by Eiffel and Ada2012. This offers easier adoption for users, since the "contract language" is really the same as the programming language itself, and this approach allows a gradual shift from simple testing to fuzzing to more static verification styles. The main drawback is that the expressive power of the contracts is limited to expression forms that can be compiled.

A middle ground, which we propose, is to go for "both" - a contract language that supports both static and dynamic verification, or some mix of both styles in a single program.  Such an approach provides developers a "smooth glide path" from their existing test based workflow to the world of formal verification, allowing them to add static verification to their code as needed.  Recent versions of SPARK (from Ada2012 onwards) support this approach.  This requires careful design work to balance the expressive power needed to support verification of non-trivial properties, with the need to ensure that language has a reasonable semantics and performance when dynamically evaluated.  One option is to define an executable subset of the contract language, with non-executable extensions whose behaviour decays in a sensible manner when evaluated concretely.  Since the language is intended for both static and dynamic application, we must guarantee "semantic consistency" of programs that are compiled with or without the dynamic contracts.  A contract language must protect against the possibility of side-effects, undefined behaviour, and panics that might be raised in the evaluation of a contract.


## Core Language Extensions

In designing a contract language, we should also consider changes to the core Rust language that might help, either by making contracts easier to write and verify, or ideally by making them superfluous by ensuring that code is “specified by construction”. 

Some of these extensions (e.g. Ghost State) could potentially be handled using proc-macros, although there are difficulties regarding the type system and object layout that would have to be carefully considered. Others, such as  Trait Laws, Object Invariants, Purity, etc., would be difficult to implement without some degree of language support, particularly if we want to enable these properties to by dynamically checked at runtime.

### Ranged Types

The type system could be modified so that additional contracts are not needed. An obvious example would be the provision of integer types with a user-defined range - a feature which is ubiquitous in Pascal and its offspring. Ranged types are also essential if proof tools are to achieve an acceptable false-alarm rate for proving the absence of arithmetic overflow.  This would also have the side benefit of providing a user-accessible mechanism to enable niche optimization.

### Predicated or "Liquid" Types

A second (and more advanced) example is the so-called "predicated type" - where the set of valid values for a type is defined by a general Boolean predicate, not just a simple range of values. These have enormous utility in both static and dynamic verification. They appear in other languages in various guises: "Liquid Types" in Haskell and OCAML, "predicated subtypes" in Ada/SPARK, and "subset types" in Dafny.  Following our running example, we could define Cursor as a type which enforces the is_valid() predicate described above. In pseudocode, one might write

```
// THIS IS PSEUDOCODE. WE CAN BIKESHED THIS LATER.
struct Cursor<T> {
    data : &mut [T],
    read : usize,
    write: usize,
} with Invariant {read <= write && write <= data.len}
```

When testing/verifying their code, the user could define at which points the type invariant should be checked: at every update to a cursor object, or perhaps as an additional pre/post-condition on functions which accept/generate Cursor objects. 

As noted above, we might consider an uplift to Rust's "expression" language to include more forms that are useful in contracts - notably quantified expressions, a notation to denote the initial value of an object in a post-condition, and so on.

A recent addition to the Rust community is the [Flux project](https://liquid-rust.github.io/index.html), led by Ranjit Jhala at UCSD. Flux extends the type system with “logical constraints” that can be checked at compile time via a compiler plug-in. Such constraints are limited to “...a subset of expressions from logics that can be efficiently decided by SMT-solvers.” We look forward to more information on Flux.


### Valid Bit-Pattern Intrinsic

It is considered UB for a Rust datatype to contain an invalid bit-pattern.  Invalid bit-patterns can happen when getting data from untrusted sources, e.g. unsafe calls, FFI calls, and deserialization.  There is currently no good way to defend against this in specifications, since accessing the invalid object causes UB.  We propose a new intrinsic that safely returns a boolean without causing UB, and can never be “optimized away” by the compiler. An intrinsic that performs this function is present in Ada, so compiler support in LLVM must already exist.  

An interesting question is whether this should apply to all types, or whether it is sufficient to have the intrinsic for primitive types, plus a derive macro to allow it to apply to higher order types as needed.


### Pointer Validity Intrinsics

The soundness of unsafe code often depends on properties of the pointer, such as whether it is allocated, the size of its backing store, its alignment,  etc.  For e.g. consider this std library function https://doc.rust-lang.org/stable/std/primitive.pointer.html#method.offset. If we wish to validate its preconditions, we need intrinsics which allow us to check properties such as `can_read_at_offset(&self, offset: usize)`.  These are extremely useful for Kani verification, and could potentially also be evaluated concretely if the runtime keeps additional metadata about allocations, as address sanitizers do.  In cases where this information is not tracked, the primitive could decay to true.

### Purity

One challenge in implementing a contracts language is ensuring that the contracts themselves are side-effect free when executed.  It would be useful to have a language level concept of “purity” which could attach to expressions, functions, etc.  This would also be useful for enabling extended compile-time calculations, e.g. for initialisers for static variables. Note that purity needs to be transitive in the call tree.  One interesting question is how purity interacts with the heap: is a function with no observable side-effects, but which allocates temporary variables on the heap, considered pure? 

Both Prusti and Creusot insist on function purity in contracts at present. Prusti also allows for “predicate” functions that have extended expressive power.


### Ghost Entities

Some languages, such as Dafny and SPARK, allow for "ghost" entities in a program that only exist for the purposes of static verification, and are eliminated or ignored when a program is compiled. For Rust, both Creusot and Prusti support some form(s) of "ghost" code or functions.  The implementation of ghost code will likely require careful extensions to the type system to ensure that the side-effects of ghost-code remain contained within ghost-state.

Ghost code typically also requires ghost state, for example 


### Trait Laws

Although Rust has a strong type system for Traits, they often come with conditions that are difficult to express with the current type-system, and impossible to test using a library.  For example, it is a requirement of `Eq` that it be reflexive, symmetric, and transitive.

This means, that in addition to a == b and a != b being strict inverses, the equality must be (for all a, b and c):
reflexive: a == a;
symmetric: a == b implies b == a; and
transitive: a == b and b == c implies a == c.
*This property cannot be checked by the compiler,* and therefore Eq implies [PartialEq](https://doc.rust-lang.org/std/cmp/trait.PartialEq.html), and has no extra methods.





## Static and Mutable State

Real-world programs have static mutable state (aka "global variables"). Contracts need to deal with these. Failure to do so introduces soundness issues for a start.

As programs grow, contracts must also scale at a reasonable rate, and avoid a "postcondition explosion" problem.

There is also an obvious tension between the needs to write "honest" contracts about mutable state and its visibility. How does one write a contract on a publicly visible function if that function mutates a static variable which is private?

Initialisation of such states needs to be covered, possibly through some sort of "initialisation postcondition" for each module that defines what is true about a static variable at program start-up.

Finally, we might need a contract to express invariant conditions that exist either for a single static variable, or for an entire group of them.

## Read/Write Sets

In the presence of static mutable states, some languages have a contract to explicitly define which set of those states may be read and/or written by a function. In some languages, this set of names is known as the "frame" or the "footprint" of a function. Knowing that set can be critical for verification tools to model what variables can and can't change in a function call. In particular, if the stated write set is too small (i.e. it does not specify the modification of a variable that a function really does mutate), then soundness issues can arise for callers of that function.

## Volatility and I/O

Real-world programs (especially embedded, real-time systems) have to deal with volatile states, including I/O devices. Dealing with them in contracts is tricky, since reading a volatile more than once can introduce unintentional order-dependence in a program. Verification systems need to know that a variable is volatile, so that its value is not trusted and might change even if there is no obvious modification to it in the program itself.

## References, Borrows and the Heap

**Contracts relating multiple (future) states**
The integration of Rust's ownership rules (in particular for mutable borrowing) with method contracts makes for an interesting design space in terms of contracts. A method such as `write_bytes` above (taking a mutable reference as argument) already motivates the need for contracts that relate two states of the program execution (using `old` or `UNCHANGED` or similar annotations to make the distinction). For a static checker of these contracts verifying these kinds of relational properties is usually straightforward; for runtime checking it typically requires "saving" the values of certain expressions in advance (e.g. spotting that we will use `old(self.write)` in a postcondition, and so saving `self.write` in the pre-state of the call).

In the presence of *reborrowing*, these relational contracts get more interesting still. For example, consider a (not necessarily super meaningful) additional `Cursor` method which lends out a pointer to the `write` field of the cursor (this would be more common for lending out references partial contents of a container data structure, e.g. [get_mut on Vec](https://doc.rust-lang.org/std/vec/struct.Vec.html#method.get_mut)):

```
 pub fn lend_write (&mut self) -> &mut usize
      precondition(self.is_valid())
      postcondition(??)
```          

Importantly, after calling this method `self` is blocked and can't be accessed until the returned reference (call it `result`) expires. This means, for example, trying to evaluate `self.write` *or* `self.read` etc. would not be possible in e.g. a runtime-checked contract on function return. On the other hand, there is information a caller will want to know: (0) what state `result` is in (what is its value), (1) what changes to `self` have *already* been made (but we can't refer to `self` yet, at least in a way Rust would allow), (2) what effect *subsequent* modifications via `result` will have on the eventual state of `self` when `self` is unblocked.

(0) can be specified with a regular postcondition (e.g. `*result == old(*self.write)`), but specifying (1) and (2) in a way which adheres to Rust's borrowing rules requires contracts expressed in terms of Rust expressions that *will be evaluated in the future* (when borrows expire). Both Prusti and Creusot support such specifications in different ways.

**Prusti**
Prusti proposed [*pledges*](https://dl.acm.org/doi/abs/10.1145/3360573) as a means of relating what we know now to what will be guaranteed by the time the reborrowed `result` expires. Pledges are written via extra modifiers around expressions which (like the familiar `old`) can be understood as changing the point in program execution at which the wrapped expression is evaluated.

`after_expiry(e)` wraps an expression `e` with the meaning that `e` should be considered evaluated after the reborrow expires; inside `e` one can also use `before_expiry(e')` to denote expressions to be evaluated *just before* this expiry time (both constructs have also been generalised with lifetime parameters, but this generalisation hasn't been implemented in the tool yet). For example, one can write `after_expiry(*self.write == before_expiry(*result) && *self.read == old(*self.read))` to express that the new value of `self.write` will be whatever is stored in `result` by the time its lifetime expires, while the new value of `self.read` will be unchanged. These modifiers can also be thought of as expressing where to evaluate/copy certain Rust expressions in order to runtime-check the analogous properties (but statically the proof of the pledge happens as part of verifying `lend_write`, i.e. before it will actually take effect).

One way to think about a pledge is: a borrow expiry behaves like an implicit function call that consumes the capabilities for the expiring reference and restores access to the blocked lender. Then the pledge is just a different way of specifying a contract for this future expiry call:
 * `after_expiry` introduces a postcondition for the call (what will be guaranteed after the borrow expires)
 * `before_expiry` plays the analogous role to `old` in a normal postcondition
 * Prusti also supports a third [`assert_on_expiry`](https://link.springer.com/chapter/10.1007/978-3-031-06773-0_5) modifier, which allows one to specify a *precondition* on the borrow's expiry, which can be used in particular to ensure that a mutable borrow is left in a state which guarantees some invariant.

These modifiers pin down the points in program execution at which to think about the nested expressions being evaluated (like a generalisation of `old`); importantly, the specifications inside can be any pure Rust expressions. This makes a translation to analogous runtime checks or other tools that support Rust expressions at the core of their specification languages potentially quite direct (similar to what is needed to support `old` expressions, but with more instrumentation needed).

<!-- wip(xavier) -->
**Creusot**: Creusot is built around the observation that we don't need to model the heap to reason about safe Rust programs and this simplifies verification drastrically. This corresponds to the notion of 'mutable value semantics', developped to reason about Swift programs. 
To model Rust programs, including mutable borrows a key technical tool called 'prophecy variables' is used, which in short associates to each mutable borrow (`&mut T`) a *final* value, the value the borrow has at the end of its lifetime. 
By allowing this final value to be used freely in specifications we can write concise and expressive contracts which would be hard using more traditional tools and logics. 

Consider the case of `Vec::index_mut`, this function takes a mutable borrow `self: &'a mut Vec<T>` and an index `ix: usize` and returns a mutable borrow `&'a mut T` to the `ix`-th index in the vector. 
How would we specify this function? As a precondition, we must require that `ix` is inbounds, which we could write as `ix < self.len()`, but what of the postcondition?
The returned value should be a pointer into the vector, specifically at the `ix`-th cell.
More than that, we should be able to specify that the modifications we are *yet to make* will be reflected in this vector. 
These are mutations happening far outside of `index_mut`, they could happen at any point during the borrow's lifetime `'a`. Nevertheless, we can say that after the dust settles the vector will contain the final state of those mutations. In Creusot we can express this using the *final* operator `^` which can be thought of as a 'future-dereference'. We would write the specification of `index_mut` in the following manner:

```rust
#[requires(ix < self.len())]
// The result points to `ix`
#[ensures(*result == *self[ix])] 
// The final value of the vector (after `'a`) will be updated with changes made to `result`
#[ensures(^result == ^self[ix])]
// Every other cell in the vector is unchanged. 
#[ensures(forall<i : _> 0 <= i && i != ix && i < self.len() ==> ^self[ix] == *self[ix])]
fn index_mut(&mut self, ix : usize) -> &mut T
```

What is surprising is that this single `^` operator is sufficient to handle all the specifications of mutable borrows in Rust. Moreover, because this is just an operation on mutable borrows, we can use it to build abstractions like we do using `*` in normal Rust code. For example, we could refactor the prior contract and move the last line to a helper method:

```rust
#[predicate] fn unchanged_outside_of(&mut self, ix: usize) -> bool {
    forall<i : _> 0 <= i && i != ix && i < self.len() ==> ^self[ix] == *self[ix]
}
```

Finally, in [prior work](https://dl.acm.org/doi/abs/10.1145/3519939.3523704), it was shown that this prophecy variable approach can be integrated with unsafe code, so long as a *different* technique is used to verify the unsafe parts. 

**TBD**. What does Kani do about this? 


## Concurrency

Something of the elephant-in-the-room. Over and above Rust's existing support for concurrent programming, can contracts add anything and/or enable verification of correctness properties that are currently out of reach?

## Libraries

Given a standard contract language, then it would seem prudent to supply suitable contracts on the "core" and (some subset of) the "std" libraries. A useful exercise would be to add preconditions that guarantee safe behaviour to all functions in "core", even if the functions are marked "unsafe". This would be a good litmus test of the expressive power of our contract language. For example, the unsafe [`get_unchecked`](https://doc.rust-lang.org/core/primitive.slice.html#method.get_unchecked) function for slices could have a precondition that guarantees that the requested index value is within bounds for the given slice.

What verification tools and approaches could then be applied to the implementation of those libraries? Who would vouch for the trustworthiness of that verification? If a library had a proof of correctness with respect to its contracts, would the proof ship with the library as standard?



# Prior Art

Prior art in the Rust community:


* MIRAI's annotations crate (https://crates.io/crates/mirai-annotations)
* Creusot's contracts and Pearlite (https://github.com/xldenis/creusot#writing-specs-in-rust-programs)
* Kani (https://model-checking.github.io/kani/kani-tutorial.html)
* Prusti's contract language (https://viperproject.github.io/prusti-dev/user-guide/verify/summary.html)
* Flux blog (https://liquid-rust.github.io/) and GitHub Repo (https://github.com/liquid-rust/flux).
* Contracts crate (https://crates.io/crates/contracts)

Prior art in other languages:


* SPARK and Ada (https://learn.adacore.com/courses/intro-to-spark/index.html)
* Frama-C and ACSL (https://frama-c.com/html/acsl.html)
* Eiffel (https://en.wikipedia.org/wiki/Eiffel_(programming_language) and AutoProof (https://se.inf.ethz.ch/research/autoproof/tutorial/)
* C++ (https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2020/p2182r1.html)
* CBMC (https://diffblue.github.io/cbmc/contracts-mainpage.html)



# Unresolved Questions

TBD

# Future Possibilities

TBD

# Discussion questions

## How do I write a question?

pnkfelix: like this! (i.e. include your name with the question, with a separate, short `##`-prefixed header/summary)

## Clarifications

Static vs dynamic:

* They are different
* Dynamic contracts can paint us in a corner if we're not careful; spec language is not entirely a prog lang
    * Rod: static vs dynamic brings about the most tension, but would love to move in the direction of getting benefits with a path to move towards more powerful spec in future

## Should we do this at all?

xavier: A specification language is a huge undertaking, building a shared one for multiple tools even more so. This will likely have a sizeable impact on the compiler and language, is this something we actually want to do?

nikomatsakis: To add onto this, can we do this in a "piece-meal" way that adds value at each point? To tip my hand, I think that dynamically checked contracts could be really useful.

dsn: +1 on dynamically checked contracts. That's the "foot in the door" for formal methods. And that is something that would require compiler changes, especially for things like type invariants and trait laws

tucker taft: preconditions and postconditions are probably the easiest place to start, and give easy to measure value immediately.

---

joshtriplett: wouldn't want to see this done by having a "magic escape hatch" where all the work is handled by an external tool. If we can do this in a way such that the Rust compiler understands it, and we are spec'ing it in some way, even if it's working together with another tool.. that make sense. Part of the point of this document was talking about standardizing across tooling.

sacha: a question that's important to ask is whether the rust compiler should care only about syntax/type-checking, or whether it should actually do something with the semantics.

pnkfelix: way doc is written is centered around verification, hard problem. deciding whether to do this depends on the outcomes we expect. dynamic validation has seen value, e.g., contracts for higher-order languages is a much less ambitious goal. Allows for a more expressive language (which can be an anti-pattern, for verification). That's part of the problem. If we're going to aim for full verification, have to think about constraints on language itself. If we ack up-front that we're not expecting to get end-to-end verification, doesn't seem to be yielding fruit, that would let us loosen up the bounds of what we allow in the first place. I'm not putting my foot in one side but trying to counterbalance.

dsn: I think dynamic is really imp't and the compiler is absolutely needed for it. Simple things like right now there's no way to have trait laws and say that when I write Eq I expect it to be reflexive. Having a way to put that on a trait and be injected as a check would need compiler support. There will be a compiler semantics to this.

Xavier: Original purpose for asking this question of whether to do it at all -- do we believe we can build a common enough spec language that it's worth the work? Realistically, any real impl of a common spec language that does anything beyond `assert`, is going to involve sig. work in the compiler, add complexity, show up in the code where people who know nothing of verification will have to interact with it. Seems likely to be some complexity there. Important design consideration beyond whether we do dynamic, static.

joshtriplett: Question of whether we can handle that level of complexity in compiler is not a lang-team question; we have lots of things in the compiler that are understood primarily by a subset of folks who work on compiler, including the details of the type system. Certainly precedent to be had.

James Parker: worthwhile to do just to provide a consistent user ex for users, so they can easily switch between tools to check the properties they care about. Integrating with the language means you can integrate it with the ecosystem, ship spec with crate, that's a huge win for users.

Tucker Taft: we added contracts to Ada in 2012. This turned out to be the one feature that got people to move off of older versions of Ada. From Ada's POV, all dynamic though designed to support formal verification under certain circumstances. When doing formal verification, can't always prove anything, so it's not like the only thing you can write. When it wasn't dynamically checkable, which is how SPARK was before, you often spent a long time trying to prove things that are not true. When you think of pre/post conditions, not a huge leap for most people. When you get into things like type invariants, it's a bit more, but you can often express it as an assumption coming in, something going out. I think you underestimate ability for average programmer to understand these things. Giving them dynamic semantics makes it easier to understand, too. 

nikomatsakis: wanting to narrow this question, I think the subquestion worth having is "what are we trying to do". I see some of the benefits being dynamic, but also high leverage. Benefits fuzzing, benefits other tools, and static verification. 

josh: rather than continue to discuss in framework of "should we do this", maybe we should see how much we can get to the rest if we assume we will in some form.

celina val: if you really want contracts to work, have to be in the language. Tools having their own contracts, because they depend on dependencies, it doesn't work. 

nikomatsakis: who's against it in this room, if anyone?

celina: I'm a bit skeptical. When we talk about contracts, is it complete? Giving benefit even for incomplete contracts -- same way they write random assertions. That's how you can ease them in.

rod chapman: really important people can write partial specifications or a specific property they want to be true. What Tucker referred to was getting this consistent semantics for static and dynamic. So that people can do this mix-mode on-ramp. Then they can say "for that one package I want to prove it statically". The other modules are just testing dynamically. That's awesome. Contract language + prog language being the same makes it a lot easier to teach people.

joshtriplett: in terms of skepticism, I want to see this in accessible form, very much don't want to see Rust turned into Coq or another proof-based language. Incredibly impressive technology, but inaccessible to most people. Want to make sure we have something as accessible as the rest of Rust. Incrementalism really important.

## Logic language vs "be just like Rust"

pnkfelix: One contingent is strongly putting forward the viewpoint that spec needs to be a logical language. I'm not sure if that's meant to imply that programming is not a logic. How much is that in tension with the other side that is saying that it needs to be like Rust?

sacha: cannot be resolved today, obviously, but if you don't have a logic, you use the language itself, you're going to have to reduce your expressivity somehow, or have weird intrinsics, which adds to the incomplete spec things, which is good for dynamic checks, or tools like kani, whcih basically run code + checks in code, but as soon as you're partial, you're giving up compositionality and scale. Maybe there's not an in-between but maybe you should be able to do both and there should be some kind of parametermization, probably won't be able to satisfy everyone here.

xavier: my point when saying spec is logic is not saying we can't have rust syntax or rust expression semantics. It's about the fact that if we want something that corresponds to both static + dynamic verificaiton, and in general if we want to be writing specifications, very nature is that you're descriving an abstract behavior, you're writing a predicate, a kind of math formula, but if we want them to be usable with static verification at all, they reason in terms of logics. We have to think about these concerns from the outset. Not something you can retrofit easily once you've settled into concrete choices for how they'll be evaluated. We don't have to include universal quantifiers from the start, but there's design work that needs to be done up front if we want something that can both be used for static / dynamic checking

Nico: maybe as important as "if we should do it" is "should we do it *now*". I'm of the opinion that there's still a lot of open questions. Pre and post conditions that people have been developing are different. Maybe an alternative is to say "don't do it now" but there are a lot of features compiler could implement that would make experimentation easier. That could be an alternative.

Felipe: Just wanted to add data points from our experience of C verification. A couple of key things we observed with C-- first was that as close to lang as possible. A lot of users were happy just by writing boolean predicates as they would normally write in C, and they could cover a lot of things that way already. If we focus on first thing we could support, pre- and post-conditions supporting boolean predicates in the lang, would already be a good advantage. The more they were using that, the more expressiveness they wanted. For many of our projects, deveopers were not writing proofs, but many of them continued to write the pre/post conditions, those were really easy to do, and they're still doing that today. Focusing on this first step would be useful.

joshtriplett: clearly we need to do another meeting, maybe a longer one.  let's do async.

dsn: agree with what felipe said. I Think there's a common core that's useful to everyone that just does simple expressions but doesn't lock us in to not doing more later. I was thinking it might be very useful to do a working backwards process, some examples of code where we'd like to specify, think through what are the interesting properties on this code, how much can we express. Having some set of challenge problems sounds like it would be useful.

tucker: I don't see as strong a dichotomy between static and dynamic. A lot of the static analysis tools are trying to deal with reality of overflows anyway. Fact that you're using the language -- yes, they have to deal with overflow, frequently there's a big-int sort of thing you can use-- that dichotomy isn't necessarily a problem. How much do you have to say? Are you trying to achieve full functional correctness? In early days we thought that was the only motivation, but a lot of people just want to prove it doesn't do something really bad. Many intermediate levels where you just try to prove you don't have a runtime failure. Don't have to suddenly be in the coq world. Can use other kinds of testing to feel like program is doing the right thing in general.

tmandry: going back to what Nico was saying of "shoudl we do this now" and "how much should we do now". Seems like maybe there is a common core we can start with. After we get past that, likely to run into some difficult questions. How do you write assertions about what happens a mutable reference expires, or other things the doc talks about. I'm wondering if we can setup a structure that has some kind of .. that is really there to encourage existing proof checkers to align on their approaches. More alignemnt, commonality that we can see across these projects, faster we can start to adopt that into the language. Should decide relatively quickly how we want to approach.



---

## Why do this in the language vs procedural macro or in projects?

nikomatsakis: There's an assumption that contracts would be integrated into lang, but it's worth discussing procedural macros, annotations, and how viable they are here.

## Verification vs "Blame Assignment"

pnkfelix: What do you all see as the key goal/deliverable of a contract/specification language? Is it to reach verification of partial/full-correctness, or is it merely to know which party is to *blame* when a contract is violated? (See e.g. Robby Findler's work on contracts for higher-level languages.)

Josh: I have a similar thought when I see descriptions like "defensible"; the goal shouldn't be whether you can defend that you *tried*, the goal should be to *succeed*.

## Core interface / approach

xavier: Will we be using a contract-based approach to specification or a refinement type based approach?

dsn: is this an either/or? Why not support both"

## Underlying logic?

xavier: What is the underlying logic used by the specification question? 
This question has to do with the broader point of having a shared understanding that a specificaiton language *is not* the same as a programming language.
Additionally, if this contract system is meant to handle both unsafe and safe code, is there any choice other than a flavor of separation logic?

xavier: What about *mathematical* types like integers or sequences? 

jhjourdan: should the contract language permit specifying ownership patterns (in addition to those stated by the types), for example in the case of raw pointers etc...

Lennard: Probably there isn't a one-size-fits-all solution that covers all use cases of existing verifiers or the potential demands for future verifiers. Should there be extension points in the specification logic to accomodate that? (e.g. for embedding separation logic things for unsafe verification, etc) What would that look like and what are the tradeoffs?

## Compiler APIs

xavier: As a stop-gap to a fully shared specification language, there may be compiler level api changes that would enable tools to more effectively build their own specificaiton languages. Is this an alternate possibility? 

jhjourdan: it seems important for verification tools to allow macro to generate ghost code which may or may not be borrow checked. It seems to be a shorter-term goal that is useful and can be achieved before a full-fledge specification language.

xavier: as verification tools often add new 'lang concepts', they may introduce new traits that need to provide impls for opaque types like closures, but there is no support for this in the compiler apis. 

dsn: having a standard way of assocating metadata with types/traits would make it easier to support trait laws/liquid types.

## Interaction with Rust specification work?

Josh: How does this interact with proposals for Rust specifications, such as Ferrocene? Is there a concrete plan to coordinate between contract-language development and the needs of specification work?

dsn: My understanding is that ferrocene is about specs for Rust itself, whereas this is about embedding specs in rust code



## Purity, termination, and const fn

dsn: checking purity will be important for specs

jhjourdan: is it easy to have a definition of purity that we all agree on? For example, are function using Box::new pure? What about functions that use traits (such as Ord) that can be pure or not?

nikomatsakis: What does it mean for a function to be "pure"? How different is this from our existing concept of `const fn`? For example, do users have to prove termination? Is it allowed to use ref-cell or cell in 'pure' functions? Are there differences between what (say) creusot requires vs other tools (e.g., tools based on separation logics)?

## Read/write sets

nikomatsakis: It seems like read/write sets would, ideally, be integrated with the borrow checker; something like [view types] could be used to increase precision.

[view types]: https://internals.rust-lang.org/t/blog-post-view-types-for-rust/15556

## Organization & Outcomes

xavier: this is likely to be a *major* and *long* term project, how do we plan to organize and coordinate the work that will need to be done?

Josh: This seems like a great deal of work that may take many years to fully realize. Can we define this in a manner amenable to incremental completion, making useful work available along the way?

## Avoiding extensions

Josh: Some of the proposals in this area have seemed like arbitrary extension-points for external tools. Can we ensure that all aspects of the contract language are at least parsed and verified (if not directly implemented in) the Rust compiler, to ensure that we do not find ourselves with tool-specific extensions?

## Covergence with type-level where clauses and everyday Rust code

We may want to extend where clauses with more sophisticated features (e.g. more powerful quantification, implication) that are analogous to the static/symbolic verification features.

Can we move in a direction that allows us to converge the syntax for these things? Can we avoid an outcome where we end up with at least three sub-languages (expressions, where clauses, and contracts)?