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

If your comment requires more than 5-10 minutes of reading and discussion to understand and effectively respond to, consider filing a meeting proposal (TODO: link) instead. This will give you ~60 minutes to present your question and the lang team more time to analyze it. We may punt the question back to you with an ask to do so if the question isn't answerable in our triage meeting time.

## How quickly will the lang team answer?

We try to be prompt, but sometimes we are not. Othertimes, we discuss the issue, but fail to leave the follow-up comment, because we're only human. Please feel free to raise the topic on Zulip or reach out to a lang team member.

## Repositories where we look for nominations

Nomination is currently supported on the following repositories:

- rust-lang/rfcs
- rust-lang/rust
- rust-lang/reference
- rust-lang/lang-team

(This set is defined by the `nominated` list in the [triagebot source code](https://github.com/rust-lang/triagebot/blob/master/src/agenda.rs))



[rustbot]: https://github.com/rust-lang/triagebot/wiki
