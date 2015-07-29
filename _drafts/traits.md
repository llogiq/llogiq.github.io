---
title: Rust's Built-in Traits, the When, How & Why
---

As the title not quite subtly hints, today I'm going to write about the
*traits* that come with [Rust](http://rust-lang.org)'s standard library,
specifically from the context of a library writer yearning to give their users
a good experience.

Rust uses traits for a good number of things, from the quite obvious operator
overloading to the very subtle like `Send` and `Sync`. Some traits can be
*auto-derived* (which means you can just write 
`#[derive(Copy, Clone, PartialEq, Eq, Debug, Default, Hash, …)]` and get a
magically appearing implementation that usually does the right thing (with
Send and Sync, you actually have to actively opt out of implementing them).

So I'll try to go from the obvious and specific to the nebulous and (perhaps) 
surprising:

### (Partial)–Eq/Ord

`PartialEq` defines partial equality, `Eq` is used as a marker to declare that 
`PartialEq` is actually reflexive (that means `a == a` for all `a` of the 
respective type). It is useful to implement them, for a good number of `std`'s 
types use them as trait bounds for one thing or another, e.g. `Vec`'s `dedup()` 
function. Auto-deriving PartialEq will make the `eq`-method check equality of 
all parts of your type (e.g. for `struct`s, all parts will be checked, while 
for `enum` types, the variant along with all its contents is checked).

Since `Eq` is basically empty (apart from a pre-defined marker method that is 
used by the auto-deriving logic to ensure that it actually worked and probably 
shouldn't be used anywhere else), auto-deriving has no chance of doing 
something interesting, so it won't.

`PartialOrd` defines a partial Order, and extends the equality of `PartialEq` 
by the `Ordering` relation. `Ord` requires a full order relation. In contrast 
to `PartialEq`/`Eq`, those two traits actually have a different interface (the 
`partial_cmp(…)` method returns `Option<Ordering>`, while `Ord`'s `cmp(…)` 
returns the `Ordering` directly), and their only relation is that if you wish 
to implement `Ord`, you have to implement `PartialOrd` as well, for the latter 
is a trait bound for the former. Auto-deriving both will order structs 
[lexicographically](https://en.wikipedia.org/wiki/Lexicographical_order), enums 
by the appearance of the variant in the definition (unless you define values 
for the variants).

The order imposed by (`Partial`)`Ord` is used for the `<`, `<=`, `=>` and `>`
operators.

### Arithmetic Operators

The following table shows the relation between arithmetic operators and traits:

Operator|Trait
--------|-----
`a + b` |`Add`
`a - b` |`Sub`
`-a`    |`Neg`
`a * b` |`Mul`
`a / b` |`Div`
`a % b` |`Rem`

Apart from `Rem`, which is an abbreviation for *Remainder*, also  known as 
`mod` in some other languages, those are pretty obvious. The binary operator
traits all have a `RHS` (=right-hand-side) generic type bound which defaults to 
`Self`, as well as an associated `Output` type that the implementation has to
declare.

This means you can implement e.g. addition of a `Foo` and a `Bar` to return a 
`Baz` if you so desire. Note that while the operations do not constrain their
semantics in any way, it is strongly advisable not to make them mean something
entirely different than their arithmetical counterparts shown above, lest your
implementation become a footgun for other developers.

### Bit-Operators

The following operators are defined to be used bitwise. Note that unlike the 
`!`-Operator, the short-circuiting `&&` and `||` cannot be overloaded – because
this would require them to avoid eagerly evaluating their arguments, which 
isn't possible in Rust.

Operator|Trait
--------|-----
`!a`    |`Not`
`a & b` |`BitAnd`
`a | b` |`BitOr`
`a ^ b` |`BitXor`
`a << b`|`Shl`
`a >> b`|`Shr`

Like with all operators, be wary of implementing those for your type unless you 
have specific reason to, e.g. it may make sense to define some of them on 
`BitSet`s.

### Index and IndexMut

The `Index` and `IndexMut` traits specify the indexing operation with immutable
and mutable results. The former is read-only, while the latter allows both
assigning and mutating the value, that is calling a function that takes 
a `&mut` argument (note that this may, but need not be self).

You will most likely want to implement them with any sort of collection 
classes. Apart from those, use of those traits would be a footgun anyway.

### Fn, FnMut and FnOnce

the `Fn*`-traits abstract the act of calling something. The difference between
those traits is simply how the `self` is taken: `Fn` takes it by reference,
`FnMut` by mutable reference and `FnOnce` consumes it by value (which is after
all why it can only be called once, as there is no `self` to call afterwards).

Note that this distinction is just about `self`, not any of the other 
arguments. It is perfectly fine to call a `Fn` with mutably referenced or even
owned/moved arguments.

The traits are auto-derived for functions and closures and I have yet to see
a different case where they are useful. stebalien also points out that they
actually *cannot* be implemented in stable Rust.

### Display and Debug

`Display` and `Debug` are used for formatting values. The former is meant to 
produce user-facing output and as such cannot be auto-derived, while the latter 
will usually produce a JSON-like representation of your type and can safely be 
auto-derived for most types.

Should you decide to implement `Debug` manually, you may want to distinguish
between the normal `{:?}` format specifier and the pretty-printing `{:#?}` one.
The easiest way to do this is to use the Debug Builder method. The 
[`Formatter`](http://doc.rust-lang.org/std/fmt/struct.Formatter.html) type has
some (unfortunately unstable as of yet, but soon to be stabilized) very helpful
methods, look for `debug_struct(&mut self, &str)`, 
`debug_tuple(&mut self, &str)`, etc.

Otherwise you can do this by querying the `Formatter::flags()` method, which 
will have the `4` bit set (which I found out by 
[experiment](https://play.rust-lang.org/?gist=f9024a36b7e0e61c1ce7&version=stable)). 
Thus, if `(f.flags() & 4) == 4` is `true`, the caller asked you to produce 
pretty-printed output. Note that this is expressly *not* a public part of 
`Debug`/`Formatter`'s interface, so the Rust gods could change this the moment 
I write this. Seriously, if you can help it, use auto-derived Debug or 
builders.

### Copy and Clone

Those two traits take care of duplicating objects.

`Copy` declares that your type can be safely copied. This means that if you 
copy the memory a value of your type resides in, you get a new valid value that 
has no references to data of the original. It can be auto-derived (and requires 
`Clone`, because all `Copy`able types are also `Clone`able by definition). In 
fact there is no use in implementing it manually anywhere.

There are exactly three reasons *not* to implement `Copy`: 

1. Your type cannot be `Copy`able, because it contains mutable references
   or implements `Drop`.
2. Your type is so big that copying it would be prohibitively expensive (e.g. 
   it could contain an `[f64; 65536]`)
3. You actually want *move-semantics* for your type

The third reason is something that should be explained. By default, Rust (like
C++) has *move* semantics – if you assign a value from `a` to `b`, `a` no
longer holds the value. However, for types that have a `Copy` implementation,
the value is actually copied (unless the original value is no longer used, in 
which case LLVM *may* elide the copy to improve performance). The 
[docs](http://doc.rust-lang.org/std/marker/trait.Copy.html) for `Copy` go into
more detail.

`Clone` is a more generic solution that will take care of any references. You
will probably want to auto-derive it in most cases (as being able to clone
values is rather useful), and only implement it manually for things like custom
refcounting schemes, garbage collection or something similar.

In contrast to `Copy` which actually alters assignment semantics, `Clone` is 
explicit: It defines the `.clone()` method which you have to call manually to 
clone something.

### Drop

The `Drop` trait is about giving back resources when things go out of scope.
Much has been written about it, and how you shouldn't rely on it being called
should something go wrong. Still, it's very nice especially for wrapping FFI
constructs that have to somehow be reclaimed later, also it's used on files,
sockets, database handles and the kitchen sink.

Unless you have an instance where this applies, you should refrain from 
implementing `Drop` at all – your values will be `Drop`ped correctly even if 
you don't. A (temporary) exception is to insert some tracing output to find out 
when a specific value has been dropped.

### Default

`Default` is a trait to declare a default value for your type. It can be
auto-derived, but *only* for `struct`s whose members all have a `Default` 
implementations.

It is implemented for a great many types in the standard libraries, and also
used in a surprising number of places. So if your type has a value that can
be construed as being "default", it is a good idea to implement this trait.

A great thing with `struct`s that have a `Default` implementation, is you can
instantiate them with only the non-default values like:

```Rust
let x = Foo { bar: baz, ..Default::default() }
```

and have all other fourtytwo fields of Foo be filled with default values. How
cool is that?

### Error

`Error` is a base trait for all values representing an error in Rust. For those
coming from Java, it is akin to `Throwable` – and behaves similarly (apart from
the fact that we neither `catch` nor `throw` them).

It is a *very* good idea to implement `Error` for any type you intend to use in
the latter part of `Result`. Doing so will make your functions *much* more
composable, especially when you can simply `Box` the Error as a trait object.

Look at the [Using `try!`](http://doc.rust-lang.org/nightly/book/error-handling.html#using-try!)
section of the Rust book for further information.

### Hash

Hashing is the process of reducing a bag of data into a single value that still
distinguishes different data items while returning the same value for equal 
items without requiring as much bits as the processed data.

In Rust, the `Hash` trait denotes values to which this process can be applied.
Note that this trait does not relate any information about the hash *algorithm*
used, it basically just orders the bits to be hashed.

Aside: This is also the reason why `HashMap` does not implement `Hash` itself, 
because two equal hash maps could still store their contents in different 
order, resulting in different hashes, which would break the hashing contract.
Even if the items were ordered (see `Ord` above), hashing them would require
sorting, which would be too expensive to be useful.

Unless you have some very specific constraints regarding equality, you can 
safely auto-derive `Hash`. Should you choose to implement it manually, beware
not to break its contract.

### Iterator and Friends

Rust's `for` loops work can iterate over everything that implements 
`IntoIterator`. Yes, that includes `Iterator` itself. Apart from that, the
`Iterator` trait has a lot of cool methods for working with the iterated
values, like `filter`, `map`, `enumerate`, `fold`, `any`, `all`, `sum`, `min`
and much more.

Did I tell you I love iterators? If your type contains more than one value of
something, and it makes sense to do the same thing to all of them, consider
providing an `Iterator` over them just in case. :-)

Implementing `Iterator` is actually pretty easy – you just need to declare the
`Item` type and write the `next(&mut self) -> Option<Self::Item>` method. This
method should return `Some(value)` as long as you have values, then return 
`None` to stop the iteration.

Note that if you have a *slice* of values (or an array or vec, from which you
can *borrow* a slice), you can get its iterator directly, so you don't even
need to implement it yourself. This may not be as cool as auto-deriving, but
it's nice nonetheless.

While writing [optional](https://github.com/llogiq/optional), I found that 
using a const slice's iterator is faster in the boolean case, but creating a 
slice of the value is still slower than copying it for most values. Your 
mileage may vary.

### From, Into and Various Variations

I said it before, whoever designed the `From` and `Into` traits is a genius.
They abstract over *conversions* between types (which are used quite often) and
allow library authors to make their libraries much more interoperable, e.g. by
using `Into<T>` instead of `T` as arguments.

For obvious reasons, those traits cannot be auto-derived, but writing them
should be trivial in most cases. If you choose to implement them – and you 
should wherever you find a worthwhile conversion! – implement `From` wherever
possible, and failing that implement `Into`.

Why? There is a blanket implementation of `Into<U>` for `T` where `U: From<T>`.
This means if you have implemented `From`, you get an `Into` delivered to your
home free of charge.

Why not implement `From` everywhere? The orphan rule unfortunately forbids 
implementing `From` for types not defined in other crates. For example, I have
an `Optioned<T>` type, that I may want to convert into an `Option<T>`. Trying
to implement `From`:

```rust
impl<T: Noned + Copy> From<Optioned<T>> for Option<T> {
	#[inline]
	fn from(self) -> Option<T> { self.map_or_else(|| none(), wrap) }
}
```

I get an error: type parameter `T` must be used as the type parameter for some 
local type (e.g. `MyStruct<T>`); only traits defined in the current crate can 
be implemented for a type parameter `[E0210]`

Note that you can implement `From` and `Into` with multiple classes, you can
have a `From<Foo>` and a `From<Bar>` for the same type.

There are a good number of traits starting with `Into` – `IntoIterator`, which
is stable and which we already have discussed above, just being one of them.
There also is `FromIterator`, which does the reverse, namely constructing a
value of your type from an iterator of items.

Then there is `FromStr` for any types that can be parsed from a string, which 
is very useful for types that you want read from any textual source, e.g. 
configuration or user input.

### Send and Sync

TODO

### Deref(Mut), AsRef/AsMut, Borrow(Mut) and ToOwned

Those all have to do with references and borrowing, so I grouped them into one
section.

The prefix-`*`-operator *dereferences* a reference, producing the value. This 
is directly represented by the `Deref` trait; if we require a mutable value 
(e.g. to assign somehing or call a mutating function), we invoke the `DerefMut` 
trait.

Note that this does not necessarily mean *consuming* the value – maybe we take 
a reference to it in the same expression, e.g. `&*x` (which you will likely 
find in code that deals with special kinds of pointers, e.g. `syntax::ptr::P` 
is widely used in [clippy](https://github.com/Manishearth/rust-clippy) and 
other lints / compiler plugins. Perhaps `as_ref()` would be clearer in those 
cases (see below), but here we are.

The `Deref` trait has but one method: `fn deref(&'a self) -> &'a Self::Target;`
where `Target` is an associated type of the trait. The lifetime bound on the
result requires that the returned value live as long as self. This requirement
restricts the possible implementation strategies to two options:

1. Dereference to a value *within* your type, e.g. if you have a 
`struct Foo { b: Bar }`, you could dereference to `Bar`. Note that this doesn't
mean you *should* do it, but it's possible and may in some cases be useful.
This obviously works as long as the part's lifetime is the one of the whole,
which is the default with Rust's lifetime elision.

2. Dereference to a constant `'static` value – I 
[do this](https://github.com/llogiq/optional/blob/db2b4c742e41f4607e64b9e855ae4638d839e828/src/lib.rs#L76) 
in optional to have `OptionBool` dereference to a const `Option<bool>`. This 
works because the result is guaranteed to outlive our value, because it is 
alive for the rest of the program. This is only useful if you have a finite 
value domain. Even then, it is probably clearer to use `Into` instead of 
`Deref`. I doubt that we will see this too often.

`DerefMut` only has the former strategy. Its usefulness is limited to 
implementing special kinds of pointers.

To see why no other implementation can be possible, let's make a thought
experiment: If we had a return value that is neither static, nor bound to the
lifetime `'a` of our dereferenced value, it would by definition have a lifetime
`'b` that is *distinct* from `'a`. There is no way we could unify those two
lifetimes – QED.

As for the other traits, they exist mainly to abstract away the act of
borrowing / referencing for some types (because e.g. with `Vec`s it is possible 
to borrow a slice of them). As such, they fall into the same category as the
`From`/`Into` traits – they don't get invoked behind the scenes, but exist to
make some interfaces more adaptable.

The relation between `Borrow`, `AsRef`/`AsMut` and `ToOwned` is as follows:

From↓ / To→|Reference      |Owned
-----------|---------------|-----
Reference  |`AsRef`/`AsMut`|`ToOwned`
Owned      |`Borrow`(`Mut`)|(perhaps `Copy` or `Clone`?)

For an example where this applies, look no further than my earlier
[detective story about `std::borrow::Cow`](/2015/07/09/cow.html).

Should you decide to implement `Borrow` and/or `BorrowMut`, you need to ensure
that the result of `borrow()` has the same hash value as the borrowed original
value, lest your program fail in strange and confusing ways.

In fact, unless your type does something interesting with ownership (like 
`Cow` or [`owning_ref`](https://github.com/Kimundi/owning-ref-rs)), you should
probably leave `Borrow`, `BorrowMut` and `ToOwned` alone and use a `Cow` if
you want to abstract over owned/borrowed values.

I have not yet divined in what cases `AsRef`/`AsMut` may be useful unless you 
count the predefined `impl`s that `std` already provides.

----

Thanks go out to stebalien and carols10cents for (proof)reading a draft
of this and donating their time, effort and awesome comments! This post
wouldn't have been half as good without them.

Have I missed, or worse, misunderstood a trait (or a facet of one)? Please 
write your extension requests on [/r/rust](https://reddit.com/r/rust) or 
[rust-lang users](https://users.rust-lang.org).
