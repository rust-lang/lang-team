# Eager drop design note

- Project proposal [rust-lang/lang-team#86](https://github.com/rust-lang/lang-team/issues/86)

## Observations

### Any attempt to make drop run more eagerly will have to take borrows into account

The original proposal was to use "CFG dead" as a criteria, but it's pretty clear that this will not work well. Example:

```rust=
{
    let x = String::new();
    let y = &x;
    // last use of x is here
    println!("{}", y);
    // but we use y here
    ...
}
```

Here, the fact that `y` (indirectly) uses `x` feels like an important thing to take into account.

### Some destructors can be run "at any time"...

Some destructors have very significant side-effects. The most notable example is dropping a lock guard.

Others correspond solely to "releasing resources": freeing memory is the most common example, but another might be replacing an entry in a table because you are done using it.

### ...but sometimes that significance is only known at the call site

However, it can be hard to know what is significant. For a lock guard, for example, if the lock is just being used to guard the data, then moving the lock release early is actually _desirable_, because you want to release the lock as soon as you are doing changing the data. But sometimes you have a `Mutex<()>`, in which case the lock has extra semantics. It's hard to know for sure.

### Smarter drop placement will mean that adding uses of a variable changes when its destructor runs

This is not necesarily a problem, but it's an obvious implication: right now, the drop always runs when we exit the scope, so adding further uses to a variable has no effect, but that would have to change. That could be surprising (e.g., adding a debug printout changes the time when a lock is released).

In contrast, if you add an early drop `drop(foo)` today, you get helpful error messages when you try to use it again.

In other words, it's useful to have the _destructor_ occurring at a known time (sometimes...).

### Today's drop rules are, however, a source of confusion

The fact that `let _ = foo` drops `foo` immediately is a known source of confusion, along with the lifetimes of temporaries in `match` statements and the like:

```rust=
match foo.lock().data.copy_out() {
    ...
} // lock released here!
```

The `let _guard = foo` pattern is probably what prople want, but it's not necessarily obvious to readers when the destructor runs.

`let guard = foo; ...; drop(guard);` has the advantage of explicitness, so does something like `foo.with(|guard| ...)`

### Clarity for unsafe code can be quite important

There are known footguns today with the timing of destructors and unsafe code. For example, `CString::new().as_ptr()` is a common thing people try to do that does not work. Eager destructors would enable more motion, which might exacerbate the problem.

## Alternatives

- Scoped methods
- let blocks
- "Defer" type constructs or scoped guard type constructs from other languages
  - Go
  - D
  - Python
