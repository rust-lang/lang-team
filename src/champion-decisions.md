# Champion decisions

Champion decisions do not represent team consensus. Rather, they indicate that *somebody* on the team is willing to champion an idea. We use them to begin experiments and for other decisions where we want to move quickly and iterate.

## When to use champion decisions

* Starting a [lang-team experiment](./how_to/experiment.md)
* Closing an RFC or issue that is clearly not going to be accepted
* Other lightweight decisions that don't make a durable commitment

## Process

### For the champion (lang team member or advisor)

1. Write up your proposal on a GitHub issue or PR, explaining the decision and context.
2. [Nominate](./how_to/nominate.md) it for discussion at a lang-team triage meeting.
3. At the triage meeting, the team will discuss and raise any concerns.
4. If no one requests FCP escalation, you can proceed.

### Handling concerns

When people raise concerns:

* **Treasure dissent.** Engage with them and make sure you understand the concern.
* Note concerns on the tracking issue as "unresolved questions" or things to explore—they shouldn't be forgotten.
* Concerns don't *block* you from proceeding, but they may give you pause, particularly if they are shared by multiple team members.

### FCP escalation

Any team member can request FCP escalation at any time—during the triage meeting or later. This converts the decision to an [FCP decision](./fcp-decisions.md), which follows the full FCP process.

Once a champion decision has been escalated and FCP'd, it becomes durable: reversing it would require another FCP.

No justification is required to request escalation. The request itself signals "I think this matters enough to need the full FCP process."

## For other team members

* Your approval is not required for champion decisions.
* If you disagree with the decision or think it's a bad idea, say so (constructively)!
* For experiments, ask yourself: What are the "weak spots" that the experiment ought to probe? What information can we gather?
* If you feel strongly that this should not proceed without team consensus, request FCP escalation.
