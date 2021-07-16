# "Frequently asked questions" about initiatives

## Does the initiative owner make decisions?

The initiative owner drafts the proposed design and takes feedback from the liaison and team about what direction to take. This feedback can take the form of "you need to do X", but typically it is more about "you need to address this scenario". Or, put another way, if the initiative owner doesn't like the proposed direction, it's up to them to find an alternative that they do like which meets those same constraints, or to argue why the constraints are not necessary.

Note that serving as initiative owner is a high level of responsibility and may not be a good "starting place" for involvement within the Rust project. In practice, initiative owners should be experienced enough that they could mentor others to do the implementation work. If you don't know the language or system well enough to do that, then you probably are not ready to be an owner -- but you may be ready to be mentored by the owner!

## Is the word of a lang team member law?

Of course not. Well, ok, sometimes. For the most part, initiative owners are encouraged to treat lang team members like any other member of the community -- this implies a lot of respect for their opinions, since they are experienced, knowledgeable people, but the initiative owner still ultimately owns the design and should use their own judgement about what things to recommend. However, lang team members do have the option of adding constraints that must be met, and they can override the initiative owner if necessary. That is typically done by raising the concern with the rest of the team/leads in a more formal way.

## What happens if an owner stops working on things?

Initiative owners are often volunteers and may have changes in priorities or find they don't have as much time as they thought they did. In that case, they can simply step back. The liaison can then either find a new initiative owner, or perhaps assume initiative owner duties themselves but find a new liaison. If they are not able to do that, the initiative will be closed as "paused".

## What if we decide a initiative is a bad idea?

Sometimes, in the course of trying to design a initiative, we decide it was the wrong direction. That's ok! At any point the liaison can decide that the initiative isn't working out and close it. However, in doing so, they should document WHY they feel it did not work out -- and identify potential conditions where the idea may make sense later on. This documentation will typically take the form of a [design note] in the lang-team repository.

If there are concerns about this, those concerns can be raised with the lang team leads.

Closed initiatives will be removed from the project board and the code for them will be removed from the compiler.

[design note]: ../design_notes.md
