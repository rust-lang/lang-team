# Decision-making process, detailed description

- Entering a "decision period" can be done by having a team member tell rustbot
  an initial status (`merge`, `stabilize`, or `close`; or, `reversible` or
  `irreversible` with a custom identifier).
  - All other team members have no initial status set.
  - During a reversible decision period, if later commenters indicate the
    decision is irreversible, the decision changes to irreversible.
  - During an irreversible decision period, if all commenters change their
    status to a reversible status (most commonly `close`), the decision becomes
    reversible.
  - Bot commands accept `mut`/`mutable`/`rev` as synonyms for `reversible`, and
    `immut`/`immutable`/`irrev` as synonyms for `irreversible`.
- Once the "decision period" has begun, a clock of 10 days starts. The clock is
  never paused unless an explicit `@rustbot restart` command is given.
- The `restart` command sets the decision period back to its initial state
  (members' current statuses are also set back to blank, though the history of
  their previous statuses is preserved as with any other status change).
- A decision is reached when the following conditions are met:
  - At least 10 days have elapsed since the decision period began (or when it
    was last `restart`ed).
  - Everyone is set to the same status, or to `abstain`, or to `dissent` (with
    at most one `dissent` status), or (for a reversible decision only) blank.
- Member may place a *hold* (raise a concern) at any phase of the process:
  proposal, experimentation, testing, stabilization. Ideally, concerns should
  be raised as early as possible.
  - In general, one of the liaison's jobs is to anticipate concerns that may
    arise and reach out proactively to the members of the team. If we find that
    team members regularly need to place serious holds for similar reasons,
    that may indicate we need to work harder at team calibration.
- "Holding" a decision is simple. If you have a potential concern that you
  haven't had a chance to fully articulate yet, you get periodic pings.
- In order to proceed after a concern is raised, whether sustaining the concern
  or overriding the concern, the team must understand the concern. This
  understanding should be expressed in writing rather than just verbally.
  Commonly, the owner of an initiative/proposal may incorporate the concern
  into the proposal and address it there (whether via their own words or those
  of a team member).
  - Ideally, a concern is only considered "understood" if the objector agrees
    it has been. If that is not possible, then everyone on the team other than
    the objector must unanimously agree that there is no further understanding
    to be gained. (In practice, this agreement is determined by whether anyone
    on the team is willing to support the objection.)
- If the owner feels that the concern has been adequately addressed, they can
  produce a write-up that describes the concern and request a poll of the team
  members to see where everyone stands.
  - Sustaining an objection requires one team member other than the objector. A
    team member should sustain an objection if either they believe the
    objection has been understood and they agree that it must be addressed, or
    if they believe the objection has not been fully understood (whether they
    personally agree with it or not).
  - If everyone else agrees that the concern has been understood *and* that the
    current design is sufficient, then the concern is *overridden*.
  - Team members are encouraged to regularly sustain objections they don't
    personally agree with, if they believe the objection has not yet been fully
    understood; this is a normal and valuable part of the process.
- Whenever a concern is overridden, team members are encouraged to add a
  **dissent** into the document to describe their concern and why they don't
  agree with the decision. They (or another team member) should then set their
  status to `dissent`.

# Delegation of decisions

We trust other teams to make decisions about their areas of expertise, without
waiting on language team approval on every such decision.

If the matter being decided seems non-trival or subtle, then we do wish to be
part of the decision-making process. But if the answer for a problem seems obvious,
and the corresponding decision is reversible, then it need not wait on T-lang nomination and approval.

For example, we trust the compiler team to fix obvious bugs that are not
intended to be part of the Rust language and have minimal known breakage. See
for instance [PR #129422](https://github.com/rust-lang/rust/pull/129422), which
corrected a mistake in the implementation of `#[repr(Rust)]`). Since the correction
being applied here was changing a no-op to an error, we have the option of
reversing the decision (going from an error back to a no-op), and thus this decision
can be delegated without having to go through T-lang nomination and approval.

However, when another team makes a language-relevant decision, the language team
also wants to be aware of it. This way we can ensure that the set of delegated
decisions do not drift out of their intended scope.

If your team is making a decision that impacts the language, please ensure that
a link to the decision (and any related discussion) appears in the agenda for
the language team's triage upcoming meeting.

(If a decision cannot be easily descibed and justified in a short snippet of
text, then it possibly needs to be brought to the attention of the language team
through the normal nomination process.)
 
# Rustbot commands

The following commands are accepted by rustbot. (Commands written below omit
the required `@` on rustbot to avoid invoking rustbot when quoting the
documentation.) A number of comments take one or more optional `@member`
arguments, denoted `@member*`; if supplied, the command is issued on behalf of
those member(s), instead of the person writing the command. (The `@` for each
member is required.) It is also permitted to write `@rust-lang/team` to select
all members of the team. rustbot will always link from each member's row in the
table to each comment changing their status.

- `rustbot @member* reversible ident` or `rustbot @member* mutable ident`
  (unambiguous prefixes such as `mut` or `rev` also work)
  - If not in a decision period: begin a reversible ("mutable") decision,
    proposing the outcome `ident`; all other members are set to a blank status.
  - If in a decision period: set yourself to `ident`. (Decisions do not proceed
    unless all members have the same status or `abstain`.)
- `rustbot @member* irreversible ident` or `rustbot @member* immutable ident`
  (unambiguous prefixes such as `immut` or `irrev` also work)
  - If not in a decision period: begin an irreversible ("immutable") decision,
    proposing the outcome `ident`; all other members are set to a blank status.
  - If in a decision period: set yourself to `ident`. If the decision was
    previously considered reversible, change it to irreversible. The decision
    now requires full team consensus. (Members may set a status of `abstain` on
    the decision if they wish.)
- `rustbot @member* merge`
  - Alias for `rustbot @member* reversible merge` - a reversible decision with
    the proposed outcome `merge`.
- `rustbot @member* close`
  - Alias for `rustbot @member* reversible close` - a reversible decision with
    the proposed outcome `close`.
- `rustbot @member* stabilize`
  - Alias for `rustbot @member* irreversible stabilize` - an irreversible
    decision with the proposed outcome `stabilize`.
- `rustbot @member* abstain`
  - If not in a decision period: error
  - If in a decision period: set your status to `abstain`. This status does not
    block a decision.
- `rustbot @member* hold`
  - If not in a decision period: error
  - If in a decision period: set your status to `hold`
    - Every N days while a hold persists, rustbot will ping all members who
      have status `hold` (and potentially other involved team members as well)
- `rustbot @member* dissent`
  - Equivalent to `abstain`, except that it sets a status of `dissent`. This
    status on one team member does not block a decision. A status of `dissent`
    on two or more team members will block a decision.
  - Note that `dissent` should not be set when first raising a concern, only
    after attempts to resolve the concern have been unsuccessful.
- `rustbot restart`
  - If not in a decision period: error
  - If in a decision period: set all members other than the one issuing the
    command to have a blank status. Preserve the last set status of the person
    issuing the `restart`.
  - Note: if the last set status of any team members were irreversible, the
    decision will continue to be treated as irreversible until all such members
    set an explicit status otherwise.
- `rustbot cancel`
  - If not in a decision period: error
  - If in a decision period: cancel the decision period.
  - Note that if a subsequent decision is started in the same issue, rustbot
    should link to the previous decision summary table.

# Frequently asked questions

## Why can members override other members positions?

It's quite common to want to check boxes and similar on behalf of all the
people in a meeting. It's also annoying to have to restart a decision process
(e.g. closing and reopening with rfcbot) just to be able to close concerns on
someone else's behalf (e.g. people who have left the team, or people who raised
a concern on behalf of someone else not on the team, or similar). We can trust
each other on the team. If people abuse `rustbot` to disrupt the process, that
isn't a problem to be solved with tooling.

## Do we want to require "all but N" people to affirm a decision, as `rfcbot` does?

We opted to require 100% participation on irreversible decisions such as
stabilization, but not on very lightweight reversible decisions such as
starting an initiative. We believe that lang team members can be expected to at
least leave a comment (even if just `rustbot abstain`), but in the limit we
also have the option to set a status on another member's behalf.

## Why does the timer start when the decision period *starts*, and not when a consensus is reached?

The current `rfcbot` starts the 10 day FCP timer once a consensus is reached.
However, we have observed that, in practice, the "holdouts" on consensus are
typically the lang team members. Further, this seems to assume a "two-phased"
decision making model where folks outside the team are commenting only after
the lang team has reached a consensus. In practice, both decision-making and
commenting tend to be more fluid, and hence we would prefer not to have a long
delay after we reach consensus.

## Why not use checkboxes?

This process recognizes that sometimes, in the flow of conversation, the
decision to merge can transmute into a decision to close, or there may be
multiple potential outcomes.

The presentation of statuses also makes it easy for `rustbot` to present the
history of status changes, with links to team members' messages.

## Why is `close` always reversible?

Because, well, it is! We can always re-open a PR or issue, after all.

## What about "postpone"?

Just write it in the comment for a `@rustbot close`. In practice, a "close"
does not preclude reopening later, and a "postpone" does not guarantee
reopening later.

## Is "Don't force an irreversible decision?" absolute?

No; it's a strongly held principle, but not an absolute one. Sometimes we may
have to make an irreversible decision, even in the face of dissent. However, we
should be *extraordinarily* careful when doing so, and in particular, we should
consider very carefully whether we could make a reversible decision, or find a
better consensus, or whether the consequences of *not* making the decision
outweigh the consequences of making it. This should be an extremely rare event.

The previous decision-making process based on `rfcbot` allowed indefinitely
blocking concerns. This new process introduces a means of carefully resolving
such concerns, and a very careful means of proceeding *despite* such concerns
while ensuring those concerns are understood and recorded and considered.

## What purpose does `restart` serve?

Sometimes, a proposal has changed enough to warrant re-checking people's
positions, but has not changed enough to warrant closing it and starting the
process over. `restart` clears people's statuses to ensure that they have the
opportunity to re-confirm (or raise a hold or concern) before the decision
proceeds.
