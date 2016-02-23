---
title: More Type-Level Shenanigans
---

[last time](/2015/12/12/types.html) we looked at how we can (mis?)use Rust's 
type system to perform arbitrary computations. There was one problem though: 
Restricting types with `where` clauses could cause the compiler to do an 
exponential-time search in the type space, in the worst case DoSing the 
compiler.

While I knew of the issue, I could not offer a solution at the time. However, 
redditor [ZRM2](https://www.reddit.com/user/ZRM2) 
[found](https://www.reddit.com/r/rust/comments/46l8jg/inconsistency_in_recursive_trait_dependencies/) 
a possible solution, which I will try to describe here.

Taking a step back, the problem is that rustc cannot know which *concrete* type 
is behind the *existential type* (=trait bound), so it has to search all 
possible types until it comes up with a solution. Unfortunately our misuse of 
generics means that the solution can be quite complex, and reaching that 
solution with a search is costly – as in waiting a few seconds or until the 
heat death of the universe, depending on the type whose existence rustc tries 
to prove.

But what if we could tell rustc to use a specific concrete type? Then it could 
do its thing without searching, compile times would evaporate and we'd be so 
much happier. I know, it's just a what if scenario...but can it be possible?

Turns out it can.

To do this, we need to take a concrete type and implement our trait for it, 
then use the type coerced to our trait bound instead of a generic we pull out 
of thin air (where the compiler will search for it). A good pick for our 
concrete type would be unit (spelled `()` in Rust lingo), because it's 
zero-sized and already available.

So let's say we have a trait `Foo` that we want to impl a `Blah` operation on:

```rust
trait Blah {
    type Output;
}

impl<X: Foo> Blah for X {
    type Output = ..; // here we trigger exponential search
}
```

we can instead

```rust
trait PrivateBlah<F> {
    type Output;
}

impl PrivateBlah<Foo> for () {
    type Output = ..; // same as before, but rustc is happy
}

trait Blah {
    type Output = ..;
}

impl<F: Foo> Blah for F {
    type Output = <() as PrivateBlah<F>>::Output;
}
```

Now rustc does a plain trait lookup on `()` instead of searching the type space 
for a generic type. Trait lookup is a linear racket (proportional to the number 
of trait implementations the type has in scope), and the number of trait impls 
is usually manageable.

----

As a sort of bonus, the original solution had a lot of `unreachable!()`s 
whenever methods were implemented. However, this entails some generated code 
that need not be there. We can get rid of those if we encode our atoms as empty 
enums by relying on the fact that an empty `match {}` matches all possible 
values of an empty enum. So if we have

```rust
enum Foo {}
```

we can write

```rust
impl<B: Bar> Add<B> for Foo {
    type Output = ...;
    fn add(self, _: B) -> Self::Output { match self {} }
}
```

The compiler will (as of Rust 1.8 nightly) generate a `core::panic` (though it 
probably could just generate an empty function) instead of a more costly 
`unwind` (which will just bloat our code with landing pads needlessly).

----

What stuff do you misuse the type system for? Discuss on
[rust-users](https://users.rust-lang.org) and/or
[r/rust](https://reddit.com/r/rust)!
