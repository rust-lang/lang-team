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
