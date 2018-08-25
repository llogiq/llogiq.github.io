---
title: Easy `proc_macro_derive`s with `synstructure`
---

Recently, I found myself in the market for some quickcheck. However, there were
custom types, which had no `Arbitrary` implementation. Wondering if someone had
already written a procedural macro to derive it, I found panicbit's
[`quickcheck_derive`] crate. However, to my dismay, it was severely limited in
that it could only derive `Arbitrary` for structs.

"How hard could it be?" thought I, forked the repo and looked at the code,
which appears to be written roughly a year ago. The used `syn` was also quite
old, so I looked around to find if there were some improvements I could use.

I was in luck – what I found was the ['synstructure`] crate which makes writing
procedural macros a breeze (as a testament, the git stats show that I deleted
more lines than I added, despite adding a sizable chunk of new functionality).
Also the crate is well-documented, so you can find examples for most things you
may like to do if you search around for a bit.

`synstructure` gives us a few macros to work with structs and enums. First, we
import the necessary stuff:

```rust
extern crate syn;
#[macro_use] extern crate synstructure;
extern crate proc_macro2;

use proc_macro2::TokenStream;
```

This allows us to write our procedural macro, which has the form
`fn my_derive(s: synstructure::Structure) -> TokenStream` and can be registered
with `decl_derive!([MyDerive] => my_derive);`.

`synstructure::Structure` has many useful methods to construct or match over
values. The `variants()` method returns a slice of `VariantInfo` for all
variants (one for structs, any number for enums), which one can iterate to
generate code for each variants. There is also an `.each(..)` method to
generate matches.

To allow for easy testing, there's a `test_derive!` macro which you feed a
struct or enum plus the expected generated code and it will expand and compile
both and check if they are equal. This looks like the following:

```rust
#[test]
fn test_my_derived_struct() {
    test_derive!{
        my_derive {
            struct MyStruct(u32);
        }
        expands to {
            ..
        }
    }
}
```

Pro-tip: Don't write the expected result by hand. Instead write `fn eek() {}`
and copy the expected result from the error message. See? I didn't even have to
*write* many lines of code!

So if you want to create macros to derive your trait, give `synstructure` a
try. You won't be disappointed.

[`quickcheck_derive`]: https://crates.io/crates/quickcheck_derive
['synstructure`]: https://crates.io/crates/synstructure
