# Auto traits

Auto traits permit automatically implementing a trait for types which contain
fields implementing the trait. That is, they are fairly close to an automatic
derive. They describe properties of types rather than behaviors; current stable
Rust has several auto traits: `Send`, `Sync`, `Unpin`, `UnwindSafe`,
`RefUnwindSafe`.

`Freeze` is also an auto trait indirectly observable on stable; it is used by
the compiler to determine which types can be placed in read-only memory, for
example.

Auto traits are tracked in [rust-lang/rust#13231], and are also sometimes
referred to as OIBITs ("opt-in built-in traits").

As of November 2020, the language team feels that new auto traits are unlikely
to be added or stabilized. See [discussion][freeze discussion] on the addition
of `Freeze` for context. There is a fairly high burden to doing so on the
ecosystem, as it becomes a concern of every library author whether to implement
the trait or not.

Each auto trait represents a semver compatibility hazard for Rust libraries, as
adding private fields can remove the auto trait unintentionally from a type.

Stabilizing the ability to define auto traits also allows "testing" for the
absence of a specific type:

```ignore
auto trait NoString {}
impl !NoString for String {}
```

This is not something we generally want to allow, as it makes almost any change
to types semver breaking. That means that stabilizing defining new auto traits
with no restrictions on negative impls is currently unlikely.

## Restrictions on negative impls

Not all use-cases for auto traits require unrestricted "type absence testing"
capabilities. `pyo3`'s [`Ungil` trait][Ungil] provides a good case study. `pyo3`
is a Python interoperability library, and it uses the `Ungil` trait to denote
types that are safe to access when the Python Global Interpeter Lock (GIL) is
not held. `pyo3` provides explicit `!Ungil` impls only for types defined in the
`pyo3` crate itself. Because of this, all types that do not implement `Ungil`
are defined either by `pyo3` itself, or by downstream dependencies of `pyo3`. As
long as `pyo3` does not introduce new negative impls for existing types in new
minor versions of itself, there is no backward compatibility hazard.

`obj2c` uses its [`AutoreleaseSafe` auto trait][AutoreleaseSafe] in a similar
manner.

[As of October 2023][2020-10-03 triage meeting], the lang team no longer feels
that stabilizing any form of auto traits is entirely out of the question.

[rust-lang/rust#13231]: https://github.com/rust-lang/rust/issues/13231
[freeze discussion]: https://rust-lang.zulipchat.com/#narrow/stream/213817-t-lang/topic/Freeze.20stabilization.20and.20auto-trait.20backcompat/near/214251682
[Ungil]: https://docs.rs/pyo3/0.20.0/pyo3/marker/trait.Ungil.html
[AutoreleaseSafe]: https://docs.rs/objc2/latest/objc2/rc/trait.AutoreleaseSafe.html]
[2020-10-03 triage meeting]: https://hackmd.io/tRjWZXdJQyOoD1d9rLeRGQ?view#%E2%80%9CPrepare-removal-of-auto-trait-syntax-in-favor-of-new-attribute-rustc_auto_trait%E2%80%9D-rust116126
