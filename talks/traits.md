---
marp: true
author: Andre 'llogiq' Bogus
title: From Deref Into Borrow
description: A Primer On Standard Library Traits
class: invert
---
<style>
:root {
  background: #000;
}
.no { border: 0px; background: #000; }
</style>
# From Deref Into Borrow
### A Primer on Rust Standard Library Traits
<!--
What A Wonderful Code

I see structs with fields and enum types
the variants clear, the memory tight
and I think to myself: what a wonderful code.

I see ints and strings and booleans too
where one may be false or otherwise true
and I think to myself: what a wonderful code.

The traits implemented for all respective types
Generics avoiding the same code written twice
I see clauses that guard, lifetimes that bound
patterns that match, loops goin' round,

I see methods try, I watch them Err
with question mark they don't need to care
and I think to myself: what a wonderful code

Yes I think to myself: what a wonderful code
-->
---
## Traits

*Trait items*

- Methods
- Types
- Associated Constants

---
## Traits

- contract between *users* and *implementors*
- abstracts *behavior* from a type
- can be used in generic bounds
- can have their own bounds (e.g. `trait Eq: PartialEq`)

---
## Traits

- allow calling methods on implementing types without naming them
- the trait must be in scope (e.g. `use std::fmt::Write as _`)
- if a function is provided by multiple traits, disambiguation is needed

```rust
trait Boo { fn boo(&self); }
trait Bah { fn boo(&self); }
// x implements Boo and Bah
x.boo() // Error
Boo::boo(x) // OK
x.boo::<Boo>() // Also OK
```

<!--
When looking up methods, the compiler will automatically check all traits implemented on the receiver type to see if they have a method of that name.
If there's more or less than one impl, you'll get an error. You can avoid the error on multiple applicable impls by using the turbofish syntax, as shown in the example.

You can import traits with `as _` to have them in scope for method lookup, but avoid putting them in the type namespace, which also means they cannot be named in the code.
-->
---
## Traits

Generics vs. Associated Types

```rust
trait ReturnGeneric<T> {
    fn as_return(self) -> T;
}

trait returnAssociatedType {
    type T;
    
    fn as_return(self) -> T;
}
```

<!--
Sometimes we want to have a generic return type. But now we have two exclusive options:
* If we are generic over the return type (as in `AsRef` or `Into`, the trait solver needs to do more work, as there may be multiple implementations on a type. This might lead to the dreaded turbofish to disambiguate the result type.
* On the other hand, if we use an associated type, we can have but one implementation per type. The nice thing here is that the trait solver knows the result type from our implementation, and there is no ambiguity.

This distinction will become relevant later.
-->
---
## Traits

`dyn` compatible traits

- Lets us cast a type to a trait object (e.g. `&dyn Foo`, `Pin<Box<dyn Foo>>`)
- all traits bounds must be dyn-compatible (notably, `Sized` isn't)
- cannot have associated constants
- no associated type may contain generics
- restrictions around dispatchable functions

<!--
By default, Rust monomorphizes our code. However, we can also choose the trait implementation at runtime with trait objects. Since edition 2018, we have the `dyn` keyword to mark those. They are implemented as fat pointers, containing a base pointer and a pointer to a so-called vtable, which is a data structure the compiler automatically generates for us and which contains the information on where to find the relevant trait functions. The implementation details of this will fill its own talk, so if you're interested, feel free to chat me up later.

In general, some restrictions apply. Notably, the compiler won't do type checks at runtime, so we cannot have any associated types with generics. The compiler cannot do associated constants through trait dispatch because of the unclear type at runtime, and the vtable won't contain any functions that have any type parameters (because we wouldn't know how to dispatch on those either) or an opaque return type because again, the compiler may not know the resulting type at compile time. There are also some restrictions regarding the receiver type, which I won't get into. We call functions that follow those restrictions "dispatchable".

Now with that all said, let's go into actual standard library traits.
-->

---
## Arithmetic

Let's start simple:

```rust
let x = a + b * c - d / e % f;

// is the same as

use std::ops::{Add, Div, Mul, Rem, Sub};
let x = a.add(b.mul(c)).sub(d,div(e).rem(f));
```

<!--
Note that it is considered unidiomatic to implement those traits unless the types really work as arithmetic.

Case in point: In the runup to Rust 1.0.0, I suggested implementing `Div` on `Path`s. It was considered too cute and thus denied.

On the other hand, the arithmetic traits are often implemented for numbers on one level of references for self, the right-hand-side or both. This is so we don't need to besmirch our formulae by random dereferences.
-->

---
## Bit-operations

```rust
a << b | c ^ d & e >> f

// is the same* as

use std::ops::{BitAnd, BitOr, BitXor, Shl, Shr};
a.shl(b).bitor(c.bitxor(d).bitand(e.shr(f)))
```

<!--
We also have bit operations that are mostly what you'd expect.

There are three things of note here:
1. The circumflex is an xor, not an exponential
2. The bitand and bitor operators unlike the double ampersand and pipe are not short-circuiting
3. The shifts have some custom typechecking logic so that the output type is expected to be the type of the receiver, although the Shl and Shr traits don't require this
This can be relevant when replacing the operator with a method call, as the compiler may require a type annotation in some cases.

Like the arithmetic traits, the bit ops traits are implemented for numbers on references for self, the right-hand-side or both.
-->

---
## Assigning Operators

All of the aforementioned operators have -`Assign` variants, e.g.

```rust
x += 1;
y ^= x;

// is the same as

use std::ops::{AddAssign, BitXorAssign};
x.add_assign(1);
y.bitxor_assign(x);
```

<!--
The assign variants take a mutably borrowed self and a right-hand side.

Unlike their plain sibling traits, assign variants are usually not implemented for references.
-->

---
## Equality

```rust
if a == b || c != d { .. }

// is the same as

use std::cmp::PartialEq;
if a.eq(b) || c.ne(d) { .. }
```

<!--
Here we're dealing with equality. If you've been listening to your math teacher, equality relations must be
* symmetric (a == b => b == a)
* transitive (a == b && b == c => a == c)
* consistent (a != b) == !(a == b)

Unsafe implementations may not rely on those properties for soundness! Otherwise implementing those traits would be unsafe.

There's also the `Eq` trait to mark types whose `PartialEq` implementation is reflexive (for all a, a == a).
-->

---
## Equality

```rust
#[derive(Eq, PartialEq)]
struct Foo(u64, bool);

// this generates roughly
impl PartialEq for Foo {
    fn eq(&self, rhs: &Self) -> bool {
        self.0 == other.0 && self.1 == other.1
    }
}
impl Eq for Foo {}
```

<!--
We can derive PartialEq and Eq or manually implement them.
-->

---
## Equality

```rust
#[derive(PartialEq, Eq)]
struct AllEqual;

struct NotEqual;
impl std::cmp::PartialEq for NotEqual {
    fn eq(&self, rhs: &Self) -> bool { false }
}
```

<!--
The following type exemplifies a degenerate, but still correct case. All `AllEqual` instances are indeed equal.

Quiz question: Could we correctly implement `Eq` for `NotEqual`? Would it be unsafe to do so?
-->

---
## Ordering

```rust
if a < b || c >= d { .. }

// is the same as

use std::cmp::PartialOrd;
if a.lt(b) || c.ge(d) { .. }
```

<!--
This trait implements an order over a type. Implementing it will usually require the following properties:
* antireflexivity (!(a < a))
* antisymmetry (a < b => b > a)
* transitivity (a < b && b < c => a < c)

Again, no unsafe code may rely on those properties, as the traits are safe to implement.
The `Ord` trait marks an ordering as a total order.

Note also that implementations should be careful to keep the equality and ordering implementation consistent.
-->

---
## Hashing

```rust
// in `std::hash`
pub trait Hash {
    fn hash<H: Hasher>(&self, state: &mut H);
}

pub trait Hasher {
    fn write(&mut self, bytes: &[u8]);
    fn finish(&self) -> u64;
}
```

<!--
We have two traits related to hashing, the Hash and Hasher trait. This lets us decouple the data to hash from the hash algorithm. This lets us write 1 Hash implementation per type we want to hash and 1 Hasher implementation per hash algorithm we want to use. Notably, the standard library's hasher function is relatively slow, but offers good hashdos protection. The Hash trait is usually derived, as is the `PartialEq` trait.

There is a relation between these traits, in that if two values are equal according to PartialEq, they should also be hashed to the same hash value. Otherwise HashMap/HashSet lookup will fail to find values or return wrong values. Implementations may not require this relation between `Hash` and `PartialEq` for safety though, because both are safe to implement. Also it is acceptable if two unequal values return the same hash.
-->

---
## Hashing

```rust
pub struct DegenerateHasher;

impl std::hash::Hasher for DegenerateHasher {
    fn write(&mut self, bytes: &[u8]) {}
    fn finish(&self) -> u64 { 42 }
}
```

<!--
This is a valid implementation of Hasher. It fulfils the contract of Hash. Also it is probably the fastest Hasher implementation one can build. Still, it is a very poor implementation, and using it will lead to bad performance, because it hashes all inputs to the same value, thus making HashMaps and HashSets doing a linear scan on lookup and inserts, which turns inserting multiple values into a quadratic operation.
-->

---
## Cloning and Copying

- `std::clone::Clone::clone` creates a new instance of with the dame data as the given value
- This does *not* guarantee deep cloning, internal references _may_ be shared (e.g. via `Arc`)
- The `clone_from` method may be overridden to reuse existing resources
- `std::marker::Copy` is a marker trait to guarantee that the clone is always a bytewise copy
- Both are usually derived. Both are in the standard library *prelude*

<!--
Clone is also known as the "making borrow checker happy" trait. It guarantees not creating mutable shared references. Usually it will do a deep clone of the data, unless inner types share data over their `Clone` implementation.

Copy is a marker trait to declare that cloning is a bytewise copy. If you derive Copy, you should also derive Clone, because there is no room for customizing a bytewise copy.

Clone is rarely manually implemented, unless there is something special to do (e.g. resources to manage, ref counters to increment, etc.)
-->

---
## Dropping

- We can `drop` any value, no matter whether it implements `std::ops::Drop`
- Calling `Drop::drop(_)` directly is an error
- The trait allows us to specify the behavior at destruction of an object
- Only the types that manage their own memory need to implement `Drop`
- Implementing `Drop` is incompatible with implementing/deriving `Copy`
- Don't panic! Or at least don't panic twice

```rust
impl Drop for Foo {
    fn drop(&mut self) {
        if std::thread::panicking() { return; } // avoid drop → panic → drop
        //TODO
    }
}
```

<!--
The drop trait lets us customize behavior on destruction of a value. This is mostly interesting for types that manage resources, such as
* collections, smart pointers and other memory-allocating types, as well as file or network handles
* guard types (e.g. std::mem::DropGuard, still nightly only, but also MutexGuard)
-->

---
## Conversions

- `std::convert::From` / `Into`
- `std::convert::TryFrom` / `TryInto`
- `std::convert::ÀsRef` / `AsMut`
- `std::str::FromStr`
- `std::iter::FromIterator`

<!--
Rust's standard library is very keen on conversions. Which is understandable, because those come up a lot in code. Let's go through them one by one.
-->

---
## `std::convert::From` and `Into`

- Infallible conversions
- Blanket `impl<T, U: From<T>> Into<U> for T`
- Therefore implement `From`, use `Into` in generic bounds

<!--
Those are the traits for direct conversions between types that cannot go wrong.
There is no guarantee that the conversion will be cheap.
The traits are not defined to be transitive, so if you have type B that implements From<A> and type C that implements From<B>, you don't automatically get From<A> for C.

One can use `impl Into<T>` generics in arguments, but it is not commonly done, because it usually hurts compile time. I personally might use it in short, rarely used functions.
-->

---
## Interlude: `From` vs. `FromIterator`

- `FromIterator<A>` is generic over `T: IntoIterator<Item = A>`
- A comparable `From` implementation for `Vec` would look like this:

```rust
impl<A, T: IntoIterator<Item = A>> From<T> for Vec<A> {
    fn from(t: T) -> Self {
        Self::from_iter(t)
    }
}
```

- However, this would mean a lot more possible type resolution ambiguity w.r.t. `From`
- Having `.collect()` work with any `FromIterator` impl expresses intent better

<!--
We have a `FromIterator` for the special case of building a collection from an iterator. While we could use `From<T: IntoIterator>` instead, having this trait allows us to clearer express the intent, and
-->

---
## `std::convert::TryFrom` and `TryInto`

- Fallible conversions
- don't panic in `From`, implement `TryFrom` instead
- Error type implementation defined. Don't use ! here, use `From` instead
- As with `Into`, there's a blanket `impl<T, U: TryFrom<T>> TryInto<U> for T`
- Also, there's a blanket `impl<T, U: Into<T>> TryFrom<U> for T`
- The trait makes no statement about the cost of the operation

<!--
Sometimes conversions are only partially defined, or less academically, those conversions can err.

The TryFrom and TryInto traits define such conversions. They are blanket defined for infallible conversions. For historical reasons, the error type is defined as `Infallible` and not the never type, even when it is stabilized in the next stable version.

The biggest benefit of this trait compared to a plain function is that it can be used in generics.
-->

---
## Interlude: `FromStr` vs. `TryFrom`

- The `FromStr` trait is far older than the `TryFrom` trait
- Even if it wasn't, we couldn't change the bound of the `str::parse` method
- Besides, there is some documentation value to having the trait mean parsing
- The trait does not guarantee, but suggest roundtrip-ability with `Display` if possible

```rust
assert_eq!(format!("{foo}").parse::<Foo>(), foo);
```

<!--
As with `FromIterator`, the `FromStr` trait doesn't introduce any additional invariants compared to `TryFrom`. However, there is the suggestion of roundtrip-ability with regards to `Display`, but this is not always possible. With that said, the trait is only useful for parsing a string that's already completely in memory, so it e.g. cannot stream.
-->

---
## `std::convert::AsRef` and `AsMut`

```rust
pub trait AsRef<T: ?Sized> {
    fn as_ref(&self) -> &T;
}
```

- Infallible non-consuming conversions between reference types
- `MyType: AsMut<T>` requires `MyType: AsRef<T>`
- Don't require sized types behind the reference (`T: ?Sized`)
- Don't have any blanket implementations

<!--
This works like `Into` but for references. It is meant for *cheap* reference-to-reference conversions. Note that unlike `From` and `Into` there is no inverse, because it can be its own inverse (`T: AsRef<U>` vs. `U: AsRef<T>`)

As with `Into`, the target type is generic, so there can be multiple `AsRef` implementations for one type. For example there are impls for OsStr with a `T` of `Path` and `OsStr` (no blanket impls, remember?).
-->

---
## `std::borrow::Borrow` and `BorrowMut`

```rust
pub trait Borrow<Borrowed: ?Sized> {
    fn borrow(&self) -> &Borrowed;
}
```

- Basically the same as `AsRef`
- There is also `Borrow_mut`, which is the equivalent to `AsMut`
- *but* both require transparency with regards to `Eq`, `Ord` and `Hash`

<!--
The astute readers here will have noticed that apart from the names being different, the traits are exactly the same. So the difference is in the *meaning* of the trait. The `Borrow` trait has some additional requirements regarding the relation to the `Eq` and `PartialEq`, `Ord` and `PartialOrd` as well as `Hash` trait.
-->
---
## `std::borrow::Borrow` and `BorrowMut`

```rust
assert_eq!(x == y, x.borrow() == y.borrow());
assert_eq!(x < y, x.borrow() < y.borrow());
assert_eq!(x.hash(), x.borrow().hash());
```

- This lets us use it in collections, e.g. `HashSet`, `std::borrow::Cow`
- collections may return wrong results for types implementing `Borrow` wrongly
- but may never rely on the implementation correctness for safety!

<!--
I have put those requirements in Rust pseudo code here.
-->

---
## `std::borrow::ToOwned`

- generalizes `Clone` to work not only as `&T → T`, but also e.g. `&str` to `String`
- this means there is a blanket impl for `T: Clone`, plus some other impls
- used as bound in `std::borrow::Cow`

<!--
Clone is only defined on the type itself, not allowing to return a different type. So we have the `ToOwned` trait, which is a superset of `Clone`, adding some extra conversions between e.g. `str` and `String`.

This is also basically the inverse to `Borrow` (and also used as such in the Cow type, which by the way is still underused in my not so humble opinion).
-->

---
## `std::ops::Deref` and `DerefMut`

- Is used to allow customizing dereference operations for smart pointer implementations (e.g. `Box<T>`)
- Inserted by the compiler in many circumstances ("Deref coercion")
- Unlike `AsRef`, the target type is an associated type: We can only have one `Deref` impl per type
- Can be misused to make one type pose as another – note that this is often seen as an antipattern
- Using it in trait bounds is not recommended, because it clashes with deref coercion

<!--
`Deref` is basically like `AsRef` … again. But the main difference is that its operation is inserted by the compiler. Therefore it cannot be generic over the target type, but uses an associated type for that. Otherwise the compiler couldn't know which type to choose in most cases.

Implementing it for a type is a powerful technique, as it allows method calls on the type a value dereferences to. However, it can also hide what's happening, so for the sake of being able to reason about the code, it is discouraged in the general case. An exception is custom smart pointers and/or guard types which should dereference to what they're pointing to or guarding.
-->

---
## Reminder: `AsRef` vs. `Borrow` vs. `Deref`

- `AsRef` is the most general. Its generics allow T x U implementations, and it requires no correctness invariants
- `Borrow`'s implementation is exactly the same as `AsRef`, but it carries additional requirements around hashing, equality and ordering
- `Deref` allows only one implementation per type, and should be used for guards or smart pointers

---
## `Send` and `Sync`

- `Send` is on types that can be moved to another thread
- `Sync` is on types that can be shared between threads
- There is a blanket `impl<T: Send> Sync<&T> for T {}`
- `Send`/`Sync` are automatically implemented for any type that doesn't contain any non-`Send`/`Sync` part
- You can opt out on stable Rust using `PhantomData`:

```rust
use std::marker::PhantomData;
// on nightly e.g. `impl !Send for MyType {}`
type NoSend = PhantomData<*mut u8>;
type NoSync = PhantomData<Cell<u8>>;

struct MyType(usize, NoSend, NoSync);
```
<!--
This stuff is what underlies Rust's thread safety, as it encodes in the type system whether a type can be moved or shared across threads.
The Sync blanket impl cleverly defines the relationship between the two threads: If you can send a reference, you can share read access.
-->

---
## std::default::Default

- Gets you a useful default value of that type – if there is one
- Useful in trait bounds (e.g. `Option::unwrap_or_default()`)
- Also often used in initialization, e.g. `let foo = Foo { bar: 42, ..Foo::default() };`
- Can be derived if all your fields impelment `Default`

<!--
The default trait is there to allow us to create a usable value of some type. It is implemented for many standard library types, and implementing it will unlock some methods e.g. on the Option type. Also if you have Default implemented on all field types of a struct, you can derive Default for that struct.
-->

---
## Iterator traits

- Rust owes most of its functional-language-ness to `Iterator`s
- For `for` loops, first `IntoIterator::into_iter()` is used on the iteratee, then `Iterator::next()` is called on each iteration
- `DoubleEndedIterator` lets us iterate from both sides
- `FusedIterator` guarantees the iterator will always return `None` once it returns `None`
- `ExactSizeIterator` adds a `len` and `is_empty` method and guarantees* `size_hint()` will return `(len, len)`

<!--
Iterators are one of the places where we have lazy computation – many methods only set up the iteration and only a few actually iterate. `IntoIterator` is used in `for` loops and also as a trait bound – having a trait different from `Into` has benefits for trait lookup because we don't need to search through all the other possible `Into` implementations, so we avoid running into ambiguity.

The DoubleEndedIterator trait allows us to iterate from both sides. It also allows us to call `.rev()` to reverse the iteration. Interestingly, `rev` is defined on the `Iterator` trait, but only for types that also implement `DoubleEndedIterator`.

Iterators have a `fuse` call which means that once iteration is done which is signalled by `next` returning `None`, they will always return `None`. The `FusedIterator` trait can mark iterators to allow the `Fuse` type to specialize for types that need no further checks to uphold this. If you implement `FusedIterator` on a type that is not by itself fused, your code may run into an infinite loop, return wrong results, etc. Fusedness is not a safety guarantee though, so you may not rely on that property in unsafe code.

ExactSizeIterator allows us to get the number of items this iterator holds. It requires that the `size_hint` is in accordance with the result of the supplied `len()` method. This requirement is, again, not a safety guarantee. There is the unstable and unsafe `TrustedLen` iterator trait for that.
-->
---
## Functions and Closures

There are three traits to handle calling functions and closures:

- `FnOnce` is for a closure that will be consumed by the call (e.g. `move || ()`)
- `FnMut` is for a closure that binds by mutable reference (e.g. `|| *x += 1`), also requires `FnOnce`
- `Fn` is for closures binding immutably (e.g. `|| x + 1`), also requires `FnMut`
- `fn` (note the lowercase `f`) is *not* a trait, but a special type family for function pointers

You generally don't manually implement any of those traits, but you can use them in generics.
<!--
There's a trick where you can implement `Deref` into a closure type that lets you call your own types. However, this is quite hacky, and I would avoid it.
-->
---
## std::future::Future

```rust
pub trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}
```

- The `Future` trait is for things that can be `poll`ed for a result which may or may not be ready at this time
- Most async engines (which we should probably call "future pollers" instead) also require `Send + Sync + 'static`

<!--
Async Rust is great because it allows us to have a fairly minimal language surface that lets us build all those state machines to run concurrently.

The standard library also contributes a few traits. Notably the future trait is for computations that may finish in the future. Poll by the way is an enum that can be Pending or Ready containing the output.

Note that the receiver has a custom type here. I won't go into pinning here though.
-->

---
## std::future::IntoFuture

```rust
pub trait IntoFuture {
    type Output;
    type IntoFuture: Future<Output = Self::Output>;
    fn into_future(self) -> Self::IntoFuture;
}
```

- The `IntoFuture` trait is like `IntoIterator`, but for futures instead of iterators. It gets called automatically on `.await`
<!--
All your async fns implicitly return a custom type that implements `IntoFuture`. The only cleverness here is the binding of the IntoFuture type to a Future trait with the right output. This is the main reason we cannot use `Into` here, because we don't know what type will the future be turned into.

There is a blanket implemenetation for all futures.
-->

---
## `std::string::ToString`, `std::fmt::Display` and `Debug`

- Rust's formatting infrastructure relies on `Display` and `Debug` to create string representations of our values
- `Debug` is often derived. Be sure to manually write an impl for all secrets-holding types
- `Display` is for user-facing code, `Debug` for internal code
- `ToString` has a blanket implementation for `T: Display` and shouldn't be implemented directly

```rust
use std::fmt::{Debug, Formatter, Result};
impl Debug for MyType { // Display is mostly the same
    fn fmt(&self, f: &mut Formatter<'_>) -> Result {
        if f.alternate() {
            // you can supply alternate outputs for Debug (`:#?`) and others
        } else {
            // or not
        }
    }
}
```

<!--
For manually implementing Debug, Formatter has a few helpful methods such as .debug_{struct, tuple, list, map, set}

Otherwise, feel free to provide an alternate output format people
-->
---
## Alternative `std::fmt` Display Traits

- Your types can implement various traits to format differently based on format specifiers
- The traits look exactly like `Display` and `Debug`
- You can choose the alternate form with a hash sign, e.g. `{:#o}`

|Format  | Trait      | Normal Form | Alternative Form |
|--------|------------|-------------|------------------|
| `{:o}` | `Octal`    | 42          | 0o42             |
| `{:x}` | `LowerHex` | 5e          | 0x5e             |
| `{:X}` | `UpperHex` | 5E          | 0x5E             |
| `{:b}` | `Binary`   | 10010111    | 0b10010111       |
| `{:e}` | `LowerExp` | 10e5        |                  |
| `{:E}` | `UpperExp` | 10E5        |                  |
| `{:p}` | `Pointer`  | 0x450f23ae2 |                  |

<!--
There are basicall three kinds of alternative forms: Different number bases for integers, exponential forms for decimal numbers and floats, and pointers.

If they are not implemented, the formatting machinery will use the `Display` implementation as a backup.
-->

---
## Alternative `std::fmt` Display Traits

```rust
use std::fmt::{Formatter, Octal, Result, Write};

struct Pair<T>(T, T);

impl<T> Octal for Pair<T> {
    fn fmt(&self, f: &mut Formatter<'_>) -> Result {
        write!(f, "({:o}, {:0})", self.0, self.1)
    }
}
```
<!--
For example, you could have a type that is a pair of values, and by implementing the alternative form traits, you could have the output customized for number types.
-->

---
## `std::error::Error`

- Implement this on your error types
- The trait implies `Debug` and `Display`
- You don't need to implement any of the methods
- Many frameworks exist to do error handling for you

<!--
The `Error` trait has more deprecated methods than non-deprecated ones. It is still good practice to implement it for your error types. The trait is often used with `dyn` trait types.

Usually people will use an error library (thiserror, anyhow or recently eros) to implement errors for them.
-->

---
## `std::marker::Sized`

- Most types are `Sized` by default
- Unsizedf types fall in three categories:
  - Slices (or types that contain them as a last member)
  - dyn Trait objects
  - extern unsized types
- All trait bounds have `Sized` implicit in them unless you opt out with a `?Sized` bound
- There's also unstable extensions (`MetaSized` and `PointeeSized`)

<!--
Rust with very few exceptions keeps track of the sizes of values in our programs:
With slices, you usually have them behind a reference, which is internally a fat pointer containing a base pointer + length
With dyn trait objects, you get a different kind of fat pointer – you get a pointer to a vtable instead of the length
With extern unsized types, we only get a pointer, and we will have to call into external methods to handle everything
MetaSized is for both dyn traits and slices – the compiler knows how to obtain the size from the pointer metadata at runtime
PointeeSized is for extern unsized types

"Unsized" doesn't mean the value is not in memory, it just means we don't know the size at compile time.
-->

---
## Notable Mentions

There are other traits that I've not discussed for lack of time, e.g. `std::pin::Unpin`, `std::future::AsyncDrop`, `std::str::Pattern`, `std::alloc::GlobalAlloc`.

Feel free to read up on them.

---
## That's all, folks!

Questions?
