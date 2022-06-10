+++
title = "Obscure Rust: early- and late-bound generics in functions"
date = 2022-06-10

[taxonomies]
tags = ["rust", "function", "generics", "lifetime"]
+++

This is a strange edge in type system I ran into when I was working on another attempt at representing
[contexts](@/rust_contexts.md).
If contextual functions aspire to be anything like normal ones,
they need to properly interact with late-bound lifetimes.
But to do that we need to understand how those work in normal functions in the first place.

This is something like a third attempt at explaining this, even though the first two were a lot less formalized.
I'm fairly confident in my claims, but I'm learning this along with you.
If you find any mistakes make sure to tell.
It is also important to note that I was studying this interaction from inside the language.
So, narrative is likely to differ from the one used by whoever designed the feature.
For language designer, late-bound lifetimes is a solution to specific problem - one out of many.

And yes, guessing from the post name, there are more obscure features coming up.
But, *boundness* first.

# Set up

Probably the best documentation you can find on distinction between early- and late-bound generics is
[guide to `rustc`](https://rustc-dev-guide.rust-lang.org/early-late-bound.html) development.
Not the first resource to turn to and frankly speaking it is... eh, *concise* at best.

Another reference to late-bound parameters can be found inside [HRTB RFC][rfc:hrtb] (we will talk about them later).
This is the only mention of late-bound generics in RFCs that I was able to find.

Let's expand on them.

## Function shape

From the standpoint of Rust type system a typical function is zero-sized type which implements `Fn` trait:

```rust
#![feature(fn_traits)]
#![feature(unboxed_closures)]
#![allow(non_camel_case_types)]

struct i_am_function;

impl FnOnce<()> for i_am_function {
    type Output = ();

    extern "rust-call" fn call_once(mut self, args: ()) -> Self::Output {
        self.call_mut(args)
    }
}

impl FnMut<()> for i_am_function {
    extern "rust-call" fn call_mut(&mut self, args: ()) -> Self::Output {
        self.call(args)
    }
}

impl Fn<()> for i_am_function {
    extern "rust-call" fn call(&self, (): ()) -> Self::Output {
        todo!()
    }
}
```

Note that we need two unstable features `fn_traits` and `unboxed_closures` to compile examples.
ABI specifications are important but don't really matter to us.
And, yes, this is properly integrated into the language, you can use `i_am_function` as any "normal" function:

```rust
fn main() {
    i_am_function() // This works
}
```

While functions implement all three `Fn*` traits, in the future I will only show `FnOnce` as
it contains all necessary information.

With this we are ready to see what exactly is happening behind the scenes.
Time to bring in generics.

## Using generics

Applying generics is simple:

```rust
struct generic_function<'a, T>(PhantomData<fn(&'a T)>);

impl<'a, 'b, T, U> FnOnce<(&'b U,)> for generic_function<'a, T> {
    type Output = ();

    extern "rust-call" fn call_once(self, _args: (&'b U,)) -> Self::Output {
        todo!()
    }
}
```

Rust requires us to use generic parameters inside the struct, therefore `PhantomData`.
Exact way it is used can get complicated to achieve correct [variance][nomicon:variance],
but that is an entirely separate topic.

Here you probably noticed something interesting: obviously, `impl` block requires generics on it,
but `generic_function` type doesn't have any use for parameters.
So the choice whether to put it there feels arbitrary.
Both cases look valid.

Situation become clearer if we give things proper names.

Generics defined on function type are part of that type, they must be decided at the moment function type is resolved,
and this might happen at a completely different place from where function is called.
Such generics are defined *early*, hence they are *early-bound*.

Generics defined on `impl` block but not on type can only be substituted when function is called.
This can happen multiple times in different places.
Such generics are defined *late*, hence they are *late-bound*.

Process of substituting generics into function type is, well, *monomorphization*.
Nothing surprising here.

[nomicon:variance]: https://doc.rust-lang.org/nomicon/subtyping.html#variance

## Importance of monomorphization

To put everything together we need to apply one last secret ingredient:

> Every function value is always valid to call.

Explaining where the rule comes from is a non-trivial task, so for now, let's put this concern aside.
This formulation is definitely vague, so let's see how it interacts with above definitions.

For example, early-bound parameters work just fine:

```rust
struct foo<T: Clone>(PhantomData<fn(T)>);

impl<T: Clone> FnOnce<(T,)> for foo<T>
{
    type Output = ();
    
    extern "rust-call" fn call_once(self, _args: (T,)) -> Self::Output {
        todo!()
    }
}
```

We can create instance of type `foo<u32>` and then call it.
We cannot call `foo<&mut u32>`, but that isn't a problem: `foo<&mut u32>` is not a valid type,
values of one cannot be constructed, so no conflict with our rule.

Conversely, not every late-bound parameter obeys it:

```rust
struct foo;

impl<T: Clone> FnOnce<(T,)> for foo
{
    type Output = ();
    
    extern "rust-call" fn call_once(self, _args: (T,)) -> Self::Output {
        todo!()
    }
}
```

Even though `foo` is a proper value,
if you attempt to call it with `!Clone` type it will result in compilation failure,
i.e. we cannot call function with such parameter!

With this experiment, we can phrase our expectations about late-bound parameters:

> Every value of late-bound parameter when substituted must result in a "well-formed" function.

It can be tempting to reformulate it in terms of `where` blocks, such as:

> Late-bound parameters cannot participate in `where` clauses.

However, this formulation is too strict.
We can produce however many meaningless bounds, for example,
`'static: 'a` is a valid bound which accepts every possible lifetime `'a`.

Interestingly, the last formulation was used historically (possibly with implied exclusion of meaningless bounds).
I managed to track Niko Matsakis' [blog post][smallcultfollowing.com:intermingled_parameter_lists]
which likely served as a foundation for existing design.
The problem it addresses is so old that proposed changes exist only on mailing list with no RFC in sight.

[smallcultfollowing.com:intermingled_parameter_lists]: http://smallcultfollowing.com/babysteps/blog/2013/11/04/intermingled-parameter-lists/
[rfc:hrtb]: https://rust-lang.github.io/rfcs/0387-higher-ranked-trait-bounds.html?highlight=late%20bound#distinguishing-early-vs-late-bound-lifetimes-in-impls

# Implications

That was a lengthy setup.
Let's toy with those definitions a bit, shall we?

## Type

Can we have late-bound type parameters within the structure?
Actually, we can:

```rust
impl<T> FnOnce<(T,)> for foo {
    type Output = ();

    extern "rust-call" fn call_once(self, _args: (T,)) -> Self::Output {
        todo!()
    }
}
```

We can substitute any type `T` in here, and it will work.
However, using reference to `T` is trickier:

```rust
impl<'a, T> FnOnce<(&'a T,)> for foo
where
    T: ?Sized
{
    type Output = ();

    extern "rust-call" fn call_once(self, _args: (&'a T,)) -> Self::Output {
        todo!()
    }
}
```

Rust automatically adds `Sized` bound to generic parameters, so we need to relax it.
It is not possible (at least right now) to pass unsized types by value,
so we didn't have to worry about it in the previous case.

Looking at the result, it is hardly exciting.
There isn't much you can do with "bare" type besides moving and dropping.
Giving functionality to a type means applying a trait bound to it,
which sounds impossibly in conflict with the no-errors-please policy.
But we can get sneaky, for example:

```rust
impl<T> FnOnce<(T,)> for foo
where
    T: From<T>
{
    type Output = ();

    extern "rust-call" fn call_once(self, _args: (T,)) -> Self::Output {
        todo!()
    }
}
```

`From<T>` has a [blanket `impl`][std:from_t] for all `T`,
so technically this cannot produce an error (hush, negative trait bound worshippers!).

Still, there isn't any actually *useful* functions we can abstract over in such way.

[std:from_t]: https://doc.rust-lang.org/std/convert/trait.From.html#impl-From%3CT%3E-12

## Lifetime

This is where things get complicated.
It is easier to enumerate cases when lifetimes are forced to be early-bound.

Most frequent case is lifetime binding a type, `T: 'a`.
This is commonly required when creating a reference to `T`:

```rust
struct foo<'a, T>(PhantomData<fn(PhantomData<&'a ()>, T)>)
where
    T: 'a;

impl<'a, T> FnOnce<(PhantomData<&'a ()>, T)> for foo<'a, T>
where
    T: 'a,
{
    type Output = &'a T;

    extern "rust-call" fn call_once(self, _args: (PhantomData<&'a ()>, T)) -> Self::Output {
        todo!()
    }
}
```

It seems as if such lifetime must always be early-bound, however this is not the case!
To make it late-bound we need to make sure that `T` is valid for every possible lifetime,
but this is the same as saying `T: 'static`:

```rust
struct foo<T>(PhantomData<fn(T)>)
where
    T: 'static;

impl<'a, T> FnOnce<(PhantomData<&'a ()>, T)> for foo<T>
where
    T: 'static,
{
    type Output = &'a T;

    extern "rust-call" fn call_once(self, _args: (PhantomData<&'a ()>, T)) -> Self::Output {
        todo!()
    }
}
```

This holds true for concrete types as well, except there compiler doesn't need guidance.
It can always infer if type is `'static` or not and render lifetime early- or late-bound as necessary.
The whole process is invisible to user.
In fact this situation covers most (if not all) practical uses for late-bound lifetimes.

You may notice that the function from example is weirdly shaped and this is for a reason.
Because from this point things become *appropriately* confusing.

What happens if we try to pass `&'a T` as an argument instead?
If we imagine for a second that `T` is late-bound, then `'a` can be late-bound too!

```rust
impl<'a, T> FnOnce<(&'a T,)> for foo
where
    T: 'a
{
    type Output = ();

    extern "rust-call" fn call_once(self, _args: (&'a T,)) -> Self::Output {
        todo!()
    }
}
```

This one took me some time to sift through.
We know that constructing `&'a T` requires `T: 'a` bound, so why?
The answer is simple: the bound cannot be violated.
If your hand is habitually reaching for `unsafe`, stop it, we are looking to produce compile-time error, not UB.
And it turns out that Rust simply doesn't allow us to construct reference type where `'a` outlives `T`.

This doesn't seem related to practice, after all in real Rust there are no late-bound types.
But remember, we can always put a concrete type in place of a generic one.
For example, if we take some non-`'static` type like `&u32` then `&T` becomes `&'a &'b u32`.
To be valid it must satisfy bound `'b: 'a`, but we already deduced that it cannot be violated!
We can claim that both `'a` and `'b` can be bound late.
And the most crazy part: compiler [agrees][playground:a_b_u32] with us!

This situation is result of how Rust treats references.
Along with reference we receive *an implied bound* which always holds
because language forbids construction of a type that could possibly violate it.
I'm not aware of other unusual interactions which can affect lifetime "boundness".

The rest is a lot more boring compared to what we just got through.
Second case is similar to the first one, except lifetime binds other lifetime `'a: 'b`.
Unless bound is implied, neither `'a` nor `'b` can be late-bound for obvious reasons:

```rust
struct foo<'a, 'b>(PhantomData<fn(&'a u32, &'b mut &'b u32)>)
where
    'a: 'b;

impl<'a, 'b> FnOnce<(&'a u32, &'b mut &'b u32)> for foo<'a, 'b>
where
    'a: 'b,
{
    type Output = ();

    extern "rust-call" fn call_once(self, args: (&'a u32, &'b mut &'b u32)) -> Self::Output {
        *args.1 = args.0
    }
}
```

There is one more situation where lifetime is forced to be early-bound.
This is when it appears only in return type:

```rust
struct foo<'a>(PhantomData<fn() -> &'a ()>);

impl<'a> FnOnce<()> for foo<'a> {
    type Output = &'a ();

    extern "rust-call" fn call_once(self, _args: ()) -> Self::Output {
        todo!()
    }
}
```

For soundness reasons compiler requires every generic to appear as part of trait, `where` clause or `Self` type.
But we cannot oblige to either of the first two:
being part of trait means lifetime appears in arguments (which we claimed it doesn't) and
being meaningful part of `where` clause means we break our no-errors rule
(and no, clever tricks like `'static: 'a` don't work).
The only solution is to put lifetime into function type, making it early-bound.

Another piece of trivia:
lifetimes in return type used to be late-bound until [soundness issues][rust:issue32330] were discovered.

Every other use is safe to be late-bound.

[rust:issue32330]: https://github.com/rust-lang/rust/issues/32330
[playground:a_b_u32]: https://play.rust-lang.org/?version=nightly&mode=debug&edition=2021&gist=0149ebbcaae35bda4c5aabb4c2242c42

## Higher-rank trait bounds

HRTBs is one of those language features which is *hard* to wrap your hand around.
I imagine there are many who tried to use it and got hammered by compiler errors (including myself).
With a single [nomicon][nomicon:hrtb] page to guide you, no wonder it's difficult to take it past basic examples.

However, the reality cannot be more prosaic.
HRTB is simply a counterpart feature to late-bound parameters.

To illustrate, imagine type `foo` is a function:

```rust
impl<'a> FnOnce<(&'a u32,)> for foo {
    type Output = ();

    extern "rust-call" fn call_once(self, _args: (&'a u32,)) -> Self::Output {
        todo!()
    }
}
```

and we try to return it from other function as an `impl Trait`:

```rust
fn bar() -> impl FnOnce<(&'??? u32,), Output=()> {
    foo
}
```

Rust requires us to introduce a lifetime for the reference, but where that lifetime comes from?
Compiler helpfully indicates that we can put it on `bar`:

```rust
fn bar<'a>() -> impl FnOnce<(&'a u32,), Output=()> {
    foo
}

let also_foo = bar();
```

Except... `also_foo` has semantics different from original `foo`!
`FnOnce` trait is only implemented for one specific lifetime `'a` with which `bar` was called.
Trying to call `also_foo` with any other disjoint lifetime is doomed to [fail][playground:1].
It is as if we fix `'a` at the point where `bar` is called, far away from the point where resulting function is used.

This sounds eerily similar to early-bound lifetimes, and indeed it is the case!
Another important observation, there is nothing you can do to make it late-bound.
As soon as lifetime appears as part of `bar`'s generics (or anything else be it `struct` or even `impl` block)
it becomes early-bound with respect to `foo`.
Such lifetime can never be "free" because it's chosen by someone else prior to `foo` entering the scope.

HRTB allows us to bypass such restriction and decouple the lifetime from `bar`:

```rust
fn bar2() -> impl for<'a> FnOnce<(&'a u32,), Output = ()> {
    foo
}

let also_foo2 = bar2();
```

In this implementation the choice of lifetime `'a` is delayed to the point where function is called,
which turns `'a` into late-bound parameter.
HRTB helped us to smuggle it past type erasure.

And, this is it, really.
All HRTB does is introduce a late-bound lifetime to allow you to construct correct trait object.

Somewhat tangentially, HRTB also uncovers a relation between differently bound generics.
You can always convert a late-bound one into early-bound one like in the "faulty" `also_foo` example;
obviously, if it is implemented for every lifetime `'a`, then we can choose singular `'specific` lifetime out of those.
But the reverse transformation is impossible.
Unfortunately, in this case compiler cannot say "I expect an early-bound lifetime instead of late-bound one here"
because this is a hidden mechanic, it is never explained anywhere, and compiler doesn't expect you to know.
Instead, it has to resort to a more generic "Implementation is not general enough" statement,
which leaves anyone seeing it for the first time scratching their head and asking in confusion
"What the hell is *this* supposed to mean?!".

[nomicon:hrtb]: https://doc.rust-lang.org/nomicon/hrtb.html
[playground:1]: https://play.rust-lang.org/?version=nightly&mode=debug&edition=2021&gist=24bcbf21fa73b891593acdebfcfc6868

## Why late-bound parameters matter

This is the point where you should ask yourself: why do we need late-bound parameters at all?
What is so important about them to not only to put up with complexity of separating them out,
but also to create a brand new confusing language feature?
Isn't it easier to make everything early-bound?

I sort of spoiled the answer in the previous section.
The primary distinction between early- and late-bound parameters in that
they interact with type erasure in fundamentally different ways:

* Early-bound parameter is part of the type, so it must be known *before* type erasure happens.
  This means such parameter is effectively immutable and often unknowable to the user.

* Late-bound parameter can be decided *after* type erasure happens.
  Choice of such parameter is deferred to type user instead of type creator.

It cannot be understated how important this distinction is.

The simplest case of type erasure is a well-known and cherished... generic parameter!

```rust
fn foo<T>() {}
```

This is a case of *static* type erasure, but type erasure nonetheless.
Original type behind `T` is hidden, so `foo` doesn't know on which exact type it operates on.
You can tell how `T` can behave through trait bounds,
but the real type can never leak into `foo`'s implementation.

Technically, in such cases we can bypass the restriction by introducing more bounds on `T`
(which often happens to be the right choice anyways),
however *runtime* type erasure has no such luxury.
Late-bound parameters is a necessary ingredient allowing us to pass lifetimes through trait objects:

```rust
let f: dyn for<'a> Fn<(&u32, &'a u32), Output=&'a u32>;
```

On related note, remember that functions are zero-sized *types*.
So converting one to function pointer is also a form of type erasure!
Which explains why `fn` pointers can make use of HRTBs too:

```rust
let f: for<'a> fn(&u32, &'a u32) -> &'a u32;
```

Observing this brings chilling thoughts.
Without late-bound generics simply abstracting over references in functions become impractical.

## Why early-bound parameters matter

Still there is one last question left without satisfying answer.
Where the rule about callability comes from?
We saw a lot of existing language structure appear from it, but why does that rule exists in the first place?

Well, something I didn't spell out loud is the fact that function monomorphization happens in *two steps*:
first, on function type, then - on `impl` block.
So, the only way to break original rule - while writing functioning code, of course - 
is to fail second monomorphization step.
Which can be reformulated as

> `Fn*` trait impl block cannot produce monomorphization errors.

Which... sounds even more arbitrary than the first one!

Everything past this point is my best guess.

The real reason behind the phenomenon is that compiler's understanding of a function is different from ours.
Compiler's job is to lower Rust into assembly,
but for assembly function is a single set of machine instructions with an associated label.
It doesn't know anything about generics and this is where the need for monomorphization arises.

But this is also where other problems begin.
In the function model we use monomorphization happens in *two* places.
Which one is real?
Rust's decision was to associate function's monomorphized type with resulting assembly function.
However, this ruling has side effects.

Now, every `Fn*` `impl` block must result in exactly one set of assembly instructions - 
regardless of how monomorphization on it goes!
This leaves `impl` block in an awkward position:
on the one hand it can (and wants to!) introduce new generics, but it must be extremely careful.
Monomorphization at this step can no longer fail (sounds familiar?) or
result in multiple paths invocation could take. 
Latter, by the way, is the real reason why there are no late-bound types.
(And if you wonder why this logic doesn't apply to lifetimes,
lifetimes are just compiler annotations used to prove certain properties of the code,
they don't directly participate in code generation and are entirely harmless when late-bound.)

# Conclusions

There you have it.
You can probably imagine that those decisions happened in a lot more haphazard way in the early days of Rust,
and it left us with some interesting consequences.

First, `Fn*` trait implementors must satisfy an extra condition of 
being compiled down to a singular set of machine instructions, so they are... *function-like*?
At this point, any applicable terminology is slipping away from me.

Second, since current HRTB rules are based off `Fn*` trait logic,
only *function-like* trait objects can be properly expressed.
This potentially hurts us in terms of language richness:
for other traits the rule is entirely artificial and doesn't necessarily make sense.

Another pain point is `unboxed_closures` feature.
If it is stabilised as-is anyone will be able to write "wrong" functions in stable Rust.
What are we going to do about that?

And, just maybe, this is a glimpse of a clash between abstractions.
We tried to glue Rust's function types directly to assembly, but could it be that we are wrong?
Could there be some room for improvements here?

I hope you found this journey both interesting and educational.
See you next time!