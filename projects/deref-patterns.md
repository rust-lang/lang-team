# Proposal

Allow pattern matching through types that impl `Deref` or `DerefMut`. 

## Summary and problem statement

Currently in rust, matching is blocked by bounderies like smart pointers, containers, and some wrappers. 
To solve this problem you would need to either use if let guards (unstable), or nested match/if-let. 
The former is limited to one such level, and the latter can become excessive for deeply nested types. 
To solve this, it is proposed propose that "deref patterns" be added, to allow for such matching to be performed.

An exception to the above problem, is that `Box<T>` can be matched with `feature(box_patterns)`. 
However, this is magic behaviour of box, and a goal of this project is to remove or reduce that magic. 

The proposed solution has a number of unanswered questions, including the syntax for patterns, whether or not to limit to standard library types,
 and how to allow exhaustive patterns soundly if not limited to standard library types. 

### Exhaustive Patterns and Soundness

One current issue with the proposed deref patterns is that if applied generally to any type that implements `Deref`, 
it cannot soundly permit exhaustive pattern matching, as a malicious `Deref` impl could return a different value each time.
An trivial example would be  `Deref` impl that returns a static reference to an enum value that is chosen randomly
There are currently 3 different possible solutions:
* Do not treat deref patterns as exhausitve
* Restrict deref patterns to (possibly a subset of) standard library types
* Expose an unsafe lang item trait called `DerefPure`, and restrict deref patterns to implementors of that trait

Part of the Project Group would be to evaluate the viability of each solution, and any other reasonable solutions which may come up. 


## Motivation, use-cases, and solution sketches

Recursive types necessarily include smart pointers, even when you could normally match through them.
For example, in a work in progress proc-macro to support restricted variadic generics, the parser needed to match "fold expressions", which take the form `(<pattern> <op> ...)`. With deref patterns, this could be implemented using `Expr::Paren(ParenExpr{expr: Expr::Binary(ExprBinary{ left, op, right: Expr::Verbatim(t), ..}), ..})`. However, this is currently not possible, and required nested matches.  
This generalizes to any case where you need to check some pattern, but hit a deref boundery. 



## Prioritization

This likely does not fit into any of the listed priorities, though it may be considered "Targeted ergonomic wins and extensions".

## Links and related work

This has been discussed on the Rust Internals Forum at <https://internals.rust-lang.org/t/somewhat-random-idea-deref-patterns/13813>, 
as well as on zulip at <https://rust-lang.zulipchat.com/#narrow/stream/213817-t-lang/topic/Deref.20patterns>. 

A tracking document of all currently discussed questions and potential answers can be found here <https://hackmd.io/GBTt4ptjTh219SBhDCPO4A>. 

Prior discussions raised on the IRLO thread:
* https://github.com/rust-lang/rfcs/pull/462
* https://github.com/rust-lang/rfcs/issues/2099
* https://github.com/rust-lang/rfcs/blob/master/text/0809-box-and-in-for-stdlib.md

## Initial people involved

Initially, Connor Horman and Nadriel on zulip would be involved
