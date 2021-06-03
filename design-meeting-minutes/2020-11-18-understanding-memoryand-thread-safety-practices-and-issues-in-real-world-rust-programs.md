# 2020-11-18: "Understanding Memory and Thread Safety Practices and Issues in Real-World Rust Programs" 

* Lang-team issue: [#53](https://github.com/rust-lang/lang-team/issues/53)
* [PDF of paper](https://cseweb.ucsd.edu/~yiying/RustStudy-PLDI20.pdf), and a [summary of key takeaways](https://hackmd.io/@Lokathor/rkb09qtVP) prepared by Lokathor (thanks!)

## Questions to ask (please add your name)

* Josh: I think we should ask some open-ended questions and listen for the first ~15 minutes, before we ask more focused and specific questions.
* Josh: What is the biggest thing we're missing, in terms of catching errors? Is there something well-established, currently done in some other language, that we're not doing anything to detect and should?
    * Linhai Song: Better GUI tools. Great interaction between developer and compiler, but would like information surfaced during development.
    * Linhai Song: Double locking. Would like to see a way to explicitly drop a lock. Much higher instance of this type of error compared to C and C++. (Paper also mentioned highlighting the scope of such locks in IDEs.)
    * Niko: following up on double locking, regarding temporary lifetimes. Locks didn't get assigned to temporaries, locks lived longer than anticipated.
    * Yiying Zhang: Cases where the lock guard lives in a compiler-generated temporary rather than an explicit variable are the source of many of the errors. Also, even if this doesn't cause a bug, the bug, the critical section may be longer than expected. Would like to see this highlighted.
    * Yiying Zhang: Perhaps a way to mark pre/post conditions of unsafe code.
    * Yiying Zhang: Functions marked as unsafe that don't have unsafe code, but could cause other code to be unsafe. Thoughts about whether all such functions should be marked unsafe, or just constructors? Some common patterns there.
    * Felix: questions about the patterns around locks, is it that explicit variables are not convenient at all or that they're not convenient because method chaining and similar encourage not having an explicit variable?
    * Josh: Would something like a `with_lock(|| ...)` method potentially be a good fit?
    * Linhai Song: Yes, in some cases. I think giving developers more flexibility would be helpful. In C/C++ there is explicit lock/unlock which may help as well.
    * Josh: (Idea for later discussion in the lang team: could we make a way to "name" anonymous temporaries by way of the thing that keeps them alive? given `let x = lock.get().xyz();` could we name `x.some_syntax_here.lock`?)
* Josh: Across the unsafe code you looked at, did you find any common patterns for which it seemed like there *should* be a safe approach? What direction should we look to expand the capabilities of safe Rust in, if we want to help people use less unsafe?
    * Linhai  Song: In some cases, unsafe code is required. Implementing low-level systems requires accessing hardware and that requires unsafe code. People use it for better performance, such as dropping bounds checks. Maybe we can apply analysis to help figure out where things are correct.
    * Josh: So improving performance of safe code so that folks feel less need to use unsafe for performance is one way. Are there maybe other ways?
    * Yiying : People need to use external libraries, too, that are not yet supported by Rust. That's not a language design issue, but as more libraries are added to Rust that will be reduced.
* Niko: One finding that I thought was quite promising was Insight 4, that all memory-safety issues involved unsafe code. This has always been a key selling point of Rust in my mind -- it's not that you can't make mistakes, but you know where to start looking. 
    * Similarly, I liked your suggestion of using unsafe code blocks as a hint for where to start linting.
    * However, it was interesting that this was not true of concurrency -- this isn't surprising, given that `Atomic` types offer safe methods. Lately we've been working towards exposing other sorts of "safe variants" of unsafe methods, such as safer transmute.
    * However, this threatens to further limit our ability to vet "tricky" code.
    * Do you think this may suggest value in having some kind of "intermediate" level of unsafe, to flag "easily misused" operations like atomic ops or safe transmute? Or do you think it might be better to offer targeted lints?
    * Yiying: Yes, that would be useful
    * Felix: like `mem::forget`, right?
* Josh: If you could have us change one thing in Rust, what would it be?
    * Niko: it seemed to me that temporary lifetimes were the most concrete source of bugs
    * Linhai: a tag to identify interior unsafe functions require careful analysis, as a way to label said functions
    * Niko: ways to make contracts more explicit, like pre/post conditions or unsafe fields, seem relevant
    * Some discussion about what point at which the unsafe is encapsulated "enough" (otherwise it would affect every caller recursively all the way up).
    * Niko: something I've noticed in my own unsafe code is that the invariants are clear up front, but they change over time. I'd be curious to know how often bugs are introduced "up front" or as a result of changes over time.
    * Yiying: Interesting idea, we didn't look at that, but we did gather some data I think about "does the fix introduce a new bug"
    * Linhai: we didn't record it in the paper though
    * Yiying: we talked about extending the paper to a longer journal paper, and we have a bit more data that we could add
    * Felix: the bugs you surveyed came from existing codebases, right? Often people will log the point where a bug was introduced, so you could extract it.
    * Linhai: we could definitely check whether those lines of buggy code to understand if an existing invariant was invalidated or a new invariant was introduced. 
    * Yiying: would some way to write out the list of invariants
* Niko: My impression from the discussion of condition variables and channels in the paper was that Rust didn't present any unique challenges. In other words, the kinds of bugs that one sees here are the same sorts of bugs you might find in other languages. Do you agree with that?
    * Linhai: very similar to Go language, but one difference is that Rust will panic if there is no receiver.
* Felix: did you ever see cases where the lock lifetime was *shorter* than expected?
    * Linhai: No, we never saw that, because you have to keep the lock guard active to access the data inside. All the bugs we studied were caused by holding a lock longer than expected.
* Niko: you mentioned a number of lints and analyses in the paper, is there any interest in getting those tools upstreamed?
    * Linhai: how perfect does it have to be? Are false alarms a problem?
    * Josh: if you give a warning, and it has rare false positives, but it's usually right, that can be ok, especially if the code that generates false positive would probably be better to change anyway. We do that in some cases already.
    * Niko: as an example, I think that it could be useful to have a lint for "temporaries with non-trivial destructors whose lifetimes wind up surprisingly long", but we'd have to tune it. 
    * Felix: clippy is a good place for this
    * scottmcm: temporaries getting explicit names might be a good example
    * Niko: another candidate might be loads/stores of atomics that are occurring in close succession, that might be a bit trickier, though I think adding `#![allow]` is ok in cases like this *personally*
* Yiying: what has changed, what's the future plan for Rust?
    * Niko: I do think that unsafe code is going to be a big focus in a number of ways. I would like to see us focusing on safety in practice, not just safety in theory.
    * Felix: IDE support is also going to be an interesting thin
    * scottmcm: paper fits well into the secure code working group as well, and the library changes around things like uninitialized memory. 

## Questions we didn't ask or which got covered "in passing"


* Niko: Identifying data sharing in buggy code -- you mentioned an analysis that looks for internal mutation with `&self` methods and the like, as well as considering when `Sync` is implemented. Did this detector encounter any previously unknown bugs?
* Niko: Did you write any kind of lint that targets atomic writes? For example, the move from separate load/store to a combined exchange seems like something we might be able to lint against, although I worry that it would ultimately lead to too many false warnings.
* Niko: In Section 6.2, after Insight 8, the paper writes "When multiple threads request mutable references to a `RefCell` at the same time, a runtime panic will be triggered." However, `RefCell` are meant to be confined to a single thread. The paper goes on to talk about mutex poisoning -- are the references to `RefCell` here actually referring to a read-write lock or a Mutex?
* Niko: When it comes to mutex poisoning, I'd be curious to hear your take. Did your analysis of concurrency identified bugs specific to "tearing down" or "handling exceptional conditions"? Mutex poisoning, along with errors are writing to dead channels and the like, were meant to help in propagating panics across threads, but it's never been clear to me how successful this strategy has been in practice (this was in part a response to our removal of a "linked failure" mechanism, which occurred when we moved from a green-threaded runtime to native threads).
* Niko: Your analysis suggests a few potential lints, some of which I think you implemented yourselves:
* Felix: Suggestion 7 says Rust should an explicit unlock API of `Mutex`. Am I right in inferring that this is asking for something where the `Mutex` itself offers an `unlock` method, that would (somehow) have the same effect that dropping the return-value of `lock()` has today?
* Niko: In section 4.1, the paper notes that one pattern for unsafe constructors is to capture the case where there is some invariant that must be maintained, even though the code itself would not require unsafe.
    * One solution that has been proposed for this is "unsafe fields", where we tag fields as unsafe to capture the idea that there are invariants that must be maintained that involve the value of this field.
    * I'd like to get your feedback on this idea but also whether there are other kinds of additions that might be useful for capturing unsafe invariants?
* Niko: One area that you highlighted as a real source of confusion is what I call "temporary lifetimes". These involve the lifetime of intermediate values that are created in the course of evaluating an expression, such as the mutex guard when you do `foo.lock().bar`.
    * The paper says "Rust's complex lifetime rules together with its implicit unlock mechanism make it harder for programmers to write blocking bug-free code" -- I think that temporary lifetimes are what you mean when you say "complex lifetime rules", right?
    * You mentioned that you implemented an analysis targeting these kinds of mistakes when it came to "use after free" and what sounded like a *separate* analysis for double locks.
    * I am wondering if it would make sense to try to create some built-in lints targeting temporary lifetimes, perhaps looking for cases like "creating a raw pointer to a temporary" and/or "tempories whose destructors are declared to have side-effects".
    * In general, one thought I've had for some time is that it is unfortunate that we don't distinguish `Drop` destructors which "simply free memory" and those (like mutex guards) that have side-effects.
