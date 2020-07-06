# const-eval skill tree meeting notes

* [Recording available](https://youtu.be/b3p2vX8wZ_c)
* [Zulip topic](https://zulip-archive.rust-lang.org/213817tlang/64901designmeetingtoday.html)

## skill tree

https://static.turbo.fish/const-eval-skill-tree/skill-tree.html

A dependency (directed) graph of features you need to implement a given feature

nodes with high in-degree are things that have high-value, in terms of many other things depending on them

goal for today's meeting: determine set of features that can be moved from lang-team more directly to the const-eval group, so that they can make decisions directly without blocking on lang team.

ralf, ecstatic-morse and oli are aligned with respect to big picture

main questions:
* How can we ensure rest of lang team is aligned with their plans, not just now but also in future?
* prioritization: which items do we want to do in the near term
* scheduling
* how do we set up future conversations with lang team

Felix: does const-eval group *need* a lang team member present as a rep of lang team continuously, to help decide which things need to be bubbled up to lang team? Or can const-eval group decide on its own which things justify being bubbled up?

Oli: A mix of both. Example: we had a big RFC which went unaddressed for 9 months, and in the end was sort of abandoned as the const-eval group took smaller steps and prototyped MVPs.

josh: Personally, easier to look at and make decisions about smaller stuff rather than trying to tackle a big RFC with everything.

generally, happy to let prototyping proceed on nightly, which turns questions into T-compiler questions rather than T-lang questions

(much discussion of skill tree document:)
* reasonable to have content available in linked issues
* hard to identify some of the edges because their arcs collide in rendering 

what to focus on:
* ralf wants to talk about the unsafe operations
* josh: missing/misplaced nodes seem like relevant topic
* felix: is current skill tree the MVP? or a superset of that?
    * oli: its a wishlist
    * ralf: mvp is what we already have had for years

josh: an inside-rust blog post explaining the skill tree would be good

(discussion of graphviz)
* main point: prettiness does not matter to us, apart from making it possible to identify what are the start and end points of edges

what to prioritize?

*  `format!`` is out of scope for prioritization because it essentially depends on everything
* looping control flow would be good to do soon, so people do not encode it via general recursion
* josh says they would like `assert!`, even one without message formating, so that people stop using hacks like indexing slices of length `0` with `1` to get errors at compile-time.

### what are open questions about unsafe const fn?

* union field accesses are allowed in const expressions... so we have to keep it.
* but union field accesses not allowed in `const fn`
* question 1: new kind of "bad behavior": doing things that are impossible at compile-time
    * main one: doing something that depends on concrete value of a pointer
    * josh: is the cast to `usize` itself erroneous? Or is it inspecting the resulting `usize`'s value?
    * ralf: have abstract representation for pointers (block+offset). Casts are no-ops.
    * ralf: but then trying to divide one of these abstract addresses (as `unsize`) by two causes a const-eval error
    * josh: so what about inspecting alignments? Or doing offsetof?
    * ralf: now we're getting into interesting questions
    * ralf: If the two pointers we are subtracting are in different allocated blocks, then nothing we can do (error). But if they are from same allocation, then we can compute the difference.
        * josh: this is not a problem; detecting and rejecting that at compile time is awesome.
    * ralf: the problem comes with `const fn`. Want to check its body for all reasonable inputs, but we cannot check this "same block" condition reasonably at time we are lookikng at definition of the `const fn`.
    * josh: can you write tests of `const fn` that run at const-eval time?
        * ralf: seems expressible via `const` item in `#[cfg(test)]`
        * Once we have `const {}` blocks, we could use those inside a `#[test]`
    * felix: surely there are other analogous problems, of things that are errors that you can only detect during dynamic evaluation of a given `const fn` invocation?
    * ralf: yes, but the analogous cases often correspond to panics during normal evaluation. Here we have dynamic misbehaviors that are only bad at const eval time?
    * ralf: what should we do for expressions that can cause these new bad behaviors? Should we require them to be in `unsafe` blocks? Or have a new kind of block (with a different keyword) to annotate such cases?
* question 2: what happens when CTFE code causes UB?
    * ralf: there is a spectrum of implementation choices here; on one end of scale, we could always catch the errors at compile-time. At the other end, we could cause UB at compile-time (which is not absurd; consider JIT'ing the const code and running it with no sandbox).
    * niko: with the dream of a miri that detects all UB, it would catch the erroneous cases?
    * ralf: today we run CTFE post mir-optimizations, so if mir-opt is leveraging UB, then we already will not catch such problems.
    * scott: can these things evaluate to something that causes UB at runtime?
    * ralf: we try to prevent that as best we can, but the check is incomplete.
    * oli: notably, the check inherently cannot cover user-invariants, such as some extra requirement the user is putting on particular raw pointers

ralf's solutions


* The `unconst` problem
  1. We could just declare such operations UB at CTFE time, and treat it like other CTFE UB. This is the easiest solution and forward-compatible with the others.
  2. make bad ("unconst") behaviors require `unsafe` even though they are not UB, as a kind of lint.
  3. ignore it. Won't cause UB, just clean interpreter shutdown.
  4. have a different keyword, like `unconst`, with blocks containing such behaviors.
 
 ```rust
 // Contains encapsulated "unconst" code but not unsafe code.
 const fn slice_eq(x: &[u32], y: &[u32]) -> bool {
    if x.len() != y.len() {
        return false;
    }
    // equal length and address -> memory must be equal, too
    if unconst { x as *const [u32] as *const u32 == y as *const [u32] as *const u32 } {
        return true;
    }
    // assume the following is legal const code for the purpose of this function
    x.iter().eq(y.iter())
}
 ```
 
 * Solution space for the UB-at-const-eval-time problem
     1. Reliably detect all UB (e.g. via miri) and then error. (This is costly in time and memory; e.g. need separate copies of MIR for each fn to run CTFE on the unoptimized MIR, need to validate invariants all the time, requires a fully precise spec of UB which we do not have yet, ...)
     2. Guarantee that compilation won't go wrong, but the resulting object code may see any value for the `const` -- effectively, consts that cause UB are uninitialized memory. This can cause UB when the program is run.
     3. Guarantee nothing; maybe even compilation will go wrong and cause UB at compile-time.

Currently, it sounds like (2.) above is most realistic w.r.t. cost-benefit and it is also what is currently implemented.
