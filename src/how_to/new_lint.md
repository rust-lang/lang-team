# How do I propose a new lint, or extend an existing one?

For small lints, you can follow the [policy for smaller changes](./propose.md): implement the lint, open a PR, and then [nominate](./nominate.md) it for lang-team attention. 

We may request an RFC if:

* the lint is on (warn or higher level) by default, and is expected to affect a lot of users
* the lint is controversial
* the lint sets a (new) direction for Rust -- for example, changing an existing pattern to a different one, even if the pattern isn't widely used
   * e.g., deprecating a syntax to make room for a possible new language feature

If in doubt, you can always raise the idea on Zulip first.