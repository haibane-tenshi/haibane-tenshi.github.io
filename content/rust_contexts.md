+++
title = "Future Rust: context desugaring"
date = 2021-12-24

[taxonomies]
tags = ["rust", "contexts"]
+++

Following recent discussion of [contexts](https://tmandry.gitlab.io/blog/posts/2021-12-21-context-capabilities/)
and [capabilities](https://jam1.re/blog/thoughts-on-contexts-and-capabilities-in-rust),
let us explore a potential design within current reach of Rust.

<!-- more -->

Exploring concrete desugaring can help us find the limits and answer some of those difficult questions
that were raised.
This isn't a concrete proposition, rather a loose (and incomplete!) sketch of
how we can potentially express contexts in today's Rust.

Also, here's the [playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=6da521cc5f27588edada2a2516e33df6)
which I used for exploration (warning: **very** NSFW).

## Representation

We will use global allocator as a strawman for typical use.
Allocation is one of the more pervasive ambient capabilities in every program,
so it makes for perfect target to test the design.

### Capabilities

In order to encode presence of capability we need to lift it into type system.
The easiest way to achieve that is newtype wrapper:

```rust
capability alloc: GlobalAllocator; 
```

expands into

```rust
struct __alloc(GlobalAllocator);
```

Very straightforward.

### Traits: context

We should be able to call a function from different places with potentially different contexts,
so it makes sense for context to be generic.
On the other hand, we also need to be able to extract capabilities from it - that's the whole point.
We can achieve this with traits, `Get`, `GetMut` and `GetOnce`:

```rust
trait Get<'a, T> {
    fn get(&self) -> &'a T;
}

trait GetMut<'a, T> {
    fn get_mut(&mut self) -> &'a mut T;
}

trait GetOnce<T> {
    type OtherCx;

    fn get_once(self) -> (T, Self::OtherCx);
}
```

It is important to bundle lifetime along references in `Get` and `GetMut` -
we don't want to be bound to lifetime of context itself.
`GetOnce` is also peculiar, it returns a different "leftover" context back to us.
After `get_once` is called we can no longer acquire the same capability from current context,
but to encode that in type system we need to drop `GetOnce` trait itself.
This is done by exchanging current context type for a different `OtherCx` type
which (presumably) carries the other capabilities.
Constraining it in `where` clause is certainly a pain.
I wonder if there is a better way.

Combining with previous section we can already express capability requirements as normal `where` blocks:

```rust
fn f<Cx>(args: (), cx: Cx)
where
    for<'a> Cx: Get<'a, __interner>,
{
    let alloc: &__interner = cx.get();
    // We can now use `alloc` to allocate, nice!
}
```

### Traits: `CxFn*` family

In Rust functions and operators are expressed as traits, so we will probably need traits for our new functions too.
Let's call new traits `CxFn`, `CxFnMut` and `CxFnOnce` (or `CxFn*` family) standing for "contextual function".
I can already sense bikeshedding team mobilize.

They closely follow `Fn*` family for obvious reason and defined like following:

```rust
trait CxFnOnce<Args, Cx> {
    type Output;
    
    fn cx_call_once(self, args: Args, cx: Cx) -> Self::Output;
}

trait CxFnMut<Args, Cx>: CxFnOnce<Args, Cx> {
    fn cx_call_mut(&mut self, args: Args, cx: Cx) -> Self::Output;
}

trait CxFn<Args, Cx>: CxFnMut<Args, Cx> {
    fn cx_call(&self, args: Args, cx: Cx) -> Self::Output;
}
```

This looks quite natural: we preserve existing relationship within `Fn*` trait family and traits look very much alike
just with an addition of extra `Cx` parameter.

There are probably a few points which capture your attention.

First, it may feel more natural to put `Cx` as generic bound on *method* instead of trait,
but that gives us wrong semantics.
`Cx` param on method says "function is callable with every possible `Cx` as long as trait is implemented",
however this is wrong.
What we want to say "function is callable only with *good* `Cx`",
i.e. it needs to conditionally implement trait depending on context.

Second, we take `Cx` by value.
This is analogous to normal arguments, any kind of more sophisticated ownership control will come from individual
fields inside the context.

## Desugaring

### Concrete functions

Let's put it all together:

```rust
fn foo() 
with &alloc
{
    let mem = alloc.allocate()
}
```

becomes

```rust
struct __foo;

impl<Cx> CxFn<(), Cx> for __foo
where
    for<'a> Cx: Get<'a, __alloc>,
{
    fn cx_call(&self, (): (), cx: Cx) {
        let __alloc: &__alloc = cx.get();
        let mem = __alloc.0.allocate();
    }
}
```

I omitted implementations of `CxFnOnce` and `CxFnMut` as they are analogous.
I will do so for future examples too to keep it brief.

Observations:
1. All functions taking contexts are generic by necessity, but we already implied that.
2. Capabilities translate into trait bounds,
   so you **must** specify capability if you use it directly.
   This is very much like Rust, it forces you to explicitly state your intentions.
3. You **must** explicitly specify desired ownership for every capability.
    I use fictional `&alloc`, `&mut alloc` and `move alloc` syntax to represent that.
    This is again analogous to normal parameters.

### Constraint propagation

The interesting part is how we propagate constraints up.
Let's take a look:

```rust
fn bar() {
    foo()
}
```

translates into

```rust
struct __bar;

impl<Cx> CxFn<(), Cx> for __bar
where 
    __foo: CxFn<(), Cx>
{
    __foo.cx_call((), cx)
}
```

The magic happens in `where` clause.
Normally in generic code if a function has `Cx: Trait` bound you replicate it on caller,
but that states constraint explicitly as opposed to smuggling it through.
Instead, we do something rather roundabout: we just require that *`foo` is callable with current context*.
Compiler is smart and will deduce required bound on `bar` from bound on `foo`.

Smuggling is successful!

There is a thought to be had here.
Unlike with previous case this extra bound on `bar` is added silently which can be upsetting to explicit gang.
We can consider to explicitly write it in `bar`'s signature

```rust
fn bar()
with foo // some arbitrary syntax
{
    foo()
}
```

but this will probably take it to untenable level of verbosity besides being a SemVer hazard.
It will also look somewhat weird to read, we write: `foo` must be callable to call `bar`.
Newcomers without knowledge of contexts will likely get greatly confused.

### Traits

`with` on trait simply reapplies bound to all methods.
There is no shared `Cx` type at trait level, so this is the only possible behaviour.

### Context creation

One last bit to look at is context creation.

```rust
with alloc = GlobalAllocator::new() {
    bar()
}
```

becomes

```rust
{
    let alloc = __alloc(GlobalAllocator::new());

    struct __ScopedContext<'a> {
        alloc: &'a __alloc,
    }

    impl<'a> Get<'a, __alloc> for __ScopedContext<'a> {
        fn get(&self) -> &'a __alloc {
            self.alloc
        }
    }

    let __cx = __ScopedContext { alloc: &alloc };

    __foo.cx_call((), __cx)
}
```

Quite straightforward, again.

If there is an outer context we can use `Get*` traits to pull extra capabilities from it in case those are needed.

## Interaction with other language parts

### `Fn*` traits

Because `Fn*` traits don't accept context parameter, we can think of them as contextual functions callable
with empty context.
This brings us to redefinition:

```rust
trait FnOnce<Args>: CxFnOnce<Args, ()> {
    fn call_once(self, args: Args) -> Self::Output;
}

trait FnMut<Args>: FnOnce<Args> + CxFnMut<Args, ()> {
    fn call_mut(&mut self, args: Args) -> Self::Output;
}

trait Fn<Args>: FnMut<Args> + CxFn<Args, ()> {
    fn call(&self, args: Args) -> Self::Output;
}

impl<Args, T> FnOnce<Args> for T
where T: CxFnOnce<Args, ()>,
{
    fn call_once(self, args: Args) -> Self::Output {
        self.cx_call_once(args, ())
    }
}

impl<Args, T> FnMut<Args> for T 
where T: CxFnMut<Args, ()>,
{
    fn call_mut(&mut self, args: Args) -> Self::Output {
        self.cx_call_mut(args, ())
    }
}

impl<Args, T> Fn<Args> for T
where T: CxFn<Args, ()>,
{
    fn call(&self, args: Args) -> Self::Output {
        self.cx_call(args, ())
    }
}
```

Even blanket impls are possible!
We expect implementation of `CxFn*` to be generic over *all possible* `Cx` it can accept,
so it is only required to test one.
Unit type is perfect.

Observations:
* Every function now implements `CxFn* + ?Fn*` bound instead of `Fn*`.
* Which means `CxFn*` trait is now defining characteristic of being a function, not `Fn*`.
    I guess it's a good thing `unboxed_closures` and `fn_traits` are not stabilized?
    Also writing function objects just became even harder.
* Most functions just stopped being `Fn*` too: our global allocator capability made sure of that.
    We will look at repercussions in a second.

### Closures

The defining mechanic of closures is capture of local variables.
Context is also desugared into a local variable, so it should be able to hack into existing capture mechanism.
This feels like most natural behaviour, and it is likely to align with user expectations.

This brings important implications:
* Closures always implement `Fn*`.
    This is a very important property to preserve (at least by default) otherwise all callback-based code 
    will probably break.
* Therefore, closures is the primary mechanism for constructing `Fn*` functions.
* Calling a closure represents discontinuity in context propagation.
* Closures have to rely on compiler magic to capture minimal context, but that is already expected for normal captures.
    Still it potentially presents a bit more work for compiler due to context bundling/unbundling. 
* Context is part of closure capture, which means automatic propagation of lifetimes and auto-traits.

Latter is really cool to get out of the box.
Imagine writing this code:

```rust
fn foo() -> impl Fn() {
    with cx = MyContext::new() {
        || cx.do_something()
    }
}
```

Compiler can immediately flag this *using already existing error*:
"[[E0373]](https://doc.rust-lang.org/stable/error-index.html#E0373):
closure may outlive the current function, but it borrows `cx` ...",
and suggest you explicitly move context into closure instead.

But what about *contextual closures*?
Surely there is some madman out there already contemplating a use for them.
Unfortunately it implies we are able to construct a closure generic over `Cx` type,
but we will need to figure out how to make generic closures first.
I haven't seen any RFC on the subject, but it may be in the works somewhere.

### Callbacks

Callbacks can choose whether they want to propagate their own context or not.
Because whole ecosystem currently lives with `Fn*`, this will probably become a golden standard.
It is also almost impossible to use normal functions as callbacks, which can break existing code.
Mitigation is easy: just wrap your callback in closure so it captures all necessary context.

Contextual callbacks is a potential direction for future, but without contextual closures such API is *hard* to use
on caller side.

### Multithreading and `async`

Now, what about sending functions to other threads?
Currently `thread::spawn` is defined as 

```rust
pub fn spawn<F, T>(f: F) -> JoinHandle<T>
where
    F: FnOnce() -> T,
    F: Send + 'static,
    T: Send + 'static,
{
    //...
}
```

and I think it doesn't have to change.

You can think of contexts as extra information necessary for computation which is passed to a function by its parent.
But by sending `f` to another thread it itself becomes the root of call tree and single source of truth;
it doesn't have a parent to ask about (and no, `main` doesn't count - it may have terminated long ago!).
It should have all necessary information baked in by the time we send it over.
And this is exactly what `FnOnce` bound implies: function is self-sufficient.

Scoped threads *can* provide a parent function to ask for context,
but conceptually it at most lifts `'static` bound on `f`.
Context still needs to be shared and passed over somehow,
so it makes sense to bake it into function call or function object itself,
which leads to `FnOnce` bound once more.

By extension, `main` should also implement `Fn`, but that's debatable, we will come back to this point in a moment.

What about `Send` (or `'static`)?
`Send` on the closure translates naturally onto `Send` on captures, which includes `Cx` type -
and compiler can take it from there.
On the other hand, function can directly set a capability instead of capture, leading to a thread-local value.
This is a very nice result, programmer is still in control of what is happening:
every capability can be shared or thread-local depending on how you use it.

Situation with async is mostly identical.

### Visibility

Visibility leans into already existing visibility for types and works just like you expect.
This snippet

```rust
mod private {
    // not pub
    capability interner: Interner;
    
    pub fn foo()
    with &interner
    {}
}
```

causes public function `foo` to have `where Cx: Get<__interner>` block on `CxFn` impl.
But that is already flagged by compiler:
"[[E0446]](https://doc.rust-lang.org/stable/error-index.html#E0446): private type `__interner` in public interface".

Which implies that any public functions have to take care and properly set every private capability
in case they want to use it or call any other function that do.
Great!

### Defaults

What about global defaults?
Most programs are probably content having a global allocator set for it by `std`.
Can we do this?

Sure, there are two simple steps:

1. Provide default value on capability itself.
    It must be const evaluatable, remember
    [rule 1.4](https://doc.rust-jp.rs/the-rust-programming-language-ja/1.9/complement-design-faq.html#there-is-no-life-before-or-after-main-no-static-ctorsdtors):
    no life before `main`.
2. Write `with interner = Capability::default()` on top of your function.

Wait, what? Where did my default go? Isn't setting it by hand defeats the point?

To understand the problem, consider this example:

```rust
mod private {
    capability interner: Interner = Interner::new();
    
    pub fn foo() {
        interner.intern("s");
    }

    pub fn bar() {
        with interner = Interner::other() {
            foo()
        }
    }
}

fn main() {
    bar()
}
```

Alright, let's unpack.

First, `interner` is private to module.
As we already deduced, it cannot appear in public APIs.
Second, `foo` uses `interner`, so it must be set before the call.
Third, compiler sees public `foo` which has unsatisfied capability `interner`,
but `interner` cannot be set in anywhere outside the module,
so when `foo` is called from outside `interner` can only be set inside `foo`.
Fourth, `interner` has a default, so user expects this to be automatically set.
Following this logic, compiler silently adds `with interner = Capability::default()` to the top of `foo`
as the only way to provide the default.

Now, when we call `bar` it sets `interner` but that gets immediately overridden inside `foo`.
What an elegant footgun, activated by simply adding a `pub`!

All of this happened because compiler tried to second-guess what we mean.
Let's keep it explicit, shall we?

### Defaults and `main`

If capability is reachable from `main` and have a default we call it *ambient capability*.
Naming is analogous to *ambient authority*, it represents a capability which is always naturally available.

This is one case where this explicitness certainly hurts.

In case we guarantee that `main` requires no context, 
every ambient capability provided by any of your dependencies must be set by hand.
On the other hand ambient capabilities are extremely pervasive, so it is preferable to set them on top of `main`.
But many applications don't even have access to the function!

And then, just imagine starting every doc-test with 

```rust
fn main() {
    // We probably need allocations, right?
    with alloc = Capability::default() {
        //...
    }
}
```

Options are not very abundant.
We can rely on compiler to enumerate all existing ambient capabilities and silently add them on top of `main` -
for everyone's sanity.
Alternatively, we can drop `Fn` requirement on `main` and pass in some prebaked global context
(which should be acceptable if all capability defaults are const-evluatable).
Effectively, both amount to the same thing with difference in details.

### Object safety

Contextual functions are not object safe.
Shouldn't be a surprise - we cannot predict with which concrete type of `Cx` it will be called while being type-erased.
The only way to make them object safe is to turn `Cx` into trait object:

```rust
struct __foo;

trait __MyContext: Get<__alloc> + Get<__interner> {}

impl CxFn<(), Box<dyn __MyContext>> for __foo {
    fn cx_call(&self, (): (), cx: Box<dyn __MyContext>) {
        //...
    }
}
```

But this looks like a lot of extra magic and a lot of extra discussions.

### `fn` pointers

ZST `Fn` objects can be converted to `fn` pointers.
Since almost no normal functions implement `Fn` now, they are not convertible to `fn` either.
This is OK with me, but not so OK with FFI gang.

### FFI

FFI requires function pointers, function pointers require `Fn`.
I offhandedly mentioned that most normal functions don't implement `Fn`, but it doesn't mean we cannot craft one.
We just need to make sure it is callable with empty context, so it is enough to set all required capabilities
at the beginning of the function to achieve that:

```rust
fn ffi_safe() {
    with 
        alloc = GlobalAllocator::new(),
        interner = Interner::new()
    {
        // we can now call contextual functions which use `alloc` or `interner`
    }
}
```

In other words, it means all exported functions **must** properly set all required capabilities on Rust side.
This makes a lot of sense: even if we allow other language to do it, how do they know how to do it correctly?

The problem here is already existing FFI-exported functions - every singly one will get broken as soon as some of the
more contagious capabilities get introduced (like `alloc`).
Some cases may require creating extra wrappers as well.

There is another interesting implication: Rust-to-Rust communication through FFI naturally does not share global
context - each side is required to set its own when being called by the other.
This, surprisingly, also makes a lot of sense.
Anyone who happened to debug global statics in dynlibs in C++ 
[will understand](https://stackoverflow.com/questions/19373061/what-happens-to-global-and-static-variables-in-a-shared-library-when-it-is-dynam).
This also begets a thought, that if direct Rust-to-Rust communication ever happens it *probably* should follow the suit.


## Potential breakage

* All of exported FFI
* Normal functions as callbacks - auto-fixable.
* Certain closures uses - sometimes auto-fixable.

Because closures now silently capture more things, it can lead to breakage 
when multiple closures are converted to `fn` pointers:

```rust
let f: fn() -> Vec<_> = if cond {
    // Doesn't allocate, ok!
    || Vec::new()
} else {
    // Oops, this now have to capture `alloc` capability
    || vec![0_usize, 1, 2, 3]
};
```

Capturing any context is enough to prevent it from being convertible to a function pointer.
This example is stupid, but I actually used this approach a couple of times in my own projects,
it simplifies functional-style processing.

Fixing this in general case is hard.
You can try to define closures outside of `if`/`match` and convert them to `dyn Fn` instead:

```rust
let first = || Vec::new();
let second = || vec![0_usize, 1, 2, 3];  

let f: &dyn Fn() -> Vec<_> = if cond {
    &first    
} else {
    &second
};
```

but capturing variable local to branches makes boxing hard to avoid.

## Case study: global allocator

Global allocator is determined by [`GlobalAlloc`](https://doc.rust-lang.org/std/alloc/trait.GlobalAlloc.html) trait,
so global allocator itself basically is just a vtable (+maybe some data).
To allow overriding we need type erasure;
the textbook solution is to use `Box<dyn GlobalAlloc>` trait object, but we can't, creating box requires allocation!
Instead, we have to keep it as a reference only:

```rust
pub capability alloc: &dyn GlobalAlloc;
```

However, there is something our pedantic compiler will not be happy with.
If this is a reference, how long does it live?

Well...
For how long referenced object lives.
Proper solution (and we are getting way ahead of current discussion!) is to make it generic over lifetime:

```rust
pub capability alloc<'a>: &'a dyn GlobalAlloc;
```

It would allow you to optionally override global allocator with locally-scoped variant (even using stack memory!),
but for now let's set it to `'static`.
This will provide us with the same overall semantics as `#[global_allocator]`.

```rust
pub capability alloc: &'static dyn GlobalAlloc;
```

Setting default is the easy part: `std` default allocator is ZST, so we can just put it in:

```rust
pub capability alloc: &'static dyn GlobalAlloc = &std::alloc::alloc::Global;
```

## Conclusions

This looks fun and way more tractable that it looked at first.
Rust still keeps surprising me in how *powerful* it is!

There is certainly a lot more to investigate (interaction with lifetimes? generic code? constraining as part of trait?),
but this post is already way too long. 

Hope you had fun too, see you next time!
