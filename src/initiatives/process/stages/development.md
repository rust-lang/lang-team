# Stage 3: Development

After an RFC is approved, the initiative enters "development" stage. The only difference from the "experimental" stage is that the feature gate can now be marked as "non-experimental" and hence used without any sort of warning.

During this stage, the focus is on implementing the complete feature and on resolving any unresolved features.

## Exit: Group declares feature "feature complete".

At some point, the group can declare a feature to be "feature complete". This requires the following materials to be available:

- Accessible documentation in the "unstable Rust" user's guide, if appropriate.
- All unresolved questions from the RFC have documented answers.
- Tests are written covering all major points in the RFC.
  - Ideally, those tests should be documented, as well, though we don't have a real convention here.

## Exit: Go inactive

It often happens that initiatives "stall out". This could be because some of the problems seem insurmountable, because the people involved wind up not having enough time to continue, or because other things take priority. At any point, the owner can decide to step back, at which point the initiative becomes [inactive](./inactive.md). If an initiative has not made progress in several months, the team may also opt to move it to inactive status.
