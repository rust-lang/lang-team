# Start an experimental feature

Sometimes it is helpful to be able to do experimentation before authoring the RFC. **We allow experienced Rust contributors to create an experimental nightly feature gate even before an RFC is authored, if you can find a lang-team champion for the idea.**

[rfc]: https://github.com/rust-lang/rfcs/#when-you-need-to-follow-this-process

## Roles

When doing an experiment, there are two roles:

* An **experimentor** who is driving the work. They need to be an experienced contributor and able to devote enough time to keep the idea moving forward steadily.
* A **lang-team champion** who will [second](../decision_process.md) the PR. 

These can be the same person, but in that case, you should find someone else to second the PR.

## Process

The process is as follows:

* The **experimentor** should open a PR adding a new feature gate to the compiler, along with an associated tracking issue. They should include a short write-up documenting the motivation and outline of what they are trying to achieve. 
* The **lang-team champion** can "second" the PR, starting an FCP. Once the FCP completes, the PR can land and implementation work begins (always gated under the new feature gate).
    * **Approving a new feature gate does not imply support for the feature.** It implies only that the lang team thinks it is worth doing the experiment to see what results.
    * *Note to lang team members:* If you have concerns about the feasibility or wisdom of the feature, the right course of action is usually to allow experimentation to continue, but ensure that your concerns are noted on the tracking issue. This allows the experimentors to try and gather data and address your concern.

**In order to be stabilized, a full RFC will be required.** The goal of the experimentation period is simply to gain experience and information so that a better RFC can be authored.

[members]: https://www.rust-lang.org/governance/teams/lang

## Frequently asked questions

### What is the role of the lang team champion?

The lang team champion is the connection to the lang team. They should be available to discuss progress and generally track what's going on, and they can also raise questions to the broader lang team during triage meetings and the like (of course, the meetings are open, so you're also welcome to join if you are able).

### I've got an idea, how do I find a lang-team champion?

We don't really have a process for that, but circulating the idea on Zulip is a good idea, or perhaps reach out to lang-team members that you know.

### Why is experimentation limited to experienced contributors?

We've found that it works best when the person driving the experiment is able to move independently and without mentoring. Most folks on the lang team have limited bandwidth, so when they agree to serve as champion, they are committing to meet regularly, give feedback on your progress, and to circulate ideas within the lang team, but they are not necesarily going to have time to help find solutions to problems beyond that ((many lang team members aren't that familiar with the compiler details anyway). 

### What if I've got an implementation on a branch already?

What we're really looking for in experimentors is commitment and the ability to see the work through. If you're able to implement the idea in a branch, that's good evidence. See if you can find a lang-team champion.

### Can a lang-team member propose an experiment, too?

Lang-team members can be the one to propose an experiment, and can serve as their own champion, but they should find someone else to "second" the FCP on the feature gate PR.

### What if I'm not an experienced contributor, but I have a mentor who is?

That's fine, you can still open the PR, but your mentor should be the one to nominate it for lang-team consideration.

### What if I'm an experienced contributor, and I want to mentor someone?

See the previous question.
