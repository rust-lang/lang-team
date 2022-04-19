# Triage meeting

The weekly triage meeting is when we go over the newly filed project
proposals along with issues that have been nominated for lang-team
feedback. We also get regular updates from the active project groups
so we can stay on top of what is going on.

## Can I attend?

Yes! The triage meeting is open to the public. You'll find the details
on our [calendar](src/../../calendar.md).

## How do I get something on the agenda?

The easiest way to get something on the agenda is to [nominate it](../how_to/nominate.md). The agenda is automatically built by triagebot from [this template](https://github.com/rust-lang/triagebot/blob/master/templates/lang_agenda.tt). We review pending [initiative proposals](../how_to/propose.md), [nominated](../how_to/nominate.md) issues and PRs from a variety of repositories, as well as pending RFC requests.

[triagebot]: https://github.com/rust-lang/triagebot

## Can I generate the agenda myself?

Sure, clone the [triagebot] repo and run this

```bash
> cargo run --bin lang agenda
```

If you install the `hackmd-cli`, you can do this:

```bash
cargo run --bin lang agenda | hackmd-cli import 
```

## Where can I find the minutes?

[Triage meeting minutes are available in this directory.][tmm]

[tmm]: https://github.com/rust-lang/lang-team/tree/master/minutes
