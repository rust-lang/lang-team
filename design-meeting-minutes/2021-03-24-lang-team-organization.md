# Lang team organization

* [Watch the recording](https://youtu.be/_4XO3YNsuLY)

# Meeting agenda

* 10 minutes: 
    * Niko gives an overview of the doc for those who haven't read it
    * People can add FAQs to the list below as they go
* 5 minutes (if needed):
    * People add to the FAQ below with questions or thoughts
    * People add their names to FAQs they would like to discuss
* 10 minutes:
    * Top voted FAQ #1
* 10 minutes:
    * Top voted FAQ #2
* 10 minutes:
    * Top voted FAQ #3
* 5 minutes:
    * Spillover because schedule was too ambitious
* 10 minutes:
    * Summarize and consider next steps

# Motivations

## Observations

* Lang team has a need for many different skill sets (sometimes found in a single person, sometimes not):
    * Managing large feature proposals
    * Fixing nits and papercuts
    * Monitoring "zeitgeist" of recent discussions, looking for ideas in the wild
    * High-level stuff like async-await, borrow-checker
    * Low-level stuff like "C unwind" or FFI interop details
    * Formal rules (e.g., unsafe code guidelines)
    * Maintaining reference material
* Lang team needs to grow, but a flat structure is ill suited to growth
    * Reaching consensus with too many people is untenable
    * Want to maintain a coherent "overall design"
* Meeting maintenance tasks are overwhelming at present, and mostly done by Niko
    * Uploading videos
    * Writing and uploading minutes
    * Blog posts announcing meetings
    * Blog posts summarizing meetings
    * Updating calendar

## Tension between keeping cohesion and growing capacity

Under the current structure, the only way to participate in the lang team is either to lead a project or to become a full member (and thus decide on RFCs and the like). This creates an inherent tension: if we grow the set of full members, we will have more capacity overall, but we will also have more people to weigh in on every decision, and create greater risk of design incohesion. Further, the only path to membership in a system like this is to work with a project group, which is detailed activity that is not well suited to all participants or experience levels.

## Different modes of interaction

To address the previous points, we would like to create multiple "modes" of interaction, supporting people with different amounts of availability and experience:

| Style of interaction                           | Role                   |
| ---                                            | ---                    |
| Tactical and not requiring extended engagement | triage                 |
| Broad interaction, potentially sporadic        | shepherd               |
| Deep, focused interaction                      | project member or lead |
| High commitment and responsibility             | lang team member       |

There are plenty of folks for whom it makes sense to serve as a shepherd, project group, or triage member, but not as a full lang team member. They may either lack the time or experience for the commitment and responsibility of deciding on RFCs; or, alternatively, there may simply not be room to add new members to the team without making decision making too unwieldy.

There are also people for whom it would make sense to play multiple roles -- for example, lang team members are typically expected to lead projects from time to time, and may well also serve as a shepherd (Josh Triplett is an example of someone who often plays both roles).

# Teams, subteams, and project groups

Clarify and alter the structure of the lang team and associated subteams.

* **Lang team** *decides on the syntax/semantics of the language*
    * Guide the development of new language features
    * Decide which project proposals to accept (i.e., which projects to officially start)
    * Resolving questions that come up in open PRs or on issues about the finer points of language semantics
    * Deciding which lints to accept
* **Shepherds subteam** *liaises with the greater community*
    * Monitor conversations on internals, zulip, and elsewhere
    * Identify possible project proposals and help people to develop them
    * Help sharpen project proposals by summarizing or encourage authors to summarize feedback
    * Identify project proposals that may be ready to be approved and take them to the lang team
* **Triage subteam** *prepares the meeting agenda*
    * Check out nominated issues to ensure that the question to be answered is clear and relevant background material is available
    * Identify subgroups that can weigh in on the question if it is more narrow
    * Perhaps forward some questions to shepherds
* **Documentation subteam** *maintains the reference, nomicon, and others*
    * Vet, write, and merge changes to the document
* **Projects and project groups** *pursue the design and implementation of a specific feature*
    * Goal is to prepare recommendations for the lang team and elaborate the reasons for those recommendations, along with possibly a suggested course of action
    * Chartered based on a project proposal, has assigned lead(s)
    * Some groups have regular meetings, others do not, depends on how the leads want to run things
    * These intersect other teams, like compiler and libs

# Meetings

* Team triage meeting (weekly)
    * Review nominated questions, as prepared by the triage subteam
    * Judge when project proposals 
    * Make decisions on smaller matters
* Planning meeting (monthly)
    * Review check-ins and status updates from active projects and project groups
    * Decide what meetings to hold the rest of the month
* Design meetings
    * In-depth discussion on particular topics
    * Each meeting has a particular lead -- typically member of lang team or shepherd
    * Write-up to be available 24hr before; canceled if no write-up is available

# Zulip streams

* t-lang -- general discussion
* t-lang/shepherds -- area for shepherds to talk, identify interesting threads, coordinate
* t-lang/doc -- doc team discussion
* t-lang/triage -- triage team can be active
* t-lang/meta -- discussion of lang team procedures
* t-lang/projects

# Lang team "user interface"

*Status:* Very draft

* How do I propose a new project?
    * Discuss on internals or elsewhere
    * If you'd like feedback or advice, contact a shepherd
    * Open a [Project Proposal]
* Get clarification about intended language semantics in an issue ("is this a feature or a bug?", "what should this corner case do?")
    * Add I-Nominated label to something tagged with T-lang
* Ask lang team to look at an issue or a question and give their opinion or make a decision
    * Add I-Nominated label to something tagged with T-lang
* Propose tweaks to existing features or resolve corner cases with a PR
    * Add I-Nominated label to something tagged with T-lang
* Propose a new lint
    * If you want feedback before doing the work, open a [Project Proposal]
    * Write the lint, open a PR, tag with T-lang and nominate 
        * You may be asked to author a project proposal or RFC, if the lint proves to be controversial or complex.
* Tweak an existing feature or clear up a minor inconsistency
    * If you want feedback before doing the work, open a [Project Proposal]
    * Open a PR, tag with T-lang and nominate 
* Stabilize a currently unstable feature
    * Write a stabilization report (FIXME: should link to a template here)
* Active project: deliver an update to the lang team
* Lang team -> project: request an update for the monthly meeting

[Project Proposal]: https://lang-team.rust-lang.org/proposing_a_project.html

# Path to membership

## First steps

* Lang team:
    * Get involved in some subteam or specific effort
* Lang/Shepherds:
    * Hang out in the shepherds zulip channel? unclear
* Triage:
    * Participate in meetings?
    * Hang out in triage Zulip channel, volunteer, do triage work on GitHub
* Project groups:
    * Propose a project 
    * Join a project that is being led by others
* Docs team:
    * Unclear <!-- Lokathor: Participate in Rust Reference PRs -->

## Key point

* Membership in shepherd or triage group is also an "end state"
* For many, that's the right role -- some folks will be both shepherds and participate in the main team, etc
* To join main team, also expected to help lead some projects (possibly as liaison) and demonstrate

# Discussion questions

During the meeting, add questions here! Or put thoughts on partial answers!

* Some people enjoy coming up with large feature proposals and some people prefer finding nits and papercuts of the language and working through them. Are both these groups being served equally?
    * I think this impacts the 'project' and 'project group' procedures in particular --nikomatsakis
    * Interested: cramertj
* Can we offload enough work that we no longer feel chartering a project group inherently costs the lang team ongoing bandwidth (rather than one-off bandwidth)? Can we accept a project proposal with an expectation of reasonably *bounding* the amount of time?
    * Interested: josh
* Some language features are high level (e.g., error handling) and some are very low level (e.g., how do we expose control of the linker to users). Should the team divide these responsibilities more formally?
    * interested: pnkfelix, nikomatsakis
* Who is responsible for interacting with other teams in the Rust project?
    * For specific efforts, project groups can and should include members from intersecting teams. This will almost always include the compiler team but could include cargo, libs, etc depending on the nature of the project.
    * Beyond that, lang and shepherd each seem to have a role, maybe worth discussing and expanding on.
* How is lang team process improved upon? For example, if the RFC process needs improvement, who leads this effort?
    * Seems like a task for the lang team directly?
    * Interested: josh
* How can we make the triage meetings more efficient?
* Are there roles and skillsets missing from the list?
* Are there large items missing from the "user interface"?
* What are the reasons that people nominate issues/PRs/etc?
    * Interested: nikomatsakis
    * Can we make a template to get more specific information
    * Perhaps some of those reasons might be better served by another mechanism?
* What kind of SLA can we offer for items in the "user interface"?
    * (Is "none" an option?)
        * Would framing it as 'expectations' help? I don't really think it's an *SLA* per se
    * In what situations are people blocked on this?  Or hurt by waiting (bitrot, etc)?
    * Interested: scottmcm
* How can we offer better visibility into the status of active projects?
    * Probably worth holding a separate meeting
* Who might be good lead(s) for the shepherd, triage effort?
    * (Think on this and send me names, probably best not discussed in a recorded meeting)
* Can shepherds offer services?
    * Interested: nikomatsakis, cramertj
    * What would that look like?
    * Tag team someone to give advice, offer an opinion, answer questions?
* From which subteam(s) will we draw project liaisons from? Shepherds mention "liaises with the greater community", but that doesn't seem the same as the project liaison role. Or are we folding that into "projects and project groups"
    * Interested: josh, nikomatsakis
* What are the primary goals of having shepherds as an official role? Recognition? Helping community members find folks to talk to? Or do they have more official capacity? What are the expectations of someone fulfilling a shepherd role? Would shepherds still be assigned to specific RFCs / topics?
    * Interested: cramertj
* Which of the maintenance tasks are making the most work? Are those tasks providing sufficient value to motivate the amount of work they require?
    * Are people getting high value out of the recorded meetings?
    * Interested: nikomatsakis, josh, pnkfelix, scottmcm
* From which subteam(s) do you intend for us to draw project liaisons from? 
    * Shepherds mention "liaises with the greater community", but that doesn't seem the same as the project liaison role.

* pnkfelix: So who (from which (sub)team) is doing "maintenance tasks" (video uploads, meeting summaries, etc)? Some of that is entirely mechanical; but writing *good* summaries may require someone with somewhat in depth knowledge and/or investment/interest in meeting topic.
    * Interested: nikomatsakis, pnkfelix, cramertj

* mark: what are our concrete goals for immediate improvements? what are the most painful points of today's experience?
    * if we could change one thing today, what would that be? more people? a change in process?
    * (pnkfelix is interested, or at least wondering about ways to interpret Q...)
* How can we make it easier to close things without prejudice? 
    * interested: pnkfelix, scottmcm
    * The other side of that, is there a low-effort way to "straw poll" or something the lang team?  What's the minimum amount of effort that we can require of people to get a "it's useful to put more work into this" answer.
* Maybe some groups in this set can periodically revisit older things to see if they can be moved along?

## Discussion

### Which of the maintenance tasks are making the most work? Are those tasks providing sufficient value to motivate the amount of work they require?

* Are people getting high value out of the recorded meetings?
* Interested: nikomatsakis, josh, pnkfelix, scottmcm

* Before the (monthly) planning meeting: Niko would like to...
    * pinging groups to get updates
* After the (monthly) planning meeting: Niko has 1h scheduled to
    * write a blog post
    * create calendar events
    * leaves comments on issues
* Triage meeting
    * generating today's agenda (15 min)
        * does not include looking at I-Nominated issues
        * does not include pre-checking if there's a clear question/summary for the lang team
        * does not include ensuring that we get to tasks that have been overlooked 
        * providing summaries
    * uploading video, minutes, linking things (30min)
        * could be made better
* Triage process:
    * job is to set things up to make triage meeting as efficient as possible
        * up to and including a recommended course of action, if one can be assembled
        * sometimes even just "we did this, fyi"
    * would be nice if the process were that we were leaving comments on the issues
        * and these are scraped by a tool
    * share with compiler team prioritization

### Some language features are high level (e.g., error handling) and some are very low level (e.g., how do we expose control of the linker to users). Should the team divide these responsibilities more formally?

* Do we need formality for who is the expert for particular areas?
* Are there areas that we should not own that could be seen as "lang"? Possible items (not consensus yet):
    * Linkers
    * Macros, name resolution
        * "Things requiring deep compiler knowledge?"
    * Trait semantics
    * Formal semantics -- or lang *does* own this?
* Another angle of attack:
    * Some details or problems tend to come out during implementation
    * There is also the bigger question of like "should we do this" 
    * Some are "tactics" vs "strategy"
* Example from traits:
    * Will adding negation break the logic? (Tactics, requires deep knowledge)
    * vs
    * Is the convenience of implied bounds outweighed by the semver implications? (Strategy, lang team)
* Existing "expert" teams?
    * e.g., borrow checker? 
    * are they teams or something else?
    * is this currently "pull-based", not "push-based" -- does it happen when something comes up
* Pull-based (who do we ask) vs push-based (who is driving efforts in the area) or both?
* There are groups in github for things like LLVM "icebreakers" and Windows people -- is there something like that which would could let people opt-in for pings about trait rules or hygiene or \_\_\_\_\_\_\_?
* Want to be wary that we don't put too many expectations on the group of people, that can lead to collapse
    * i.e., if we try to create too much structure up front
* But if we call things a team, there is an expectation that I can get involved -- is that the case here?
    * Is there something to get involved *in*?
    * also, the expectation is also that there's a lot of value being assigned to being a team member, and a pull-based team makes it harder to set minimum expectations.
* Can and should we document additional lists of topic area experts?
