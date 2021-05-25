# Auto traits

Auto traits permit automatically implementing a trait for types which contain
fields implementing the trait. That is, they are fairly close to an automatic
derive. They describe properties of types rather than behaviors; current stable
Rust has two auto traits: `Send` and `Sync`.

Auto traits are tracked in [rust-lang/rust#13231], and are also sometimes
referred to as OIBITs.

As of November 2020, the language team feels that new auto traits are unlikely
to be added or stabilized. See [discussion][freeze discussion] on the addition of `Freeze` for
context. There is a fairly high burden to doing so on the ecosystem, as it
becomes a concern of every library author whether to implement the trait or not.

Each auto trait represents a semver compatability hazard for Rust libraries, as
adding private fields can remove the auto trait unintentionally from a type.

Stabilizing the ability to define auto traits also allows "testing" for the
absence of a specific type:

```ignore
auto trait NoString {}
impl !NoString for String {}
```

This is not something we generally want to allow, as it makes almost any change
to types semver breaking. That means that stabilizing defining new auto traits is
currently unlikely.

[rust-lang/rust#13231]: https://github.com/rust-lang/rust/issues/13231
[freeze discussion]: https://zulip-archive.rust-lang.org/213817tlang/73585Freezestabilizationandautotraitbackcompat.html
