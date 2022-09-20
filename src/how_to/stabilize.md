# Stabilize a feature

The final step in the [language-change process](./propose.md) is to **stabilize** a feature. Stabilization works as follows:

* Author a [stabilization report][sg]:
    * Briefly recap the feature's design from the RFC -- you don't have to go into detail, we can re-read the RFC.
    * Give detailed descriptions of how the feature's design has changed since the RFC was approved!
    * Summarize any major decisions that were made during the implementation process.
    * Verify that the feature is fully implemented. Look for tests covering all the major pieces of the RFC and include them in the stabilization report.
    * Provide answers to any "unresolved questions" listed in the RFC.
    * Describe the implementation history of the feature (optional).
* Prior to stabilizing, we need to coordinate with other teams:
    * An open PR editing the reference to describe the change is required. (You don't personally have to author it, but there needs to be an open PR, and ideally one that has been edited and is generally accepted.)
    * If the feature affects the type system, you should ping @rust-lang/types to check for their approval.
    * If the feature adds new syntax, you should ping the style team.
* The lang team will read the report and eventually move to FCP. Per our [decision process](../decision_process.md), full consensus is required, as this is an irreversible change.
    
See also the rustc-dev-guide's [stabilization guide][sg].

[sg]: https://rustc-dev-guide.rust-lang.org/stabilization_guide.html