# const generics

## Summary and problem statement

Implement and design the `const_generics` feature with
the initial goal of stabilizing the `min_const_generics` subset.

## Prioritization

This project group will pretty much directly work on the third highest [lang team priority] by trying to support const generics on stable.

[lang team priorities]: https://lang-team.rust-lang.org/priorities.html

## Motivation, use-cases, and solution sketches

Const generics allows us to to parameterized types by values. Most noticably
this allows us to abstract over array lengths.

The initial goal is to stabilize `min_const_generics` in a way which is both usable
while preventing any future developments.

To do so we will impose two restrictions on const generics:

- They can only have the type of an integral primitive type (integers, bool, and char).
- Expressions used in const generic position must be either:
  - Just a const generic name in scope (e.g. N)
  - A const expression which contains on no free type or const parameters.

There are also some additional concerns which require attention before stabilizion `min_const_generics`, most of which are hopefully minor. 

The currently most interesting problems after `min_const_generics` is how to assert
the well-formedness of const generics and how to introduce lazy normalization.

## Links and related work

- [original RFC](https://github.com/rust-lang/rfcs/pull/2000)
- [`min_const_generics` compiler MCP](https://github.com/rust-lang/compiler-team/issues/332)
- [`min_const_generics` tracking issue](https://github.com/rust-lang/rust/issues/74878)
- [lang team meeting](https://github.com/rust-lang/lang-team/issues/37)

## Initial people involved

@lcnr

*List the lang-team liaison who picked up the proposal along with the project group lead(s) and any other folks who would like to participate in some capacity. If you have some idea who might implement the idea, that's good to list too.*

