# Planning meeting 2021-06-01

Attendance:

* Team: Niko, Scott, Josh, Taylor, Felix
* Others: simulacrum, Mara

## Design meeting proposals

* [Design note catch up #95][]
* [Structural equality #94][]
* Status updates from projects
* Lang team process proposal

[Design note catch up #95]: https://github.com/rust-lang/lang-team/issues/95

[Structural equality #94]: https://github.com/rust-lang/lang-team/issues/94

## Calendar this week

### Availability

* Today: Design note catch up
* June 9 -- Niko, Felix not available
* June 16 -- Structural equality
* June 23 -- Possible: Atomic follow-up
* June 30 -- Lang team process proposal

## Design note notes

Goal: 

* Role of a design note is to document the design space that is explored and tradeoffs uncover
* Consensus does not represent lang team agreement to accept the feature, just that the document is generally accurate
* Someone from lang team was going to own and merge when they felt it was ready, running it by the appropriate stakeholders

### "Initial draft of copy ergonomics design note" lang-team#62

**Link:** https://github.com/rust-lang/lang-team/pull/62

* Niko for Mara: range discussion from libs team?
* Niko: could reference match ref bindings interactions
* Scott: feel pretty good, looks like there've been a number of updates
* Josh: added a comment with a suggested change
* Felix: I don't know if it is sound for `MaybeUninit<T>` to be copy?
    * cramertj: When `T: Copy`, right?
    * scott: it's safe to copy, it's the "assume uninit" that is not
* cramertj: today, copy is the only way to know whether something can be memcpy'd, but I think we don't necessarily want that to be implicitly copyable.
    * Some types should be memcpy'able but not necessarily implicitly copyable
    * Others could plausibly be "implicitly cloneable" (rc, arc), even though they are not memcpy.
    * Mem copy may be important for specialization.
* Mark to add notes
    * may want to link to https://github.com/rust-lang/rfcs/pull/2111
* Assign to pnkfelix

### "Autoref/autoderef for operators" lang-team#63

**Link:** https://github.com/rust-lang/lang-team/pull/63

* Niko: it would be nice to have more canonical examples, what are the pain points?
* link https://github.com/rust-lang/rfcs/pull/2111 in this note
* Mark: String + &str, String + str etc
    * depends impls can be fn(self, Other) or fn(&self, &Other)
* Felix: do we expect to be able to avoid the inference breakage if we add this feature?
    * Josh: I think part of the point was to have the compiler be able to figure out which impl it needs better
    * Mark: if you know that these impls are all equivalent might mean you can be more intelligent
        * implications of adding in generic code (e.g., `T` + `&T`)
* Niko: examples of breakage are useful
    * Scott: adding `Add` to `u32`
    * `0 + s.parse().unwrap()`, plausibly 
* Mark to add notes
* Review: scottmcm

### "Auto trait design note" lang-team#69

**Link:** https://github.com/rust-lang/lang-team/pull/69

* Scott: Covers semver and reflects the conversation
* Niko: I'd be up to merge as is, and maybe add some notes later on related topics
* Niko to own merging, Mark to quickly add forgotten auto traits

### "Add design notes for function-type `Default` implementation discussion" lang-team#71

**Link:** https://github.com/rust-lang/lang-team/pull/71

* Josh: Feels sort of like a rebuttal
* Niko: seemed to cover all the points in a satisfactory way
* cramertj: Not the use of the word thunk that I'm familiar with. 
    * Felix: me too
    * cramertj: I'm accustomed to a "thunk" being a function that you call that produces more steps
    * Josh: I think I can see how the term "thunk" would apply
    * scottmcm: Trampoline?
    * cramertj: not very clear to me how you would call `add_one_thunk`, given that you can't name the function type?
        * niko: possibly with a named impl trait
* josh/pnkfelix: I'd like to see definition plus usage
* cramertj: wouldn't the use cases here be covered by a fn trait that doesn't take self?
    * niko/scott: fnstatic is in the doc
* mark: I think that, unlike the other design notes, this feels more like it's a summary of how you could achieve the feature
    * niko: I'm good with that I just think it needs a clearer title
    * josh: it feels like "we want to add this thing, here' show to do it"
    * niko: I'm ok with that, but I think it should just be "if we want to add this thing..."
* mark: I think it'd be useful to have more examples of how this might look
    * in particular how do you use this thing and why
* pnkfelix: what about generators? of course, if there are no upvars...
    * ...at least for opt-in trait impls, you might want that for generators, right?
    * niko: that sounds like a consideration to add
* scottmcm: I don't know if this needs to be in the doc, but all the questions around specialization makes some of the leakage things different, but maybe that's not specific to this note in particular, and is just something for the stabilizing-specialization conversation.
* Niko to follow up, suggest and/or make a few minor edits, then merge

### "Add draft of variadic notes" lang-team#76

**Link:** https://github.com/rust-lang/lang-team/pull/76

* cramertj: to follow up