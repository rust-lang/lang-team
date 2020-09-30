# Declarative macro repetition counts

## Summary and problem statement

Add new syntax to allow declarative macro authors to easily access the count
or index of declarative macro repetitions.

## Prioritization

This project fits in the "Targeted ergonomic wins and extensions" category.

## Motivation, use-cases, and solution sketches

Macros with repetitions often expand to code that needs to know or could
benefit from knowing how many repetitions there are, or which repetition is
currently being expanded.  Consider the standard sample macro to create a
vector, recreating the standard library `vec!` macro:

```
macro_rules! myvec {
    ($($value:expr),* $(,)?) => {
        {
            let mut v = Vec::new();
            $(
                v.push($value);
            )*
            v
        }
    };
}
```

This would be more efficient if it could use `Vec::with_capacity` to
preallocate the vector with the correct length.  However, there is no standard
facility in declarative macros to achieve this.

There are various ways to work around this limitation.  Some common approaches
that users take are listed below, along with some of their drawbacks.

### Use recursion

Use a recursive macro to calculate the length.

```
macro_rules! count_exprs {
    () => {0usize};
    ($head:expr, $($tail:expr,)*) => {1usize + count_exprs!($($tail,)*)};
}

macro_rules! myvec {
    ($(value:expr),* $(,)?) => {
        {
            let size = count_exprs!($($value,)*);
            let mut v = Vec::with_capacity(size);
            $(
                v.push($value);
            )*
            v
        }
    };
}
```

Whilst this is among the first approaches that a novice macro programmer
might take, it is also the worst performing.  It rapidly hits the recursion
limit, and if the recursion limit is raised, it takes more than 25 seconds to
compile a sequence of 2,000 items.  Sequences of 10,000 items can crash
the compiler with a stack overflow.

### Generate a sum of 1s

This example is courtesy of @dtolnay.
Create a macro expansion that results in an expression like `0 + 1 + ... + 1`.
There are various ways to do this, but one example is:

```
macro_rules! myvec {
    ( $( $value:expr ),* $(,)? ) => {
        {
            let size = 0 { $( + { stringify!(value); 1 } ) };
            let mut v = Vec::with_capacity(size);
            $(
                v.push($value);
            )*
            v
        }
    };
}
```

This performs better than recursion, however large numbers of items still
cause problems.  It takes nearly 4 seconds to compile a sequence of 2,000
items.  Sequences of 10,000 items can still crash the compiler with a stack
overflow.

### Generate a slice and take its length

This example is taken from
[https://danielkeep.github.io/tlborm/book/blk-counting.html].  Create a macro
expansion that results in a slice of the form `[(), (), ... ()]` and take its
length.

```
macro_rules! replace_expr {
    ($_t:tt $sub:expr) => {$sub};
}

macro_rules! myvec {
    ( $( $value:expr ),* $(,)? ) => {
        {
            let size = <[()]>::len(&[$(replace_expr!(($value) ())),*]);
            let mut v = Vec::with_capacity(size);
            $(
                v.push($value);
            )*
            v
        }
    };
}
```

This is more efficient, taking less than 2 seconds to compile 2,000 items,
and just over 6 seconds to compile 10,000 items.

### Discoverability

Just considering the performance comparisons misses the point.  While we
can work around these limitations with carefully crafted macros, for a
developer unfamiliar with the subtleties of macro expansions it is hard
to discover which is the most efficient way.

Furthermore, whichever method is used, code readability is harmed by the
convoluted expressions involved.

### Proposal

The compiler already knows how many repetitions there are.  What is
missing is a way to obtain it.

We propose to add syntax to allow this to be expressed directly.

As an initial suggestion, we are considering `${function(...)}` where function
can be `count` or `index`.  For example:

```
macro_rules! myvec {
    ( $( $value:expr ),* $(,)? ) => {
        {
            let mut v = Vec::with_capacity(${count(value)});
            $(
                v.push($value);
            )*
            v
        }
    };
}
```

In the future this could be used for other things, but anything other than
the repetition count or index are **out of scope** for this project.

## Links and related work

[MCP issue](https://github.com/rust-lang/lang-team/issues/28)

## Initial people involved

Initial people: markbt, Lokathor

