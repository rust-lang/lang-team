# Welcome

Welcome to the repository for the Rust Language Design Team.  This
page stores our administrative information, meeting minutes, as well
as some amount of design constraints. It's always a
work-in-progress (insert omnipresent mid 90s logo for under
construction here).

## What is the lang team

The lang team generally governs the "surface area" of the language, meaning both what code compiles and what happens when it executes. For new language features, we also assume a general "project management" role, in that we track the feature as it progresses from an idea, to implementation, and finally to stabilization.

Note that implementation itself is governed by the [compiler team](https://github.com/rust-lang/compiler-team). To be very concrete, the lang team controls the spec, and the compiler team approves the implemenation and ensures that it meets the spec. Naturally, though, development is a collaborative process, and it often happens that the spec is altered in response to concerns that arise during implementation.

We also work closely with the [types team](https://github.com/rust-lang/types-team), which owns the details of the type system design in much the same way as the compiler team owns the details of the implementation.

Finally, there are often "grey areas" between the language and library teams, such as the addition of a new standard library trait that reflects a core system capability (e.g., `Future`). In those cases, the lang team is generally reponsible for deciding if we want the core capability, the libs team owns the API details.

## Key links

* [Active initiatives project board](https://github.com/orgs/rust-lang/projects/16/)
    * Shows you the things that are currently under development (or exploration) within Rust.
    * You can [read more about initiatives here](./initiatives.md).
* [Meeting calendar](./calendar.md), [triage meeting minutes](https://github.com/rust-lang/lang-team/tree/master/minutes), [design meeting minutes](https://github.com/rust-lang/lang-team/tree/master/design-meeting-minutes)
    * You can [read more about our meetings here.](./meetings.md).
