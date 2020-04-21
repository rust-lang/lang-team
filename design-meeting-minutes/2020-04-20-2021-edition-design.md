# 2021 Edition Design Meeting

* [recording](https://www.youtube.com/watch?v=uDbs_1LXqus)

Changes prepping for the edition should all start materializing by October it
seems, based on comments throughout the meeting

## Implied bounds

Apparently there are API compatibility hazards here, where if implied bounds
are depended upon by an external crate removing a bound from the struct would
be a breaking change.

This can be mitigated a few ways, either restrict implied bounds to only the
crate, make implied bounds be publicly exportable, or maybe some fancy stuff
based on usage that I didn't totally understand.

## Specialization

Not much backwards incompatibility so not very relevant to edition planning.

There's talk of making it eventually opt-out by default, but it doesn't seem
likely that this would happen short term, so specializable impls must be marked
`default` initially.

Also they may initially stabilize the `min_specialization` subset being used in
the compiler right now which I think only allows specializing for concrete
types, not for things like `T where T: Trait`.

## if let or exit function

They talked about implementing an inverse fallible binding for making a binding
or exiting early that might require a small  backwards incompatible change that
doesn't affect any known code.

The main syntax proposal seems to be

let Some(x) = y else {
    return Err(BrokenThingsAlert);
}

println!("{}", x);

This also ties into or patterns, they might be looking for ppl motivated to
work on this

## Unsafe in Unsafe Function

They're working on making the bodies of unsafe fns safe by default, but due to
the size of the change they're considering breaking this up across multiple
editions I think.

They're definitely planning on handling this with `rustfix` but its unclear how
well they can do this, so its unclear but it might be that in 2021 its only a
warning if you use an unsafe expr in an unsafe fn without an unsafe block.

## Early return with error

Assuming that a keyword is picked in time (presumably before october) the plan
is to reserve that keyword as part of the next edition.

## Impl Trait

Seems to be progressing nicely, there are some discrepancies between the RFC
and the implementation that need to be reviewed.

This will apparently help with async fns, tho not the ones with GATs, but niko
seemed to think that will be done by 2021 as well maybeee ??

## Implicit modules

Explicit modules can be confusing and lead to confusing errors that arise due
to lacking a `mod foo` statement, even for experienced users. However, last
year we did hit some real challenges with removing them, both because of
disturbing users' existing workflows and technical limitations associated with
scanning directories on networked file systems and the like.

Conclusion was that we're not really inclined to "re-open" this issue -- we did
a lot of work on modules and we want to move onto other areas at the moment.
However, as a simple step, we would be interested in adding a lint that warns
about missing `mod foo` statements and improving error messages as a starting
point (because the lint can be disabled, or disabled by default, that would
address the problems around networked file systems).

## Crate visibility

We held off on stabilizing `crate fn` because of concerns around a parser
ambiguity. We'd like to revisit that, it seems like a relatively small change
and it's still definitely a "nice to have". We do have to investigate the
options for resolving the ambiguity, which had to do with types like `struct
Foo(crate ::bar)`. (There is a listing of some of the options in the paper
doc...)

## Type Ascription (and Named Arguments)

They're particularly worried about not conflicting with future named arguments
syntax, so it seems like they might deprecate some syntax to allow named args
to use `=` as a separator so type ascription can safely use `:`
