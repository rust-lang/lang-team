This file will contain summaries from the meetings where the
unsafe-code-guidelines WG synchronizes with the lang team and broader
community. That hasn't happened yet. =)

# First meeting agenda (Date TBD)

## Review of how UCG has operated thus far

- The UCG reference lives on the repository
- At any given time we have an "active area of discussion":
    - This is proposed in a PR which describes an initial set of discussion topics
      ([example](https://github.com/rust-lang/unsafe-code-guidelines/pull/54)).
    - When an issue seems to reach consensus, somebody writes up a
      summary in the form of a PR that edits the reference
      ([example](https://github.com/rust-lang/unsafe-code-guidelines/pull/57)).
    - Sometimes, the consensus contains further points of contention
      -- in this case, we open up new issues representing just those
      points (and reference them from the reference).
- Weekly meetings on Zulip; mostly try to look for issues that
  seem to have reached consensus and assign work, don't usually involve
  a lot of technical discussion.
  
Experience with this system? It seems to be working reasonably well,
though we've not had as much participation as I had hoped, and we've
been moving a bit slower than I might've thought. I think this is
largely because these discussions are nobody's top priority right
now. -nikomatsakis

## General roadmap

Our immediate goal is to produce an "RFC" covering two inter-related areas:

- **Layout:**
  - What are the layout rules that unsafe code authors can rely on?
  - When do we guarantee ABI compatibility between Rust types and C types?
  - When can you rely on "enum layout" optimizations?
- [**Validity invariants:**](https://github.com/rust-lang/unsafe-code-guidelines/blob/master/active_discussion/validity.md)
  - 
