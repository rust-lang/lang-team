# Backlog bonanza

Backlog bonanza is a particular kind of design meeting. We often schedule a backlog bonanza for those weeks where we don't have more specific things to discuss. The idea is to go through each tracking issue and "disposition them". The goal is to identify what we ought to do with this particular unstable feature; e.g., what is blocking this from being stabilized? Do we still want this? Is it perma-unstable?

## When does backlog bonanza take place?

Backlog bonanza meetings are typically scheduled as [design meetings](design.md).

## Can I attend?

Yes! Design meetings are open to the public. You'll find the details on our [calendar](../calendar.md).

## What labels do we apply to issues?

Here are the labels we apply during the process and their meaning:

* S-tracking-ready-to-stabilize: Needs a stabilization PR (good to go :train:)
* S-tracking-needs-to-bake: Needs time to bake (set a date? other criteria?)
* S-tracking-impl-incomplete: Not code complete or blocking bugs
* S-tracking-unimplemented: Implementation not begun
* S-tracking-design-concerns: Blocking design concerns
    * This might be "doesn't quite seem to deliver value we hoped for" or "something doesn't feel right"
* S-tracking-perma-unstable
    * Internal implementation detail of rustc, stdlib
* S-tracking-needs-investigation

## Where can I find the minutes?

Currently the minutes are tracked in the issues themselves, but we also create hackmd documents in the [Rust lang team](https://hackmd.io/team/rust-lang-team?nav=overview).