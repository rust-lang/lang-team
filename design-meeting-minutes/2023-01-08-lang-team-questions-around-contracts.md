# T-lang Questions about Contracts

:::info
Currently this doc is a sketch of a list of questions, as well as *potential* answers when pnkfelix saw them. The core point, though, is for T-lang to agree about what **Questions** it wants answered, and *not* to dive deep into the answers, apart from when it helps us better understand poorly phrased questions (which should be subsequently revised).

However, if there are some constraints on the answers that T-lang already *knows* it wants to communicate, then we should include that in the document we deliver to the Rust contracts design stakeholders.
:::

## Related work:

* [Draft RFC 2022-11-25][]

[Draft RFC 2022-11-25]: https://hackmd.io/w8AS2N09R76aXDFK9ogHBQ?view

## Purpose of Contracts

**Q1:** What is the primary purpose of contracts?

More specifically, are contracts primarily:
* a specification tool? (i.e. meant for *proofs* of correctness properties, showing them to hold prior to the program execution)
    * correctness proprties might include:
        * (full-blown) functional correctness (total or partial)
        * safety (i.e. lack of UB)
        * panic-freedom
        * maintenance of a [representation invariant][rep invariant]
* an assertion encoding mechanism (i.e. a way to catch mistakes dynamically)?
    * If so, are the assertions guaranteed to hold?
        * (This is meant in the sense of `assert!`, which one can rely on in all builds, versus `debug_assert!`, which one should not assume holds when it comes to e.g. safety conditions in release builds.)
    * The concerns of any assertion mechanism overlaps with the criteria for "a specification tool" above (e.g. one might attempt to encode total correctness properties, or just a subset thereof), but an assertion mechanism changes the goal posts so that one need not always demand ahead-of-time proof; mere *eventual* and/or *potential* detection at runtime suffices here)
* a blame-assignment mechanism? (i.e. a an aid for identifying which modules injected errors)
    * If so, what are the parties (functions, traits, modules, crates, or some subset thereof)?
    * What guarantees are made? E.g. if blame is assigned to a party, then it will be assigned correctly (a kind of "blame soundness")? will blame always be assigned to some party (a kind of "blame completeness") 
* a documentation tool?
    * (pnkfelix: This is an oft-touted benefit of contracts, but its hard to believe the effort spent developing a contract system would pay off if improved documentation were the *primary* purpose.)

[rep invariant]: https://en.wikipedia.org/wiki/Class_invariant


#### Why this question matters

Different purposes entail different sets of constraints on the design space.

For example, an assertion mechanism whose guarantees can be assumed by unsafe code in relase builds is one that needs to *always be proven to hold* (dynamically if necessary). That in turn may entail different constraints on how *expressive* the contract language is allowed to be.

## What is value-add for Contracts?

**Q2:** What would your *minimal acceptable form* of Contracts provide over mere `assert!`?

**Q3:** What be provided via "natural" anticipated extensions to the aforementioned minimal form?

Ideas for answers to the second question:

* Automatic injection of checks provided by trait definition into object code of (every) corresponding impl
* Automatic generation of test inputs and corresponding validity checks of resulting outputs

**Q4:** What is value-add for building into "Rust core" (versus delivery via third-party crate)?

* Encourage uniformity and ubiquity
* Can be incorporated into Rust `std` crate.

**Q5:** How are contracts best understood by the programmer? 

In particular, see https://www2.ccs.neu.edu/racket/pubs/icfp16-dnff.pdf which points out that neither a denotational or operational view is ideal, at least for a rich dynamic contract language like Racket's.

#### Why these questions matter

We have `assert!` in Rust today. We need to understand why this new mechanism is being added to the language.

## Prior art

**Q6:** What is prior art specifically for Rust?

e.g. Attribute-based (i.e. proc-macro) contract encoding

 * https://crates.io/crates/contracts
     * `#[requires(...)]` and `#[ensures(...)]`
     * `#[invariant(...)]`
     * all three of above have `debug_`-prefixed variants
 * [Prusti](https://github.com/viperproject/prusti-dev):
     * `#[requires(...)]` and `#[ensures(...)]`
     * `#[trusted]`
     * `#[pure]`
     * `#[after_expiry(...)]` aka Pledges, etc
 * [Creusot](https://github.com/xldenis/creusot)
    * `#[requires(...)]` and `#[ensures(...)]`
    * `#[invariant(...)]` (on loop expressions)
    * `#[trusted]`
    * uses Pearlite, a pure immutable fragment of Rust with logical operators and mappings to "models" of objects

#### Why this question matters

Any divergence from existing contract systems needs to be properly motivated. In other words, it would be good to provide an easy upgrade path for people using existing third party crates (especially the successful ones).

**Q7:** What is *successful* prior art independent from Rust?

(TODO: validate the success of any item listed below...)

e.g.
 * [seL4](https://en.wikipedia.org/wiki/L4_microkernel_family#High_assurance:_seL4)
 * [Java Modeling Language (JML)](https://en.wikipedia.org/wiki/Java_Modeling_Language)
* Contracts for Dynamic Checking
    * [Ada](https://learn.adacore.com/courses/intro-to-ada/chapters/contracts.html)
    * [Kotlin ?](https://blog.kotlin-academy.com/understanding-kotlin-contracts-f255ded41ef2)
    * Racket: Findler/Felleisen
    * Eiffel
* Contracts for Static Reasoning
    * Ada
        * note Ada has distinct `with Static_Predicate => ...` and `with Dynamic_Predicate => ...` in its DSL
    * Java (see JML mentioned above, which has ESC/Java)
* (Are there examples available of temporal protocols modeled via contracts?)

#### Why this question matters?

This is a heavily explored space and it would be foolish for us to not ensure we are aware of what has worked and what hasn't worked for other languages.

## Varieties of Contract Language

**Q8:** Where are Contracts meant to be applied?

(This is somewhat connected to the question above about "who are the parties" if you think of contracts as being about "blame assignment")

* Arbitrary functions?
* Methods of traits?
* Any associated items of traits?
* Types
    * to express [representation invariants][rep invariant]
* Traits themselves
    * (to express abstract invariants that hold for any impl of the trait, potentially written against so-called "ghost state" attached to the trait)
* Loop headers (to express "loop invariants", a crucial component of correctness proofs for imperative program)
* Modules (e.g. for mutable state stored within)



**Q9:** What are criteria for a useful "contract language"?

The [draft RFC][Draft RFC 2022-11-25] mentioned that there are a number of research tools, each developing its own "contract language"

What are the criteria for such "contract languages"?

From RFC:
  * expression of "key correctness properties"
  * maps to (or serves directly as) testable + verifiable documentation
  * strong guarantees at module boundaries (enables modular verification effort)

In addition 1:
  * understandable to typical Rust developer
  * writable by (advanced?) Rust developer (or at least typical Rust *library* developer?)
  * executable?

In addition 2:
  * provides operators not available in "normal" Rust code? (E.g. Larch IL for C has `minIndex(p)` and `maxIndex(p)` to bound the valid offsets from arbitrary pointer `p`)
      * (This may be in tension/conflict with "executable" above, depending on expressiveness of any given operator)
      * for example, Prusti's ["Predicate" system](https://viperproject.github.io/prusti-dev/user-guide/verify/predicate.html) allows definition of helper functions that are solely invocable from specifications
      * [Ada's contract system](https://learn.adacore.com/courses/intro-to-ada/chapters/contracts.html) has quantifiers like `for all` and `for some`. (pnkfelix is not yet clear on whether they are dynamically enforced in all cases)
 * mocking?
     * see also type-modeling ([prusti](https://viperproject.github.io/prusti-dev/user-guide/verify/type-models.html))
     * (pnkfelix *thinks* "modelling" is current term-of-art for describing values in the abstract domain that objects represent; i.e., an abstraction function maps concrete data to their corresponding abstract values in the model)
 * intra-specification checks? i.e. encoding statements that validate parts of the *specification itself*, rather than merely validating that the implementation satisfies the specification
     * For example, the Larch Shared Language (LSL) enables one to encode  semantic claims about the specification itself, independent of any implementation. See section 7.1 of the [Larch Book](https://www.cs.cmu.edu/afs/cs/usr/wing/www/publications/LarchBook.pdf) for more discussion on this point.


#### Why these questions matter

The answer to this question helps inform the scope of the planned contract language and the potential constraints on its design. 

The answer should establish a shared notion of the scope of the design space and the constraints that the stakeholders are bringing to the table.

## Question/Discussion Topic Queue

### How do I write a question?

pnkfelix: you put it here!

### What questions are missing?

pnkfelix: I want to know what T-lang members think I accidentally left out when I wrote this.

josh: don't think anything was "missing" per se. More that there are cross-cutting issues that are somewhat assumed that we need to resolve. E.g. are they a verification-mechanism only? Or are they meant to be a way to increase program *expressiveness* (e.g. allowing cases that are provably impossible to be removed entirely from the program).

pnkfelix: indeed. I don't want to *require* a theorem prover. But it probably doesn't suffice (or is uninteresting) to weaken contract system to point where it could be eliminated entirely and still have valid.

joshtriplett: Not proposing to require a theorem prover, but we could have assertions that we can then rely on, after checking them. The compiler would have to understand what the contract is provding/asserting.

pnkfelix: I'm definitely seeing dilemma here, between wanting simple assertion semantics versus wanting expressiveness of e.g. universal quantification.

tmandry: similar. Is there an assertion that cannot be checked dynamically?

pnkfelix: go-to example is universal quantifier. "for all X, P(X)" is typically not an assertion you can check when you hit it. But you can seek *counterexamples* dynamically (i.e checking against the X's you have in your hand)

joshtriplett: seems like there might be value in having distinct notations for the things that are guaranteed to be checked versus things that are "nice to check" but not guaranteed.

tmandry: seems related to some of the issues we have with `const fn` vs non-const `fn`.

### Contract purpose: can they affect the interpretation/capabilities of the program, or only constrain them?

joshtriplett: Today, traits and other aspects of the type system can *add* capabilities. If you know that a type implements a trait, that lets you call methods of the trait on that type, or similar. Contracts *could*, in theory, provide guarantees that the program could rely on. Down that road lies full dependent typing, and things like "this array has at least one element" or "these two arrays are the same length". Do we want a contract system that Rust code can *rely* on for things like that, such that it can then do things it otherwise couldn't? Or is this *just* a verification system that adds zero capabilities to the language?

joshtriplett: If we *commit* to a system that will never add expressive power to the language, that may make us more capable to add hooks for third-party tools to grab on to, but *also* seems like a one-way door.

joshtriplett: similarly, we may have one-way doors w.r.t. static versus dynamic. What can we allow at runtime, versus what we might allow one to express but solely for the compile-time checkers?

tmandry: Why is it a one-way door to commit to not adding expressive power to the language?

joshtriplett: if it adds expressive power to the language, it has to be implemented in rustc. If it is not implemented in rustc, then it cannot add expressive power to the language.

joshtriplett: We could skirt the one-way door by having two distinct systems, but that seems unfortunate.

tmandry: when we talk about relying on things like "this number is in this range", my brain immediately goes to exhaustiveness checking. Is that the kind of thing you're expecting in terms of increasing expressive power?

joshtriplett: If you're relying on something in unsafe code, the compiler doesn't have to understand. If you're relying on something in safe code, the compiler has to understand the assertion you've made or the contract that holds.

### Constraints on defining the contract language

joshtriplett: We should talk about the degree to which we implement contracts in rustc vs third-party tools, and the degree to which we are willing to rely on the latter. This would constrain our usage of them in various ways, such as relying on them for capabilities, or adding functionality to the compiler to support them.

pnkfelix: my main concern here is that we may need to constrain the contract language in order to satisfy the needs to the static reasoning folks, even if the compiler itself doesn't guarantee any form of static reason in its interpretation of the contracts

joshtriplett: We don't want to have "dialects" of Rust - if there's a requirement that contracts need to meet (e.g. "only call pure functions"), that requirement *must* be enforced by rustc.