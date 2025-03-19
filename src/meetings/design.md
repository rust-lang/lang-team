# Design meetings

We reserve a weekly slot for our planning and design meetings. 
A **design meeting** is a one-hour deep-dive discussion into some particular
topic. Each meeting is centered around a document that is prepared for
that meeting explaining the details of what is to be discussed; we begin by reading
the document and then discussing its contents. These meetings are used for all kinds
of purposes, such as brainstorming, getting feedback on an idea, or building
consensus around a specific proposals.

## How are design meetings scheduled?

To schedule design meetings, we hold a special **planning meeeting** once per month.
In that meeting, we choose what design meetings we will hold the rest of the month.

To generate the agenda for the planning meeting, you can use the following link and then copy/paste the generated text into a fresh hackmd page:

<https://triage.rust-lang.org/agenda/lang/planning>

## How do I propose a design meeting?

You need to open an issue, [as described here](../how_to/design_meeting.md).

## Can I attend?

Yes! Design meetings are open to the public. You'll find the details on our [calendar](../calendar.md) and a list of the upcoming meetings in the [design meeting schedule][ghp].

## How does a design meeting work?

Before the meeting starts, someone has to prepare a document -- we recommend using hackmd and using [this template](https://hackmd.io/VJrbVMeqT4uUDBRVncHyTw). 

When the meeting starts, send out the link to your document on Zulip (and on Zoom, if you like). Everyone will start to read it. **There is no expectation that people will read the document in advance.**

As they read, people will append questions to the end of the document -- the template has a space for this. We recommend making each question into a markdown section (e.g., `### Why is this document so great?`). People will append their question in that section.

After everyone is done reading, whoever is driving the meeting will pick questions to discuss. Typically we go in linear order but that's not required, we can go in whatever order seems best.

## Where can I find the minutes?

The [design-meeting-minutes directory][dnm] contains the document from each meeting along with any questions that were asked and the ensuing disceussion.

[dnm]: https://github.com/rust-lang/lang-team/tree/master/design-meeting-minutes
[ghp]: https://github.com/orgs/rust-lang/projects/31/views/10
