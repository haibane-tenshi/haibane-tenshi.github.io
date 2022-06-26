+++
title = "Obscure Rust: reborrowing is a half-baked feature"
date = 2022-06-26

[taxonomies]
tags = ["rust", "reborrowing"]
+++

Let's start with simple motivating use case.

```rust
fn mutate(i: &mut u32) -> &mut u32 {
    *i += 1;
    i
}

fn mutate_twice(i: &mut u32) -> &mut u32 {
    mutate(i);
    mutate(i)
}
```

This innocuous example hides a huge can of worms behind apparent simplicity.

`&mut T` implements neither `Copy` nor `Clone` - and it makes sense, copying unique reference aliases it.
It means mutable references are move-only, so first call to `mutate` consumes `i`.
But then how does the second call to `mutate` works?!
The answer is in the title: reborrowing.

# Reborrowing

Let's take a slightly simpler example.

```rust
let mut num = 32_u32;

let a = &mut num;
let b: &mut _ = a;
*b += 1;
*a += 1;
```

To make this work, compiler inserts a reborrow on your behalf:

```rust
let b: &mut _ = &mut *a;
//              ^^^^^^
```

It seems like just a new reference is created, but in fact it does something much more interesting.
There can exist only one mutable borrow, and to achieve that parent borrow `a` is suspended,
then `b` becomes an active borrow in its stead.
There is no problem with us using `b` at this point as compiler knows that this is the only borrow that *can* be used:

```rust
let mut num = 32_u32;

let a = &mut num;
let b: &mut _ = a; // Create reborrow
*b += 1;           // `b` has all privileges, so we can use it
                   // `b` goes out of scope
*a += 1;           // It is OK to use `a` again
```

However, this works only as long as `a` is inactive.
If you try to use `a` while `b` is still alive...

```rust
let mut num = 32_u32;

let a = &mut num;
let b: &mut _ = a;
*a += 1; // Mutation order is changed
*b += 1; 
```

...you will be greeted with [angry compiler messages][playground:main1].

Last thing to mention, there is a number of places where [implicit borrows][reference:implicit_borrow]
can be inserted, but it can only happen when compiler is aware that a reference is expected.
For example, generic non-reference parameters are [not automatically reborrowed][playground3]:

```rust
fn take<T>(_: T) {}

let mut num = 3_usize;

let a = &mut num;
take(a);
take(a); // Uh-oh
```

But it is always possible to achieve this [manually][playground4].

[reference:implicit_borrow]: https://doc.rust-lang.org/stable/reference/expressions.html?highlight=implicit,borrows#implicit-borrows
[playground:main1]: https://play.rust-lang.org/?version=nightly&mode=debug&edition=2021&gist=1190c9555095a3fe8b245938c0a745a0
[playground3]: https://play.rust-lang.org/?version=nightly&mode=debug&edition=2021&gist=dd76f60c941e526083f92c6aa3478636
[playground4]: https://play.rust-lang.org/?version=nightly&mode=debug&edition=2021&gist=1c9c87689289cb39cbbccf37ebb98394

# Problems

That concludes the short prelude, let's talk business.
There are glaring issues with this model of reborrowing.
More specifically, there is one problem that was annoying me for the better part of last few months:

> Reborrowing cannot be expressed in type system.

I.e. we cannot abstract over it.

This may sound controversial.
People seemingly tried this [for years][stackoverflow],
there is even [a crate][docs.rs:reborrow] doing just that.
Is it all a lie?
Well, sort of.

[docs.rs:reborrow]: https://docs.rs/reborrow/latest/reborrow/
[stackoverflow]: https://stackoverflow.com/questions/43036156/how-can-i-reborrow-a-mutable-reference-without-passing-it-to-a-function

## Emulating reborrow

We will be talking only about mutable references from now on, as this is the interesting case.

Every attempt at expressing reborrow in type system boils down to writing a function
which transforms `&mut &mut T` into `&mut T`:

```rust
fn reborrow<'a, 'b, T>(r: &'a mut &'b mut T) -> &'a mut T {
    r
}
```

This should look familiar: `Clone::clone` is very similar in both structure and rationale.
We want to keep original reference intact, so we pass it by reference (resulting in double reference).
Also, outer reference needs to be mutable, so that compiler is able to infer uniqueness of resulting reference.

The cool part, [it works][playground:emu1]!

```rust
let mut num = 32_u32;

let mut a = &mut num;
let b: &mut _ = reborrow(&mut a);
*b += 1;
*a += 1;
```

Which gives a naughty feeling as if we slandered the type system by doing the impossible.
*Except*, this approach is imperfect.
Remember example at the start?
It [no longer compiles][playground:emu2]:

```rust
fn mutate_twice(mut i: &mut u32) -> &mut u32 {
    mutate(reborrow(&mut i));
    mutate(reborrow(&mut i))
}
```

:(

[playground:emu1]: https://play.rust-lang.org/?version=nightly&mode=debug&edition=2021&gist=3c0b652692a370b4186764424122991c
[playground:emu2]: https://play.rust-lang.org/?version=nightly&mode=debug&edition=2021&gist=a411b9c755033d2f296478eb49c2d227

# Alternative view

It isn't trivial to rationalize this difference in behavior,
so here's my personal mental model of lifetimes.
It doesn't pretend on anything, but it makes the problem we ran into above painfully visible.

Every lifetime is a link between data (place in memory) and
referent (shared/mutable reference) which grants access to it.
We can imagine that lifetime is combination of *two* scopes, not one:

* *liveness* scope - which delimits how long data exists, and
* *referential* scope - which delimits how long referent can be used.

The separation is surprisingly clean.
Every scope has its own responsibility and those are orthogonal to each other.

Purpose of *liveness* scope is to ensure that we access alive objects,
i.e. to prevent use-after-free and uninitialized memory access.
The scope is property of data and is effectively immutable.
There is nothing you can do through a reference to shorten/lengthen lifetime of underlying data,
and Rust prevents us from dropping the data prematurely as long as any referents exist.

Purpose of *referential* scope is to ensure aliasing rules.
Unfortunately, simply scoping references is not enough, so we get complicated interaction rules and
this is the reason for [Stacked Borrows][github:stacked_borrows] to exist.
One of the most important properties of *referential* scopes is ability to be narrowed
to allow multiple references to coexist.

It is also plain to see that those two scopes barely interact.
Since liveness scope is immutable it is just carried around and inherited by derived lifetimes,
and referential scope can contract or even expand at will - as long as it stays within liveness scope.

Here's an interesting observation: in current Rust those two scopes are almost always identical.
And, that's right, reborrowing is the reason for the *almost* part.
Reborrowing creates a new reference which keeps liveness scope intact but has narrower referential scope.
It allows lifetime to properly interact with aliasing analysis, without losing information about 
how long data actually lives. 
This operation is notably unique.
No other language feature allows us to separate the two scopes.

Inventing new questionable syntax, if `'l` is liveness scope, `'r` is referential scope and
`'l+'r` is combined actual lifetime, we can express true reborrowing as:

```rust
fn true_reborrow<'l, 'r0, 'r1, T>(&'l+'r1 mut &'l+'r0 mut T) -> &'l+'r1 mut T;
```

As opposed to emulation attempt we did earlier:

```rust
fn reborrow<'l, 'r0, 'r1, T>(&'l+'r1 mut &'l+'r0 mut T) -> &'r1+'r1 mut T;
//                                                          ^^^
//                                 liveness information is lost!
```

[github:stacked_borrows]: https://github.com/rust-lang/unsafe-code-guidelines/blob/master/wip/stacked-borrows.md

## Scope expansion

If it did end there, the model wouldn't be particularly useful.
However, it supports somewhat unnatural operation: referential scope expansion.

Let's annotate original example:

```rust
fn mutate_twice(i: &mut u32) -> &mut u32 { 
    // i belongs to scope '0
    // return type is expected to be the same
    
    mutate(i);         // reborrow for scope '1
    let r = mutate(i); // reborrow for scope '2
                       // r inherits the scope '2
    
    r                  // we return reference with scope '2
                       // as if it belongs to scope '0! 
}
```

The last line is interesting.
`r` is reference with associated lifetime `'l+'2`, but function is expected to return reference with `'l+'0`!
We implicitly expanded the referential scope to fit function signature.

How valid is such operation?

Both references share *liveness* scope, so no memory-related problems to be had here.
From perspective of *referential* scopes, as long as aliasing rules are not broken everything is fine.
We can see that original `i` is no longer used, so this definitely holds.
You can imagine that `r` assumed `'0` scope by rendering `i` inaccessible.

Summarizing, referential scope narrowing is *reversible*.
This is where the emulation attempt fails spectacularly.
If we look at the pseudo-signature another time...

```rust
fn reborrow<'l, 'r0, 'r1, T>(&'l+'r1 mut &'l+'r0 mut T) -> &'r1+'r1 mut T;
```

...recovering original reference would mean converting `'r1+'r1` lifetime into `'l+'r1`,
but it is considered taboo by compiler!
Expanding liveness scope certainly leads to memory problems, so it is flat forbidden.

## Implications

My biggest gripe with current state of reborrowing in that it is so tightly coupled to references.
**Reborrowing is about manipulating lifetime scopes to assist aliasing analysis**,
it isn't just a gimmick to sneakily manage mutable references!
This is a fundamental operation inside type system and should be treated as such.

The coupling spells a lot of trouble.
Custom types parametrized on lifetimes cannot be reborrowed.
Given type...

```rust
struct MyRef<'a>(&'a mut ());
```

...the only way to reborrow one is to destructure, reborrow inner reference and reconstruct the value,
provided you have access to innards.
And you cannot even move operation outside current function:
reborrow must be used from the same scope where it was created.
Nightmare fuel.

It gets worse yet.
If the type is hidden behind generic parameter `T` it is just over.
There is *nothing* you can do to tap into true reborrowing.
Emulation while cute is sorely inadequate as a replacement.

`unsafe` is maybe heralded as an escape hatch, but it fails miserably to resolve the situation
because converting references to pointers erases all lifetime information.
This is not what we want.

And this is exactly the edge case I ran into.

Also, as a tangent, I wonder if this is a part of why lifetimes are confusing to newcomers.
We have two different meanings and semantics jumbled into single object.
Teaching resources often treat it as one or the other depending on situation, which doesn't add clarity.

# Conclusions

I don't know what conclusions to make here.
It is obvious that 99.9% of Rust ecosystem doesn't care about this.
Trying to fix lifetimes considering backwards compatibility and all related work is maddeningly complicated.
Unfortunately, I have hard time imagining contexts work in practice without solving this issue.
I also wonder what other potential applications are inhibited by current situation.

Anyways, have a good day, and see you next time!
