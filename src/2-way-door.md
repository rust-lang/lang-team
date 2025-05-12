# Reversible decisions

Reversible decisions do not represent team consensus.
Rather, they indicate that *somebody* on the team is in favor of the idea.
We use them to begin experiments and do other small things.
We never use them for decisions that impact stable code, such as stabilizing a feature.

## Examples where this process is appropriate

* Championing a lang-team experiment
* Closing an RFC --
    * Technically reversible (you can always re-open the PR), but particularly when external contributors are involved it's best to discuss this with the team beforehand. It's very confusing for people to have their PR closed and re-opened and can lead to burned bridges.

## Process

### Decision maker (lang team member or lang team advisor)

* Write a comment on the Github issue or PR that
    * Explains decision and context
    * Labels the issue with `T-lang` (`@rustbot labels +T-lang`)
    * Starts a `@rfcbot poll` -- typically the issue should *only* have `T-lang` to avoid excessive pings
* If people raise concerns, particularly Rust team members:
    * Remember to **treasure dissent**. Engage with them and make sure you understand it. Note all the concerns raised on the tracking issue as "unresolved questions" or things to explore so they don't get forgotten.
    * Concerns don't *block* you from going forward, but they may give you pause, particularly if they are shared by many on the team.

### Other team members

* Since this action is reversible, your approval is not required. However, please check your box to indicate you reviewed the issue, it's useful feedback.
* If you disagree with the decision, or think the experiment is a bad idea, say so (constructively)! 
* For experiments, ask youself: What are the "weak spots" that the experiment ought to probe? What information can we gather?
