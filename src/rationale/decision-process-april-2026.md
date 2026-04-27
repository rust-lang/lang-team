# Summary

This documents the changes made in [PR #360](https://github.com/rust-lang/lang-team/pull/360/):

* The role of the **champion** is better defined:
    * Champions are drawn from lang-team and lang-team advisors
    * Champions drive experiments and have the power to make reversible decisions
    * Champions are expected to keep the lang-team abreast of updates and decisions and to bottom out lang-team concerns
* The formal decision making process is modified:
    * FCP decisions are made by the team
    * Concerns can now be made by **lang-team members only**
    * Concerns can be resolved either by
        * the person who raised the concern may opt to withdraw it
        * a FCP on a *resolution document* that proposes a resolution
            * This resolution document can be sponsored by any team member.
            * The person who raised the original concern cannot raise concerns on the resolution ("no recursive blocking").
* The ideal sizeof the team is defined as 4-8 members.

Changes from the past include:

* Older material concerning an alternative rfcbot procedure is removed, as it is has not been enacted (we may reconsider it at a future date).
* Champions can no longer raise concerns; in practice, the tooling has never supported this, and we expect that concerns from a champion will typically be seconded by a lang team member.
* There is now a clearer path for resolving concerns; in the past, the procedure said that concerns had to be "seconded" but the means of doing so was ill-defined.

# Motivation

The motivation for these changes was to increase individual autonomomy. We wish for the lang team to retain its role of shaping and vetting language features but we wish to ensure that the champions and owners driving a particular feature feel empowered to make forward progress.

# Design axioms

* **Perfection is a process.** We do our best to get designs right but also recognize the limits of deliberation. Many design constraints only become apparent (or relevant) once a feature has been adopted. For complex designs, we prefer to ship incrementally, addressing the most important needs first and using the experience gained to inform future design work.
* **Treasure dissent.** The best designs arise out of finding a creative way to overcome two seemingly incompatible constraints. When someone raises a concern that blocks your progress, you should look on it as an opportunity to improve your design so it can meet more needs.

# Frequently Asked Questions

## Why are Champion Decisions "unblockable"?

In the process as written, Champion Decisions (e.g., to start or stop an experiment) cannot be blocked by members of the lang-team. We wish to lean in favor of autonomy and easy experimentation and avoid the perception that champions need to wait for the lang-team to "weigh in" before making lightweight decisions (e.g., how to focus their time). The intended flow is rather that champions document those decisions for periodic review by the lang-team after the fact. Any concerns held my members of the team can be raised there; champions would do well to heed them, as those concerns will simply arise later during the FCP process.

## Why not permit "recursive blocking"?

The rationale and discussion was captured in [resolution issue #365](https://github.com/rust-lang/lang-team/issues/365). The decision was made to favor "perfection is a process" over "treasure dissent", which meant that we favor the ability to take measured steps forward over the ability for an individual to block progress. This decision was controversial and Josh Triplett has opted to include a dissent

> **The crux of the debate**
>
> Both views acknowledge they carry risk. Both have mitigation strategies. The difference is **who bears the burden** and **what question they're asking**.
>
> Let us assume that there has been significant discussion and there is a proposal to move forward, but the dissenter that raised the original concern is not satisfied:
>
> - Under **View A,** the dissenter must decide if their concern is serious enough to be worth blocking forward progress. The team cannot proceed if the dissenter decides it is.
> - Under **View B,** the rest of the team must ask, "Have we engaged genuinely, and can we course-correct if we are wrong?" They can proceed if they all agree the answer is "yes", even over the dissenter's objection.
>
> Put differently: when, after thorough discussion, a single team member remains unconvinced - which way do we tilt? View A tilts toward the dissenter's judgment. View B tilts toward the team's collective judgment.

Do **not** add recursive blocking. The person who raised a concern cannot block the FCP to resolve that concern.

However, add process scaffolding that encouages genuine engagement with concerns, particularly in cases where the concern-raiser does not sign off on the resolution:

- Template for resolution write-ups, including specific reflection questions (see the FAQ for details)
- Option for concern-raiser to request a design meeting before final decision is complete (where they provide the document) 

### Dissent by Josh Triplett

> Josh Tripplet did not agree with the decision to remove recursive blocking and wished to record a dissent. This spot is left for him to add the dissent in a future PR.
