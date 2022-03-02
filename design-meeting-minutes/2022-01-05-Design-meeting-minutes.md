---
title: Planning meeting 2022-01-05
tags: planning-meeting
---

# T-lang planning meeting agenda

* Meeting date: 2022-01-05

## Attendance

* Team members: nikomatsakis, pnkfelix, Josh
* Others: simulacrum

## Meeting roles

* Action item scribe: 
* Note-taker: nikomatsakis

## Proposed meetings
- Backlog bonanza
    - Jan 19
- "lang team check-in" [lang-team#128](https://github.com/rust-lang/lang-team/issues/128) 
    - last meeting had some areas for follow-up
    - to make a next meeting useful
    - Jan 26

## Things that will happen async

- niko and simulacrum to work async to dig up lint policy from edition guide

## Other potential meeting topics

- pnkfelix: [enum issue (lang-team#134)](https://github.com/rust-lang/lang-team/issues/134)
    - mark: does this need a meeting?
    - josh: I think it probably just needs a clear vision, could be done with a doc
    - nikomatsakis: I'm concerned this won't happen
    - pnkfelix: what do we lose if we get the doc done early, circulate it, and decide to cancel the meeting?
    - nikomatsakis: I think that's unlikely to occur in time, but sure
    - nikomatsakis: this is kind of a new-ish process for us, I think it will work better
- nikomatsakis: dyn + async things
    - lots of design work that needs to be communicated, is a meeting the right way to do it?

## Deferred meetings

### "Non-terminal divergence between parser and macro matcher " [lang-team#111](https://github.com/rust-lang/lang-team/issues/111)

### ""things other teams want"" [lang-team#130](https://github.com/rust-lang/lang-team/issues/130) 

- seems worth doing, important, but not urgent

### "Lint policy" [lang-team#132](https://github.com/rust-lang/lang-team/issues/132) 

- Unlikely to be ready this January
- February?
- Need an owner-- niko wants to, but doubts he'll have time, my guess is Ryan is super busy
- Josh is interested in contributing/collaborating
- simulacrum: my recollection is that edition work kind of produced a doc and it is 90% of what you want you need need for a meeting
    - if you have an idea of where to get that doc from, I feel like if I had a start, I might have enough time to polsh it
- nikomatsakis: I can find the doc and send it to you and we can discuss from there

### dyn upcasting

- nikomatsakis: I want to stabilize dyn upcasting, I will revamp the doc and josh and I can discuss
- nikomatsakis: working on a project that is very painful without this feature, want to push it through

## Calendar

- 2022-01-12 - enum repr/discr (owner: Felix)
- 2022-01-19 - backlog bonanza (Felix is out)
- 2022-01-26 - lang-team check-in (owner: Niko/Josh)

## Active initiatives

no updates this time

## Status updates

* nikomatsakis: let's talk -- why aren't they happening? should we give up?
* pnkfelix: goal was to have status updates about all the things at every planning
* nikomatsakis: all the active initiatives, distinct from "all the tracking issues that exist"
* cramertj: I have pinged the one person actively working on and they said "they didn't have time"
    * nikomatsakis: in my mind, "no progress" is a perfectly valid update
* pnkfelix: compiler's team's model is to rotate through getting some updates every week from a different group
    * not that it works great
    * but having the focus on 1 or 2 things for the week
    * aided by the compiler team having the active group building the agenda
    * they pre-build agenda and post it in prep
    * if team meeting arrives and people haven't said anything...
        * we do tend to get things written
* nikomatsakis: I do like that. I had forgotten about that model
* simulacrum: what is the goal for status updates?
    * compiler team initiatives are predom. focused on impl work, not so much design work, not an expectation of people giving guidance
    * lang team intent as I understand it was that you give status updates on the decisions so that lang team roughly says "sounds right" 
* nikomatsakis: I think that's interesting
* pnkfelix: if info flow is coming from the domain up to the whole, cadence doesn't matter so much, if there's no reverse feedback, having a more frequent cadence doesn't matter so much
    * I know we wanted to avoid talking about cadence, compiler team model means more projects get less synchronization
* nikomatsakis: two goals
    * to identify conflicts earlier
    * to be of service to the initiative, identify places where they are struggling and where we might HELP
        * "oh, yes, now that you say it, I do think we could have a meeting"
* simulacrum: now that you say it, these groups do give an opportunity to advertise, give kudos
* pnkfelix: aided by the zulip driven meeting
* nikomatsakis: in my ideal world, people show up, we chat, there's a blog post, and it gets discussed on reddit or whatever
* nikomatsakis: it seems like having focus on 1 group may serve our goals better
    * more exciting, more thoughtful feedback
    * yes there is a bottleneck but ...
* simulacrum: I don't think we have that many truly active initiatives in any case
* pnkfelix: nonetheless, we have to make the updates more frequent than monthly
    * if we are focused on indiv teams 
    * if you want them at the triage team
* nikomatsakis: what if we have a "spotlighted" group, but we also ping others to give them a chance in case there's a question
* josh: so kind of a "required" one that comes every time 
* pnkfelix: compiler team we have 2x each week, partly because often 1 team has not made progress, 
* mark: we have roughly 10 in lang
* cramertj: if we do it in the triage meeting
* so plan is
    * open the triage meeting with a spotlighted initiative
    * ping the liason the week before to tell them to talk to the owner and write an update with a summary, active questions etc
        * @Liaison, can you talk to @Owner and prepare an update for next week's triage meeting? Here is a template. (Topic subject "Status update for YYYY-MM-DD lang meeting")
        * in Zulip stream for project.
    * prior to planning meetings
        * reminder that we have a planning meeting, if you have active questions that could use a design meeting, or have smaller topics that might be good to have discussed, 
* discussion about how to automate this
* simulacrum: if after no status updates twice in a row, we mark it as inactive? Doesn't mean anything but if it's inactive for long enough.
* nikomatsakis: Maybe just after 6 months we mark it as inactive and you have to at least ping to get a liaison reassigned?
* simulacrum: If we say, "hey after N months of being inactive ..."
* cramertj: Relatedly, what's our policy on cookie licking?
* pnkfelix: COOKIES?
* simulacrum: shall we pick someone for next week? I can do it for never type!
* *general admiration abounds*

