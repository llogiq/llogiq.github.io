---
title: Footguns in SIMD land
---

If you use the newly stabilized `std::arch` module, you may find a few, well,
surprises:

If you fail to guard your SIMD code with `#[cfg(and(target_arch = "..",
target_feature = ".."))]`, you'll miss out on inlining. There is not even a
warning for this yet (clippy issue #????)

Both 128 and 256 bit types have three variations: One for {4, 8}×`f32`, one
for {2, 4}×`f64` and one bag of bits that can be sliced to various integer
lengths and signedness. For historical reasons, the `__m128`/`__m256` types
denote multiples of `f32`, whereas the `f64`-based types get a `d` suffix, and
the integer types get an `i` suffix. C/C++ programmers may feel at home here.

The type constructors aren't const yet. This means that you cannot have static
`__m128i`s, for example.

The `_mm_set` function and the `__mm_loadu`/`__mm_storeu` functions take their
arguments in inverse order. For example:

```rust
let mut y = [0xf64; 2];
let onetwo = _mm_set_pd(1.0, 2.0);
__mm_storeu_pd((&mut y).as_mut_ptr(), onetwo);
assert_eq!([2.0, 1.0], y); // who'd have thunk?
```

Speaking of which, the `_mm_store_*` functions don't seem to be faster than the
`_mm_storeu_*` functions (at least on my skylake), so it's unclear if defining
a 16-byte-aligned type for the former would be worth the hassle. The same goes
for `_mm256_store_*` and 32-byte-aligned types.

----

What problems have you found with Rust + stdsimd? Discuss on [r/rust] or
[rust-users]!

[r/rust]: https://reddit.com/r/rust
[rust-users]: https://users.rust-lang.org

