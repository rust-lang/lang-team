# Stage 1: Proposal

## Summary

The proposal stage is where specific ideas start. To make a new proposal, [open an "Initiative Proposal" issue on the lang-team repository][open-proposal]. It should include:

- Motivation and general idea
- Any relevant background links
- Enough detail to understand the problem and some sense of how it might be solved
- Someone interested in being the [owner] for the initiative, if any.

[open-proposal]: https://github.com/rust-lang/lang-team/issues/new/choose

Proposals are meant to be raised early. The idea should have undergone some amount of iteration, perhaps on internals or elsewhere, but it doesn't have to be -- and ideally is not -- a fully formed, concrete proposal. It can be a sketch of "here is a problem and here are a few general ideas for how to address it".

Once a proposal is opened, it can be discussed asynchronously by lang team members. If some member likes the idea, they first check with the team leads. If the leads agree, the member can **second** it, which means the member agrees to serve as the lang team [liaison].

[owner]: ../roles/owner.md
[liaison]: ../roles/liaison.md

- Proposal does not have to include:
  - Specific details or a known plan, uncertainty is expected
- Lang team members will review, consider, and discuss

While a proposal is open, it can undergo refinements simply by editing the issue text. Discussions typically take place on Zulip, but it is expected that regular summaries will be posted.

## Timeframe

Proposals represent pending decisions and are not meant to stay open. The decision about whether to accept a proposal or not is typically made within 1-2 weeks and is meant to decided within one month. In some cases, we may close a proposal but continue discussing and decide to open a fresh proposal shortly thereafter.

## Exit: Seconding a proposal

Any team member can **second** a proposal: this means that they volunteer to serve as initiative [liaison]. Before seconding a proposal:

- The proposed owner should ensure that all relevant conversation from the Zulip threads, internals forum, and other forums is summarized in the issue.
  - If they are not willing/able to do this, they are likely not a good choice to act as owner.
- The liaison should check with the team lead(s).
  - This gives the leads a chance to discuss whether the initiative seems like a good fit and whether the proposed initiative owner/liaison is a good choice (and, if not, why not).
  - Team lead(s) should check with the moderation team to see if the proposed owner (or prominent likely members) has any prior history that may not be known to them. The leads do this to keep this info on a narrow basis.
  - It's better to have those kind of sensitive discussions before things have been said publicly.

After this is done, members may second the proposal by writing `@rustbot second` in the issue thread along with a short comment identifying the owner. Seconding a proposal will cause it enter an FCP period. Team members may opt to suspend the FCP period by issuing `@rustbot pause`, this will cause the FCP to suspend. This is typically done to raise a concern which can then be discussed. The FCP can be resumed by any member saying `@rustbot resume` (to re-enter FCP) or `@rustbot cancel` (to cancel FCP).

Once a proposal is seconded, the next step depends on its complexity:

- Most initiatives proceed to [Stage 2 (Experimental)](./experimental.md), which is focused on authoring an RFC.
- However, simple initiatives can skip directly to [Stage 3 (Development)](./development.md).
  - This indicates that the design is well understood and there aren't any complex tradeoffs to explore and document.
  - A common example of this is for new lints.

Types of objections that make sense at this period:

- Technical concerns:
  - These are typically managed by adding the scenario or concern to a list of constraints to be taken under consideration and addressed.
  - **It is absolutely not necessary to have answers to all the technical problems before seconding a proposal!**
- Overload concerns:
  - A single owner or liaison should not be involved in too many things.
- Prioritization concerns:
  - This idea might seem premature or like a poor choice of resources.
- People concerns:
  - Concerns about the people involved are best raised by taking directly to the leads first.

## Exit: Proposal is not accepted

In some cases, proposals will not be accepted. This could happen because there is nobody who wants to second it at this time. It could also happen because the lang team leads decide that the proposal doesn't it the current priorities of the team, or because the lang team leads feel that the task is not a good fit for the proposed owner or liaison (for example, the task may require specialized skills or more time than they have available). In these cases, the proposal will be closed.

We typically do not close proposals just because lang team members have technical objections: instead, those objections can be logged and resolved through the experimentation period. However, if it is clear that the proposal has no pathway to being accepted, we would try to take that into account.