# Checklists

Here are some checklists of the exact procedural steps to take.

## Open a proposal

* [Owner] should [open an "Initiative Proposal" issue on the lang-team repository][open-proposal].

[open-proposal]: https://github.com/rust-lang/lang-team/issues/new/choose

## Second a proposal

* If there has been significant conversation, [liaison] should ask [owner] to summarize that on the issue.
* [Liaison] should write `@rustbot second` which will trigger the start of the 10-day Final Comment Period (FCP).
* Any lang team member who has concerns should leave them in Zulip and on the issue during the final comment period.
    * Owner and liaison are responsible to describe how they expect to resolve the concern. This can be something simple, like "we will do a write-up and a design meeting on this point".

## Approve a proposal

Once a proposal has been seconded and the FCP has expired:

* If the initiative wants its own repository, then open an issue on the [infra-team repository](https://github.com/rust-lang/infra-team/) and request that the infra team clone the [initiative-template repository](https://github.com/rust-lang/initiative-template) with whatever name you need (`your-initiative-here`).
* Create a tracking issue to track the initiative:
    * If should be on `rust-lang/your-initiative-here` if you have a repository
    * Otherwise it should be `rust-lang/rust` or `rust-lang/lang-team`
    * Tag it with `C-tracking-issue` and `T-lang`
* Add the tracking issue to the [project board][pb]:
    * Set the status appropriately (typically "experimental").
* If necessary, add a feature-gate to the compiler and tag it as "experimental".

## Close a proposal

* After 4 weeks without a second, rustbot will post (not yet implemented) and add "disposition-close" and "final-comment-period":
    * "Hi there @rust-lang/lang and @author! It has been 4 weeks and this initiative has not been seconded. Initiatives that are not seconded after 6 weeks are automatically suggested for closure. Members of @rust-lang/lang, please take a look and see if you want to second this! -- your friendly neighborhood rustbot"
* After 6 weeks without a second, rustbot will post the following and add "timer-elapsed"
    * "Hi there @rust-lang/lang and @author! It has been 6 weeks and this initiative has not been seconded. It seems that there isn't bandwidth to take up this proposal right now. I am tagging this issue to be closed at the next triage meeting. Please feel free to reopen it in the future when more bandwidth is available, especially if you are able to find a lang team member who says they will second it. -- your friendly neighborhood rustbot"
* Human can then close the issue from the triage meeting with comment sort of like:
    * "Thanks for the proposal @author. Unfortunately there wasn't a second at this time and so I am going to close the issue."

## Exit the [experimental] stage and enter [development] stage

* Write an RFC
* Get the RFC approved
* xxx fill this in

## 

[pb]: https://github.com/orgs/rust-lang/projects/16/
[proposal]: ./initiatives/process/stages/proposal.md
[experimental]: ./initiatives/process/stages/experimental.md
[development]: ./initiatives/process/stages/development.md
[feature complete]: ./initiatives/process/stages/feature_complete.md
[stabilized]: ./initiatives/process/stages/stabilized.md
[Stage]: ./initiaives/process/stages.md
[Owner]: ./initiaives/roles/owner.md
[Liaison]: ./initiaives/roles/liaison.md
