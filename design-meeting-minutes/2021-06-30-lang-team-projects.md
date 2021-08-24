# Lang team initiatives

## Goals

- **Empower individuals and give ownership:** 
    - Each initiative in this proposal is ultimately owned by a single person who drafts the proposals and recommendations. 
    - The role of team is to review the designs, provide feedback, and ultimately decide whether to accept the design.
    - The team can introduce constraints and requests that the owner should either satisfy or explain why they are not able to do so.
- **Minimize friction for "reversible" decisions:** 
    - We often require checkboxes or other things for decisions that are ultimately reversible. I am trying to shift us towards a model where we trust individuals to make decisions, but those decisions are logged and reviewed, and we can reverse or halt them if there are concerns.
- **Liaison as mentor and decision maker:**
    - The liaison role has always been a bit unclear. I have tried to clarify it considerably.

## Summary

| Phase | Goal | Permits | Successful exit |
| --- | --- | --- | --- |
| Proposal | Find a team liaison | Discussion | Team member seconds, thereby agreeing to act as liaison |
| Experimental | Refine the design and work towards an RFC | Active zulip stream; tracking issue / repository; code can land under "unstable" feature gate | RFC is approved by team |
| Development | Finalize design and implementation | Removing the "experimental" tag on a feature | Liaison declares the proposal feature complete |
| Feature complete | Gathering feedback | Advertisting the initiative as "feature complete" | Stabilization proposal approved |
| Stabilized | Enjoy | Use on stable branch | (none) |

## Roles and concepts

* **Initiative:** An effort with a clear goal or deliverable. Typically this will be a change to the language, but it could also be a piece of documentation, a specification, or something internal to the lang team.
    * *Side note:* We have usually called these "projects", but I opted for a different term here because I've several times encountered confusion where people thought I meant "the Rust project".
    * Alternatives: "project", "feature", or "deliverable" as alternatives.
* **Initiative owner:** The person who is driving the initiative. They are the one who drafts the design and who is ultimately responsible for the initiative.
    * **Tasks:** (note that in practice these may be delegated or done in concert with a group)
        * Exploring the design space and preparing the final design
        * Escalating tricky decisions by defining the 'menu' of choices and alternatives for the team to consider (with recommendations, where appropriate)
        * Documenting the design space and alternatives that were not chosen (and why)
        * Interacting with people on Zulip or elsewhere who are offering feedback and ideas, incorporating those ideas where appropriate into the final design
        * Developing and writing the code for the feature
        * Documenting the feature in the Rust reference or other sites as appropriate
    * **Pre-requisites:**
        * EDIT: Sufficient experience to perform or mentor the above tasks independently
        * EDIT: Demonstrated good judgement
    * **Estimated time commitment:** 
        * Varies, but anywhere from 8-40 hours per week is reasonable.
* **Team liaison:** A lang team member who is coordinating the initiative from the lang team side (they cannot be the same person as the initiative owner).
    * **Tasks:** (note that in practice these may be delegated or done in concert with a group)
        * Mentoring the initiative owner and helping them to decide when facing difficult design decisions.
        * Preparing a summary for the monthly planning meeting documenting the decisions that were made and why.
        * Escalating important decisions to the team where appropriate for broader feedback.
        * Generally serving as a kind of "outside voice" where necessary.
    * **Pre-requisites:**
        * Lang team member
        * Interest in the initiative and sufficient context to help guide (don't have to be an *expert*, but should be able to recognize what you don't know)
    * **Estimated time commitment:**
        * 15-30 minutes per week (regular sync meeting) and the occasional deep dive. If the liaison is spending much more time than this, then they may in fact be playing the role of owner, and that is a problem.

## Details

### Stage 1: Proposal

#### Summary

The proposal stage is where specific ideas start. A *proposal* is, like today, an issue on the lang-team repository. It should include:

* Motivation and general idea
* Any relevant background links
* Enough detail to understand the problem and some sense of how it might be solved

Proposals are meant to be raised early. The idea should have undergone some amount of iteration, perhaps on internals or elsewhere, but it doesn't have to be -- and ideally is not -- a fully formed, concrete proposal. It can be a sketch of "here is a problem and here are a few general ideas for how to address it".

Once a proposal is opened, it can be discussed asynchronously by lang team members. If some member likes the idea, they can **second** it, which means they agree to serve as the lang team liaison. (EDIT: They need to check with leads first, see below)

* Proposal does not have to include:
    * Specific details or a known plan, uncertainty is expected
* Lang team members will review, consider, and discuss

While a proposal is open, it can undergo refinements simply by editing the issue text. Discussions typically take place on Zulip, but it is expected that regular summaries will be posted.

#### Timeframe

Proposals represent pending decisions and are not meant to stay open. The decision about whether to accept a proposal or not is typically made within 1-2 weeks and is meant to decided within one month. In some cases, we may close a proposal but continue discussing and decide to open a fresh proposal shortly thereafter.

#### Exit: Seconding a proposal

Any team member can **second** a proposal: this means that they volunteer to serve as initiative liaison. Before seconding a proposal:

* The proposed owner should ensure that all relevant conversation from the Zulip threads, internals forum, and other forums is summarized in the issue.
    * If they are not willing/able to do this, they are likely not a good choice to act as owner.
* The liaison should check with the team lead(s). 
    * This gives the leads a chance to discuss whether the initiative seems like a good fit and whether the proposed initiative owner/liaison is a good choice (and, if not, why not).
    * Team lead(s) should check with the moderation team to see if the proposed owner (or prominent likely members) has any prior history that may not be known to them. The leads do this to keep this info on a narrow basis.
    * It's better to have those kind of sensitive discussions before things have been said publicly.

After this is done, members may second the proposal by writing `@rustbot second` in the issue thread along with a short comment identifying the owner. Seconding a proposal will cause it enter an FCP period. Team members may opt to suspend the FCP period by issuing `@rustbot pause`, this will cause the FCP to suspend. This is typically done to raise a concern which can then be discussed. The FCP can be resumed by any member saying `@rustbot resume` (to re-enter FCP) or `@rustbot cancel` (to cancel FCP).

Types of objections that make sense at this period:

* Technical concerns:
    * These are typically managed by adding the scenario or concern to a list of constraints to be taken under consideration and addressed.
    * It is absolutely not necessary to have answers to all the technical problems before seconding a proposal!
* Overload concerns:
    * A single owner or liaison should not be involved in too many things.
* Prioritization concerns:
    * This idea might seem premature or like a poor choice of resources.
* People concerns:
    * Concerns about the people involved are best raised by taking directly to the leads first.

#### Exit: Proposal is not accepted

In some cases, proposals will not be accepted. This could happen because there is nobody who wants to second it at this time. It could also happen because the lang team leads decide that the proposal doesn't it the current priorities of the team, or because the lang team leads feel that the task is not a good fit for the proposed owner or liaison (for example, the task may require specialized skills or more time than they have available). In these cases, the proposal will be closed

Sometimes proposals will be moved 

### Stage 2: Experimental

Once a proposal is seconded, it enters the "experimental" state. The goal at this stage is to iterate on exploring and documenting the design space and preparing an RFC, or even multiple RFCs. These initiatives have their own Zulip stream and can land code in the compiler.

Being in the experimental stage does not represent a commitment to land the code. There may well be team members with very live concerns about the feature and it may well get removed if those concerns cannot be resolved.

Initiatives in the experimental stage can have the following resources:

* Their own dedicated Zulip stream (`#project-xxx`)
* A tracking issue
* A repository if desired
* They can land code in the compiler under an "experimental" feature gate (i.e., one that warns when you use it)
    * Ideally we would warn that this represents an early stage, "experimental" proposal

#### During this stage: updates to the team

During this stage, the owner and the liaison should meet on a regular basis. The owner should update the liaison about major design directions and seek their guidance on complex issues (particularly if the owner is not a member of the team). The liaison is responsible for documenting these updates and preparing a monthly update to the team as a whole. They are also responsible for deciding when an issue should be escalated to a lang team design meeting. Sometimes it makes sense to have a design meeting even if there isn't a decision to be made, just to update the team about the overall progress.

#### Exit: RFC approval

To exit the Experimental stage, you need to prepare an RFC on the rust-lang/rfcs repository. This RFC needs to be approved by the lang team. The RFC can include "unresolved questions" to be resolved during the "development" phase.

### Stage 3: Development

After an RFC is approved, the initiative enters "development" stage. The only difference from the "experimental" stage is that the feature gate can now be marked as "non-experimental" and hence used without any sort of warning.

During this stage, the focus is on implementing the complete feature and on resolving any unresolved features.

#### Exit: Group declares feature "feature complete".

At some point, the group can declare a feature to be "feature complete". This requires the following materials to be available:

* Accessible documentation in the "unstable Rust" user's guide, if appropriate.
* All unresolved questions from the RFC have documented answers.
* Tests are written covering all major points in the RFC.
    * Ideally, those tests should be documented, as well, though we don't have a real convention here.

### Stage 4: Feature complete

"Feature complete" initiatives are awaiting community experimentation and stabilization. Typically this is done by writing a blog post (on Inside Rust, perhaps) encouraging experimentation and feedback. That feedback should be gathered and summarize in the monthly reports. It is particularly useful to have lists of users.

The owner should continue to meet regularly with the liaison at this time, though the meeting frequency is often just once a month and quite short.

#### Exit: Stabilization report prepared and approved

To exit the stage, the owner prepares a stabilization report. Stabilization reports 

### Stage 5: Stabilized

This is the final stage. The development is done and support transitions to the regular team members. However, the stream often sticks around as a convenient place to ping the people who were involved in implementing the feature for follow-up questions and for fixing bugs or other maintenance. The expectation is that if you helped to develop a feature, you will stay involved for some period of time after it hits stable to help in resolving problems that arise.

## Frequently asked questions

### What are the "on ramps" to lang team membership under this proposal?

This document doesn't talk explicitly about membership requirements. However, the expectation is that serving as a **initiative owner** for one or more initiative is a stepping stone towards becoming a lang team member. A more complete proposal on those matters will be forthcoming.

### Does the initiative owner make decisions?

The initiative owner drafts the proposed design and takes feedback from the liaison and team about what direction to take. This feedback can take the form of "you need to do X", but typically it is more about "you need to address this scenario". Or, put another way, if the initiative owner doesn't like the proposed direction, it's up to them to find an alternative that they do like which meets those same constraints, or to argue why the constraints are not necessary.

Note that serving as initiative owner is a high level of responsibility and may not be a good "starting place" for involvement within the Rust project. In practice, initiative owners should be experienced enough that they could mentor others to do the implementation work. If you don't know the language or system well enough to do that, then you probably are not ready to be an owner -- but you may be ready to be mentored by the owner!

### Is the word of a lang team member law?

Of course not. Well, ok, sometimes. For the most part, initiative owners are encouraged to treat lang team members like any other member of the community -- this implies a lot of respect for their opinions, since they are experienced, knowledgeable people, but the initiative owner still ultimately owns the design and should use their own judgement about what things to recommend. However, lang team members do have the option of adding constraints that must be met, and they can override the initiative owner if necessary. That is typically done by raising the concern with the rest of the team/leads in a more formal way.

### What happens if an owner stops working on things?

Initiative owners are often volunteers and may have changes in priorities or find they don't have as much time as they thought they did. In that case, they can simply step back. The liaison can then either find a new initiative owner, or perhaps assume initiative owner duties themselves but find a new liaison. If they are not able to do that, the initiative will be closed as "suspended".

### What if we decide a initiative is a bad idea?

Sometimes, in the course of trying to design a initiative, we decide it was the wrong direction. That's ok! At any point the liaison can decide that the initiative isn't working out and close it. However, in doing so, they should document WHY they feel it did not work out -- and identify potential conditions where the idea may make sense later on.

If there are concerns about this, those concerns can be raised with the lang team leads.

Closed initiatives will be removed from the project board and the code for them will be removed from the compiler.

### How should we handle current projects (now called initiatives) and the owners and liaisons of those projects?

We should review existing projects. Some projects have completed, and can be moved to an equivalent of stage 4, with the owner and team members remaining known for future consultation. Some projects have become defunct, and should be closed. For the remaining projects, we should evaluate whether we have a clear owner and liaison that fit the roles (and expected time committments). If we don't, we should close the project. If we do, we should determine what stage the initiative is in.

Guesses for existing projects:

| Project | Owner | Liaison | Stage | Notes |
| --- | --- | --- | --- | --- |
| Half open ranges | workingjubilee | | Feature complete | |
| Inline assembly | Amanieu | joshtriplett | Feature complete | |
| Instruction Set | Lokathor |  | Development | Seems vaguely stalled |
| Macro Metavariable Expr | | | Development | |
| RFC 2229 | nikomatsakis | | Feature complete | Group is doing the work, but nikomatsakis is really leading |
| Never type stabilization | mark-simulacrum | nikomatsakis | Experimental | |
| ffi-unwind | BatmanAod | nikomatsakis | Development | Stalled around impl issues |
| let-else | Fishrock123 | joshtriplett | Experimentation | |
| const-evaluation |  |  | Experimentation | |
| async-foundations |  |  | Experimentation | |
| const-generics |  |  | Experimentation | |
| project safe transmute | jwrenn | joshtriplett | Experimentation | |
| deref patterns |  |  | Experimentation | |

---

# Discussion notes

## Talking about groups

* We don't want to make a requirement that you have a group
* But it is commonly very useful, and it's great to be able to recognize people who contribute
* We should include "entry in the Rust team repo" in the list of things a project group gets
* Josh: thinking of on ramps, having a group makes it useful to see the skills needed for lang team membership
    * it shouldn't necessarily be a hard pre-requisite, but it is a good thing for us to be able to evaluate
    * it would let us see how much you can do coordination, reporting, division of labor
    * might make sense to do a small project, start with a large project, etc
    * someone who has experience elsewhere in the community might want to go to directly

## What kind of "implementation" work is expected of owners

* They don't need to be doing the hacking but should be able to delegate it
* Drive it

## Q

* Mark: Is it intentionally a partial solution to lang team processes? For example, triage is not readily a project but seems necessary.
    * also related to on-ramps to lang membership, if those are limited to initiative owners, that is a high bar to clear. Maybe warranted, maybe not. Lang team duties seem to not completely overlap with solid expertise/desire to drive a particular idea forward - room for other paths seems desirable.

* Niko: Yes, this is intentionally not meant to cover triage
    * some of the work we do -- e.g., triage etc -- doesn't fit into this
    * regarding on-ramps, I agree, but we need to have good ways to demonstrate you are doing the job before taking the position

## Q


* Josh: How do we handle large initiatives where different pieces are in different stages? For instance, inline assembly is *mostly* feature-complete and we want to encourage people to try it on nightly, but it has a few things that need fixing, and there may also be further improvements planned. const-eval is even more of an umbrella initiative. How do we track and manage things like that?

* Niko: FFI is another related example
* Niko: have been debating about it.
* Niko: In case of ffi-unwind, we extended the charter.
* Niko: but a better idea might have been to spin off a separate initiative, e.g. ffi:longjmp
* Niko: Same for inline assembly. Spinning off an initiative is meant to be lightweight. It could even have the same owners.
* Niko: Two scenarios: idea, but not quite done -- ideas for extensions, etc. The other: work ends. And then there's new problems (or maybe the original was a true MVP, and now we want something non-minimal). See e.g. async.
* Josh: Seems good to keep the structure of the group, because that keeps track of the brains that we can reference in the future, who can provide advice.
* Josh: In some cases forming a new group with the same members may make sense. But overall, it seems good to keep the structure by default.
* Niko: I'd like it if we broke off separate sub-groups or sub-projects to track new sub-initiatives. Don't need whole new zulip streams in those cases. Example includes FFI having multiple sub-initiatives.

## Q

* Mark: Suggestion for a post-stabilization or intermediate stages of "retro" on how things went. Maybe outside process, but seems valuable.

* Niko: Good idea!

* Niko: Don't know what shape it should take, but even just an ad-hoc meeting with the owner could be useful.

* Mark: I was thinking even just an inside-rust blog post on each transition from one stage to the next.

* Niko: I think that makes a lot of sense. 
* NIko: We may evolve a template, so that when you transition between stages, you have template of what info to include ready to go.

## Q

* Mark: How do we expect initatives which are not intended for stabilization (e.g., dropck eyepatch, etc.) in their current form to move through this? Is there a need for a "idle" state you can drop into?

* Niko: We should probably have a state analogous to stabilization ... 
* Niko: consider case of "an owner disappears". An initiative can end via: stabilization, or we decide its a bad idea, or the energy fizzles and people disappear. But maybe the idea is still good. Specialization could be considered in the latter state.
* Niko: So: If we reach a point where we are happy with status quo, but we would not want to stabilize it.
* Niko: So then you ask: Is it "we will *never* stabilize this", or is it something where we would be willing.

* Mark: It might make sense to allow groups to wind down, rather than pause. Where liason closes down issues and posts updates. You could always spin it up again if energy comes back.
* Niko: In governance WG, we talked about this. Came to conclusion that you should end in "inactive state", with various reasons (stabilization or abandoned both fit)
* Niko: I like the idea of at least posting a status saying "this issue was part of this group, but the group is currently inactive"

## Q

* Scott: Is there a piece here for things that are more compiler projects than lang projects?
* Niko: sometimes?
* Niko: sometimes things are not user visible but *enable* things that are user visible. E.g. that implementation detail recently discussed that enabled Default for all zero-length arrays
* Scott: So when user-visible things are exposed, but the underlying guts are not, and activity has ceased, is that stabilized, or inactive?
* Niko: We should rename the state "stabilized" to "inactive" -- that covers both cases, and just tell people to write up the description of the state of things at the time that the initiative went inactive. That can then include all the detail about which parts are exposed and which parts are hidden implementation details.

* Josh: If new energy comes in, we want to encourage it. In particular, we don't want to fall into the trap of taking someone providing small energy and pointing them at a massive all-encompassing effort, when the right answer is: "Lets try to carve out a bite-size piece appropriate for what you want to accomplish"

* Niko: Also important to note: Not every feature-gate is going to be on our project board. That's why we need to categorize stuff into active vs inactive.

## Q

* Mark: 8 hours a week *minimum* implied is a really high bar I think, and doesn't seem necessary for many of our projects. Maybe ought to be lowered? Removed entirely?
  - Josh: :+1:. I think there's a certain minimum expectation for a non-idle initiative, but lower seems reasonable for some initiatives.
  - Mark: Would suggest instead requiring e.g. a monthly meeting with the liason, for however long is appropriate. This sets the bar roughly for regular updates and implies progress, but is lightweight.
  - "for however long is appropriate" seems like it would make the liaison take on more of the issue; part of the goal is to get the owner to take more ownership, and if they can't take that ownership then the initiative itself might not progress.
       * note that "15-30 minutes per week (regular sync meeting)" is expected of the liaision in this proposal. I think that's rather high. I basically would suggest 1 hour/month as minimum.

* Niko: Yes. The 8 hours was a bit of guess
* Niko: The more crucial thing is the upper-bound on the work done by the liason. I found that I had counted myself as a liason when in fact I was acting like more of an owner, and I couldn't scale myself appropriately.

* Felix: mixed feelings. 8 hrs/week may be appropriate to set expectations appropriately. Being a lead *is* work.
* Niko: Yes. But there are some things that are progressing, just slowly. And there are others that are simply blocked due to lack of time.

* Mark: Maybe instead of setting a time bound, we can give the liason the ability to observe that the liason isn't seeing enough progress to warrant continued effort in the space, and suggest winding down the group.
* Niko: I like that a lot. Might just be the inactive state, e.g. "paused for the summer"
* Josh: Depends on size of initiative too. Small initiatives might only requre 1-1.5 hrs/week. Making steady progress, and it will get done when it gets done. While larger initiatives (and especially those requiring coordination of multiple people) simply need more investment to make forward progress, and we need to guide the initiative about what to do in that case.
* Niko: Seems like tying it to progress is good: If we observe many months with no progress, then its time to have a conversation.

## Q


* Josh: How do we want to handle capacity planning? Do we want a rough guideline for how many initiatives one liaison should simultaneously manage (even if they can otherwise manage the time committment), to limit total team bandwidth usage? 

* Niko: Seems like conversation from prior question sort of addressed this.
* Josh: I'm not talking about initiatives not making steady progress. I'm talking about language team not having capacity for the amount of work going on. Even if a team member has time for their own meetings, the lang team as a whole might not have time for the combined set of initiatives.
* Mark: One natural feedback mechanism is observing whether the planning meeting is consistently running out of time, then that is a sign that we are over capacity, while if we have 30mins left consistently, then we are under capacity.
* Niko: One of the goals here is to enable liasons to make independent choices/feedback without having to engage the whole team, to enable better scaling.
* Niko: Its tough to come up with an upfront number here. Maybe 1 major and 2 minor? ("major" and "minor" initiatives)
* Mark: another area of capacity planning is the amount of time each liason has for the 1:1 updates with the initiative leads.
* Josh: The liason capacity seems like one form of capacity planning, and it may be necessary to additionally add whole-team capacity planning.
* Niko: The leads probably take ownership on planning the team capacity.

* Mark: its worth noting that the list of action items is mostly niko, and that is itself a sign of a problem.
* Josh: As if Niko is acting as liason for all of the lang initiatives.
* Niko: Yes.

* Mark: Might be good to engage with Aidan, Manish, or Jane. They are working on charter stuff and some of the structure here may align well with what they are exploring.
* Niko: Okay.

## Conclusion

* Niko: Seemed like no one had serious problems with the document
* Scott: Overall I was reading it and said "oh yes, we should do this, yes."
* Josh: Would be happy to second this (after revisions based on this meeting).