---
marp: true
author: Andre 'llogiq' Bogus
title: An Option? Take a Slice!
description: The story of how `Option::as_slice` came to be
class: invert
---
<style>
:root {
  background: #000;
}
code { color: #fff7f0; }
</style>
# An Option? Take a Slice!

Andre 'llogiq' Bogus

<!--
Buddy coding Go make a big splash
be a CEO all your code be web scale someday
You got a data race
Big disgrace
Who needs all the five nines anyways

|: We will we will Rust you :|

Buddy writing Java it's all en-
terprisey as heck, will be legacy code someday
You got pointers to chase
Big disgrace
Write an AbstractObjectFact'ryInterface

|: We will we will Rust you :|

Buddy doing Python it's the best
And you're gonna hang out with the AI bros someday
You got startup delays
Big disgrace
But at least your code uses consistent space

|: We will we will Rust you :|

Buddy writing C be all macho
Just gotta git gud
Create less CVEs per day
it segfaults in your face
Big disgrace
Somebody better put you back into your place

|: We will we will Rust you :| 4x
-->

---
# The Problem

(simplified)
```rust
enum MyEnum {
    OneOpt(Option<String>),
    Many(Vec<String>),
}
// ...
let my_iter = match &my_enum {
    MyEnum::OneOpt(option) => option.iter(),
    MyEnum::Many(vec) => vec.iter(),
};
```

<!--
While working at jot.ai in 2021, a colleage came to me with a snippet very alike the one I'm presenting here. He had an enum with one variant containing an option and another with a Vec.

Unfortunately for him, it wouldn't compile.
-->
---
# The Problem

```sh
Compiling [REDACTED] v0.1.23 ([REDACTED])
error[E0308]: `match` arms have incompatible types
  --> src/[REDACTED].rs:31:30
   |
29 |       let my_iter = match &my_enum {
   |  ___________________-
30 | |         MyEnum::OneOpt(option) => option.iter(),
   | |                                   ------------- this is found to be of type `std::option::Iter<'_, String>`
31 | |         MyEnum::Many(vec) => vec.iter(),
   | |                              ^^^^^^^^^^ expected `std::option::Iter<'_, String>`, found `std::slice::Iter<'_, String>`
32 | |     };
   | |_____- `match` arms have incompatible types
   |
   = note: expected struct `std::option::Iter<'_, String>`
              found struct `std::slice::Iter<'_, String>`

For more information about this error, try `rustc --explain E0308`.
error: could not compile `[REDACTED]` (bin "[REDACTED]") due to 1 previous error
```

<!--
Error E308 is something most of you will have seen: The compiler wasn't able to unify two types during inference.

As we know, in Rust each value has exactly one type. But here we try to put two differently-typed iterators into one value.

Since he wasn't completely firm in Rust, I gave him an explanation along with three solutions
-->

---
# Solution 1

```rust
let my_iter = match &my_enum {
    MyEnum::OneOpt(option) => option
        .as_ref()
        .map_or(&[][..], std::slice::from_ref),
    MyEnum::Many(vec) => &vec[..],
}.iter();
```

<!--
This is what a competent Rustacean would write back then. No unsafe code, and though the `as_ref` may look a tad complex, it captures what needs to be done. It still incurs one branch in the option match arm.
-->

---
# Solution 2

```rust
let my_iter: Box<dyn Iterator<Item = &String>> = match &my_enum {
    MyEnum::OneOpt(option) => Box::new(option.iter()),
    MyEnum::Many(vec) => Box::new(vec.iter()),
};
```

<!--
This solution makes use of dynamic dispatch to unify the iterator types. It incurs no extra branch, but one indirection for each iterator call, in addition to one heap allocation.
-->

---
# Solution 2 unboxed

```rust
let my_iter: &mut dyn Iterator<Item = &String> = match &my_enum {
    MyEnum::OneOpt(option) => &mut option.iter(),
    MyEnum::Many(vec) => &mut vec.iter(),
};
```

<!--
Nowadays, we can use dynamic dispatch without using `Box`, so we no longer need the heap allocation, but still incur the dynamic dispatch. This wouldn't have been available back then, because the compiler didn't yet figure out the lifetimes and would have balked at the `&mut _` in the match arms.
-->

---
# Solution 3

```rust
use either::Either;

let my_iter = match &my_enum {
    MyEnum::OneOpt(option) => Either::Left(option.iter()),
    MyEnum::Many(vec) => Either::Right(vec.iter()),
};
```

<!--
This is the option my colleague chose. The `Either` type, supplied by the `either` crate, implements `Iterator` when both `Left` and `Right` types also implement `Iterator` with the same `Item` type. Apart from needing an extra crate, this version incurs a branch on each iteration. On the plus side, the code doesn't allocate, is easy to understand, and here wasn't on the hot path anyway. Besides the either crate already was in our dependencies.
-->


---
# However…

An `Option` is conceptually a collection of zero or one element.

- `None` → `[]`
- `Some(1)` → `[1]`

What if we had an `as_slice` method like `Vec` has?

<!--
But I felt something was missing. I am also the author of the `optional` crate, which implements a few specialized option-like types that brought some performance benefits mainly before niche optimization was a thing, and for my types I remember I had implemented an `as_slice` method, because it was easy to do, and it gave me an iterator for free. So why didn't `Option` have `as_slice`?
-->

---
# Thinking of a better solution…

```rust
let my_iter = match &my_enum {
    MyEnum::OneOpt(option) => option.as_slice(),
    MyEnum::Many(vec) => &vec[..],
}.iter();
```

<!--
if we had an `as_slice` method, we could simply get a slice from an Option directly, which would make the whole example more readable and less arcane. And get rid of all indirection or allocation.
-->

---
# What's in an Option?

```rust
/// The `Option` type. See [the module level documentation](self) for more.
#[doc(search_unbox)]
#[derive(Copy, Eq, Debug, Hash)]
#[rustc_diagnostic_item = "Option"]
#[lang = "Option"]
#[stable(feature = "rust1", since = "1.0.0")]
#[allow(clippy::derived_hash_with_manual_eq)] // PartialEq is manually implemented equivalently
pub enum Option<T> {
    /// No value.
    #[lang = "None"]
    #[stable(feature = "rust1", since = "1.0.0")]
    None,
    /// Some value of type `T`.
    #[lang = "Some"]
    #[stable(feature = "rust1", since = "1.0.0")]
    Some(#[stable(feature = "rust1", since = "1.0.0")] T),
}
```

`Option` is an `enum` of two variants.

<!--
This is the type as it appears in libcore. As we see if we ignore the slew of annotations, it's an enum with two variants and the `None` variant comes first, which will be useful later.

But how does rustc actually put an option in memory?
-->
---
# `Option` In-memory layout

For `T` without a niche

```
          discriminant padding   Some(T)
         .------------.·······.-----------.
None     |     0      |       | <padding> |
         )------------(·······)-----------(
Some(1)  |     1      |       |     1     |
         `------------'·······`-----------'
```
<!--
We get a discriminant, which may be one or four bytes, depending on the architecture (some architectures have problems addressing single bytes).

Then we may have some padding and finally a place to put the value if any.
-->

---
# `Option` In-memory layout

Slight niggle: *Niche optimization*, e.g. Option<&T>

```
         Some(&T) / None
         ,-------------.
None     |    null     |
         )-------------(
Some(1)  |  0x60517109 |
         `-------------'
```

<!--
Rust has this thing called niche optimization, where a bit pattern that cannot appear in a valid value will be used to denote `None'. In the case of references, that bit pattern is `null` (usually zeroes).

In this case, we can still get the comparison with the None pattern without a branch (on x86 for example that'll be a `sete` instruction)

Now let's compare that to how a slice looks in memory
-->

---
# `Slice` In-memory layout

```
             ptr         length
         ,------------.-----------.
&[]      |  not null  |     0     |
         )------------:-----------(
&[1]     |     o      |     1     |
         `-----+------^-----------'
               |    ,---.
               `--->| 1 |
                    `---'
```

<!--
As we can see, the length of the slice is exactly the discriminant of the `Option`. If the length is zero, the type's invariants dictate that the pointer is non-null, so when Rust constructs an empty slice, the pointer will be set to a "dangling" value, that is it doesn't point anywhere useful, but it is not null.

The specific position the pointer points to is "don't care" for our intents and purposes, so long as the pointer is correctly aligned and non-null.
-->

---
# A plan

In both cases, we can get the discriminant without a branch

Then we can call `std::slice::from_raw_parts(some_value, discriminant)`, which will also basically become a no-op.

...if we could get `some_value`...

Can we?

---
# A plan

* The `offset_of!` macro back then didn't support enums
* But if we have a `Some(T)`, we could calculate the offset via `opt.as_ref().byte_offset_from(&opt)`
* But we cannot get a `Some(T)` in generic context (our `Option` might be `None`)
* What if there was a `U` with the same layout of `T` that we could get?

<!--
The first thing I looked at was whether we already had some method of getting the offset. Unfortunately, the `offset_of!` macro, which was designed for this very purpose was still incomplete back then, notably lacking the capabilities to work with enums at all.

So I could either wait or choose a different way to get the offset. If I had `&Some(value)`, I could get the offset by getting a pointer to the `value` and subtracting a pointer to the `Option`.

The problem was that our `Option` could be `None`, leaving us without a `value`. So either I would have to ask users to supply `Some(value)` as an extra argument that would then be removed at compile time, or I would have to find another way.
-->

---
# Searching for `U`

Rust has a few special types to deal with memory.

One of them is `MaybeUninit`.

- `mem::size_of::<MaybeUninit<T>>() == mem::size_of::<T>()`
- `mem::align_of::<MaybeUninit<T>>() == mem::align_of::<T>()` (*)
- We can get one with `Some(Maybeuninit::uninit())` at zero runtime cost

<!--
MaybeUninit fit the bill

I added the asterisk because the folks who later founded the opsem group decided that they don't want to rule out ABIs where this property isn't guaranteed even if no niche optimization takes place.

But for the moment, all looked fine, so I prepared to get this implementation into the standard library.
-->

---
# API Change Proposal

[ACP #150](https://github.com/rust-lang/libs-team/issues/150)

- suggests `Option::as_slice` and `Option::as_slice_mut`, along with
`Option::<&T>::into_slice`
- `as_slice_mut` was renamed to `as_mut_slice` for consistency
- `into_slice` was dropped after finding no use for it in the wild

<!--
The way to introduce a library feature into Rust's standard library is to make an ACP and debate for the feature's inclusion. I did so fairly convincingly, and had an implementation ready, so the debate was more on the details.
-->

---
# The first implementation (by me)

```rust
const fn get_some_offset() -> isize {
    if mem::size_of::<Option<T>>() == mem::size_of::<T>() {
        // niche optimization means the `T` is always stored at the same position as the Option.
        0
    } else {
        assert!(mem::size_of::<Option<T>>() == mem::size_of::<Option<mem::MaybeUninit<T>>>());
        let some_uninit = Some(mem::MaybeUninit::<T>::uninit());
        // SAFETY: This gets the byte offset of the `Some(_)` value following the fact that
        // niche optimization is not active, and thus Option<T> and Option<MaybeUninit<t>> share
        // the same layout.
        unsafe {
            (some_uninit.as_ref().unwrap() as *const mem::MaybeUninit<T>)
                .byte_offset_from(&some_uninit as *const Option<mem::MaybeUninit<T>>)
        }
    }
}
```

<!-- walk through each line, we've got time -->

---
# The first implementation (by me)

```rust
#[inline]
#[must_use]
#[unstable(feature = "option_as_slice", issue = "108545")]
pub fn as_slice(&self) -> &[T] {
    // SAFETY: This is sound as long as `get_some_offset` returns the
    // correct offset. Though in the `None` case, the slice may be located
    // at a pointer pointing into padding, the fact that the slice is
    // empty, and the padding is at a properly aligned position for a
    // value of that type makes it sound.
    unsafe {
        slice::from_raw_parts(
            (self as *const Option<T>).wrapping_byte_offset(Self::get_some_offset())
                as *const T,
            self.is_some() as usize,
        )
    }
}
```

<!-- walk through each line, we've got time -->

---
# Usage in the compiler

In [#108678](https://github.com/rust-lang/rust/pull/108678), I used `as_slice` within the compiler, changing e.g.

```rust
let arg_tys = match arg_ty {
    None => &[],
    Some(ref ty) => slice::from_ref(ty),
};
```

to

```rust
let arg_tys = arg_ty.as_slice();
```

<!--
I added another Pull Request to use the new method in the compiler to simplify some code.

All was good. Now we could end the talk here, right?
-->

---
# However…

My implementation was soon after ([#108623](https://github.com/rust-lang/rust/pull/108623)) deemed potentially unsound and replaced with an implementation that would fall back to a slower operation if the compiler couldn't prove the guessed address was equal to the payload reference:

```rust
let payload_ptr: *const T =
    if let Some(payload) = self {
        payload
    } else {
        let self_ptr: *const Self = self;
        // SAFETY: `SOME_BYTE_OFFSET_GUESS` guarantees that its value is
        // such that this will be in-bounds of the object.
        unsafe { self_ptr.byte_offset(Self::SOME_BYTE_OFFSET_GUESS).cast() }
    };
```

I'm generally not too happy about silently regressing performance, so...

<!--
As I alluded before, the opsem folks later decided that they did not want to guarantee that `MaybeUninit`, despite being `#[repr(transparent)]` would have the same layout (including niches) as its component value.
For the record, I'm not a sore loser, but I think correctness should not preclude performance.
-->

---
# Intrinsics Motiviation

I did not like the perf regression. And the `offset_of!` macro implementation was mired in syntax discussions. It was unclear when it would be ready.

So I decided to create an `option_payload_ptr` *intrinsic* and change the implementation to use that until the `offset_of!` macro was ready to work with enums.

An *intrinsic*  is a function that is just specified as a signature in the standard library, but the real implementation is in the compiler.

[#109179](https://github.com/rust-lang/rust/pull/109179)

<!--
Implementing an intrinsic sounds very involved and can be very much so. Luckily, this one simply makes a MIR place projection available where the language wouldn't, which made the implementation fairly easy, with a few pointers to know where to look.
-->

---
# Intrinsic Ingredients

- The intrinsic definition itself
- Some typechecking code to give a signature to the type checker
- Code generation – in this case, just some MIR

<!--
From my work on clippy, I know a small bit about MIR, the Rust compiler's Middle Intermediate Representation. It has a thing called "place projection", which encodes things like field accesses. I needed to create a field projection to an enum variant field no matter whether I had that variant. Luckily, all the variant checking is done in the language before it gets compiled to MIR, and constructing the required field projection is quite tractable.
-->

---
# The intrinsic definition itself

```rust
#[cfg(not(bootstrap))]
/// This method creates a pointer to any `Some` value. If the argument is
/// `None`, an invalid within-bounds pointer (that is still acceptable for
/// constructing an empty slice) is returned.
pub fn option_payload_ptr<T>(arg: *const Option<T>) -> *const T;
```

<!--
As we can see, the intrinsic definition is just a plain function signature without a body.

What we cannot see here is that the signature is just for documentation. The only thing the compiler takes from it is the function name. It would even work if we had omitted the arguments and return type. This is because intrinsics are exempt from signature type checking, because some of them need to do weird things to their type, such as accepting arguments of multiple types – but only some of them.
-->

---
# Creating the signature for type checking

This constructs a signature of `fn(*const Option<T>) -> *const T`.

```rust
sym::option_payload_ptr => {
    let option_def_id = tcx.require_lang_item(hir::LangItem::Option, None);
    let p0 = param(0); // one unbounded type parameter `T`
    (
        1, // ← the number of type arguments, in this case 1 (`<T>`)
        //   ↓ creates a pointer to a type
        vec![tcx.mk_ptr(ty::TypeAndMut { // ← the input types
            ty: tcx.mk_adt( // ← This is a struct or enum
                tcx.adt_def(option_def_id), // ← `Option`
                tcx.mk_substs_from_iter([ty::GenericArg::from(p0)].into_iter()),
                // ↑ This makes `Option<T>` out of `Option`
            ),
            mutbl: hir::Mutability::Not, // ← `const *` …
        })],
        // ↓ the return type: `*const T`
        tcx.mk_ptr(ty::TypeAndMut { ty: p0, mutbl: hir::Mutability::Not }),
    )
}
```

<!--
To be able to typecheck the intrinsic, there's some code that creates a signature to hand to typecheck. This is one arm of a match that is supposed to return a tuple with the number of generic arguments, a vec of input types and the return type.

The observant folks in the audience will have noted the `#[lang = ".."] annotation when I showed the Option type, which is what gives us this lang item.

In typecheck, we don't really care about the names of generics, we only care about their identity. Which is why our p1 is just another type parameter.

As you can see, this is normal Rust code. It arguably looks more intense because I added some explanatory commenbts.
-->

---
# Lowering the intrinsic

```rust
sym::option_payload_ptr => {
    if let (Some(target), Some(arg)) = (*target, args[0].place()) {
        let ty::RawPtr(ty::TypeAndMut { ty: dest_ty, .. }) =
            destination.ty(local_decls, tcx).ty.kind()
        else { bug!(); }; // This is basically another type check

        block.statements.push(Statement {
            source_info: terminator.source_info,
            kind: StatementKind::Assign(Box::new((
                *destination,
                Rvalue::AddressOf( // This creates a *reference* Rvalue
                    Mutability::Not, // it's an immutable reference
                    arg.project_deeper(
                        &[
                            PlaceElem::Deref,
                            PlaceElem::Downcast( // This unconditionally
                                Some(sym::Some), // selects the `Some` variant
                                VariantIdx::from_u32(1),
                            ),
                            PlaceElem::Field(Field::from_u32(0), *dest_ty),
                        ], // ↑ This selects the only field of that variant
                        tcx,
                    ), // ↑ This call creates the MIR field projection
                ),
            ))),
        });
        terminator.kind = TerminatorKind::Goto { target }; // ← end statement
    }
}
```

<!--
Now that we've typechecked any call to our intrinsic, we need to replace it with the right MIR.

This MIR has three parts: First, we get a "projection" that first dereferences our `*const Option` pointer, then unconditionally interprets this as the "Some" variant, and finally gets the zeroeth field (remember we're counting from zero). Second, we take the immutable address of this field. Finally, we assign that to the result.

Again, this is normal Rust code, and while I wouldn't expect a new rustacean to come up with it, writing such code is tractable thanks to the compiler docs and the very helpful folks on Rust zulip.
-->

---
# Our intrinsic-using `as_slice()`

```rust
unsafe {
    slice::from_raw_parts(
        crate::intrinsics::option_payload_ptr(crate::ptr::from_ref(self)),
        usize::from(self.is_some()),
    )
}
```
<!--
And with that, our `as_slice()` method becomes simpler, directly calling our intrinsic with a pointer to the given `Option`. We won't need to care about offsets, or niche optimizations, because the projection already does the work for us. The length part was changed from an `as` cast to a `usize::from` on request of one reviewer.
-->

---
# Happy End

* `Option::as_slice` was stabilized in version 1.75.0 [#116220](https://github.com/rust-lang/rust/pull/116220)
  It still used the intrinsic I implemented
* @GKFX replaced the intrinsic with `offset_of!` in 1.76.0 [#117525](https://github.com/rust-lang/rust/pull/117525)
* and made the functions const in 1.81.0 [#126711](https://github.com/rust-lang/rust/pull/126711)
* @scottmcm changed the length calculation to avoid compilation overhead [#127748](https://github.com/rust-lang/rust/pull/127748)

<!--
I stabilized the method in version 1.75.0 and notified my colleague that his old problem now had a better solution. He was surprised to say the least.

Fun story, some people were really unhappy about a new intrinsic in stable, so this apparently sped up the development of the `offset_of!` macro, which just a few weeks later gained the ability to unstably select enum variants.
Later the functions were marked `const` because there really wasn't anything holding that back, and the length calculation was simplified to use the discriminant directly instead of calling `is_some()` to speed up compilation.
This method would often be inlined, so this would shave a few microseconds off some builds.
-->

---
# Surprising Fact

`Option::as_slice` is a *negative* cost abstraction.

Unlike a *zero*-cost abstraction, which is as fast as what a competent practicioner would write, it is arguably faster than that.

```rust
opt.as_ref().map_or(&[], std::slice::from_ref)
// vs.
opt.as_slice()
```

<!--
Remember the original version I suggested my colleague – which was the thing a competent practicioner would have written. It took one branch and differed in the base pointer, too, which could sometimes preclude other optimizations.

So as_slice removes that potential overhead. While I doubt that it'll make a measurable difference in practice, I'd like to think that this small change has made a large number of programs infinitesimally faster.
-->

---
# Q & A

> A colleague once asked me advice\
> How to unify `Option` and slice\
> So I built an extension\
> to relieve this tension\
> the resulting performance is nice.
