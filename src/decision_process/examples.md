# Examples of the decision-making process in action

## Reversible decision: merging a proposal

The process is best described by example. Suppose that there is a pending lang
team proposal, and a lang team member would like to serve as the liaison. They
contact the team leads and receive the go-ahead. They can then write:

> @rustbot merge
>
> I propose to merge this proposal. I think it will be a great addition to
> Rust!

This indicates that they would like to merge the proposal. At the moment, there
is no decision pending, so rustbot would add a comment that looks like the
following:

> Hello! @Alan has proposed to merge this. This is a **reversible decision**,
> which means that it will be affirmed once the "final comment period" of 10
> days have passed, unless a team member places a "hold" on the decision (or
> cancels it).
>
> | Team member | State |
> | --- | --- |
> | @Alan | **merge** |
> | @Barbara | |
> | @Grace | |
> | @Niklaus | |

As the comment says, the PR is now in "pending decision" state, with Alan
having kicked off the process with a proposal to merge. Alan's status of
**merge** will link to his comment.

Now, for this particular proposal, Barbara has a concern. She thinks that the
proposal has overlooked an important consideration. She writes a comment:

> @rustbot hold
>
> Did you consider reversing the polarity? Or the impact on the flux capacitor?

At this point, rustbot updates the state; Barbara's status of
**hold** will link to her comment:

> Hello! @Alan has proposed to merge this. This is a **reversible decision**,
> which means that it will be affirmed once the "final comment period" of 10
> days have passed, unless a team member places a "hold" on the decision (or cancels it).
>
> | Team member | State |
> | --- | --- |
> | @Alan | **merge** |
> | @Barbara | **hold** |
> | @Grace | |
> | @Niklaus | |

Alan is currently busy at work, though, so by by the time that he and Barbara
get a chance to talk, 11 days have passed. (Alan and Barbara receive a ping
from rustbot after a week or so.) Once they get a chance to talk, Alan
fully addresses Barbara's concern, so Barbara posts:

> @rustbot merge

The comment is updated:

> Hello! @Alan has proposed to merge this. This is a **reversible decision**,
> which means that it will be affirmed once the "final comment period" of 10
> days have passed, unless a team member places a "hold" on the decision (or cancels it).
>
> | Team member | State |
> | --- | --- |
> | @Alan | **merge** |
> | @Barbara | ~~hold~~ **merge** |
> | @Grace | |
> | @Niklaus | |

Barbara's previous ~~hold~~ status links to her previous comment setting her
status to `hold`, and her current **merge** status links to her more recent
comment setting her status to `merge`.

At this point, all the statuses are either empty or **merge**, and more than 10
days have passed since the FCP started. Therefore, it completes immediately.

## Authoring an RFC (illustration of `rustbot restart`)

After some time, the proposal is completed and an RFC is proposed. This is a
reversible decision. Alan, as the liaison, proposes to merge the RFC with
`@rustbot merge`, and the decision making process proceeds as above.

This time, Niklaus has a concern:

> @rustbot hold
>
> I have not had time to read this yet! Give me a bit of time to write it up.

After 7 days have passed, rustbot writes to him:

> @Niklaus, I see you have placed a hold but 7 days have passed. Are you any
> closer to reaching a decision? (cc @Alan)

This continues for a week or two while Alan and Niklaus play "email tag". In
the interim, Barbara decides she agrees with the RFC, so she uses `@rustbot
merge` as well. The status now looks like this:

> Hello! @Alan has proposed to merge this PR. This is a **reversible
> decision**, which means that it will be affirmed once the "final comment
> period" of 10 days have passed, unless a team member places a "hold" on the decision (or cancels it).
> decision.
>
> | Team member | State |
> | --- | --- |
> | @Alan | **merge** |
> | @Barbara | **merge** |
> | @Grace | |
> | @Niklaus | **hold** |

Eventually, Alan and Niklaus find a time to discuss, and Alan agrees that
Niklaus's concerns are valid, so he makes some major edits to the RFC. Given
that the RFC is completely different, he decides to restart the clock and
writes:

> @rustbot restart

This strikes through the state of all team members (setting their current
status to blank, while preserving the history) and begins the clock anew.
rustbot also pings the relevant team members:

> Dear @rust-lang/team, @Alan has restarted the clock!

The status now looks like this:

> Hello! @Alan has proposed to merge this. This is a **reversible decision**,
> which means that it will be affirmed once the "final comment period" of 10
> days have passed, unless a team member places a "hold" on the decision (or cancels it).
>
> | Team member | State |
> | --- | --- |
> | @Alan | **merge** |
> | @Barbara | ~~merge~~ |
> | @Grace | |
> | @Niklaus | ~~hold~~ |

Barbara can use `@rustbot merge` to re-affirm her **merge** status, and Niklaus
can use `@rustbot merge` to set his own status to **merge** since he agrees
with the resolution of his concern.

## Authoring an RFC continued (Overriding a concern)

At this point, Grace has a concern, and explains that concern in detail:

> @rustbot hold
>
> I've thought about this a lot, and I don't think we should do this. Now that
> I see the syntax used in practice, I feel like if we do this it'll have an
> adverse effect on the ecosystem...

The status now looks like this:

> Hello! @Alan has proposed to merge this. This is a **reversible decision**,
> which means that it will be affirmed once the "final comment period" of 10
> days have passed, unless a team member places a "hold" on the decision (or cancels it).
>
> | Team member | State |
> | --- | --- |
> | @Alan | **merge** |
> | @Barbara | ~~merge~~ **merge** |
> | @Grace | **hold** |
> | @Niklaus | ~~hold~~ **merge** |

Niklaus reads this message. He feels he understands the concern well, and
agrees that this point hasn't yet been considered:

> @rustbot hold
>
> I agree. I think we should take more time to evaluate alternative syntaxes.
> What about...

Over the course of a few subsequent meetings and side conversations, Grace and
other team members discuss the concern further; the initiative owner also
considers the concern, and raises it with others working on the initiative.

The owner of the initiative updates the RFC to include a discussion of a couple
of alternative syntax proposals. The owner recommends a slightly modified
version of the originally proposed syntax, and outlines criteria that they feel
the syntax should meet in order to support the use case.

Grace agrees that her concern has been understood, but does not agree with the
proposed syntax. Grace feels the new proposal is an improvement, but her
concern remains.

Niklaus feels that the team has understood Grace's concern, and furthermore,
that the updated proposal addresses Grace's concern:

> @rustbot merge
>
> I appreciate the potential impact this may have on the ecosystem. However, I
> feel that as now described in section XYZ of the RFC, the value of A
> outweighs the risk of B, and I think C mitigates the potential risk by...

(Notice that while Niklaus feels that the team has understood Grace's concern,
he does not speak for the entire team or imply that his summary represents the
entire team. Niklaus is just withdrawing his own support for the concern.)

At this point, the entire team other than Grace agrees that the proposal should
move forward:

> Hello! @Alan has proposed to merge this. This is a **reversible decision**,
> which means that it will be affirmed once the "final comment period" of 10
> days have passed, unless a team member places a "hold" on the decision (or cancels it).
>
> | Team member | State |
> | --- | --- |
> | @Alan | **merge** |
> | @Barbara | ~~merge~~ **merge** |
> | @Grace | **hold** |
> | @Niklaus | ~~hold~~ ~~merge~~ ~~hold~~ **merge**|

(We'll assume, for this example, that Grace does not manage to convince anyone
else.)

Grace takes some time, working with the RFC author, to add a dissent, including
a specific unresolved question.

Grace then writes a comment containing `@rustbot dissent`. (If necessary, or if
Grace would prefer, another team member may issue `@rustbot @grace dissent` on
her behalf.) The status now looks like this:

> Hello! @Alan has proposed to merge this. This is a **reversible decision**,
> which means that it will be affirmed once the "final comment period" of 10
> days have passed, unless a team member places a "hold" on the decision (or cancels it).
>
> | Team member | State |
> | --- | --- |
> | @Alan | **merge** |
> | @Barbara | ~~merge~~ **merge** |
> | @Grace | ~~hold~~ **dissent** |
> | @Niklaus | ~~hold~~ ~~merge~~ ~~hold~~ **merge**|

Since all statuses are now either **merge** or **dissent** rustbot also posts a
comment:

> The final comment period has resolved, with a decision to **merge**.
>
> Note that this decision has dissents; please ensure these dissents have been
> recorded for subsequent consideration.

## Stabilizing a feature

The feature has been implemented and is now eligible for stabilization. Alan
writes a stabilization report and posts it, and then issues the command

> @rustbot stabilize

Rustbot recognizes that a "stabilization" decision is irreversible, so the
template is a bit different:

> Hello! @Alan has proposed to stabilize this. This is a **irreversible
> decision**, which means that it will be affirmed once all members come to a
> consensus and the "final comment period" of 10 days has passed.
>
> | Team member | State |
> | --- | --- |
> | @Alan | stabilize |
> | @Barbara |  |
> | @Grace |  |
> | @Niklaus |  |

This time, Barbara, Grace, and Niklaus must all explicitly provide a status
before the decision can proceed. One by one, they join the PR. They must
individually set their state using one of the rustbot commands.

Niklaus reads this and comments with `@rustbot merge`. But then, Niklaus uses
the feature and discovers a crucial flaw. He posts a comment:

> @rustbot close
>
> After more testing, I believe this is not ready for stabilization. I have
> found that it doesn't work at all like the specification in the case of foo!
> This seems closely related to Grace's concern on the RFC; I think if we
> stabilize at this point we may indeed harm the ecosystem...

Other team members test as well, and find that Niklaus is right. Alan changes
his status using `@rustbot close`, and Grace (with some relief) sets the same
status.

Once everyone has changed their status, rustbot posts a comment:

> The final comment period has resolved, with a decision to **close**.

(Note: Since `close` is an inherently reversible status (a PR can always be
reopened), rustbot can observe that everyone has set a reversible status, and
will start treating the decision as reversible; this means the final comment
period can end even if Barbara hasn't responded yet.)

This may not be the end of this feature's consideration, and the concern might
get resolved in many different ways. The initiative owner might need to do some
additional design work on how to solve the original use case and address the
concern; implementers may find a way to address the concern by improving the
implementation; or the team may change their mind.
