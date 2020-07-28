# Proposing a new project

Interested in extending the Rust language in some way? The language
design team is experimenting with a new staged process.

## First step: file a Project Proposal

The very first step is [file a **project proposal** issue]. A project
proposal is a lightweight issue that describes your motivations, any
ideas you've had thus far on the design, and other background
information.

**The primary role of the proposal is to request feedback on an idea
from the lang team.** Before creating a proposal, it's a good idea to
create a thread on internals and float the idea to get immediate
feedback on whether the problem is real, prior ideas, or related
problems. However, you don't have to have a complete solution in mind
-- and it's fine to have 2 or 3 ideas for how one might solve
something.

Once you file the issue, a Zulip stream will be created and you may
start to get feedback right away. The proposal will also be discussed
in the [triage meeting], and we'll try to post general feedback from
that meeting.

The next step depends on the scope of the proposal and on whether a
[lang-team liaison] is available who has bandwidth.

[triage meeting]: ./meetings.html#triage-meeting

### Smaller proposals that can be implemented directly

If a proposal is considered "small", then the lang-team will approve
the proposal to go straight to PR. In this case, the next step is
simply to create an implementation on the appropriate rust-lang
repository. The PR can cite the proposal page in its description.

Proposals approved to go straight to PR are tagged as
`implementation-needed`.

### Larger proposals that require an RFC

Most proposals however require some sort of RFC to be implemented. In
this case, we will create a **project group** to pursue the project.
A project group is just a group of people who are authoring the RFC
and pushing it through to stabilization (and ideally through
implementation as well, although sometimes that involves very
difficult people).

If we decide that a proposal requires an RFC, then once it is assigned
a liaison, it will be tagged with `charter-needed`.

* **Propose a charter:** The liaison will work with the author of the
  proposal (and any other interested parties) to create a project
  charter, based on the [charter template]. This is often just a copy
  of the MCP, updated with whatever new material came up during
  discussion. The charter should be added to the lang-team repository
  in a PR, and that PR can "close" the proposal issue.
  * This charter PR is approved by the lang-team with a `@rustbot fcp
    merge` command.
* **Exploration phase:** Once the project is created, it enters the "exploration
  phase". We will create a Zulip stream, project issue, and potentially a repository
  as desired. The goal of this phase is to explore the design space and to come up
  with one or more RFCs that will actually solve the problem. The project
  should keep the lang team up to date on the designs being considered during
  this phase.
  * It is also permitted to land feature-gated code in the rust
    compiler during this phase, although any such change should be
    considered highly unstable (and should ideally be minimally
    disruptive).
  * Project groups are also encouraged to write regular blog posts in the 
    Inside Rust blog documenting their progress.
* **RFC phase:** The output of the exploration phase is one or more
  RFCs that describes the proposed design; these RFCs are typically
  written by the project group collaboratively. It's a good idea to
  create RFCs early because this they often receive much wider
  community feedback and may uncover problems in the design.
* **Implementation phase:** Once the RFC is landed, implementation typically
  begins, although it sometimes happens that implementation occurs concurrently
  with authoring the RFC.
* **Evaluation phase:** Once implementation of the RFC is complete, the project
  group should write a blog post requesting evaluation. They should keep the
  lang-team abreast of any feedback that is received.
* **Stabilization phase:** Finally, the feature can be stabilized and released
  on stable Rust. This is done by authorizing a stabilization report, as
  [described here][stab].

[charter template]: https://github.com/rust-lang/lang-team/tree/master/minutes
[stab]: https://rustc-dev-guide.rust-lang.org/stabilization_guide.html

### Shortlisted proposals

In some cases, there is a liaison who really likes the idea of a
proposal, but does not presently have the bandwidth to pursue the
idea. In that case, it can be assigned to the liaison but tagged as
`shortlisted`. The liaison should comment on when they expect to have
bandwidth to help out; if other liaisons are available, they may want
to pick up the project instead.

## Frequently asked questions

### What is the role of a lang-team liaison?

The lang-team liaison has the job of monitoring the progress on the
project and posting updates for the rest of the team. These updates
can be delivered in person at the [triage meeting] or as comments on
the corresponding tracking issue for the project.

Liaisons should particularly focus on questions where the lang team
can help -- for example, if the project has narrowed down the set of
options to a few viable candidates, it would be good to seek lang team
feedback on which option seems the most promising (often this would be
a good point to request a [design meeting]).

Sometimes liaisons will also get actively involved in the design itself,
or help to mediate the exploration of the design space and ensure
that the project is pursuing all the possible designs.

[design meeting]: ./meetings.html#design-meeting

### How do I become a liaision?

Liaisons play an important role and hence must either members of the
lang-team or people that the lang team selects for the role. If you'd
be interested in serving as a liaison, feel free to ping the [lang
team leads] to express your interest!

If you're interested in eventually becoming a member of the lang team,
serving as a liaison is a necessary first step (but many people serve
as liaisons without the intention to join the lang team).

### What are the things the lang-team is looking for in a proposal?

When evaluating project proposals, the lang-team is looking to figure
out the following:

* **Does this proposal solve a problem?** Does that problem fit within our
  stated priorities?
* **Who are the people involved in this proposal?** In particular, are
  there stakeholders who have explored this design but are not
  mentioned in the proposal?
* **What is the scope of work involved and is it proportional to the
  problem being solved?** Sometimes there are problems that are really
  complex to solve and ultimately not that important.
* **How "contained" is this project?** Sometimes, even narrow-sounding
  features can wind up affecting lots of things in the language and
  requiring lots of team bandwidth.
* **How controversial is this project and is it proportional to the
  problem being solved?** Let's be honest. Lang team discussions can
  sometimes get quite heated. Sometimes we might decide that even if
  we think an idea is good, it's not worth the controversy that will
  result from pursuing it.
* **Do we have the right people involved?** Some ideas require
  specific stakeholders to be a success. This might be because of
  specific knowledge (e.g., deep understanding of the type system), or
  because we want representatives for specific use cases, or because
  we want representatives from a wider variety of
  companies/backgrounds. If there have been big projects in this area,
  we might want representatives from all those projects.
* **What other things are going on?** We are trying hard not to take
  on more work than we can handle. It's very tempting in open source
  to start a ton of different activities because there are often lots
  of people around with good ideas, but having too many things going
  on is very stressful for everyone.
