# Consensus decisions

Consensus decisions represent the official opinion of the team. They require that a majority of the team is in favor and that nobody has raised a concern that achieved [strong enough support](#threshold-to-resolve-a-concern-without-consent) to block the decision.

They are used for for decisions that cannot be reversed (e.g., stabilization decisions or decisions that affect language semantics). They are also used for RFC approvals because, while the designs described there can be reversed before stabilization, we do not wish to do so lightly.

## Definitions

The decision making process distinguishes the following roles

* **Lang team**: the [Language Design Team](https://github.com/rust-lang/team/blob/master/teams/lang.toml).
* **Lang team member**: a member of the Lang team. Lang team members are the ones who have to [approve the final decision](#the-process).
* **Advisor**: a member of the [lang team advisors](https://github.com/rust-lang/team/blob/master/teams/lang-advisors.toml). Advisors can propose a decision and can raise [formal concerns](#raising-concerns) but their approval is not required.
* **Decision document**: the RFC, issue text, or other document describing the decision being made.
* **Document author**: The person who authored the decision document. They ultimately decide what changes they wish to make to the text in response to concerns. They often do this in close consultation with a member of the lang team, but that is not required.

## The process

Consensus decisions are done using rfcbot. They always begin with a decision document authored by the [document author](#definitions). The document author can be anyone, they do not have to be a member of a Rust team.

To make a decision, a [lang team member or advisor](#definitions) should issue `@rfcbot fcp merge`. This will create checkboxes. Once [2/3 of the boxes have been checked](#doing-the-math), the decision will enter final-comment-period. During that period the team should review any comments raised — assuming no concerns are raised (see below), the decision is finalized once the FCP has expired.

*Note:* Before finalizing the decision, the team should engage with **new** points raised on the thread (particularly if they are made by people not able to raise formal concerns).

## Raising concerns

[Lang team members or advisors](#definitions) are entitled to raise a **formal concern** during the discussion process. This can be done with `@rfcbot concern concern-name`. The decision cannot be finalized until the concern is resolved, either because the person who raised the concern is satisfied and [resolves](#resolving-a-concern) it themselves, or because the team decides to [challenge it](#challenging-a-concern).

### Expectations on raising a concern

The [lang team member or advisor](#definitions) who raises a concern is expected to

* write a constructive comment explaining the concern;
* make themselves available for discussion in a reasonable fashion;
* be prepared to write a document summarizing their concern for the lang-team if requested.

### When a concern is raised

When a concern is raised, the team will review the concern in the triage meeting. Based on that discussion, the [document author](#definitions) can decide whether they wish to make changes to mitigate or resolve the concern.

## Resolving a concern

If the person who raised the concern is satisfied, either because of changes made or because they don't wish to continue blocking progress, they should resolve the concern themselves with `@rfcbot resolve`. This is the desired outcome.

If the concern has still not been resolved, there will typically be a period of deeper discussion. The goal is to [find common ground](./making_decisions.md#design-axioms), either by mitigating it directly or by side-stepping it via a subset of the design that still achieves the major goals.

### Challenging a concern

If deeper discussion does not reach a solution, a [lang team member](#definitions) can challenge a concern. This indicates that they feel they understand the concern and do not agree it is a problem.

Challenging a concern is done by scheduling a meeting in which the lang team reads and discusses a summary of the concern and the subsequent discussion. At the end of this meeting, the concern will either be *sustained* (meaning that progress is still blocked) or *resolved* (meaning that final-comment-period can continue, assuming there aren't other concerns to be resolved).

Sustaining a concern requires [1/4 of the lang team](#doing-the-math) to agree to sustain; this number includes the person who raised the concern, if they are a member of the lang team and not an advisor. Members can agree to sustain either because they with the concern *or* because they feel more time is needed for discussion.

*Note:* The document to be read during the meeting will ideally be prepared by both the person raising the concern and the people challenging it so that it fairly represents both points of view. The document should include a

1. summary of the concern and discussion;
2. coverage of possible mitigations, changes, or subsets that were considered and their pros/cons


## Doing the math


Given a team of N>=4 members, the actual threshold `R` for approving a decision is `R = floor(N * 2 / 3)`. The threshold `S` for sustaining a concern is `floor(N / 3)`. So e.g...

| N   | R   | S   |
| --- | --- | --- |
| 20  | 13  | 6   |
| ... | ... | ... |
| 12  | 8   | 4   |
| ... | ... | ... |
| 9   | 6   | 3   |
| 8   | 5   | 2   |
| 7   | 4   | 2   |
| 6   | 4   | 2   |
| 5   | 3   | 2   |

For 4 or less:

| N   | R   | S   |
| --- | --- | --- |
| 4   | 3   | 1   |
| 3   | 3   | 1   |
| 2   | 2   | 1   |
| 1   | 1   | 1   |
