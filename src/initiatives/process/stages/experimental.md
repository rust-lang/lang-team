# Stage 2: Experimental (optional)

If an initiative is sufficiently complex as to warrant an RFC, then, after being seconded, the initiative enters the "experimental" state. The goal at this stage is to iterate on exploring and documenting the design space and preparing an RFC, or even multiple RFCs. These initiatives have their own Zulip stream and can land code in the compiler.

Being in the experimental stage does not represent a commitment to land the code. There may well be team members with very live concerns about the feature and it may well get removed if those concerns cannot be resolved.

Initiatives in the experimental stage can have the following resources:

- Their own dedicated Zulip stream (`#project-xxx`)
- A tracking issue
- A repository if desired
- They can land code in the compiler under an "experimental" feature gate (i.e., one that warns when you use it)
  - Ideally we would warn that this represents an early stage, "experimental" proposal

### Skipping this stage

Some initiatives are rather simple. In that case, we can skip this "experimental" stage and go straight towards development without authoring an RFC. This often applies to small tweaks in the language, or to things like adding a new lint.

### During this stage: updates to the team

During this stage, the owner and the liaison should meet on a regular basis. The owner should update the liaison about major design directions and seek their guidance on complex issues (particularly if the owner is not a member of the team). The liaison is responsible for documenting these updates and preparing a monthly update to the team as a whole. They are also responsible for deciding when an issue should be escalated to a lang team design meeting. Sometimes it makes sense to have a design meeting even if there isn't a decision to be made, just to update the team about the overall progress.

### Exit: RFC approval

To exit the Experimental stage, you need to prepare an RFC on the rust-lang/rfcs repository. This RFC needs to be approved by the lang team. The RFC can include "unresolved questions" to be resolved during the "development" phase.

### Exit: Go inactive

It often happens that initiatives "stall out". This could be because some of the problems seem insurmountable, because the people involved wind up not having enough time to continue, or because other things take priority. At any point, the owner can decide to step back, at which point the [initiative becomes inactive](./inactive.md). If an initiative has not made progress in several months, the team may also opt to move it to inactive status.
