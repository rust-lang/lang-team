# Becoming and being a lang-team member

Lang team members are the ones who ultimately decide the syntax/semantics of the Rust language. They judge whether a given piece of Rust syntax ought to compile and, if it does, what should happen when it runs.

## Relations to other teams

Lang team members should be familiar with how the compiler works, but they don't need to drive its implementation or even work on it. Compiler implementation details are the job of the [compiler team]. Note that compiler team members are encouraged to raise concerns if they feel that the desired design cannot be implemented in a reasoable and maintainable way.

[compiler team]: https://rust-lang.github.io/compiler-team/

Lang team members have to know the language inside and out, but they don't necessarily need deep type theory background. Working out the formal semantics of the language as well as diving into very detailed questions is covered by the [types team]. Types team members are encouraged to raise concerns if they feel that the desired design cannot be made sound or is internally inconsistent.

[types team]: https://rust-lang.github.io/types-team/

## Expectations

Lang team members are expected to

* Help to advance the state of the language by some combination of participating in discussions on RFCs and Zulip, authoring and editing RFCs, and shepherding features through the stabilization process.
* Attend triage and design meetings regularly.
* Promptly respond to RFC decisions by reviewing the question and either checking box or raising concerns

## Process to add a new member

Lang team members can propose new additions to the team as follows:

* Lang team member prepares a short write-up to propose candidate to the rest of the team
    * The write-up should draw on the qualifications below, giving examples where the candidate demonstrated the various criteria
* Leads check with moderation team for known flags, surface to team if any.
* The decision to add must have unanimous consent, similar to any other decision. Objections may be raised in a private team discussion, or by contacting a team lead.

**Important:** Discussions about potential new members are kept strictly confidential. Email or voice conversation are often preferred because they doesn't keep publicly available records.

### Qualifications

These are the questions we ask ourselves when deciding whether someone would be a good choice as a lang team member.

* Has this person demonstrated **strong language design skills**?
    * Have they led an impactful initiative to completion?
* Is this person **responsible**?
    * When they agree to take on a task, do they either get it done or identify that they are not able to follow through and ask for help?
* Is this person able to **lead others to a productive conversation**?
    * Are there times when a conversation was stalled out and this person was able to step in and get the design discussion back on track?
        * This could have been by suggesting a compromise, but it may also be by asking the right questions or encouraging the right tone.
* Is this person able to **disagree constructively and empathically**?
    * The expectation is that team members go "above and beyond" the [Rust code of conduct](https://www.rust-lang.org/policies/code-of-conduct), embodying not only the letter but also the spirit.
    * When they are having a debate, do they make an active effort to understand and repeat back others' points of view?
    * If they have a concern, do they engage actively to make sure it is understood and to look for ways to resolve it?
    * Do people come away from disagreements with this person feeling respected?
* Is this person **active**?
    * Are they attending the [triage meeting](./meetings/triage.md) and [design meetings](./meetings/triage.md) regularly? (meetings are open for anyone to attend; but note that merely attending meetings is not enough to become a team member!)
    * Either in meeting or elsewhere, do they comment on disussions and otherwise?
* Does this person have an **overall desire to improve the language**, rather than a strong interest in some particular domain?
    * Everyone have preferences, but members are responsible for balancing a wide array of interests. Someone with very specialized interest may be a better choice for a lang team advisor.

Keep in mind that qualifications are not a checklist and membership decisions are ultimately made on a case-by-case basis. **If you are interested in joining the lang team, we recommend you reach out to the lead(s) to talk about the path forward.**
