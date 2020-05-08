# Can we allow integer literals like `1` to be inferred to floating point type?

## Background

In rust today, an integer like like `1` cannot be inferred to floating
point type. This means that valid-looking numeric expressions like
`22.5 + 1` will not compile, and one must instead write `22.5 +
1.0`. Can/should we change this?

## History

This was discussed on Zulip in [May 2020]. Some of the key highlights from the discussion were:

* Floating point literals are prone to many surprises that integers
  are not. For [example][], `20_000_000 + 1`, if inferred to `f32`
  type, would have the final value `20_000_000.0`. This leads some to
  conclude that ["the surprise factor of floats is so high that they
  are qualitatively different than integers"][surprise].
* But there are a lot of similar surprising things:
  * [`20_000_001_f32`, for example, is also going to be `20_000_000`.](https://zulip-archive.rust-lang.org/213817tlang/43153EditionRequestlet1beafloatliteral.html#196930252)
  * Integer overflow can mean that `255 + 1` has the value 0.
* Some key questions to consider:
  * [How often does adding `.0` result in some insight?](https://zulip-archive.rust-lang.org/213817tlang/43153EditionRequestlet1beafloatliteral.html#196926142)
  * How much would *seeing* the `.0` help to debug a tricky problem?
* Balanced against the annoyance and surprise factor.

[example]: https://zulip-archive.rust-lang.org/213817tlang/43153EditionRequestlet1beafloatliteral.html#196887384
[surprise]: https://zulip-archive.rust-lang.org/213817tlang/43153EditionRequestlet1beafloatliteral.html#196887384

[May 2020]: https://zulip-archive.rust-lang.org/213817tlang/43153EditionRequestlet1beafloatliteral.html
