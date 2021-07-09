# Process

This page describes how lang team initiatives work. This is the process to use if you have an idea for a change you would like to make in the language.

## Summary

In a nutshell, the process for a successful initiative is as follows:

- Have an idea
  - Talk about it on internals, Zulip, etc to flesh it out a bit
  - Ideally, identify a potential [owner]
- Open a [proposal] as an issue on the lang-team repository
  - A lang team member can decide to be your [liaison] and [_second_ your proposal][2nd].
  - Once that happens, we will create a Zulip stream, tracking issue, and (optionally) repository, etc.
- If warranted, [explore][experimental] the design space and author the RFC
  - In this phase, the [owner] works with the [liaison] and other contributors to expore the design space and develop the RFC
  - Code can be landed in this phase, but the feature gate is marked as "experimental"
  - Users of the feature gate will get a warning that the RFC is under development
  - Once the RFC is ready, it can be opened on the RFC repository and approved by the lang team
- Finish [development]
  - At this point, development proceeds but the feature gate does not have to be marked as "experimental"
- [Feature complete]
  - When the liaison feels that it is ready, the initiative may be declared "feature complete".
  - This is primarily a 'signaling' mechanism to encourage testing and feedback.
  - This is a good phase in which to write the Rust reference chapter and other supporting documentation.
  - Presuming feedback is positive, a stabilization report is prepared and (hopefully) approved.
- [Stabilized]
  - Done! The Zulip stream and group can stick around as a place to report bugs or for further discussion, but the initiative is complete.
  - The final step is to conduct a retrospective discussion between the [owner] and [liaison] about how the process went.

[proposal]: ./process/stages/proposal.md
[owner]: ./process/roles/owner.md
[liaison]: ./process/roles/liaison.md
[proposal]: ./process/stages/proposal.md
[2nd]: ./process/stages/proposal.html#exit-seconding-a-proposal
[experimental]: ./process/stages/experimental.md
[development]: ./process/stages/development.md
[feature complete]: ./process/stages/feature_complete.md
[stabilized]: ./process/stages/stabilized.md

## Goals

- **Empower individuals and give ownership:**
  - Each initiative in this proposal is ultimately owned by a single person who drafts the proposals and recommendations.
  - The role of the lang team is to review the designs, provide feedback, and ultimately decide whether to accept the design.
  - The team can introduce constraints and requests that the owner should either satisfy or explain why they are not able to do so.
- **Clarify the role of each individual:**
  - As described in the [roles] page, each individual and group involved in an initiative has a clear, defined role in the decision making process.
- **Minimize friction for "reversible" decisions and enable experimentation:**
  - We avoid requiring "full checkoff" from team members for things that can be readily reversed.
  - We want to make it relatively easy to start hacking and experimenting with an idea. Under this proposal, all it takes is to find an owner, a liaison, and to have the team leads approve.
  - Other team members are encouraged to log concerns and constraints that ought to be addressed in the design, rather than blocking experimentation.
- **Ensure that decisions are truly reversible:**
  - On the flip side, although we wish to make it easy for ideas to move forward, we recognize that this can create a lot of momentum that allows ideas to force their way through the process.
  - This is why code in the [experimental] phase issues a warning, for example.

[roles]: ./process/roles.md
