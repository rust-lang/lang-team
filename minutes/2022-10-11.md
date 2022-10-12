---
title: Triage meeting 2022-10-11
tags: triage-meeting
---

# T-lang meeting agenda

* Meeting date: 2022-10-11

## Attendance

* Team members: nikomatsakis, pnkfelix, scottmcm
* Others: nbdd0121

## Meeting roles

* Action item scribe:
* Note-taker: pnkfelix

## Scheduled meetings

Tomorrow: planning meeting

## Announcements or custom items

nothin'

## Action item review

* [Action items list](https://hackmd.io/gstfhtXYTHa3Jv-P_2RK7A)

## Pending lang team project proposals

None.

Niko closed these out and added existing tracking issues to the [project board](https://github.com/orgs/rust-lang/projects/16/views/1?layout=board).

## PRs on the lang-team repo
### "Note design constraints on hypothetical `DynSized`" lang-team#166

**Link:** https://github.com/rust-lang/lang-team/pull/166

* Niko read it. Left a few notes. Looks ready to merge.

### "Document membership criteria and expectations" lang-team#174

**Link:** https://github.com/rust-lang/lang-team/pull/174

* Felix needs to read this.

## RFCs waiting to be merged

None.




## Proposed FCPs

**Check your boxes!**
### "Stabilize raw-dylib for non-x86" rust#102793

- **Link:** https://github.com/rust-lang/rust/issues/102793
- [**Tracking Comment**](https://github.com/rust-lang/rust/issues/102793#issuecomment-1272151461): 
	> Team member @joshtriplett has proposed to merge this. The next step is review by the rest of the tagged team members:
	> 
	> * [ ] @cramertj
	> * [x] @joshtriplett
	> * [ ] @nikomatsakis
	> * [ ] @pnkfelix
	> * [ ] @scottmcm
	> 
	> No concerns currently listed.
	> 
	> Once a majority of reviewers approve (and at most 2 approvals are outstanding), this will enter its final comment period. If you spot a major issue that hasn't been raised at any point in this process, please speak up!
	> 
	> See [this document](https://github.com/rust-lang/rfcbot-rs/blob/master/README.md) for info about what commands tagged team members can give me.
- [**Initiating Comment**](https://github.com/rust-lang/rust/issues/102793#issuecomment-1272151453): 
	> @rfcbot merge

* related to [RFC 2627: #[link(kind="raw-dylib")]`](https://github.com/rust-lang/rfcs/pull/2627)
* folks should take a look

### "make unaligned_reference a hard error" rust#102513

**Link:** https://github.com/rust-lang/rust/pull/102513

 * already future-incompat. Its time to make it a hard-error.

## Active FCPs
### "Elaborate supertrait bounds when triggering `unused_must_use` on `impl Trait`" rust#102287

**Link:** https://github.com/rust-lang/rust/pull/102287

### "Document membership criteria and expectations" lang-team#174

**Link:** https://github.com/rust-lang/lang-team/pull/174



## P-critical issues
### "`as` cast of non-`Copy` enum is no longer a move" rust#102389

**Link:** https://github.com/rust-lang/rust/issues/102389

* added T-compiler
* group agrees: lang thinks current behavior is wrong and should be fixed.
* left comment and removed I-lang-nominated tag

## Nominated RFCs, PRs and issues discussed this meeting

(none yet, move things from the section below as they are discussed)

### "PhantomData is unsound" rust#102810

**Link:** https://github.com/rust-lang/rust/issues/102810

pnkfelix and nikomatsakis wonder if this occurs for newtypes too?

https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=19ef7d261704f445208dd7b69c60282d

This does not compile:

```rust
use core::cell::Cell;
struct PDOrBox<T> {
    m: T,
    // m: PhantomData<T>,
}
struct Foo<'a> {
  selfref: Cell<Option<&'a Foo<'a>>>,
}
impl<'a> Drop for Foo<'a> {
  fn drop(&mut self) {
  }
}
fn make_selfref<'a>(x: &'a PDOrBox<Foo<'a>>) {}
fn make_pdorbox<'a>() -> PDOrBox<Foo<'a>> {
    unimplemented!()
}
fn main() {
  let x = make_pdorbox();
  make_selfref(&x);
}
```

Not related to `Box`, as using `struct PDOrBox<T>(T);` does the same as `Box`.

But `type PDOrBox<T> = [T; 0];` works like the `PhantomData`.


* Adding use of `x` at end, does that show that this is not unsound?

https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=d89770c74b6f9a073b4fd2d6cce0d279

* Interesting point
  `let x: PhantomData<T> = ...;` 
  has an `x` that is `Copy`; it will not have drop glue.
  * But if you put it into something that *is* non-copy, then you will observe the desired failure to compile here (right?) 
     * [yes](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=2c6e1a61081f093bab2b882e9ea676ae)

Conclusion:

* PhantomData is copy, and therefore cannot require drop.
* If something requires drop (for some other reason), and it contains a `PhantomData<T>`, the `Drop` is assumed to access data of type `T`.

Niko to leave comment and seek feedback from OP.

### "RFC: Field projection" rfcs#3318

**Link:** https://github.com/rust-lang/rfcs/pull/3318

Nominated by Josh, wait for him.

### "Support upcasting of `dyn Trait` values" rfcs#3324

**Link:** https://github.com/rust-lang/rfcs/pull/3324

* Niko nominated to draw attention; it was just filed, but the topic has been previously discussed (with general sentiment being positive).

### "Support #[global_allocator] without the allocator shim" rust#86844

**Link:** https://github.com/rust-lang/rust/pull/86844

 * Rust-for-Linux project needs this

[Description](https://github.com/rust-lang/rust/pull/86844#issuecomment-894814365)

 * What *is* the insta-stable change of concern here?
     * Before this change, linking two rlibs together required rustc
     * After this change, it is possible to link two rlibs together without using rustc
* Motivation
    * RfL doesn't want to compile each drive to a static lib
        * that would mean everyone gets their own libcore

Observation:

There's not a lot of documentation. We should have an RFC or something we can point people at.

scottmcm: This should be documented with the `#[global_allocator]` attribute, is that documented in the std docs?

Looks like it's mentioned in <https://doc.rust-lang.org/std/prelude/v1/macro.global_allocator.html>, so having the PR include documentation there sounds reasonable.

nbdd0121: I think the question is whether you want to support Rust-for-Linux case of compiling each rlib and using `--emit=obj`, and then linking them together separately.

pnkfelix: But this PR does more than that, right? It affects dylibs? I admit that blocking on dylibs, which have problems of their own, doesn't seem right, but still it's a concern.

nikomatsakis: I think this warrants an RFC.

nikomatsakis: specifically, RFCs make sense when you want to remember why you did it the way you did. Doesn't have to be an RFC, could be a write-up, but an RFC is an easy way to do it.

scottmcm: when you're creating an interface for other people.

nbdd0121: seems good to figure out how it interacts with dylibs.



### "Introduce a no-op FakeRead for `let _ =`." rust#102256

**Link:** https://github.com/rust-lang/rust/pull/102256

From [previous week](https://hackmd.io/bvRI45nqS6iWi3XztIw9jA#%E2%80%9CIntroduce-a-no-op-FakeRead-for-let-_-%E2%80%9D-rust102256)...

compiles today...

```rust
let mut x = 3;
let y = &mut x;
let _ = x;
drop(y);
```

...as does this (which is why we put the rules how they are)...

```rust
let mut x = 3;
let y = &mut x;
|| { let _ = x; }; // b/c closure does not capture `x`
drop(y);
```

* intentional setup: closure captures should not inject a change in behavior here.

Crater confirms it's breaking borrowck: https://crater-reports.s3.amazonaws.com/pr-102256-1/try%2321ddd5e0c16e62c1970e7c19fea4e5d6107d122d/reg/munge-0.3.0/log.txt
```text=
[INFO] [stdout] error[E0716]: temporary value dropped while borrowed
[INFO] [stdout]    --> src/lib.rs:331:18
[INFO] [stdout]     |
[INFO] [stdout] 331 |         b = &mut MaybeUninit::uninit();
[INFO] [stdout]     |                  ^^^^^^^^^^^^^^^^^^^^^- temporary value is freed at the end of this statement
[INFO] [stdout]     |                  |
[INFO] [stdout]     |                  creates a temporary which is freed while still in use
[INFO] [stdout] 332 |         let _ = a;
[INFO] [stdout] 333 |         let _ = b;
[INFO] [stdout]     |             - borrow later used here
[INFO] [stdout]     |
[INFO] [stdout] help: consider using a `let` binding to create a longer lived value
[INFO] [stdout]     |
[INFO] [stdout] 331 ~         let binding = MaybeUninit::uninit();
[INFO] [stdout] 332 ~         b = &mut binding;
[INFO] [stdout]     |
```

scottmcm: That NLL error feels wrong to me, since "used" pointing at `_` feels wrong.

another question is would this test report an error? it may be accepted even with the PR, not sure. --nikomatsakis

```rust
union Foo {
    f: u32
}
let mut x: Foo = Foo { x: 3 };
|| {
    unsafe {
        let _ = x.f;
    }
}
```

* pnkfelix: note difference in attitude re `_` from RalfJ (vs Niko above) here: https://github.com/rust-lang/miri/issues/2360

* possible from gary/scott?: just say that 
```
let _ = *ptr;
```
is not UB in RalfJ's example from miri#2360

* pnkfelix: what about;

```
*ptr;
```
then?


* scott points out in response: this works:
```
    let x = "hello".to_string();
    let _ = x;
    let y = x;
```
This fails:
```
    let x = "hello".to_string();
    x;
    let y = x;
```
<https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=a1174672c379f22377ab1ddf700fe73e>
Because the `x;` is a read that's "thrown away by the `;`", but the `let _ = x;` isn't a read.


## Nominated RFCs, PRs and issues NOT discussed this meeting

### "impl DispatchFromDyn for Cell and UnsafeCell" rust#97373

**Link:** https://github.com/rust-lang/rust/pull/97373
