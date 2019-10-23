---
title: Towards Overflower 1.0
---

The [overflower] crate contains a helper library and procedural attribute macro
to allow specifying how overflow should be handled. Since its inception, it has
relied on specialization to change just the integer arithmetics while leaving
the other operations basically unchanged.

Well, not really unchanged. See, procedural macros know very little about the
types involved. This is why specialization is required: We simply implement the
overflower helper traits that the macro uses to replace the `std::ops` traits
in terms of the latter for all types we don't want to change behavior for.

The first version of overflower was [released] in June 2016, which means it's
now more than three years old. Specialization is still unstable, so there's no
way to use it anywhere but nightly Rust. That got me thinking: "Could we put
the specialization behind a feature and hack together a workaround for stable"
and it ocurred to me that I could actually do this.

The sole problem we solve with specialization is that without it, the helper
traits won't be implemented for all types that may be used in the code. But
perhaps if we offer a way to simply implement the trait, we could use let the
user take care of it?

After a good deal of experimentation, I now have what I would call a working
prototype. This version of overflower adds an `impls!` macro that you can use
to implement the overflower traits for all your types that might implement any
`std::ops` traits. It's used like this:

```rust,ignore
// this will expand to all the trait impls
overflower::impls!(MyType<'a> : 'a; all);
// or to be more explicit:
overflower::impls!(MyType<'a> : 'a; add sub mul div rem shl shr neg add_assign
    sub_assign mul_assign div_assign rem_assign shl_assign shr_assign);
```

I thought about writing a `proc_macro_derive` for it, but that will likely cost
even more compile time and the only benefit it offers is that you don't have to
write out your type twice. Also I couldn't use a derive to implement the traits
for `std` types (`Wrapping`, `NonZero*`, `String`, `Cow<'a, str>`).

There's another change that will require you to slightly change your code: The
`#[overflow(_)]` attribute and the helper traits are now directly importable
from `overflower`, so the `overflower_support` crate is now an implementation
detail you don't need to care about any longer.

Also in order to keep the dependencies lean, I added a `proc_macro` default
feature that you can disable if you just want to implement the overflow
handling traits for your types. So for those cases, you can depend on
overflower with `overflower = { version = "0.9", default-features = false };`
in your `Cargo.toml`'s `[dependencies]` section.

If you do this, you'll likely want overflower as an optional dependency, so
you'd use: `overflower = { version = "0.9", default-features = false,
optional = true };`. Then in your code, you can use a `#[cfg]` to let users opt
in to the trait implementations:

```rust,ignore
#[cfg(feature = "overflower")]
overflower::impls!(VeryInterestingType; div);
```

Please keep in mind that those trait implementations only forward to the
`std::ops` traits, they don't handle overflow at all, regardless of any
`#[overflow]` attribute.

The code is in the [`overflower-0.9`] branch should you want to play with it.

# Roadmap

I feel that overflower now works mostly the way I want it. Sure, some macros
could probably be simplified, but apart from a little polish here and there, I
see no big changes before we get that shiny 1.0.0. Onwards!

⚘

[overflower]: https://docs.rs/overflower
[released]: https://llogiq.github.io/2016/06/24/overflower.html
[`overflower-0.9`]: https://github.com/llogiq/overflower/tree/overflower-0.9
