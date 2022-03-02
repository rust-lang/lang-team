---
tags: design-meeting
---

# 2021-02-24: Lints and editions

## Links

* [Proposed lint/error policy](https://hackmd.io/i1Ob4XS6TwuUv-rOVEoM4A)
* [Lint categorizations and descriptions](https://hackmd.io/HETreGqPSRezlN109vgCnQ)

## Agenda

* Overview general lint policy (15 minutes)
    * Key questions:
        * 
* Overview lints (30 minutes)
    * Per
* Revisit general lint policy (15 minutes)

## General lint policy


| name | Edition N | Edition N+1 | Migrations? |
| --- | --- | --- | --- |
| repurposed syntax | allow | new meaning | mandatory |
| edition error | allow | error | mandatory |
| soft deprecations | warn | warn | recommended |
| hard deprecations | warn then deny | deny | mandatory |
| bug fix | warn then error | warn then error | recommended |

Niko's concerns

* edition error: is there syntax we remove that is NOT deprecated?
* hard deprecation: is there a reason that we would want deny in the new edition, and not error?
    * something we want people to be able to type but they have to opt-in with allow (versus some other mechanism)?
* soft vs hard:
    * hard is for likely BUGS
    * not just a matter of "it'd be nice if this were better" but more like "some horrible problem"
    * Extreme danger of misuse or unsoundness.
    * Third-party tooling may wish to drop support for the pattern (and limit itself to the newer editions).

## Table

| Lint | Current status | Proposed resolution | Bugs | Meeting consensus? |
| --- | --- | --- | --- | --- |
| BARE_TRAIT_OBJECTS | warn | soft deprecation<br>edition error | :heavy_check_mark: |
| ELIDED_LIFETIMES_IN_PATHS | allow | no consensus | |
| ELLIPSIS_INCLUSIVE_RANGE_PATTERNS | warn | soft deprecation<br>edition error |
| EXPLICIT_OUTLIVES_REQUIREMENTS | allow | revisit after bugs are fixed | [#54630]  |
| UNUSED_EXTERN_CRATES | allow | not edition (though it take the edition into account) | 53964, 44294 |
| UNREACHABLE_PUB | allow | defer, not edition |

| MACRO_USE_EXTERN_CRATE | allow | decide as part of pub macro-rules | #52043
| CLASHING_EXTERN_DECLARATIONS | allow | hard deprecation | #52043 |


[#54630]: https://github.com/rust-lang/rust/issues/54630
