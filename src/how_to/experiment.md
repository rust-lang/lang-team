# Experimental feature gates

We use "experimental feature gates" to allow experienced Rust contributors to start implementing and exploring features even before an RFC has been written. This is particularly useful for larger features where we know we definitely want to solve the problem, but there are a lot of unknowns to work out before we can really create a coherent RFC -- think of things like adding async functions or the like.

[rfc]: https://github.com/rust-lang/rfcs/#when-you-need-to-follow-this-process

## Process

If you are an experienced Rust contributor who would like to start an experiment in-tree, the process is as follows:

* Write-up a description of the problem you are trying to solve and the general shape of the solution you want to work on. Discuss it on Zulip or elsewhere to find a **lang-team liaison**:
    * The liaison is the connection to the lang-team. They can check in with you from time to time to see how the work is going and relay those updates to the lang-team (of course, you're always welcome to join meetings yourself too!). They can also help to discuss problems that arise.
* Once you've found a liaison, open a PR [adding a new feature gate to the compiler][adding] and create an associated tracking issue.
    * The PR and tracking issue should include a write-up documenting the motivation and outline of what they are trying to achieve. 
    * The feature gate should be marked as 'experimental', so that users get warnings if they try to use it. This flag has to stay until an RFC is accepted, even if the implementation is in good shape.
* The lang-team liaison will "second" the PR, starting an FCP. Once the FCP completes, the PR can land and implementation work begins (always gated under the new feature gate).
    * **Approving a new feature gate does not imply support for the feature.** It implies only that the lang team thinks it is worth doing the experiment to see what results.
    * Note to lang team members: If you have concerns about the feasibility or wisdom of the feature, the right course of action is usually to allow experimentation to continue, but ensure that your concerns are noted on the tracking issue. This allows the experimentors to try and gather data and address your concern.
* When you feel the design is ready, you write an RFC as normal with your proposal. The goal of the experimentation period is simply to gain experience and information so that a better RFC can be authored.

[members]: https://www.rust-lang.org/governance/teams/lang
[adding]: https://rustc-dev-guide.rust-lang.org/implementing_new_features.html#stability-in-code

## Frequently asked questions

### What is the role of the lang team liaison?

The lang team liaison is the connection to the lang team. They should be available to discuss progress and generally track what's going on, and they can also raise questions to the broader lang team during triage meetings and the like (of course, the meetings are open, so you're also welcome to join if you are able).

### I've got an idea, how do I find a lang-team liaison?

We don't really have a process for that, but circulating the idea on Zulip is a good idea, or perhaps reach out to lang-team members that you know.

### Why is experimentation limited to experienced contributors?

We've found that it works best when the person driving the experiment is able to move independently and without mentoring. Most folks on the lang team have limited bandwidth, so when they agree to serve as liaison, they are committing to meet regularly, give feedback on your progress, and to circulate ideas within the lang team, but they are not necesarily going to have time to help find solutions to problems beyond that ((many lang team members aren't that familiar with the compiler details anyway). 

### What if I've got an implementation on a branch already?

What we're really looking for in experimentors is commitment and the ability to see the work through. If you're able to implement the idea in a branch, that's good evidence. See if you can find a lang-team liaison.

### Can a lang-team member propose an experiment, too?

Lang-team members can be the one to propose an experiment, and can serve as their own liaison, but they should find someone else to "second" the FCP on the feature gate PR.

### What if I'm not an experienced contributor, but I have a mentor who is?

That's fine, you can still open the PR, but your mentor should be the one to nominate it for lang-team consideration.

### What if I'm an experienced contributor, and I want to mentor someone?

See the previous question.

### As the experimentor, what do I do when I feel like I am ready to write the RFC?

Glad to hear the experiment was a success! Check in with the liaison to figure out if they feel like it's time to author the RFC. In particular, if people raised concerns about the design in the beginning, make sure that you have a good answer for them. Even better, show them the answer, and see if they are convinced!

### As the experimentor, what if I feel like we don't want the feature after all?

This is also a very useful finding! In this case, it's best to write up a comment (potentially short) on the tracking issue reflecting the findings from the experimentation phase, and suggest to your lang team liaison that the tracking issue may want to be fcp'd to close so we can remove the feature from the compiler.

It's generally a good idea to do this for features that aren't being actively driven to conclusion and are in the experimental phase, as they can easily accumulate otherwise.

### As the experimentor, what if I run out of time to drive this proposal?

First off, it's always ok to take a break. If you are going to step away for a long time, you should at minimum leave a comment with your current thinking on things -- if nothing else, it'll help you remember what was going on when you come back. 

If you feel like you have to step away indefinitely, then discuss with the liaison. They may be able to find someone else, or it may make sense to simply write-up your findings and remove the feature from the compiler.

### As a lang team member, what do I do if I feel like my concerns are not being addressed?

Bring up your concerns with the lang team liaison and try to work towards finding experiments or new approaches that can resolve them. If this feature is ever going to make it to RFC, they are going to need your agreement, after all!

### What if I have a competing proposal? Can I land my own experimental feature?

Obviously the *best* is if you can combine your idea with the existing experiment, particularly if your idea is a minor variation. But it is also ok to have multiple competing experiments, if the idea is going in a different direction. Just follow the same process (find a lang team liaison, etc). Be sure to note in the PR etc that you know this overlaps with the existing experiment and are looking to explore a different part of the design space. Also, it will be really helpful if you and the other experimenters can jointly maintain a kind of FAQ comparing and contrasting the two ideas.
