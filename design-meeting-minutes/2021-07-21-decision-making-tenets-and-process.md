---
tags: "design-meeting"
---

# 2021-07-21: Lang team tenets, decision making, advisors

This document is not intended as the final word on lang team organization and processes, just an evolution we'd like to adopt. If this is approved, we'd like to start implementing the tooling to support it, and start using it as soon as the tooling is available.

This document consists of four related parts:

* Rust team tenets -- tenets related to consensus-based decision making
* Proposed decision process, illustrated -- a proposed decision process based on these tenets, illustrated through various vignettes
* Proposed decision process, detailed description -- a detailed look at how the decision making process works
* Lang team advisors -- propose adding a new subteam, the lang team advisors, somewhat analogous to the compiler team contributors

## Rust team "decision making" tenets

These are in order of priority. These are intended to be general enough that they should apply to any Rust governance team, not just the language team.

* **Treasure dissent.** When someone raises a concern, that's a chance to improve the design, and to discover and explore underlying values. Dissent should be an amicable, cooperative process.
* **Understand and cooperatively resolve concerns.** We cannot resolve a concern without first understanding it, including the underlying values motivating it. We should demonstrate that understanding by documenting the concern. We should consider the tradeoffs and the impacts on users, through the Rust design principles. We should always favor satisfying solutions over [satisficing](https://en.wikipedia.org/wiki/Satisficing) solutions.
* **Don't force an irreversible decision.** We should make as many decisions reversible as possible. Even when a decision seems irreversible (e.g., stabilization), we should strive to find a subset that still allows addressing the use case, or a subset that will support evaluation and data-gathering to enable making the decision in the future. If that's not possible, consider the null alternative; not making a change should always be the easier path, and the burden of proof to override a concern on an irreversible decision should be high.
* **Value expertise.** When cooperatively resolving a concern, or when considering overriding a concern, carefully weigh the advice and recommendations of experts. This includes team advisors, domain experts, and the owners or members of relevant initiatives.
* **Recording reasoning helps ensure good, consistent decisions over time.** Even if we decide not to sustain an objection, we should always record the objection and the reasons for our decision as a "dissent", as well as any unresolved questions for evaluation later in the process. The team member who raised the objection has the perogative to author that dissent and frame the unresolved questions (within reason).
* **Consensus doesn't mean unanimity.** Consensus means everyone feels heard and understood, and we've found a decision everyone can live with, but not necessarily the decision everyone prefers. 

## Proposed decision process, illustrated

### Reversible decision: merging a proposal

The process is best described by example. Suppose that there is a pending lang team proposal, and a lang team member would like to serve as the liaison. They contact the team leads and receive the go-ahead. They can then write:

> @rustbot merge
>
> I propose to merge this proposal. I think it will be a great addition to Rust!

This indicates that they would like to merge the proposal. At the moment, there is no decision pending, so rustbot would add a comment that looks like the following:

> Hello! @Alan has proposed to merge this. This is a **reversible decision**, which means that it will be affirmed once the "final comment period" of 10 days have passed, unless a team member places a "hold" on the decision (or cancels it).
>
> | Team member | State |
> | --- | --- |
> | @Alan | **merge** |
> | @Barbara | |
> | @Grace | |
> | @Niklaus | |

As the comment says, the PR is now in "pending decision" state, with Alan having kicked off the process with a proposal to merge. Alan's status of **merge** will link to his comment.

Now, for this particular proposal, Barbara has a concern. She thinks that the proposal has overlooked an important consideration. She writes a comment:

> @rustbot hold
>
> Did you consider reversing the polarity? Or the impact on the flux capacitor?

At this point, rustbot updates the state:

> Hello! @Alan has proposed to merge this. This is a **reversible decision**, which means that it will be affirmed once the "final comment period" of 10 days have passed, unless a team member pauses or suspends the decision.
>
> | Team member | State |
> | --- | --- |
> | @Alan | **merge** |
> | @Barbara | **hold** |
> | @Grace | |
> | @Niklaus | |

Alan is currently busy at work, though, so by by the time that he and Barbara get a chance to talk, 11 days have passed. (Alan and Barbara receive a ping from rustbot after a week or so.) Once they get a chance to talk, Alan addresses Barbara's concern, so Barbara posts:

> @rustbot merge

The comment is updated:

> Hello! @Alan has proposed to merge this. This is a **reversible decision**, which means that it will be affirmed once the "final comment period" of 10 days have passed, unless a team member pauses or suspends the decision.
>
> | Team member | State |
> | --- | --- |
> | @Alan | **merge** |
> | @Barbara | ~~hold~~ **merge** |
> | @Grace | |
> | @Niklaus | |

Barbara's previous ~~hold~~ status links to her previous comment setting her status to `hold`, and her current **merge** status links to her more recent comment setting her status to `merge`.

At this point, all the statuses are either empty, **abstain**, or **merge**, and more than 10 days have passed since the FCP started. Therefore, it completes immediately.

### Authoring an RFC (illustration of `rustbot restart`)

After some time, the proposal is completed and an RFC is proposed. This is a reversible decision. Alan, as the liaison, proposes to merge the RFC with `@rustbot merge`, and the decision making process proceeds as above.

This time, Niklaus has a concern:

> @rustbot hold
>
> I have not had time to read this yet! Give me a bit of time to write it up.

After 7 days have passed, rustbot writes to her:

> @Niklaus, I see you have placed a hold but 7 days have passed. Are you any closer to reaching a decision? (cc @Alan)

This continues for a week or two while Alan and Niklaus play "email tag". In the interim, Barbara decides she agrees with the RFC, so she uses `@rustbot merge` as well. The status now looks like this:

> Hello! @Alan has proposed to merge this PR. This is a **reversible decision**, which means that it will be affirmed once the "final comment period" of 10 days have passed, unless a team member pauses or suspends the decision.
>
> | Team member | State |
> | --- | --- |
> | @Alan | **merge** |
> | @Barbara | **merge** |
> | @Grace | |
> | @Niklaus | **hold** |

Eventually, Alan and Niklaus find a time to discuss, and Alan agrees that Niklaus's concerns are valid, so he makes some major edits to the RFC. Given that the RFC is completely different, he decides to restart the clock and writes:

> @rustbot restart

This resets the state of all team members to "abstain" and begins the clock anew. rustbot also pings the relevant team members:

> Dear @rust-lang/team, @Alan has restarted the clock!

The status now looks like this:

> Hello! @Alan has proposed to merge this. This is a **reversible decision**, which means that it will be affirmed once the "final comment period" of 10 days have passed, unless a team member pauses or suspends the decision.
>
> | Team member | State |
> | --- | --- |
> | @Alan | **merge** |
> | @Barbara | ~~merge~~ |
> | @Grace | |
> | @Niklaus | ~~hold~~ |

Barbara can use `@rustbot merge` to re-affirm her **merge** status, and Niklaus can use `@rustbot merge` to set his own status to **merge** since he agrees with the resolution of his concern.

### Authoring an RFC continued (Overriding a concern)

At this point, Grace has a concern, and explains that concern in detail:

> @rustbot hold
>
> I've thought about this a lot, and I don't think we should do this. Now that I see the syntax used in practice, I feel like if we do this it'll have an adverse effect on the ecosystem...

The status now looks like this:

> Hello! @Alan has proposed to merge this. This is a **reversible decision**, which means that it will be affirmed once the "final comment period" of 10 days have passed, unless a team member pauses or suspends the decision.
>
> | Team member | State |
> | --- | --- |
> | @Alan | **merge** |
> | @Barbara | ~~merge~~ **merge** |
> | @Grace | **hold** |
> | @Niklaus | ~~hold~~ **merge** |

Niklaus reads this message. He feels he understands the concern well, and agrees that this point hasn't yet been considered:

> @rustbot hold
>
> I agree. I think we should take more time to evaluate alternative syntaxes. What about...

Over the course of a few subsequent meetings and side conversations, Grace and other team members discuss the concern further; the initiative owner also considers the concern, and raises it with others working on the initiative.

The owner of the initiative updates the RFC to include a discussion of a couple of alternative syntax proposals. The owner recommends a slightly modified version of the originally proposed syntax, and outlines criteria that they feel the syntax should meet in order to support the use case.

Grace agrees that her concern has been understood, but does not agree with the proposed syntax. Grace feels the new proposal is an improvement, but her concern remains.

Niklaus feels that the team has understood Grace's concern, and furthermore, that the updated proposal addresses Grace's concern:

> @rustbot merge
>
> I appreciate the potential impact this may have on the ecosystem. However, I feel that as described in section XYZ of the RFC, the value of A outweighs the risk of B, and we've now mitigated the potential risk by...

(Notice that while Niklaus feels that the team has understood Grace's concern, he does not use "We" in his response, and does not imply that his summary represents the entire team.)

At this point, the entire team other than Grace agrees that the proposal should move forward:

> Hello! @Alan has proposed to merge this. This is a **reversible decision**, which means that it will be affirmed once the "final comment period" of 10 days have passed, unless a team member pauses or suspends the decision.
>
> | Team member | State |
> | --- | --- |
> | @Alan | **merge** |
> | @Barbara | ~~merge~~ **merge** |
> | @Grace | **hold** |
> | @Niklaus | ~~hold~~ ~~merge~~ ~~hold~~ **merge**|

(We'll assume, for this example, that Grace does not manage to convince anyone else.)

Grace takes some time, working with the RFC author, to add a dissent, including a specific unresolved question.

Grace (or another team member) then writes a comment containing `@rustbot dissent`. The status now looks like this:

> Hello! @Alan has proposed to merge this. This is a **reversible decision**, which means that it will be affirmed once the "final comment period" of 10 days have passed, unless a team member pauses or suspends the decision.
>
> | Team member | State |
> | --- | --- |
> | @Alan | **merge** |
> | @Barbara | ~~merge~~ **merge** |
> | @Grace | ~~hold~~ **dissent** |
> | @Niklaus | ~~hold~~ ~~merge~~ ~~hold~~ **merge**|

Since all statuses are now either **merge** or **dissent** rustbot also posts a comment:

> The final comment period has resolved, with a decision to **merge**.
> Note that this decision has dissents; please ensure these dissents have been recorded for subsequent consideration.

### Stabilizing a feature

The feature has been implemented and is now eligible for stabilization. Alan writes a stabilization report and posts it, and then issues the command

> @rustbot stabilize

Rustbot recognizes that a "stabilization" decision is irreversible, so the template is a bit different:

> Hello! @Alan has proposed to stabilize this. This is a **irreversible decision**, which means that it will be affirmed once all members come to a consensus and the "final comment period" of 10 days has passed.
>
> | Team member | State |
> | --- | --- |
> | @Alan | stabilize |
> | @Barbara |  |
> | @Grace |  |
> | @Niklaus |  |

This time, Barbara, Grace, and Niklaus must all explicitly provide a status before the decision can proceed. One by one, they join the PR. They must individually set their state using one of the rustbot commands.

Niklaus reads this and comments with `@rustbot merge`. But then, Niklaus uses the feature and discovers a crucial flaw. He posts a comment:

> @rustbot close
>
> After more testing, I believe this is not ready for stabilization. I have found that it doesn't work at all like the specification in the case of foo! This seems closely related to Grace's concern on the RFC; I think if we stabilize at this point we may indeed harm the ecosystem...

Other team members test as well, and find that Niklaus is right. Alan changes his status using `@rustbot close`, and Grace (with some relief) sets the same status.

Once everyone has changed their status, rustbot posts a comment:

> The final comment period has resolved, with a decision to **close**.

(Note: Since `close` is an inherently mutable status (a PR can always be reopened), rustbot can observe that everyone has set a mutable status, and will start treating the decision as mutable; this means the final comment period can end even if Barbara hasn't responded yet.)

## Proposed decision process, detailed description

* Entering a "decision period" can be done by having a team member tell rustbot an initial status (`merge`, `stabilize`, or `close`; or, `mutable` or `immutable` with a custom identifier).
    * All other team members have no initial status set.
    * During a "mutable" (reversible) decision period, if later commenters indicate the decision is immutable, the decision changes to immutable.
    * During an "immutable" (irreversible) decision period, if all commenters change their status to a mutable status (most commonly `close`), the decision becomes mutable.
* Once the "decision period" has begun, a clock of 10 days starts. The clock is never paused unless an explicit `@rustbot restart` command is given.
* The `restart` command sets the decision period back to its initial state (members decisions are also changed back to blank, as appropriate, though the history of their previous statuses is preserved).
* A decision is reached when the following conditions are met:
    * At least 10 days have elapsed since the decision period began (or when it was last `restart`ed).
    * Everyone is set to the same status, or to abstain, or to dissent (with at most one `dissent` status), or (for a mutable decision only) blank.
* Member may place a *hold* (raise a concern) at any phase of the process: proposal, experimentation, testing, stabilization. Ideally, concerns should be raised as early as possible.
    * In general, one of the liaison's jobs is to anticipate concerns that may arise and reach out proactively to the members of the team. If we find that team members regularly need to place serious holds for similar reasons, that may indicate we need to work harder at team calibration.
* "Holding" a decision is simple. If you have a potential concern that you haven't had a chance to fully articulate yet, you get periodic pings.
* In order to proceed after a concern is raised, whether sustaining the concern or overriding the concern, the team must understand the concern. This understanding should be expressed in writing rather than just verbally. Commonly, the owner of an initiative/proposal may incorporate the concern into the proposal and address it there (whether via their own words or those of a team member).
    * Ideally, a concern is only considered "understood" if the objector agrees it has been. If that is not possible, then everyone on the team other than the objector must unanimously agree that there is no further understanding to be gained.
* If the owner feels that the concern has been adequately addressed, they can produce a write-up that describes the concern and request a poll of the team members to see where everyone stands. 
    * Sustaining an objection requires one team member other than the objector. A team member should sustain an objection if either they believe the objection has been understood and they agree that it must be addressed, or if they believe the objection has not been fully understood (whether they personally agree with it or not).
    * If everyone else agrees that the concern has been understood *and* that the current design is sufficient, then the concern is *overridden*.
    * Team members are encouraged to regularly sustain objections they don't personally agree with; this is a normal part of the process.
* Whenever a concern is overridden, team members are encouraged to add a **dissent** into the document to describe their concern and why they don't agree with the decision. They (or another team member) should then set their status to `dissent`.

## Rustbot commands

The following commands are accepted by rustbot. A number of comments take one or more optional `@member` arguments, denoted `@member*`; if supplied, the command is issued on behalf of those member(s), instead of the person writing the command. (The `@` for each member is required.) It is also permitted to write `@rust-lang/team` to select all members of the team. rustbot will always link from each member's row in the table to each comment they made that changed their status.

* `rustbot @member* mutable ident` or `rustbot @member* mut ident` or `rustbot @member* reversible ident`
    * If not in a decision period: begin a reversible ("mutable") decision, proposing the outcome `ident`; all other members are set to a blank status.
    * If in a decision period: set yourself to `ident`. (Decisions do not proceed unless all members have the same status or `abstain`.)
* `rustbot @member* immutable ident` or `rustbot @member* irreversible ident`
    * If not in a decision period: begin an irreversible ("immutable") decision, proposing the outcome `ident`; all other members are set to a blank status.
    * If in a decision period: set yourself to `ident`. If the decision was previously considered mutable, change it to immutable. The decision now requires full team consensus. (Members may set a status of `abstain` on the decision if they wish.)
* `rustbot @member* merge`
    * Alias for `rustbot @member* mutable merge` - a mutable decision with the proposed outcome `merge`.
* `rustbot @member* close`
    * Alias for `rustbot @member* mutable close` - a mutable decision with the proposed outcome `close`.
* `rustbot @member* stabilize`
    * Alias for `rustbot @member* immutable stabilize` - an immutable decision with the proposed outcome `stabilize`.
* `rustbot @member* hold`
    * If not in a decision period: error
    * If in a decision period: set your status to `hold`
        * Every N days while a hold persists, rustbot will ping all members who have status `hold` (and potentially other team members, TBD)
* `rustbot @member* abstain`
    * If not in a decision period: error
    * If in a decision period: set your status to `abstain`. This status does not block a decision.
* `rustbot @member* dissent`
    * Equivalent to `abstain`, except that it sets a status of `dissent`. This status on one team member does not block a decision. A status of `dissent` on two or more team members will block a decision.
* `rustbot restart`
    * If not in a decision period: error
    * If in a decision period: set all members other than the one issuing the command to have a blank status. Preserve the last set status of the person issuing the `restart`. 
    * Note: if the last set status of any team members were immutable, the decision will continue to be treated as immutable until all such members set an explicit status otherwise.
* `rustbot cancel`
    * If not in a decision period: error
    * If in a decision period: cancel the decision period.
    * Note that if a subsequent decision is started in the same issue, rustbot should link to the previous decision summary table.

## Language team advisors and domains

Finally, we propose adding a lang subteam analogous to Rust contributors, called the *language team advisors*:

* Recognize people for their regularly called-upon expertise.
* Add people who we regularly consult, and whose recommendations we trust.
* Document (loosely) what domains we consider them to have domain expertise in.
    * We should provide similar documentation for the domain expertise of lang team members.
* Folks may be added by recommendation from a lang team member if leads agree and there are no objections or moderation-team red flags. (Proposed additions will be discussed privately.)
* Since the lang-team advisors will not have any regular meeting, or expectations other than responding to lang-team consultations, the team size is not limited by the logistics of live meetings. We should still keep the list reasonably manageable, though.
* Inactive advisors will be moved to "emeritus" status as usual.
* Being a member of the advisors does not grant formal decision-making authority.

## Frequently asked questions

### Do we want to require "all but N" people to affirm a decision, as rfcbot does?

We opted to require 100% participation on immutable (irreversible) decisions such as stabilization, but not on very lightweight mutable (reversible) decisions such as starting an initiative. We believe that lang team members can be expected to at least leave a comment (even if just `rustbot abstain`), but in the limit we also have the option to leave comments on others' behalf.

### Why does the timer start when the decision period *starts*, and not when a consensus is reached?

The current rfcbot starts the 10 day FCP timer once a consensus is reached. However, we have observed that, in practice, the "holdouts" on consensus are typically the lang team members. Further, this seems to assume a "two-phased" decision making model where folks outside the team are commenting only after the lang team has reached a consensus. In practice, decision making tends to be more fluid, and hence we would prefer not to have a long delay after consensus is reached.

### Why not use checkboxes?

This process recognizes that sometimes, in the flow of conversation, the decision to merge can transmute into a decision to close, or there may be multiple potential outcomes.

The presentation of statuses also makes it easy for rustbot to present the history of status changes, with links to team member's messages.

### Why is `close` always reversible?

Because, well, it is! We can always re-open a PR or issue, after all.

### What about "postpone"?

Just write it in the comment for a `@rustbot close`.

### Why can members override other members positions?

It's quite common to want to check boxes and things on behalf of all the people in a meeting. It's also annoying to not be able to clear the objections of people on vacation, or who have left the team, or whatever. We can trust each other on the team. If people abuse rustbot to disrupt the process, that isn't a problem to be solved with tooling.

## Meeting questions

(to be filled in...)

### Q: What is required to switch between mutable and immutable?

* "mutable vs immutable"
    * default is that *merge* and *close* could be mutable
    * default is that *stabilize* would be immutable
* but if there is dispute, that requires consensus to settle

### Q: What is satisficing?

* Avoiding design by committee, or something that "checks all the boxes but nobody loves it".
* What about "perfect, enemy of good"?

### Q: Relationship to draft compiler team contributor guidelines?

 * pnkfelix: I had [drafted a set of Contributor Guidelines](https://hackmd.io/TYGRjVIbSBmxpbcfDzll-w) for the compiler. We ended up not adopting them as-is, and instead planned (but didn't act) to revise them. Anyway: It might be good exercise to compare/ contrast them against what is written here.
     * josh: It seems like that draft is aiming to cover primarily interactions between the team and the rest of the community, more so than interactions within the team (e.g. decision-making and conflict resolution). Is that an accurate interpretation?

Notes:

* pnkfelix: Discussion between team members and community at large probably have a lot of things in common, but Jthis document is focused more on the formal structure for making decisions.
* Josh: This is intended to be both formal structure and process but also principles. Not intended though to be "every bit of process around the lang team", focused on decision making / conflict etc. There are other things in the Contributor Guidelines we may want to adopt as well.
* pnkfelix: I meant it as a way to explain why we make a given decision (e.g., we prioritize X over Y).
* nikomatsakis: these are more like "how" to make decisions not "what" to decide.
* pnkfelix: sounds good.

### Q: Do members start out as abstain or not? (resolved)

* Mark: There seems to be some confusion/disconnnect in some places in the text around whether a member starts out "abstaining" or is simply not registered with particular feedback; it seems useful to explicitly denote this.
    * Josh: That's an artifact of an incomplete edit most likely; could you point those out?
    		* :thumbsup: happy to do so inline, sounds good, likely on a PR or whatever.
                * hackmd has comments; could you use those to flag the text in question? (The correct version is that team members start out with no status, not "abstain"; if you see the latter please flag it with a comment.)
                * it's somewhat scattered - I think it's not major for reading. but if I notice I'll flag

### Q: "Don't force an irreversible decision?"

* Niko: The doc says

> **Don't force an irreversible decision.** We should make as many decisions reversible as possible. 

but there seems to be some "countertension" that is missing, right? For example, one way to satisfy this would be never stabilizing, which I don't think we want. It's more like "if there is some concern, sometimes it works to leave the door open to resolve that concern in the future"..?

* Niko: feels like there's a missing "don't let perfect be the enemy of the good" tenet.
* Josh: I think prioritizing that would be a challenge, since we have to decide whether to bias towards making decision. Good to recognize that they are in tension, but we also need to find.
* Niko: Could combine then into one tenet.
* Mark: The initiative process forces towards progress, since there are only so many things a liaison can do, so as they want to move on, they have to stabilize. Some push towards resolution through that.
* Josh: Agreed. Also, with an initiative open, there are multiple decision points along the way -- e.g. we may all agree that a problem needs solving, but disagree on the correct solution or quality of the impl. But saying we don't want this solution doesn't mean we don't want to solve the problem.
    * Ultimately someone might come back with "we've done more exploration and here are the other options plus tradeoffs"
    * We can re-evaluate and decide whether to either give up or go with something.
    * The fact that this is a multi-step process provides valuable countertension, we'd have to deliberately decide later to stop trying to solve the problem. We can decide against a specific solution while still wanting to solve the problem.
* Niko: I do feel we sometimes have a problem with perfect being the enemy of the good, though, so I think we should say it somewhere.
* Taylor: I think there are a number of places where having the best solution turned out to be less important than having *some* solution. Basically Worse Is Better. We probably won't settle it in 1 meeting, but...
* Josh: How much impetus there is to solve the problem is how much motivation there is to dig into the conflicts. If we find we are strongly disagreeing on something and we'd have to spend a lot of energy to resolve it, but the problem doesn't seem that important, maybe we shouldn't try to do it.
    * If the problem is very important, it may be worth spending the energy and the stress to do it.
* Taylor: What I'm saying is I'm not sure we want to have a tenet that just stands in opposition of WiB.
* Mark: Even within stabilizations, there are ways of framing decisions that aren't necessarily irreversible. So it's a useful principle that, if there is significant disagreement, find a way you can make something reversible.
* Josh: To be clear, it's not don't *make* an irreversible decision, it's don't *force* one, and there's a reason it's 3rd, not 1st.
* Taylor: Maybe I want more context about what force means.
* Niko: I agree that we should avoid something that could be misinterpreting as "don't do anything that isn't perfect"; what I had in mind for this was "sometimes it's useful to find what people agree on, do that, and come back to the question", which in a way is the opposite of "perfect enemy of good", since perfect is "solve it all"
* Josh: Note what the delta is from the status quo to this proposal. The emphasis here is mostly about finding ways to move forward when there *is* concern, a new capability for resolving concerns (by registering dissent). We can already block things indefinitely today, this is trying to give us a process for making progress instead. Don't see much danger of this making us *not* do things we would otherwise have done.

### Q: inviting concerns from non-team members

* Mark: rfcbot has the option to "invite" concern raising from someone not part of the initial list. Is this desirable?
	* Particularly may be that language team advisors should at minimum be pinged, if not blocked on?
    * Josh: At a minimum, a team member can always register a concern on their behalf and point to their comment. Worth considering additional possibilities though. But it might be painful to deal with the additional process complexity of something like "can participate but not block in the same way"; the tooling here is intended to facilitate the process rather than fully implementing/enforcing it.
* Mark: main thing I'm thinking is that, procedurally, it may make sense for the process to decouple itself from things being tagged t-lang and so forth. There are shorthands for "this group of people" but it's easy to add people. e.g. to stabilize an RFC it probably makes sense to include the initiative owner. If they're not ok with merging it, maybe we shouldn't merge.
* Niko: I'm intrigued by the idea of letting advisors block. If Ralf wanted to block an RFC, I'd want to hear him out. Obviously it does mean that we have to be careful about who we add to the set.
* Josh: We may want to have a different treatment in the process. Not totally sure how to do that. Might want a process for that, but I don't think it should be the *same* process as for team members.
* Niko: I think I'm flexible, to some extent, although I think that being an advisor should mean you can tell people to do stuff.

### Q: When does one write a dissent?

Regarding dissent:

> pnkfelix writes "I might want more guidance as to what pre-requisites for a `dissent` are, though. Is it enough to say "I have a gut feeling that this is a bad move", without concrete actionable steps for someone to take in response (beside discarding the proposal)?"

* pnkfelix: What's the expected *outcome* from a dissent? What do we expect to happen?
* nikomatsakis: I think the role I had in mind was both emotional satisfaction but also that this can occur fairly early in the process, e.g. RFC phase, and you might want to leave ideas about alternatives. It might be that with more experience people come to agree with you.
* joshtriplett: I have at times felt that I had a concern early on, but I didn't feel strongly enough about it to block, and then I saw it arise later, and people were like "the time to raise that was earlier".
* pnkfelix: That seems to imply an unresolved question attached to a dissent, no?
* joshtriplett: I do think we should at least look at dissents later on to evaluate that question. In particular, a dissent at the proposal stage should be evaluated at the RFC; RFC dissent should be re-evaluated at the feature complete, etc.
* pnkfelix: That's fine. The example I gave was "bad feeling" but that isn't something you can act on. We should be able to engage with people and "work out" what you are concerned with.
* joshtriplett: That is part of what "hold" is meant for -- a short-term hold is meant for "I'm still trying to formulate my concern". Shouldn't be indefinite, and should be collaborative. We should be able to ask questions, doesn't have to be adversarial. By the time a hold becomes a dissent, it should be better than "have a bad feeling".
* nikomatsakis: I like this. I also just realized that something you can do here is come in with a "dissent" right off the bat, if it's a debate that's been had but you don't want to block.

### Q: Do we really want restart?

* Mark: The doc emphasizes "restart"s as a mode of operation. Is this what we actually want? GitHub is bad at long threads, and restarting on a new thread may be better for that reason at least. Tracking the "phases" and concerns etc raised is also easier when you have more explicit splits.
    * Josh: I'd expect this to happen when (for instance) an RFC has changed enough to be worth re-checking with people, but not so much that it's a *completely* different proposal. It'd be valuable, in such cases, to keep it attached to the RFC/PR/etc that we want to merge.
* Niko: Good point, I have advocated in the past for closing and re-opening RFCs to help people get refocused.
* Mark: My main concern is that restart clears folks positions but doesn't draw a clear line. If we're just re-checking with people, then that seems like.. maybe.. "restart" isn't the right word then?
* Josh: "Recheck" might be better. 
* Niko: What's an example where we would want to use this?
* Mark: Here's an example. We have a discussion in triage, everyone seems in favor, a day later after everyone has registered agreement, someone points out a factual error. The thread itself is correct, but that summary is wrong. Let's clear every comments and recommence.
* Josh: A similar example came out of a libs team thing. We had cases where we're looking to stabilize, etc, etc, we kick off decision, then there's a bit of a bikeshed, we decide to change the name (but nothing else), so we get PR author to change the name, but we want to restart the check process just in case somebody is like "wait no!"
* Niko: I am hoping that the initiative process will also lead to shorter threads. Seems like we want to rename to recheck, at least, maybe hold on implementing it.

### Q: Who can start a proposal?

* Mark: Should lang advisors (or some other set not specifically lang) be allowed to *start* a proposal? Currently it is not infrequent that lang meetings arrive at "let's initiate fcp", and then are blocked on a lang member making the proposal. Particularly given the initiative work, it may be desirable for that duty to be on the initiative owner, not the lang team -- they can write a better document summarizing the state.
    * Josh: We may want the liaison to have the specific responsibility of initiating this when viewed as ready. There's no *technical* reason why we couldn't make it possible for someone outside the team to kick off FCP and just have *all* the team members start off with a blank status. But there may be process reasons not to do so, or to limit that capability to specific teams.
	* (Note that today such a technical limitation *does* exist, so this would be a change in some sense.)
* Niko: we didn't discuss this *specific point*... it'd be a good question how to know 
* Mark: I'm thinking roughly "any person on any rust team"
* Felix: there are two things, right? who has responsiblility to write the doc, and who can initiative FCP?
    * Those are separate right, i.e., liaison can go to the owner
* Josh: Mark's question I think was all about capability. Should we add the capability for someone to do this?
* Mark: It's kind of both. I do think it's valuable. Often the case that Ralf says "yes this seems vaguely good" and then we discuss it on lang. But if Ralf could say "fcp merge" directly, it puts more impetus to give a more thorough investigation.
    * e.g. to in that comment say more about it.
    * basically like niko's idea of nominating (with a template saying what is there)
* Niko: it feels analogous to compiler contributors having r+. Low risk. Advisors are people we trust to make tactical decisions in their area of expertise and to know that area.
* scottmcm: right now, we have people write it, and put I-nominated, and then we look at it. if it just showed up because it was in the list of things we started.
* niko: I like it, more empowering.
* Mark: Might also allow us to do more stuff async
* Niko: two questions:
    * who can initiative (no risk)
    * who can block 
    * might be nice if any Rust member etc can write a dissent, at least