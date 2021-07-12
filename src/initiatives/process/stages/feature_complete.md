# Stage 4: Feature complete

"Feature complete" initiatives are awaiting community experimentation and stabilization. Typically this is done by writing a blog post (on Inside Rust, perhaps) encouraging experimentation and feedback. That feedback should be gathered and summarize in the monthly reports. It is particularly useful to have lists of users.

The owner should continue to meet regularly with the liaison at this time, though the meeting frequency is often just once a month and quite short.

## Entry criteria

Entering the "feature complete" stage typically requires that there is documentation available to users about how the feature works overall (but not necessarily detailed reference material):

- Liaison agrees that the feature is feature complete.
- An explainer is prepared that explains to end-users how the feature works.
- A blog post, linking to the explainer, that announces progress and requests testing and feedback.

In addition, it's a good place to explain how any "unresolved questions" from the RFC would up being resolved.

## Preparing reference documentation

In addition to evaluating the feature, this is a good period to prepare "reference" documentation that explains the changes in depth. This is typically included in the [Rust reference] but may appear in other documentation as well, such as the Necronomicon. These changes will be reviewed as part of the stabilization report and will land after the feature is stabilized.

[Rust reference]:

## Exit: Stabilization report prepared and approved

To exit the stage, the owner prepares a stabilization report. Stabilization reports follow the templat and generally give details about:

- Final design of the feature
  - In particular, how were unresolved questions or other details from the RFC resolved?
- What is tested and where
- Link to the Rust reference materials or other documentation
