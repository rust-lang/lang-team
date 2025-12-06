# FCP decisions

FCP decisions represent a significant commitment from the team. They require sign-off from team members and follow a formal process for resolving concerns. An FCP decision cannot be reversed without another FCP—the expectation is that reversal requires some change in circumstances: new experience, new information, or a reasoned change of mind.

FCP decisions are used for stabilization, RFC approval, and other cases where we are making a promise to our users or taking a position we don't want to reverse lightly.

## Definitions

* **Lang team**: The [Language Design Team](https://github.com/rust-lang/team/blob/master/teams/lang.toml).
* **Lang team member**: A member of the lang team. Lang team members are the ones who approve the final decision.
* **Advisor**: A member of the [lang team advisors](https://github.com/rust-lang/team/blob/master/teams/lang-advisors.toml). Advisors can propose a decision and can raise concerns, but their approval is not required.
* **Decision document**: The RFC, issue text, or other document describing the decision being made.
* **Document author**: The person who authored the decision document. They ultimately decide what changes they wish to make in response to concerns.

## The process

FCP decisions use rfcbot. They always begin with a decision document authored by the document author, who can be anyone—they do not have to be a member of a Rust team.

To start an FCP, a lang team member or advisor issues `@rfcbot fcp merge` (or `fcp close` for closing). This creates checkboxes. Once enough boxes have been checked (per rfcbot's standard rules), the decision enters final-comment-period. Assuming no concerns are raised, the decision is finalized once the FCP has expired.

Before finalizing the decision, the team should engage with any **new** points raised during the FCP period, particularly from people not able to raise formal concerns.

## Raising concerns

Lang team members or advisors can raise a **formal concern** during the discussion process using `@rfcbot concern concern-name`. The decision cannot be finalized until the concern is resolved.

### Expectations when raising a concern

The person who raises a concern is expected to:

* Write a constructive comment explaining the concern
* Make themselves available for discussion in a reasonable fashion
* Be prepared to write a summary document if requested

## Resolving concerns

There are two ways to resolve a concern:

### 1. Withdrawn by the person who raised it

If the person who raised the concern is satisfied—whether because of changes made or because they've decided not to block progress—they resolve the concern themselves with `@rfcbot resolve`. This is the preferred outcome.

### 2. Resolution proposed

If the concern is not withdrawn, someone (typically the document author) can propose a **resolution**. This is a more formal process that ensures concerns are genuinely engaged with.

#### Creating a concern issue

For concerns that require discussion (judgment calls rather than simple technical corrections), create a GitHub issue to track the concern. The issue can be on rust-lang/lang-team if there is no other suitable repository.

*Note:* We recommend that rfcbot be updated to create these issues automatically. Until then, create them manually.

#### The resolution template

A resolution must include:

1. **Summary of the concern**: Demonstrate that you understand the concern by summarizing its key points. The person who raised the concern should be able to confirm "yes, you understood me."

2. **Proposed changes**: What changes (if any) are being made to the original proposal, and how they address specific points of the concern. This may be empty if no changes are being made.

3. **Rationale**: Explain the reasoning behind the resolution. If any aspects of the concern are *not* being addressed, explain why (typically because they conflict with other goals or constraints). Referencing the [design axioms](./decision-making.md#design-axioms) and their ordering can be a useful way to document this rationale.

#### FCP on the resolution

Post the resolution to the concern issue and start an FCP with `@rfcbot fcp merge`.

**Important:** The person who raised the original concern cannot block this FCP. They can (and should) comment on whether the summary accurately represents their concern, but their only path to blocking is convincing another team member that the resolution is inadequate.

Other team members can raise new concerns on the resolution, or request a design meeting for deeper discussion.

#### After the resolution

If the resolution FCP completes successfully:

* If the resolution included changes to the original proposal, **restart the original FCP** to give people a chance to review the updated proposal.
* If no changes were made, the original FCP can continue.
