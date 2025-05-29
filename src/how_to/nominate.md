# Nominate an issue

You can raise issues to lang team attention by tagging them with `I-lang-nominated`. We scan through nominated issues during our [triage meetings](../meetings/triage.md). For each issue, we try to answer questions and reach decisions on the question being discussed in the issue.

## How to nominate

Add a **self-sufficient** comment to the issue that explains why you are nominated the issue and includes the text `@rustbot label +I-lang-nominated`. For example:

```
@rustbot label +I-lang-nominated

I am nominating this for lang-team attention. We have been discussing the pros/cons of updating the type-checking rules for foo bar. The options on the table are as follows:

* Allow type mismatches: This is good because blah.
* Disallow type mismatches: This is good because blah.

Where should we go from here?
```

The ideal comment will identify precisely what question you would like answered. Please try to make the comment easy for us to parse and understand without requiring a lot of context. We encourage links to internals or Zulip so we can dive into the details, but it really helps us give useful answers if you can summarize the key details up front.

If you're not a member of the Rust team on GitHub you'll get an error from rustbot when trying to add the label - please make the comment (following the guidelines above) anyway, and ping the lang team on our (chat platform)[../chat_platform.md) asking for the label to be added.

If your comment requires more than 5-10 minutes of reading and discussion to understand and effectively respond to, consider [filing a meeting proposal](./design_meeting.md) instead. This will give you ~60 minutes to present your question and the lang team more time to analyze it. We may punt the question back to you with an ask to do so if the question isn't answerable in our triage meeting time.

## How quickly will the lang team answer?

We try to be prompt, but sometimes we are not. Othertimes, we discuss the issue, but fail to leave the follow-up comment, because we're only human. Please feel free to raise the topic on Zulip or reach out to a lang team member.

### "Drag" factor and "easy decision"

The language team is experimenting with a **"drag" factor** based system for sorting agenda items. Items are quantized into five drag factor levels, `P-lang-drag-{0,1,2,3,4}`. It is dubbed a *drag factor* to avoid conflating it with priority or "worthiness" of an agenda item, but is instead a combination of factors. Items with *lower* drag factors get ranked *higher* on the agenda. Drag factor 0 agenda items come first while drag factor 4 agenda items come last during a lang meeting.

> **Example**
>
> An agenda item may have a complex characterization: it may be of high importance, yet is very difficult, and at the same time is not urgent. Such an agenda item may receive a high drag factor. That is to say, **importance is not the only factor being considered when estimating a drag factor!**

- Most normal agenda items default to drag factor 2.
- Agenda items known to have high priority for the language team and/or compiler team may be assigned drag factor 1.
    - Examples (non-exhaustive): on-going effort that the lang team has context on, stabilizations, agenda items that seem particularly important, agenda items that seem very straightforward.
- Agenda items that are really urgent, such as backport decisions, will receive drag factor 0.
- Agenda items that seem quite challenging or not very urgent may receive drag factor 3.
- Agenda items may be assigned drag factor 4 if it might take a while to go through.
    - Examples: agenda items that need a design meeting.

Lang team may lower the drag factor for nominated agenda items that were discussed but failed to reach team consensus on.

Estimated drag factors are *relative*: just because an agenda item is marked as drag factor 4, it does not mean the item is unimportant. The drag factor is instead an estimate on the expected value that may be gained from discussing another agenda item first being higher. This often has to do with lang team bandwidth and probability of getting through the agenda item versus the value of landing a change.

Within the same drag factor level, agenda items labelled with `I-lang-easy-decision` get bumped in front of other items. An agenda item might be considered an "easy decision" if the item is most likely uncontroversial, and is not complex such that one might reasonably expect the lang team to be able to make a fast decision during a meeting or asynchronously.

## Repositories where we look for nominations

Nomination is currently supported on the following repositories:

- rust-lang/rfcs
- rust-lang/rust
- rust-lang/reference
- rust-lang/lang-team

(This set is defined by the `nominated` list in the [triagebot source code](https://github.com/rust-lang/triagebot/blob/master/src/agenda.rs))



[rustbot]: https://github.com/rust-lang/triagebot/wiki
