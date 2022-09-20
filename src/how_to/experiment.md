# Experimental feature gates

We use "experimental feature gates" to allow experienced Rust contributors to start implementing and exploring features even before an RFC has been written. This is particularly useful for larger features where we know we definitely want to solve the problem, but there are a lot of unknowns to work out before we can really create a coherent RFC -- think of things like adding async functions or the like.

[rfc]: https://github.com/rust-lang/rfcs/#when-you-need-to-follow-this-process

## Process

If you are an experienced Rust contributor who would like to start an experiment in-tree, the process is as follows:

* Write-up a description of the problem you are trying to solve and the general shape of the solution you want to work on. Discuss it on Zulip or elsewhere to find a **lang-team champion**:
    * The champion is the connection to the lang-team. They can check in with you from time to time to see how the work is going and relay those updates to the lang-team (of course, you're always welcome to join meetings yourself too!). They can also help to discuss problems that arise.
* Once you've found a champion, open a PR adding a new feature gate to the compile and create an associated tracking issue.
    * The PR should include a write-up documenting the motivation and outline of what they are trying to achieve. 
    * The feature gate should be marked as 'experimental', so that users get warnings if they try to use it. This flag has to stay until an RFC is accepted, even if the implementation is in good shape.
* The lang-team champion will "second" the PR, starting an FCP. Once the FCP completes, the PR can land and implementation work begins (always gated under the new feature gate).
    * **Approving a new feature gate does not imply support for the feature.** It implies only that the lang team thinks it is worth doing the experiment to see what results.
    * Note to lang team members: If you have concerns about the feasibility or wisdom of the feature, the right course of action is usually to allow experimentation to continue, but ensure that your concerns are noted on the tracking issue. This allows the experimentors to try and gather data and address your concern.
* When you feel the design is ready, you write an RFC as normal with your proposal. The goal of the experimentation period is simply to gain experience and information so that a better RFC can be authored.

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
