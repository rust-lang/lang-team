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

To start an FCP, a lang member or advisor issues `@rfcbot fcp merge lang` (or `fcp close lang` for closing). This creates checkboxes. Once enough boxes have been checked (per rfcbot's standard rules), the decision enters final-comment-period. Assuming no concerns are raised, the decision is finalized once the FCP has expired.

Before finalizing the decision, the team should engage with any **new** points raised during the FCP period, particularly from people not able to raise formal concerns.

### Expected contents for an FCP

FCP proposals should include:

* Motivation
    * Why is this change being made?
    * If this is reverting a prior FCP, what aspects of the original rationale no longer apply?
* Details
    * What is the old/current behavior and what will be the new behavior?
* Design considerations
    * What alternatives did you consider? Why did you choose this route over those?
    * What are common misconceptions?

FCP proposals do not have to be long, but they should capture the key rationale backing the decision. It is often helpful to link to full details.

## Blocking concerns

Lang team members and advisors can raise a **blocking concern** during the discussion process using `@rfcbot concern concern-name`. The decision cannot be finalized until the concern is resolved.

### Before raising a concern

Before raising a blocking concern, we recommend that you nominate the issue for discussion in the triage meeting. Many blocking concerns can be resolved quickly.

### Expectations when raising a concern

The person who raises a concern is expected to:

* Write a constructive comment explaining the concern;
* Make themselves available for discussion in a reasonable fashion;
* Be prepared to write a summary document if requested.

## Resolving concerns

There are two ways to resolve a concern:

### 1. Withdrawn by the person who raised it

If the person who raised the concern is satisfied—whether because of changes made or because they've decided not to block progress—they resolve the concern themselves with `@rfcbot resolve`. This is the preferred outcome.

### 2. Resolution proposed

If the concern is not withdrawn, someone (e.g. the document author) can propose a **resolution**. This is a more formal process that ensures concerns are genuinely engaged with.

#### Creating a concern issue

For concerns that require extended discussion (judgment calls rather than simple technical corrections), create a GitHub issue to track the concern. In most cases --- when the concern involves a change to the language --- the issue will be filed in `rust-lang/rust`. Otherwise, if there is no other suitable repository, the issue can be filed on `rust-lang/lang-team`.

*Note:* We recommend that rfcbot be updated to create these issues automatically. Until then, create them manually.

#### The resolution template

A resolution must include:

1. **Summary of the concern**: Demonstrate understanding of the concern by summarizing its key points. The person who raised the concern should be able to confirm "yes, you understood me."

2. **Proposed changes**: What changes (if any) are being made to the original proposal, and how they address specific points of the concern. This may be empty if no changes are being made.

3. **Rationale**: Explain the reasoning behind the resolution. If any aspects of the concern are *not* being addressed, explain why (typically because they conflict with other goals or constraints). Referencing the [design axioms](./decision-making.md#design-axioms) and their ordering can be a useful way to document this rationale.

#### FCP on the resolution

Post the resolution to the concern issue and start an FCP with `@rfcbot fcp merge lang`.

#### Blocking the resolution of a concern

Blocking a concern resolution is held to a higher standard than blocking the original decision:

* **Only full lang team members** (not lang team advisors) may raise a blocking concern on a concern resolution.
* When a lang team member blocks the resolution of a concern, that **concern must be seconded** by another lang team member.
    * If a second cannot be found within a reasonable time, then the concern must be withdrawn. The typical procedure is to discuss the concern in the next triage meeting and to withdraw the concern if nobody seconds it at that time, but leads may opt to give more or less time depending on circumstances.
    * Before the concern is withdrawn, the team member who raised it may request a design meeting. They are expected to author a document explaining why they feel the resolution of the concern is incorrect. At the end of this meeting, there will be a call for seconds from the team; if nobody agrees to second the concern, then it must be withdrawn.
    * If the second is withdrawn, then another second must be found or else the concern must be withdrawn.

The rules are set up to ensure that a single lang team member cannot block the remainder of the team if they are aligned on how the concern ought to be resolved.

#### Reasons to second a concern

There are two reasons to raise or second a concern raised on a resolution:

* You are not convinced that the resolution is the correct path:
    * You may feel that the resolution does not adequately address the concern.
    * You may feel that the resolution goes too far in addressing the concern and creates new issues of its own.
    * You may not be sure what is right and prefer to take a more conservative path (e.g., erroring or stabilizing a more narrow set of behavior).
* You feel the original concern has not been sufficiently discussed:
    * Sometimes you may not yet be convinced that the original concern is an issue, but you may feel that the champion has moved to resolve the concern too quickly, and that more discussion is warranted.

The team should be careful when opting not to second a concern raised on a resolution. If there is a "forward-compatible" option that preserves the possibility to defer the choice, that is usually better (unless that forward-compatible limitation fully prevents a primary use case).

#### After the resolution

If the resolution FCP completes successfully:

* If the resolution included changes to the original proposal, **restart the original FCP** to give people a chance to review the updated proposal.
* If no changes were made, the original FCP can continue.

# History

* In April of 2026 the process was revised in [PR #360](https://github.com/rust-lang/lang-team/pull/360). This included a significant discussion about how to manage blocking concerns with a dissent. See the [documented rationale for that PR](./rationale/pr-360.md) for more details.
* A later PR revised the process to a new rfcbot merge procedure that was never adopted in practice. This included the requirement that blocking concerns be "seconded".
* The original consensus procedure was defined in [RFC #1068](https://github.com/rust-lang/rfcs/blob/master/text/1068-rust-governance.md), which specifies consensus with the team lead empowered to resolve cases where the team cannot come to consensus.
