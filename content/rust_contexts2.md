+++
title = "Futuristic Rust: context emulation, part 2"
date = 2022-07-14

[taxonomies]
tags = ["rust", "contexts"]
+++

In [previous article](@/rust_contexts.md) we devised a context emulation method which supports access
to capabilities through shared references.
It's time to significantly improve the approach.

<!-- more -->

At some point while writing this post even I - with my unyielding love for extremely long longposts -
had to admit that it got out of hand.
There are too many details to fit on a page.
So instead you can find all of that in [the github repo][gh:rust_context_emulation].
This article is intended as high level overview of the approach.
Documentation is minimalistic, so I recommend you to look into examples first.
Those illustrate important use cases and serve as a test ground.
Feel free to clone and play with it.

Also, you can skip right to [conclusions](#conclusions) if you are only interested in TL;DR.

# Goals

So, before we start let's talk a little about what we want to achieve.

1. One of the biggest issues with previous attempt is
    contextual functions being out of touch with existing `Fn*` traits design.
    In particular, it requires late-bound types, which is more of speculative feature right now,
    also it breaks implicit assumptions around `Fn*` trait implementations.

    This has long-reaching unfortunate implications when interacting with other language parts.
    Poor interaction with traits especially stands out. 

2. Another exploration vector is better integration with Rust ownership semantics.
    It would be nice to move past shared references to mutable references or even taking capabilities by value.

3. There is desire to expand on definition of capabilities.
    So far we assumed that capability values have a single concrete type.
    Relaxing this requirement might be interesting, for ex. allowing generic capabilities.

[gh:rust_context_emulation]: https://github.com/haibane-tenshi/rust_context_emulation

# Representation

## Improving on context design

To avoid confusion, I expanded on terminology a bit.

* *Context* is a general name for a collection of capabilities a given function receives.

* `Store` is an actual type(s) which used to this purpose.

### True nature of stores
 
`Store` is a container with the following properties:

* It can hold values of different types (capability wrappers in our case), i.e. is heterogeneous.
* At most one value of each type is allowed (due to shadowing/replacement).
* Presence of individual values can be detected via traits at compile time.

It doesn't take much difficulty to put it all together: context is just *a statically-typed `anymap`*.

`anymap` container maps *type* `T` to a single *value* of `T`.
It is a relatively niche container type.
There are [articles][blog:anymap] theorizing its existence
as well as already working and widely used [implementations][docs.rs:anymap].
Still, considering the state of type manipulation in Rust,
building a *statically-typed* version sounds insane.
A perfect job for us.

[blog:anymap]: https://www.jakobmeier.ch/blogging/Untapped-Rust.html#section-1-a-heterogenous-collection-of-singletons
[docs.rs:anymap]: https://docs.rs/anymap/latest/anymap/

### Building blocks

To start we need to understand how "normal", dynamic anymap works.
The core idea is simple: if we can assign a unique hash to every type ([`TypeId`][std:type_id] could work),
we can use that hash to map onto a type-erased value of that type,
for example with the help of `HashMap<TypeId, Box<dyn Any>>`.
Because types serve as "keys", recovering actual value is a straightforward [`Any::downcast_ref`][std:any].
The problem for us is how to lift it to compile-time.

Let's impose a constraint: **every capability must be declared in code before being used**.
It doesn't really introduce a limitation, as this is an implicit assumption behind most contexts visions anyways.
However, it allows us to know all capabilities that can ever appear in the program.
This also means we already know all acceptable keys for the map, 
so it is possible to build a [perfect hash map][wiki:perfect_hash]!
It is easy to put the rest together:

* Context itself can be just a tuple.
* Every capability is assigned a (unique) index within that tuple.
* We can lift getters/setters/whatever into traits and
    implement them based on exact type residing in every position.

Let's do it!

[std:type_id]: https://doc.rust-lang.org/std/any/struct.TypeId.html
[std:any]: https://doc.rust-lang.org/std/any/trait.Any.html#method.downcast_ref
[wiki:perfect_hash]: https://en.wikipedia.org/wiki/Perfect_hash_function

## Traits: basic tuple ops

For starters, we need to abstract over tuple operations.
The following two traits summarize it:

```rust
trait Get<N: Indexed> {
    type Output;

    fn get(&self) -> &Self::Output;
    fn get_mut(&mut self) -> &mut Self::Output;
}

trait Put<T, N: Indexed> {
    type Output;

    fn put(self, t: T) -> Self::Output;
}
```

They *do* have to be traits: `get`'s output type depends on index and `put` changes type of container altogether. 

The tuple itself looks like this:

```rust
struct Tuple<T0, T1, T2, T3>(T0, T1, T2, T3);
```

We will have only 4 slots, because why not.
You can choose any number you want, it only limits how many capabilities you can put in.
Ideally, we would like it to be variadic, but the feature is far away, so a big enough number will do.

Implementation of `Get` and `Put` is straightforward.
The only mystery is `Indexed` trait:

```rust
struct Index<const N: usize>;

mod private {
    pub trait Sealed {}
    impl<const N: usize> Sealed for super::Index<N> {}
}

trait Indexed: private::Sealed {}
impl<const N: usize> Indexed for Index<N> {}
```

`Index` is just wrapper for usize value, and `Indexed` is a marker trait with standard seal -
to ensure we don't accidentally pass something wrong instead.

Normally, we should be able to use indices directly, but it is difficult at the moment.
Passing associated `const` items as const generic parameters is not stable and requires `generic_const_exprs` feature.
Currently, the feature is incomplete, and I ran into a number of bugs.
The numbers are not used in any way, so we replace them with marker types instead.

Summing it up, `Get` and `Put` traits transform tuple into... umm, heterogeneous array?
The resulting container is generic: it can hold anything and doesn't have any context-specific behaviour yet.
But before we continue we need to take a look at handles.

## Handles

A secondary goal of this exploration is providing support for mutable references.
Unfortunately this is not nearly as simple as it seems and takes us to a dark corner of the language.

The biggest challenge mutable references face is that they are move-only.
This property when taken for face value is very restrictive, for example the following snippet...

```rust
fn foo(_: &mut u32) {}

fn bar(i: &mut u32) {
    foo(i);
    foo(i);
}
```

...should not compile, contrary to [expectations](https://play.rust-lang.org/?version=nightly&mode=debug&edition=2021&gist=ac5aca26a4d39439c07a299b92048f2b).
Rust has a special mechanism for resolving this: reborrowing.
And we will need to tap into it to allow similar use cases for capabilities.
Unfortunately, we cannot express real reborrowing in the type system, so we will have to settle for an emulation.

This brings us to `HandleRef` and `HandleMut` which is just a wrapper around shared/mutable reference:

```rust
pub struct Handle<T>(T);

pub type HandleRef<'a, T> = Handle<&'a T>;
pub type HandleMut<'a, T> = Handle<&'a mut T>;
```

## Capabilities

How do we assign indices to capabilities?
With traits again:

```rust
pub trait Capability {
    type Index: Indexed;
    type Inner: ?Sized;

    fn as_ref(&self) -> &Self::Inner;
    fn as_mut(&mut self) -> &mut Self::Inner;
}
```

The interesting bit is `Index` type, the rest is just implementation helpers.
Setting indices is our responsibility since there is no way to automatically generate them.

The core idea of capability representation remains the same.

```rust
capability interner: Interner;
```

...expands into...

```rust
struct __interner(Interner);

impl Capability for __interner {
    type Index = Index<1>; // Index is picked manually.
    type Inner = Interner;
    
    // ...
}
```

## `CxFn*` family core  

Shape of `CxFn*` trait is key to this approach.

Observations:
* Every contextual function can always define a minimal set of capabilities required to execute it.
    There is never a reason to pass in anything beyond that.
* Because all possible contexts are represented by one generic container, 
    every set of capabilities (along with their access mode) can be associated with single unique `Store` type.

In other words, contextual functions don't need to be generic over contexts,
they only need to be executable with single specific context type!

This is the payoff for going through the trouble of working out representation of `Store`:
we can now define function context as an associated type on `CxFn*` traits:

```rust
pub trait CxFnOnce<Args> {
    type Output;
    type Context;

    fn cx_call_once(self, args: Args, cx: Self::Context) -> Self::Output;
}

pub trait CxFnMut<Args>: CxFnOnce<Args> {
    fn cx_call_mut(&mut self, args: Args, cx: Self::Context) -> Self::Output;
}

pub trait CxFn<Args>: CxFnMut<Args> {
    fn cx_call(&self, args: Args, cx: Self::Context) -> Self::Output;
}
```

These traits are already functional, although incomplete.
Context may contain "free" lifetimes that are not attached to anything within the trait,
which causes compiler to rightfully complain.
There are other design concerns too, so we will come back to resolve this issue in a moment.

## Finishing `Store`

Time to finish store construction.

I should mention, we don't store `Handle`'s directly.
Instead, we wrap them in special `Some` type:

```rust
pub struct Some<T>(pub T);
pub struct None;
```

Yes, this is just a type-level option.
`None` represents an absent capability.
After writing so much Rust it is almost unexpected
how easily things become messy when you don't have familiar vocabulary types to reach for.
I had to rewrite internals two times before realizing that simple `Option` is what I truly needed.

I won't mention them much past this point: ops implementations for them are about what you expect. 

### Store ops

Finally, the fun part.

`Push` trait allows us to manipulate the shape of context:

```rust
trait Push<CRef> {
    type Output;

    fn push(self, t: CRef) -> Self::Output;
}
```

`Store` implements `Push<&'a C>` and `Push<&'a mut C>` where `C: Capability`.

`ExtractRef` and `ExtractMut` traits to acquire references from context:

```rust
pub trait ExtractRef<'a, T>
where
    T: ?Sized + 'a,
{
    fn extract_ref(&self) -> &'a T;
}

pub trait ExtractMut<'a, T>
where
    T: ?Sized + 'a,
{
    fn extract_mut<'s>(&'s mut self) -> &'s mut T
    where
        'a: 's;
}
```

Note the distinction between those two.
`ExtractRef` is able to perfectly replicate original lifetime of capability, but this is not possible with `ExtractMut`.
Instead, it has to rebind resulting reference to lifetime of `self`.

This is actually a serious problem.
After you extracted a mutable reference, store becomes untouchable as long as that reference exists.
Which means we cannot call other functions nor extract more references from it!

This is not the first time such problem manifests.
For example, you cannot get mutable references to two different elements of `Vec`, at least using naive methods.
There do exist ways to work around, for example `Vec` can use [`[T]::split_at_mut`][std:slice_split_at_mut] and friends.
However, even though I feel it should be possible, I struggled to make similar API work in our case.
There are certainly GAT and HRTB limitations that make it difficult.
Let me know if you have better luck.

[std:slice_split_at_mut]: https://doc.rust-lang.org/std/vec/struct.Vec.html#method.split_at_mut

### General purpose ops

The rest of operations on `Store` are implemented by delegating them to `Handle`'s

#### Reborrowing

`HandleMut`s cannot be cloned, so reborrowing provides a replacement.
True reborrowing is out of our reach, but we can emulate it:

```rust
pub trait Reborrow {
    type Output<'local>
    where
        Self: 'local;

    fn reborrow(&mut self) -> Self::Output<'_>;
}
```

I have to concede a point here: this emulation actually replicates the behavior of true reborrowing in the current
Rust.
However, be careful to not repeat my [embarrassing mistakes](@/rust_reborrowing.md#emulating-reborrow).
Take a look at implementation on `HandleMut`:

```rust
impl<'a, T> Reborrow for HandleMut<'a, T>
where
    T: ?Sized + 'a,
{
    type Output<'s> = HandleMut<'s, T> where Self: 's;

    fn reborrow(&mut self) -> Self::Output<'_> {
        Handle(&mut *self.0)
    }
}
```

Resulting reference is scoped to lifetime of `&mut self`.
Compiler is very insistent about this point.
However, since we pass stores by value, this reference is certain to be *local*,
which means we can never return reborrowed mutable references from functions!

This sounds like a difficult situation, but in fact it isn't.
Returning reference from a function requires that reference to be scoped to whole function,
which in turn implies this is the last use of such reference.
In other words, there is no need to reborrow in first place.
At least, this is how it works with *current* borrow checker.

Importantly, the above logic works *only* for mutable references.
So, in fact, we have to implement reborrowing differently for shared references:

```rust
impl<'a, T> Reborrow for HandleRef<'a, T>
where
    T: ?Sized + 'a,
{
    type Output<'s> = Self where Self: 's;

    fn reborrow(&mut self) -> Self::Output<'_> {
        Handle(self.0)
    }
}
```

For them, reborrow is a simple `Copy`.

#### Unification

The next trouble to take care of is context unification.
Purpose of unification is to take two different contexts and produce a combination.
This is commonly required when a function calls multiple other functions or accesses any capabilities alongside that.

The trait itself is not descriptive:

```rust
pub trait UnifyOp<T>: Sized {
    type Output;
}
```

The magic happens in implementations.

Situation with `Store` is simple: types are unified slot-by-slot by delegating to `Handle`s.
And in case of multiple `Store`s participating it is done sequentially one-by-one.

Unifying two handles is much more interesting.
Every handle has three disjoint parts: lifetime, reference kind (shared/mutable) and type.
Under the assumption that both handles refer to the same data, we can do it piecewise:

* Lifetimes can only be unified when they are identical, i.e. `'a = 'b`.
* Reference kind is either `Shared` or `Mutable`.
    Unifying any kind with mutable gives us mutable, and otherwise it is shared.
* `T` and `U` can only be unified when `T = U`.

If handles refer to different data, then unification should fail.
We cannot guarantee that, but it isn't a problem.
This can only occur if you mistakenly assign the same index/context slot to two different capabilities.

I should also mention that rule about `T = U` can be relaxed, we will talk about it in section about 
[generic capabilities](#generic).

#### Coercion

Purpose of coercion is to undo unification.
It is very important piece of puzzle since context must be shaped individually for every function.

For now let's limit ourselves to transformations reversing proposed unification:

* `HandleMut -> HandleMut`
* `HandleRef -> HandleRef`
* `HandleMut -> HandleRef`
* `Some<T> -> None`

The first three correspond to transformations between different kinds of references,
the last one removes capability from store. 

This is done through coercion trait:

```rust
pub trait Coerce<Other> {
    fn coerce(self) -> Other;
}
```

## Advanced capabilities

### Dynamically sized

Something I didn't consider last time is wrapping a DST type.

```rust
capability alloc: dyn Allocator;
```

This is possible and provides a decent way to define overridable functionality:

```rust
struct __alloc_helper<T: ?Sized>(T);
type __alloc = __alloc_helper<dyn Allocator>;

impl Capability for __alloc {
    type Index = Index<0>;
    type Inner = dyn Allocator;

    // ...
}
```

And it can be instantiated like following:

```rust
let alloc: &__alloc = &__alloc_helper(std::alloc::Global);
```

Helper type is required to provide unsizing coercion.
See [relevant Rustonomicon pages][rustonomicon:dst].

[rustonomicon:dst]: https://doc.rust-lang.org/nomicon/exotic-sizes.html#dynamically-sized-types-dsts 

### Generic

It isn't difficult to extend what we did with DST to full-blown generics.
However, generic capabilities have a nasty problem: at first glance they completely ruin unification.
Given two handles `HandleRef<'_, T>` and `HandleRef<'_, V>` how does one even produce type `U` to unify `T` and `V`
when `T != V`?!

#### Coercion

It is actually easier to work the problem backwards.
Instead, let's ask, which types `U` can be safely converted to a given type `T`?

We can scramble a few examples.
`[T; N]` and `Vec<T>` can be coerced to `[T]` so it sounds OK.
It is also probably fine to convert `R: Read` to `dyn Read`.
Also, maybe `Box<T>` to `T`, since we access it behind reference, so as any other `Deref` or `DerefMut`.
Oh, also `T` may contain lifetimes, so subtyping can be relevant too...

Now, this starts to suspiciously resemble [coercion page][reference:coercion] in Rust reference.
And for a good reason: we literally are trying to list every possibility when one type can be
*implicitly* converted into another.
If reading that page induces a headache, just know, you are not alone.

Also, hey, `&mut T` to `&T` is listed as a special case of coercion, hopefully this explains name choice for the op.

Alright, to the bad news.
Abstracting over all of this is just impossible.
With help of specialization, we can cover certain bits,
but things like subtyping and `dyn` coercion are hardwired into compiler and is out of our reach.
This is *a lot* more restrictive than it sounds.
It can potentially be worked around in specific examples by implementing `Coerce` trait on capability wrappers,
but it isn't implemented at the moment.

[reference:coercion]: https://doc.rust-lang.org/stable/reference/type-coercions.html?highlight=coercion#coercion-types

#### Unification

One nice thing about generics in capabilities is that they naturally leak into function type.
If `bar` uses `alloc<A>`, `alloc<A>` will make it into `bar::Context`
and compiler naturally will ask where that `A` came from, to which you can have two answers:

* make current function generic over `A`
* specify a concrete type, like `Global`

This is almost a trick: we *don't need to produce* type `U` as unification of `T` and `V`,
compiler makes sure it is always provided by user!
The real question is when and under which conditions we can unify `T` into `U`.
Which leads to three situations:

1. Both `U` and `T` are generic.

    This is the simplest case.
    No conversions, both generics refer to the same type.
    As a nice bonus, any number of different `T`s can be unified in such fashion.
2. `U` is generic, but `T` is a concrete type.

    To make it work compiler needs to ensure that it is possible to coerce every `U` to `T`.
    It can be trivial (e.g. `A: Allocator` to `dyn Allocator`), but might ask user to specify some extra bounds on `U`.

    This case can result in trouble. 
    When you try to coerce to multiple `T`s it might be possible to produce combination of coercions 
    that is impossible to satisfy.
3. Both `U` and `T` is a concrete type.

    Again, the requirement here is that `U` is coercible to `T`.
    On the bright side, unlike 2nd case we always either succeed or get a compiler error.
    There is no way to produce uncallable functions.

Obviously, case when `U` is concrete, but `T` is generic is impossible.
We cannot ensure that coercing concrete type to any arbitrary one always works.
Solution to this case is trivial: specify `T = U`.

Generally speaking, unification in both 1st and 2nd case is always trivially achievable.
However, there are problems waiting to happen on the boundary where generic type is replaced by concrete one,
so this part might require extra work.

# Desugaring into GATs

## `CxFn*` traits

As I mentioned when introducing [newer `CxFn*` traits](#cxfn-family-core),
they are not complete, and it is time to finish them.
In this implementation we will use a GAT-based approach.
This is the first one I fully developed, and it is the one which started this whole journey.
There is another promising lead, but right now let's focus on what is in front of us.

```rust
pub trait CxFnOnce<Args> {
    type Output<'_0, '_1, '_2, '_3>;
    type Context<'_0, '_1, '_2, '_3>;

    fn cx_call_once<'_0, '_1, '_2, '_3>(
        self,
        args: Args,
        cx: Self::Context<'_0, '_1, '_2, '_3>,
    ) -> Self::Output<'_0, '_1, '_2, '_3>;
}

pub trait CxFnMut<Args>: CxFnOnce<Args> {
    fn cx_call_mut<'_0, '_1, '_2, '_3>(
        &mut self,
        args: Args,
        cx: Self::Context<'_0, '_1, '_2, '_3>,
    ) -> Self::Output<'_0, '_1, '_2, '_3>;
}

pub trait CxFn<Args>: CxFnMut<Args> {
    fn cx_call<'_0, '_1, '_2, '_3>(
        &self,
        args: Args,
        cx: Self::Context<'_0, '_1, '_2, '_3>,
    ) -> Self::Output<'_0, '_1, '_2, '_3>;
}
```

Looks quite... abominable, actually.

Lifetime count should be explainable by our `Store` only holding up to 4 capabilities.
It is not surprising to see `Context` being a GAT, but `Output` is probably unexpected.
Let's dive deep into that right away.

## Lifetimes

One of the important things we need to sort out is how contextual function handle late-bound lifetimes.
Unlike in normal functions, a lifetime can appear in one of three places: arguments, output or context.
This leads to four interesting situations:

* lifetime appears only in context,
* in context and arguments,
* in context and output,
* in all three.

The latter is a straightforward combination of previous two so we can skip it,
which leaves us with three situations to study.

And before continuing I recommend reading my [earlier post](@/rust_early_and_late_bound_generics.md)
as a refresher on this topic.

### Late-bound lifetime in context

`Context` is an associated type, so we cannot just put a lifetime in:

```rust
impl<'a> CxFnOnce<()> for f {
    type Context = &'a str; // Nope!
    
    // ...
}
```

This is required for type system soundness.
Traditionally, there are three ways to solve it:

* Put lifetime into `Self` type.
    We cannot do that, because it renders lifetime early-bound.
* Put lifetime into trait.
    It implies that lifetime is part of arguments, and we claimed it is not.
* Constrain lifetime in `where` block.
    By current rules late-bound lifetime must be universally qualified, so this option is also out.

This is where GAT comes in.
It allows us to introduce a lifetime, while keeping good appearances:

```rust
impl CxFnOnce<()> for is_empty {
    // Simplified to reduce noise.
    type Context<'_0, '_1, '_2, '_3> = &'_0 __hidden_str;

    // ...
}
```

### Late-bound lifetime in context and output

Situation gets worse when `Output` is involved.
Trying to put lifetime into output puts us basically under the same conditions as in the precious case. 
This is the reason for `Output` to be GAT as well:

```rust
impl CxFnOnce<()> for first_char {
    type Output<'_0, '_1, '_2, '_3> = &'_0 str;
    type Context<'_0, '_1, '_2, '_3> = &'_0 __hidden_str;

    fn cx_call_once<'_0, '_1, '_2, '_3>(
        self,
        args: (),
        cx: Self::Context<'_0, '_1, '_2, '_3>,
    ) -> Self::Output<'_0, '_1, '_2, '_3> {
        // ...
    }
}
```

Capability lifetime sharing is encoded in a roundabout way:
those are defined on `cx_call_once` method and then passed into `Context` and `Output`.

This is certainly my least favorite part about this approach.

### Late-bound lifetime in context and argument

Because such lifetime appears in trait, it is equivalent to "normal" late-bound lifetime:

```rust
impl<'s> CxFnOnce<(&'s str,)> for compare_to {
    // Oversimplified to reduce noise.
    type Context<'_0, '_1, '_2, '_3> = &'s __hidden_str;
}
```

Extra lifetimes from GAT can simply be ignored.
Boring.

### Insights

So what are the takeaways?

* This design is fully compatible with late-bound lifetimes.

* The only reason why we need GATs is to potentially support object safety.
    If this is not a concern, the complication can be removed.
    Personally, I don't think that abandoning it is a good idea, but it is an option.
    There could be situations where GATs are in the way of exploring designs,
    however at this point we might as well use a [different desugaring](#desugaring-into-input-lifetimes).

* An extension from second point, we need GATs in `Output` only to support sharing late-bound lifetimes
    between it and `Context`.
    But, again, I don't think it is worth to abandon.
    This case is likely to be quite prevalent.

* While we saved references to capabilities,
    any other lifetime embedded into capability itself is doomed to be early-bound.
    However, this is to be expected.
    To construct `&'cap cap<'a>` we need `'a: 'cap` bound, but spelling it renders both lifetimes early-bound.

## fn: `&capability`

Time to look at function desugarings.

```rust
fn len()
with data_store: &Vec<u32>
{
    data_store.len()
}
```

(Actually, after working with generic capabilities 
I kind of consider it a good idea to always spell out expected capability type.)

Expands into this mostrosity:

```rust
impl CxFnOnce<()> for len {
    type Output<'_0, '_1, '_2, '_3> = usize;
    type Context<'_0, '_1, '_2, '_3> =
        MakeContext<(Select<'_0, '_1, '_2, '_3, Shared, __data_store>,)>;

    fn cx_call_once<'_0, '_1, '_2, '_3>(
        mut self,
        (): (),
        cx: Self::Context<'_0, '_1, '_2, '_3>,
    ) -> Self::Output<'_0, '_1, '_2, '_3> {
        let data_store = cx.extract_ref::<__data_store>();
        data_store.len()
    }
}
```

`MakeContext` jumbles together capability references into proper `Store` type.
`Select`'s job is to pick lifetime which corresponds to `__data_store` capability.
We will talk about reasons for this complication when we get to traits.
The rest should look self-explanatory.

I will again omit implementations of `CxFnOnce` and `CxFnMut` in examples to reduce verbosity.

## fn: `&mut capability`

Mutable case looks very similar.

```rust
fn push(value: u32)
with data_store: &mut Vec<u32>
{
    data_store.push(value)
}
```

expands into

```rust
impl CxFnOnce<(u32,)> for push {
    type Output<'_0, '_1, '_2, '_3> = ();
    type Context<'_0, '_1, '_2, '_3> =
        MakeContext<(Select<'_0, '_1, '_2, '_3, Mutable, __data_store>,)>;

    fn cx_call_once<'_0, '_1, '_2, '_3>(
        mut self,
        (value,): (u32,),
        cx: Self::Context<'_0, '_1, '_2, '_3>,
    ) -> Self::Output<'_0, '_1, '_2, '_3> {
        let data_store = cx.extract_mut::<__data_store>();
        data_store.push(value);
    }
}
```

Note that our tricks about putting mutable references away doesn't absolve us from responsibility to uphold Rust rules.
In particular, we cannot keep local mutable references while calling any function
which can touch capabilities behind those.
The restriction already works in normal code, so we should be expected to follow.

## Traits

### Desugaring methods

Let's figure out how to reduce trait methods to something we can work with.

Observation: every function has a type which implements relevant `Fn*` traits and the type is what matters to us.
More than that every function type is also ZST, so function objects can be spawned from thin air with no harm.
Following this thought, it is possible to replace functions with associated types:

```rust
trait Archiver {
    fn add_record<T>(&mut self, t: T);
    fn len(&self) -> usize;
}
```

replaced by

```rust
trait Archiver {
    type add_record<T>: Default + Fn(&mut Self, T);
    type len: Default + Fn(&Self) -> usize;
}
```

There is no way to encode ZST requirement, but `Default` gives us ability to spawn function objects at will.
Close enough.
Also, GATs are required to accurately model generic parameters, but it isn't a problem for us.

### Desugaring contextual methods

It is easy to apply the technique to contextual functions.

```rust
trait MyPush {
    // This method is specified with restricted context.
    type concrete_push: Default
        + for<'_0, '_1, '_2, '_3> CxFn<
            (u32,),
            Output<'_0, '_1, '_2, '_3> = (),
            Context<'_0, '_1, '_2, '_3> = MakeContext<(
                Select<'_0, '_1, '_2, '_3, Mutable, __data_store>,
            )>,
        >;

    // This method is specified with wildcard context.
    type wildcard_push: Default
        + for<'_0, '_1, '_2, '_3, 'a> CxFn<
            (&'a Self, u32),
            Output<'_0, '_1, '_2, '_3> = ()
        >;
}
```

And it immediately brings us to an option.
We can either fully specify the context, restricting it in trait definition, or leave it unspecified.
We call unspecified context *a wildcard*.
Such context is defined by trait implementor, it allows them to request any set of capabilities
(and it can be different between different implementors).

It also makes sense that wildcard contexts are most desirable.
Often, trait implementors are going to be in other crate, far away from trait definition,
possibly using capabilities that trait itself have no information about.
Considering how popular trait usage is in Rust, without wildcard contexts the feature can never succeed.
So, we would like to ensure it works.

Wildcard contexts have a minor problem, however.
Imagine you hold on to `'alloc` lifetime, which slot of `Context<'_, '_, '_, '_>` do you
put it in?
This is important for trait user and trait implementor to agree in this instance on meaning of lifetimes,
otherwise compiler will get confused.
By extension this also must be true between all trait implementors, or even normal functions.

The easiest solution is to (globally) assign indices to capability lifetimes as well.
We already assigned indices to capabilities, so lifetimes can just reuse them. 
This is exactly what `Select` does: pick lifetime based on capability index and glue those two together.
Technically, we can get away by manually designating specific lifetime slots to belong to specific capabilities,
but it is simpler this way.

### Using contextual methods

Implementing trait is only half of a story.
Usage is where most of complexity resides.

Restricted contexts present no issue.
Because context is known at usage site, such a method is no different from plain function
at least with respect to type manipulation.

Wildcard contexts are another beast entirely.
Imagine the following setup:

```rust
impl<F, G> CxFn<(F, G)> for call_two<F, G>
where
    F: CxFn<(), Output = ()>,
    G: CxFn<(), Output = ()>,
        
    // Let's omit other bounds for brevity...
{
    fn cx_call(&self, (f, g): (F, G), mut cx: Self::Context) -> Self::Output {
        {
            let cx = cx.reborrow().coerce();
            f.cx_call((), cx)
        }

        {
            let cx = cx.coerce();
            g.cx_call((), cx)
        }
    }
} 
```

In order to make the function legal we need to convince compiler of three things:
`f`'s and `g`'s contexts can be unified,
and that unified context can be reborrowed and then coerced in this order.
All three should be true by convention.

We can type those bounds, `call_two` even compiles!
But, any attempt at calling it immediately fails.

Compiler at this point just gave up on giving intelligible errors.
To get a gist of what's going on let's zoom in on one capability.
For example, if `f` requests `&__hidden_str` capability and `g` doesn't, in particular it produces the following bound: 

```rust
struct FaultyBound
where
    for<'_0, 'local> <
                <(Some<HandleRef<'_0, __hidden_str>>, None) as Unified>::Output
            as Reborrow>::Output<'local>:
        Coerce<Some<HandleRef<'_0, __hidden_str>>>;
```

If you remember, `Reborrow::Output<'local>` has `where Self: 'local` clause, which is required for soundness.
Now it should be apparent what happened.
Combining those two bounds gives us `'_0: 'local`, but both lifetimes are universally qualified!
We need to add this clause to HRTB to make this legal.

There is a deeper issue still.
Mutable references in this situation reveal a weird edge.
Watch what happens if we replace `HandleRef` with `HandleMut`:

```rust
struct FaultyBound
where
    for<'_0, 'local> <
                <(Some<HandleMut<'_0, __hidden_str>>, None) as Unified>::Output
            as Reborrow>::Output<'local>:
        Coerce<Some<HandleMut<'???, __hidden_str>>>;
```

Reborrowing replaces `'_0` with `'local`, so for this bound to compile we need to insert `'local` in place of `'???`.
But rememeber, we are working with opaque context.
The only way to tweak expected lifetimes in such way is to spell something like
`<F as CxFnOnce<(), Output=()>::Context<'local, '_1, '_2, '_3>`.
But from inside `call_two` we don't know if such replacement is warranted!
For all we know `f` might want shared reference with the real lifetime instead.

You can witness the madness for yourself in [the corresponding example][gh:gat_widlcard01].

The most obvious next step is to get rid of this pesky reborrowing.
It implies abandoning support for mutable access as well, but we still have shared access.
Without mutable references we can consider all context to be `Copy`able.
And, lo and behold, [it works][gh:gat_wildcard03]!
At least something is going our way.

There are still a few things we can try to solve the two issues,
but those require [further modifications](#desugaring-into-hybrid) to the approach.

[gh:gat_widlcard01]: https://github.com/haibane-tenshi/rust_context_emulation/blob/main/examples/gat/wildcard01_with_reborrow.rs
[gh:gat_wildcard03]:https://github.com/haibane-tenshi/rust_context_emulation/blob/main/examples/gat/wildcard03_with_copy.rs

## Object safety

Strictly speaking, object safety of `CxFn*` traits is dependent on object safety of GATs, which is largely unexplored
at the moment.
However, since when "You cannot do that!" stop us?
We can emulate `dyn` objects too.

We will need a vtable:

```rust
type FixedContext<'_0, '_1, '_2, '_3> =
    MakeContext<(Select<'_0, '_1, '_2, '_3, Mutable, __counter>,)>;

struct CxFnVTable {
    pub cx_call: for<'_0, '_1, '_2, '_3> fn(
        *const (),                        // Self
        (),                               // Args
        FixedContext<'_0, '_1, '_2, '_3>, // Context
    ) -> (),
}
```

A pseudo-`dyn` object:

```rust
struct DynCxFn {
    vtable: CxFnVTable,
    this: *const (),
}
```

I will omit implementation of `CxFn*` traits as those are trivial.
Last, we need a way to transform normal functions:

```rust
impl DynCxFn {
    pub fn coerce_from<F>(f: &F) -> Self
    where
        F: for<'_0, '_1, '_2, '_3> CxFn<
            (),
            Output<'_0, '_1, '_2, '_3> = (),
            Context<'_0, '_1, '_2, '_3> = FixedContext<'_0, '_1, '_2, '_3>,
        >,
    {
        let cx_call = |this: *const (), args: (), cx: FixedContext<'_, '_, '_, '_>| -> () {
            unsafe {
                let this = &*(this as *const F);
                this.cx_call(args, cx)
            }
        };

        let vtable = CxFnVTable { cx_call };

        DynCxFn {
            vtable,
            this: f as *const F as *const (),
        }
    }
}
```

I know, it doesn't look very safe to use, but not much we can do.
Native `dyn` objects are unsized, however custom DSTs are far away.
It is good enough to use in a lab.

Does this function?
Actually, [yes][gh:gat_adv01]!

One nitpick that I have is that this approach is hard to generalize.
You have to produce a specialized version for each function signature.
I struggled to properly constrain `Output`'s lifetimes in generalization .
Not exactly sure what's at fault,
could be that we need proper HKTs for that.

Anyways, it works in this form, and I call this a win.

[gh:gat_adv01]: https://github.com/haibane-tenshi/rust_context_emulation/blob/main/examples/gat/adv01_dyn_cxfn.rs

## Design issues

First, most obviously, reliance on GATs.
Because GATs are central to this design we also inherit any limitations and issues,
whether existing or discovered in the future.
This isn't really a problem in itself, building new features on top of others is a natural way to develop language,
but it is something to be aware of.

Second, this design is clunky around edges.
You probably never have to interact with `Context`, but having GATs on `Output` associated type is especially bad.
Every programmer out there trying to type `my_fn::Output<'??, '??,...>` now have to think what are those lifetimes
and where they come from.
It also implies uncontrolled proliferation of HRTBs, which isn't something I welcome either.

Third, implementation of late-bound lifetimes is inconsistent.
Lifetimes that appear in arguments are defined on `impl` block

```rust
impl<'s> CxFn<(&'s str,)> for foo {
//   ^^    
    
    // ...
}
```

But the ones that don't are actually defined on `cx_call*` method

```rust
impl CxFn<()> for bar {
    fn cx_call<'_0, '_1, '_2, '_3>(
        //     ^^^^^^^^^^^^^^^^^^
        &self,
        (): (),
        cx: Self::Context<'_0, '_1, '_2, '_3>,
    ) -> Self::Output<'_0, '_1, '_2, '_3> {
        // ...
    }
}
```

This isn't immediately apparent, but it might present an issue if we ever desire to expand on definition of late-bound
lifetimes, for ex. allow bounds them.
The second group is built into definition of the trait, so we can never modify it.
Those lifetimes are forever universally qualified.

An additional concern about it is complexity.
Because source of late-bound lifetime can differ it adds extra difficulty to their handling.
This point definitely cannot be overlooked. 

# Desugaring into input lifetimes

## `CxFn*` traits

The other approach is to embed capability lifetimes directly into `CxFn*` traits:

```rust
pub trait CxFnOnce<'_0, '_1, '_2, '_3, Args> {
    type Output;
    type Context;

    fn cx_call_once(self, args: Args, cx: Self::Context) -> Self::Output;
}

pub trait CxFnMut<'_0, '_1, '_2, '_3, Args>: CxFnOnce<'_0, '_1, '_2, '_3, Args> {
    fn cx_call_mut(&mut self, args: Args, cx: Self::Context) -> Self::Output;
}

pub trait CxFn<'_0, '_1, '_2, '_3, Args>: CxFnMut<'_0, '_1, '_2, '_3, Args> {
    fn cx_call(&self, args: Args, cx: Self::Context) -> Self::Output;
}
```

Yeah, don't ask.
I have no idea how GATs came out first.

Since this post is too long already, I won't go into as much detail here.

## Comparison with GAT-based approach

In terms of feature set this approach is able to replicate everything that GAT-based one managed to achieve.
It doesn't address mutable references in wildcard contexts.

Key improvements:
* There is no bifurcation of late-bound lifetimes
* There is no GAT on `Output`

I view both as a serious issue, and it is nice to see that we can improve on them.
Another minor improvement is object safety.
Since there are no GATs anymore, traits are `dyn` safe by default.

Overall this is a pure win.

## Flavors of early-bound lifetimes

Yet, there is another distinction I have to mention.

There are two distinct ways you can implement early-bound lifetimes in this approach.

"Freeform" flavor:

```rust
impl<'_0, '_1, '_2, '_3, 'data_store> CxFn<'_0, '_1, '_2, '_3, ()>
    for freeform<'data_store>
{
    // ...
}
```

Or "connected" flavor:

```rust
impl<'data_store, '_1, '_2, '_3> CxFn<'data_store, '_1, '_2, '_3, ()>
    for connected<'data_store>
{
    // ...
}
```

The second flavor is unique to this approach.
GATs are locked only to the first one.

To be frank, I still have no idea which one is correct.
Obviously, the second flavor exposes a lifetime dependency to surrounding code,
so as long as the early-bound lifetime is exposed we have to account for and propagate this dependency.
However, so far I have not encountered any case where this actually presents a problem.

We can craft a situation
where it might be possible to observe difference in behavior by completely erasing the lifetime.
This implies:
* No direct access to function type, i.e. it is behind a generic `F` type
* Context must be wildcard, because capability lifetime is always part of it even when early-bound
* The lifetime must not appear in either arguments or output type

I [attempted that][gh_wildcard04].
It compiles when it should and doesn't compile when it shouldn't.
Beats me ¯\\_(ツ)\_/¯
The only distinction is what errors are produced.
The first flavor complains that unification and coercion bounds fail,
the second complains that `F`'s implementation is not general enough. 

If you ask, I put my money on second flavor with vague reasons like
it doesn't introduce unused late-bound lifetimes, and it propagates lifetime dependencies to caller,
which results in more sensible errors.
If you are able to clarify this contest, please do.

Despite this being said, examples in the repo almost exclusively use the first flavor.
The reasons are technical: it is way simpler to implement, and this is easier to keep it comparable to GATs. 

[gh_wildcard04]: https://github.com/haibane-tenshi/rust_context_emulation/blob/main/examples/input/wildcard04_early_bound_liftime_flavors.rs

# Desugaring into hybrid

Alright, let's tackle mutable references in wildcard contexts.
As we [previously figured](#using-contextual-methods), there are two issues:

* Lack of `where` clause in HRTB
* Incorrect shape of contexts used as coercion target

## `Reborrow2` trait

We are hardly the only people to run into the first issue.
With pending stabilization of GATs this has been raised a few times.
Sabrina Jewson nicely outlines it in [her post][sabrinajewson:gat_alternative],
and luckily for us, she also highlights a possible [solution][sabrinajewson:gat_alternative:solution].

We need to reshape `Reborrow` trait:

```rust
pub trait Reborrow2Lifetime<'this, ImplicitBound = &'this Self> {
    type Output;
}

pub trait Reborrow2: for<'this> Reborrow2Lifetime<'this> {
    fn reborrow(&mut self) -> <Self as Reborrow2Lifetime<'_>>::Output;
}
```

The only notable downside to this is lack of object safety,
but we never have to create trait objects out of this trait, so we don't care.

I don't know how to word it.
We came to a point where we emulate GATs in order to emulate reborrowing in order to emulate contexts.
I hope you appreciate how deep this goes.

[sabrinajewson:gat_alternative]: https://sabrinajewson.org/blog/the-better-alternative-to-lifetime-gats
[sabrinajewson:gat_alternative:solution]: https://sabrinajewson.org/blog/the-better-alternative-to-lifetime-gats#the-better-gats

## `CxFn*` traits

To help our coercion bounds we need to communicate to target context that 
expected mutable references' lifetimes need to be replaced by some `'local` lifetime.
This can be achieved by simply adding extra lifetime to definition of `CxFn*` traits:

```rust
pub trait CxFnOnce<'_0, '_1, '_2, '_3, Args> {
    type Output;
    type Context<'m>;

    fn cx_call_once(self, args: Args, cx: Self::Context<'_>) -> Self::Output;
}

pub trait CxFnMut<'_0, '_1, '_2, '_3, Args>: CxFnOnce<'_0, '_1, '_2, '_3, Args> {
    fn cx_call_mut(&mut self, args: Args, cx: Self::Context<'_>) -> Self::Output;
}

pub trait CxFn<'_0, '_1, '_2, '_3, Args>: CxFnMut<'_0, '_1, '_2, '_3, Args> {
    fn cx_call(&self, args: Args, cx: Self::Context<'_>) -> Self::Output;
}
```

Trait implementor is responsible to put `'m` on all mutable references.

There is definitely an option to embed it into the trait alongside of "real" lifetimes.
Combined with GAT-based approach it gives us 4 different varieties.
It is enough to check just one, we already know there isn't any meaningful difference between where to put lifetimes.

## Usage in wildcard contexts

So, what's the verdict?
Does it work?

Well, given two functions

```rust
impl<'_0, '_1, '_2, '_3> CxFnOnce<'_0, '_1, '_2, '_3, ()> for mutable_ref {
    type Output = ();
    type Context<'m> = MakeContext<(
        &'m mut __counter,
    )>;

    fn cx_call_once(self, args: (), cx: Self::Context<'_>) -> Self::Output {
        // ...
    }
}

impl<'_0, '_1, '_2, '_3> CxFnOnce<'_0, '_1, '_2, '_3, ()> for shared_ref {
    type Output = ();
    type Context<'m> = MakeContext<(
        Select<'_0, '_1, '_2, '_3, Shared, __counter>,
    )>;

    fn cx_call_once(self, args: (), cx: Self::Context<'_>) -> Self::Output {
        // ...
    }
} 
```

We can call both `call_two(shared_ref, shared_ref)` and `call_two(mutable_ref, mutable_ref)`.

Great success!

## More problems

Almost.

Unfortunately, `call_two(shared_ref, mutable_ref)` doesn't compile,
which makes very little sense.
How does *weakening* requirements on a function breaks it?

This time, error message is actually helpful, but let me replicate the bad part anyways.
This is coercion bound:

```rust
struct FaultyBound
where
    for<'_0, 'm, 'local> <
                <(
                    Some<HandleRef<'_0, __counter>>,
                    Some<HandleMut<'m, __counter>>
                ) as Unified>::Output
            as Reborrow2Lifetime<'local>>::Output:
        Coerce<Some<HandleRef<'_0, __counter>>>;
```

We already fail at unification step: lifetimes are expected to be equal!
Our initial argument behind lifetime unification was that all capability references within the same context refer
to the same data, so their lifetimes also must be the same.
But our reborrowing traits break this assumption!

Even if that wasn't an issue, and we manage to produce `HandleMut<'_0, __counter>` as result of unification,
we are still screwed.
After reborrowing it becomes `HandleMut<'local, __counter>` and original lifetime is lost.

There is something really subtle going on here.
To solve reborrowing limitation we need it to treat reference as shared while reborrowing.
It is as if we have to coerce first.
Now, if you actually think about it, this is how it is supposed to work in the first place:
we shape context into what other function expects and *only then* reborrow to "clone".
However, we are doing it the other way around.

There is a good reason for this.
Coercing can produce new types, so it must operate by value.
But we cannot move in mutable reference otherwise original context will lose it,
and the only way to make more of them is to reborrow before coercing.
We are chasing our own tail.

And even if we go the brute force way and simply combine two operations,
ultimately, what we are trying to express in this case is the following function:

```rust
fn weird<'local, 'a, T>(t: &'local mut &'a mut T) -> &'a T {
    t
}
```

Which [isn't even legal][playground1]!

I believe this is the wall.

[playground1]: https://play.rust-lang.org/?version=nightly&mode=debug&edition=2021&gist=ef43262b567aaa3f16b8f09b3dadc794

# Interaction with language

**`'static` capabilities**

Even though capabilities are expected to be local variables, there are ways to make them live for `'static` lifetime.
You can find practical applications for this.
For example, you will need a `alloc: &'static A` to put `Box<T, &'static A>` into actual static variable.

The simplest way to achieve this is to create a static variable and push reference to it into context.

Another way is to use a ZST.
Compiler treats shared references into fresh ZSTs as `'static` ones.
However, references into ZST are dangling and there is no point passing them around.
With help of specialization (or just [const generics][rust-forums:zst]?)
we might be able to push ZSTs directly into context, getting all zero-size benefits.
But then, affected types like `Box` will also be able to distinguish
them and just embed ZST value directly instead of (dangling) reference,
which sounds like a solid win.

A word of caution though.
This works only for shared references.
As soon as you produce `&'static mut T` all hell breaks loose.
There are [long-wound discussions][gh/rust:static_mut] around `static mut`, and I won't replicate it here.

[gh/rust:static_mut]: https://github.com/rust-lang/rust/issues/53639
[rust-forums:zst]:https://users.rust-lang.org/t/trait-bound-for-zst-zero-sized-types/61620

**Closures**

For closures everything is good.
There is an extra associated type involved, but functionally it doesn't require extra work from us.
I don't see any obstacles to making contextual closures.

The more interesting part, of course, is interaction with captures.
Just like the last time I would imagine closures capture context at creation site by default.
This provides us with reliable way to produce "pure" functions for various callbacks.
Because captured context is part of closure object,
it automatically becomes a subject to normal Rust's lifetime rules as well as auto-traits, like `Send` and `Sync`.
Another bright spot is that such context capture is also minimalistic 
since functions request only capabilities they need.

This capture mode doesn't interact well with mutable references to capabilities,
especially when wildcard contexts are involved.
This is a potential counterpoint to implementing mutable access.

**Threads**

For a normal thread there is no outer environment to provide and host capabilities,
so every thread is just a plain function.
There are two ways to create them:

* Set up capabilities locally
* Capture context on parent thread with help of closure

But if you take a look at `thread::spawn` method signature...

```rust
pub fn spawn<F, T>(f: F) -> JoinHandle<T>
where
    F: FnOnce() -> T,
    F: Send + 'static,
    T: Send + 'static;
```

...realistically it supports only the first method.
`F: 'static` is very restrictive.

The second mode is likely to be actively used by scoped threads (which got recently stabilized, nice!).
For them capturing references is actually sensible,
because they *do* have outer environment in the form of parent thread.  

**Extern ABI**

Non-Rust ABIs don't have contexts.
To call contextual function behind such API we need to convert it to a "pure" one,
which leads us to the same set of solutions as for threads.
Considering that closure execution outside of Rust sandbox is a risky endeavor,
we are mostly limited to the first method.

This can potentially be used to our benefit.
For example, panicking across FFI is UB, and currently you need to check it by hand.
But if every panic requires `panic_handler` capability (which is only set by `catch_unwind`),
every unhandled panic immediately turns into a compiler error!

(But I should say I don't know how well this setup integrates with existing panic mechanism.)

**Traits**

We already looked at how traits can be handled at mechanics level.

From the language perspective, we should expect trait methods to have wildcard contexts by default,
i.e. allow implementors to request whatever capabilities they want.
Concrete contexts in trait methods are still possible, but likely to be a minority.

This will also require additional syntax to replace wildcard contexts with concrete ones to
e.g. create trait objects.

**Futures/generators**

Futures are interesting in a sense that they can request capabilities in two places:
at creation point and in `Future::poll`.
This creates some interesting practical considerations,
for example is it worth capturing capability or just request it every time through `poll`?
And how do we reflect this difference in `async fn` syntax?

Also, future in this aspect behave more like scoped thread.
It presumably runs in an executor,
so it also has an outside environment (which can conveniently communicate capabilities through `Future::poll`).
This will likely require executors to expose control over what context stored futures are supposed to receive. 

# Conclusions

To be honest with you, I was *extremely* skeptical going into this part,
but the result looks almost too good to be true.

**Supported features**:

* Contextual functions are a superset of normal functions. There is no trait bifurcation.
* Only minimal (= required) set of capabilities is passed to each function.
* Usage in traits, including support for both concrete (requested capabilities are determined by trait definition)
    and wildcard (requested capabilities are determined by trait implementation) contexts.
* Ability to use functions with wildcard contexts in generic contexts.
* Ability to transparently propagate capability requirements (including received from wildcard contexts).
* Full support for access to capabilities through shared references.
* Partial support for access to capabilities through mutable references.

    Mutable references are fully usable in concrete contexts,
    but usage behind wildcard contexts is very difficult, courtesy of our reborrowing abstraction.

    Usage in wildcard contexts is flat impossible in both GAT- and input-based approaches.
    With some modifications to traits (see [hybrid](#desugaring-into-hybrid) approach)
    we can make it work, however mixing shared and mutable access (to the same capability) is still unachievable.
    I believe this to be a hard limitation.
    We will need a different way to represent reborrowing to proceed.
* Capabilities wrapping DST type.
* Type and lifetime generics on capabilities.
* Object safety.

    Object safety for other traits using contextual methods is based on object safety of `CxFn*` family,

    For GAT-based approach, `CxFn*` family object safety is deferred to GAT object safety.
    so certainly more work on GATs is required.
    Getting ahead of that, you *can* manually build a close approximation, which works.

    For input-based approach, object safety is given.

**Features, blocked behind language/compiler extensions**:

* Arbitrary number of capabilities.
    
    This is locked behind variadic types.
    We can sort-of work around when all contexts are transparent,
    but as soon as wildcard contexts are involved, variadic types are unavoidable.

    We can consider supporting only limited set of capabilities, e.g. only ones provided by `std`.
    However, I really don't like this option.
    There are plenty examples in ecosystem where capabilities seem like the right answer.

* Object safety in presence of arbitrary number of capabilities.

    To achieve this we need variadic lifetimes applied to `CxFn*` traits.
    This is way more demanding than it sounds.
    Rust doesn't allow us to manipulate lifetimes directly, the only thing we can do is to put it into a type.
    So, unless we see some improvements on this front, 
    variadic lifetimes imply *variadic type constructors* (are GATs sufficient?).
    And still there are plenty of questions, like how do we reshape lifetime list to what other function expects?
    Or how do I extract `'alloc` lifetime for local use?

    Realistically speaking, initial iterations of this would be implemented inside compiler
    with follow-up debates on whether it is even worth to expose the mess to language at all.
    This poses a real danger of `Fn*` traits being rendered forever unstable.

    We can try to avoid variadic lifetimes, however results are also not pretty.
    Without late-bound capability lifetimes all contextual functions (except "pure" ones with empty contexts)
    become `dyn`-unsafe.
    This will create an unfortunate rift between generics and `dyn` objects.
    Also people will immediately figure out that they can simply pass relevant capabilities as normal arguments
    and then push them into context inside the function,
    which is enough to make would-be-`dyn`-safe contextual functions actually `dyn`-safe.
    This potentially brings uncomfortable questions like why compiler cannot do it for us?

* Full support for mutable access to capabilities.

    The biggest blocker is reborrowing abstractions.
    Existing emulation works well enough with concrete contexts, but falls apart for wildcard contexts. 
    
    The only reason for reborrowing is that we manipulate mutable references directly.
    As a mind experiment, we can replace all references with pointers to get around reborrowing limitations.
    This makes a lot of things unsafe and difficult to use, but most importantly it renders our `Store` always `Copy`.
    And we already know that wildcard `Copy`able contexts are functional!
    So, *theoretically speaking*, both GAT- and input-based approach are able to fully support mutable access.
    The challenge is whether we can construct a reborrowing abstraction to allow doing it safely. 
    
    Let's also not deceive ourselves, difficulties go way beyond nailing reborrow trait.
    Mutable references don't play well with wildcard contexts at all.
    When borrow checker sees opaque context it has to be conservative with analysis 
    on off chance it contains a mutable reference.
    Throw in potential captures via closures, passing `T` objects around, and you get the picture.
    Likely, there will be a lot of unexpected pessimisation in generic code.

    The ironic part is that shared accesses are probably going to be an overwhelming majority.
    If we look at known motivating cases (`alloc`, `fs`, `log`, async executor, standard streams, etc.)
    they are either immutable or already want internal mutability for reasons.
    I don't doubt people will find compelling use cases for mutable access,
    but considering amount of effort required and looming restrictions
    I think this requires careful consideration.
    Do we want to support mutable access at all?
    Or can we provide feature in some limited form?

Have a nice day and see you next time!
