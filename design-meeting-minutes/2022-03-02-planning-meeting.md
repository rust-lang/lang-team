---
title: Planning meeting 2022-03-02
tags: planning-meeting
---

# T-lang planning meeting agenda

* Meeting date: 2022-03-02

## Attendance

* Team members: nikomatsakis, Felix, Josh, Scott
* Others: simulacrum, Jane, lcnr, Mara, Jakob

## Meeting roles

* Action item scribe:
* Note-taker:

## Annoucements

* Niko + Josh getting close to a lang roadmap draft
    * [draft](https://hackmd.io/JGhj3CFJSmuDvpiPxbWaTQ)
    * Desire to incorporate libs work into this roadmap as well
* Niko to attend ETAPS 2022 in Munich for https://sites.google.com/view/rustverify2022/home

## Availability

* Mar 9: pnkfelix not available
* Mar 16
* Mar 23
* Mar 30

## Proposed meetings

-  "Non-terminal divergence between parser and macro matcher " [lang-team#111](https://github.com/rust-lang/lang-team/issues/111)
-  "Lint policy" [lang-team#132](https://github.com/rust-lang/lang-team/issues/132)
    - nikomatsakis would like to see progress but doesn't feel ready to do the prep work
    - we have notes from before that we need to find first
    - https://hackmd.io/7YycPi5ORhiXbualfhaQRw -- Firefox AwesomeBar FTW!
-  "Plan for enum discrim, repr, and casting" [lang-team#134](https://github.com/rust-lang/lang-team/issues/134)
    - jswrenn was chasing this but he's busy with other things
-  "Astrologic chart (aka roadmap)" [lang-team#140](https://github.com/rust-lang/lang-team/issues/140)
    - scheduled for Mar 9 
-  "RPITIDT" -- return position impl trait in dyn trait
    - scheduled for Mar 23
- Backlog Bonanza!

### Plan

* Mar 9: Roadmap [lang-team#140](https://github.com/rust-lang/lang-team/issues/140)
    * Do async work in advance, and then incorporate feedback afterwards and publish.
* Mar 16: Backlog Bonanza!
* Mar 23: RPITIDT [lang-team#144](https://github.com/rust-lang/lang-team/issues/144)
* Mar 30: Lint policy [lang-team#132](https://github.com/rust-lang/lang-team/issues/132)
    * pnkfelix to write initial draft

### Closed during meeting

-  ""things other teams want"" [lang-team#130](https://github.com/rust-lang/lang-team/issues/130) 
    -  We should reach out to other teams, and only schedule this if a team has enough to make a meeting worthwhile.
-  "A path towards stable generic const expressions" [lang-team#138](https://github.com/rust-lang/lang-team/issues/138)
    -  lcnr: this probably doesn't make too much sense atm, currently taking a step back wrt const generics so there isn't anything there rn.
    -  nikomatsakis: should we close, lcnr?
    -  jup, will do
-  "impl trait overview" [lang-team#142](https://github.com/rust-lang/lang-team/issues/142)

## Pending lang team project proposals

Let's see if we can figure out what to do with these!

### Proposed changes:

- Can we sort these?
    - E.g. by stage
    - maybe print date
- What do we do when it is accepted
    - Open a tracking issue--
        - on a repo if it has one
        - else on lang-team repo 
        - tagged as T-lang C-initiative
        - add it to [this project board](https://github.com/orgs/rust-lang/projects/16/)
- Policy for when to move out of proposal stage
    - If there is a second, it can move forward, don't need to reach consensus
    - But we should log concerns and liaison (or owner) should document some plan for how they'll be addressed
        - run that by the person who raised concern and get feedback
- Can we make a better policy?
    - tag it with something after 1 month, give it 2 more weeks, then label again

### "Deprecate target_vendor" lang-team#102

**Link:** https://github.com/rust-lang/lang-team/issues/102

* scottmcm to second this

### "Async fundamentals initiative" lang-team#116

**Link:** https://github.com/rust-lang/lang-team/issues/116

* wanted a liaison
* maybe close and rename to async fn in traits

### "Attribute for trusted external static declarations" lang-team#118

**Link:** https://github.com/rust-lang/lang-team/issues/118

* niko has seconded it

### "Support platforms with size_t != uintptr_t" lang-team#125

**Link:** https://github.com/rust-lang/lang-team/issues/125

* Concern:
    * Although the text of this proposal indicates that we should prepare an assessment, the name of the initiative ("support platforms...") suggests a conclusion.
    * It feels like a `rustbot second` would be endorsing that we should support such platforms; it's not clear if it makes sense for someone who believes we shouldn't support such platforms to second this proposal

### "Positional Associated Types" lang-team#126

**Link:** https://github.com/rust-lang/lang-team/issues/126

* can be approved but should document plan for how we will address usability
    * "we will meet with researchers to develop experiments"

### "Heap allocations in constants" lang-team#129

**Link:** https://github.com/rust-lang/lang-team/issues/129

-> niko proposes to close

### "Interoperability With C++ Destruction Order" lang-team#135

**Link:** https://github.com/rust-lang/lang-team/issues/135

### "inner crates, aka multiple crates per file" lang-team#139

**Link:** https://github.com/rust-lang/lang-team/issues/139

### "allow construction of non-exhaustive structs when using functional update syntax" lang-team#143

**Link:** https://github.com/rust-lang/lang-team/issues/143















