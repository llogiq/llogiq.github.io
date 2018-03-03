---
title: Opportunistic Mutations
---

As you may know, my current [mutagen](https.//github.com/llogiq/mutagen)
project deals with mutation testing in Rust. However, as I remarked, Rust's
famed flexibility leaves us little room to do mutations while keeping the type
checker happy. For example, other mutation testing frameworks can mutate
`x + y` to `x - y`.

This is an interesting mutation, because it's so easy to do in languages like
Java, which have full type information available at the bytecode level and so
hard to do in Rust, because the `std::ops` traits make everything so hecking
flexible.

So my first thought was "it's generally impossible without trait lookup". But
then I remembered the trick I pulled in
[overflower](https://github.com/llogiq/overflower), which is to shanghai the
type system into doing the lookup for us via specialization: We can write our
own ops trait that delegates to the `core::ops` trait by default while
specializing for known types.

(Aside: Yes, specialization is unstable and the rules are going to be stricter
than what is currently implemented in nightly, but our usage is in line with
the rules as [outlined by Niko Matsakis])

[outlined by Niko Matsakis]: http://smallcultfollowing.com/babysteps/blog/2018/02/09/maximally-minimal-specialization-always-applicable-impls/ "Maximally Minimal Specialization"

However, unlike overflower, we want to still be generic over our types: E.g.
for replacing `+` with `-` (and vice versa) we need to be sure that the `Rhs`
type parameters are equal *and* the associated `Output` types are compatible.
So without further ado, here's our trait (I'll use a trait to mutate add to
sub as an example. Mutating other ops is essentially the same):

```rust
pub trait AddSub<Rhs = Self> {
    type Output;
    fn add(self, rhs: Rhs, mutation_count: usize) -> Output;
}
```

A default impl that unconditionally adds looks like this:

```rust
impl<T, Rhs> AddSub<Rhs> for T
where T: Add<Rhs> {
    type Output = <T as Add<Rhs>>::Output;
    default fn add(self, rhs: Rhs, _mutation_count: usize) -> Output {
        self + rhs
    }
}
```

Now we would like to require the `Output` types of `Add` and `Sub` to be equal.
This could be done with the following trickery:

```rust
/// this trait should only be implemented for pairs of the same type
trait Same;
impl<T> Same for (T, T) { }

impl<T, Rhs> AddSub<Rhs> for T
where T: Add<Rhs>, T: Sub<Rhs>,
    (<T as Sub<Rhs>>::Output, <T as Add<Rhs>>::Output) : Same {
    ..
}
```

This looks like it should work, and indeed the type system will accept it.
However it turns out we cannot implement this method, because ironically, even
knowing the types are one and the same gives us no way to convert between them.
So let's take a step back and re-evaluate our requirements:

For our specialized impl, we need a way to turn the `Output` of the `Sub`
operation into that of the `Add` operation. They don't even need to be the same
type, they just need to be convertible. Fortunately what Rust's type system
taketh away, it also giveth, in this instance with the `Into` trait, which
allows us to abstract over convertible types:

```rust
impl<T, Rhs> AddSub<Rhs for T
where T: Add<Rhs>,
      T: Sub<Rhs>,
     <T as Sub<Rhs>>::Output: Into<T as Add<Rhs>>::Output> {
    fn add(self, rhs: Rhs, mutation_count: usize) -> Output {
        if mutagen::now(mutation_count) {
            (self - rhs).into()
        } else {
            self + rhs
        }
    }
}
```

That `where` clause is a mouthful, but reading it carefully makes the intent
really clear: `T` implements addition and subtraction and the `Output` type of
`T`'s subtraction is convertible into the `Output` type of its addition. So
this specifies exactly what we need and it actually works. Neat!

There is but one niggle in that we don't want to run the ineffective mutation
via the `default` impl, but this can also easily be fixed by setting up yet
another backchannel between the non-mutated test run and the test runner that
the default impl can use. The test runner can then avoid the ineffective
mutation.

### The Clone Wat!?

The same thing goes for cloning a mutable ref (which is similar to an early
return, but without actually skipping the whole method). As we don't know if
the type in question implements `Clone`, we cannot change the code and still
compile. But what if we could?

There are two snags here:

1. We need to decide if we can clone the value
2. We need somewhere to own the cloned instance

The first one is pretty simple:

```rust
trait MayClone<T> {
    /// can we clone the current value?
    fn may_clone(&self) -> bool;
    /// only will be called on cloneables, so we can panic for non-cloneables
    fn clone(&self) -> Self;
}
```

if we specialize this trait for `T: Clone`, we can use it to prepend the
following for a function with the mutable argument `x`:

```
let mut _x_clone;
if MayClone::may_clone(x) {
    let _x_clone = MayClone::clone(x);
    x = &mut _x_clone;
}
```

Now the `may_clone` method tells us if we can clone and our `_x_clone` binding
holds the cloned value. Again, we should keep a flag to see if the mutation
was effective.

### Ineffective mutations vs. types

One small caveat about the "flag" for ineffective mutations: As functions may
be generic over some type and tests may call a function with multiple types,
thereby exercising multiple implementations, we need to keep track of the
mutation *for each test* and only declare the mutation as ineffective unless it
wasn't effective for a single test.

Note that being ineffective once during a test is not enough, it must also
never have been effective during the same test.

-----

Are there other mutations that I have not thought of that we can implement with
this technique? [Tell me about it](https://github.com/llogiq/mutagen/issues)!
