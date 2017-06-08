---
title: Rust Performance Pitfalls
---

Overall, Rust is pretty good for performance. Write the most simple, naive
stuff, and it will usually run within a factor of two from optimized C/C++
code, without any further performance work on the code. With some investment
into optimizations, matching or exceeding C's speed should be possible in most
cases. However, Rust makes some tradeoffs for different reasons than sheer
speed, so here's a handy list of some things that may bite you and how you can
speed them up.

Before we come to the list, please remember that this is only general advice
that may or may not apply to your specific situation. As Kirk Pepperdine always
exhorts: "Measure, don't guess!".

### Ask for `--release`

The first stumbling block is that Rust by default compiles (and runs) in
`debug` mode. This is fast to compile, but does next to no optimizations, so
your code will likely run slow as molasses.

Every now and then, someone logs on IRC, rust-users or /r/rust asking "why is
Rust so slow?" only to be blown away by *how fast* it is once you ask for it.
So use `cargo run --release` to run your code if you want it fast.

I'd like to add that this is actually a good default, letting you test your
code quickly without much hassle, leaving the heavyweight optimizations until
you're sufficiently confident it does the right thing. No need to waste cycle
optimizing a program that doesn't work correctly.

### Unbuffered IO

By default, Rust uses unbuffered File IO. So when you write files, wrap them in
a `BufWriter` / `BufReader`, unless you have good reason to forgo the speedup –
constrained memory, using a custom buffering scheme, specific need for high
write granularity. Also if you read/write the whole thing at once, buffering
won't help you.

Similarly, the default `print!` macros will lock STDOUT *for each
write* operation. So if you have a larger textual output (or input from STDIN),
you should lock manually.


This:

```rust
let mut out = File::new("test.out");
println!("{}", header);
for line in lines {
    println!("{}", line);
    writeln!(out, "{}", line);
}
println!("{}", footer);
```

locks and unlocks io::stdout a lot, and does a linear number of (potentially
small) writes both to stdout and the file. Speed it up with:

```rust
{
    let mut out = File::new("test.out");
    let mut buf = BufWriter::new(out);
    let mut lock = io::stdout().lock();
    writeln!(lock, "{}", header);
    for line in lines {
        writeln!(lock, "{}", line);
        writeln!(buf, "{}", line);
    }
    writeln!(lock, "{}", footer);
}   // end scope to unlock stdout and flush/close buf
```

This locks only once and writes only once the buffer is filled (or buf is
closed), so it should be much faster.

Similarly, for network IO, you may want to use buffered IO.

### Reading lines

Similarly, the `Read::lines()` iterator is very easy to use, but it has one
downside: It allocates a `String` for each line. Manually allocating and
reusing a `String` will reduce memory churn and may gain you a bit of
performance.

This:

```rust
for line in buf.lines() {
    let line = line.unwrap();
    // do something with line
}
```

should become this to remove the extra allocation per line:

```rust
let mut line = String::new(); // may also use with_capacity if you can guess
while buf.read_line(&mut line).unwrap() > 0 {
    // do something with line
    line.clear(); // clear to reuse the buffer
}
```

### `str` vs. `[u8]`

Many operations can be done regardless the string encoding. Yet Rust insists on
using UTF-8 encoding, and will check this on string creation. This is a very
good thing, as it allows us to rely on valid UTF-8 data, which allows Rust to
implement quite fast, yet correct string handling methods, once we've paid the
entrance fee by way of UTF-8 checking.

To get rid of the checks, we can either use bytes directly (usually via
`Vec<u8>` / `&[u8]`) or, if we are absolutely sure the input will be valid
UTF-8, use `str::from_utf8_unchecked(_)` (note that this will require `unsafe`
and break your code in surprising ways should the input not be valid UTF-8).

The [`regex`](https://crates.io/crates/regex) crate has a `regex::bytes`
submodule containing all functions to work with byte slices where `regex` does
with `&str`ings

Most parsing crates work on byte slices already to avoid UTF-8 checking.

I won't give an example here, as the operations are too diverse to hope
covering even the most popular cases.

### Making a Hash of it

A lot has been written about the topic of hashing (including
[by me](/2016/12/08/hash.html))

Like with UTF-8 strings, Rust chooses a sensible, safe default and allows you
to override that choice should you be so inclined. For example, the Rust
compiler uses the [`fnv`](https://crates.io/crates/fnv) crate for the `FNV`
hash function which speedily gets good results with short-ish strings.

Don't do anything before you've measured and have a good idea of your hash key
distribution. Depending on this, you may be able to get some respectable gains,
but the price in carefully choosing the hash function and setting up a good
benchmark regime is steeper than with the other suggestions.

As a baseline check, you can (if your keys implement `Ord`) replace your hash
map/set with a `BTreeMap`/`BTreeSet` and look at the performance. Also in some
cases, the [`ordermap`](https://crates.io/crates/ordermap) provides maps that
improve memory locality (good for fast iteration) at the cost of some memory.

### Don't Index, Iterate (in simple cases)

While codegen for complex iterator chains
[may be suboptimal](/2017/03/06/lifetime.html) as of yet, in most simple cases
using an iterator will be faster than an indexed loop. So this:

```rust
for i in 0..(xs.len()) {
    let x = xs[i];
    // do something with x
}
```

should really be this:

```rust
for x in &xs {
    // do something with x
}
```

However, there are some caveats:

* if you use the index `i` elsewhere, use `for (i, x) in xs.iter().enumerate()`
* the iterator loop borrows `xs` whereas the indexed loops does so only for the
indexing operation. So you may need to index to pass the borrow checker. Still,
some access patterns may be supported by iterator methods (e.g.
`xs.iter().windows(2)` if you want to access each two neighboring elements).
* note this also means you cannot modify `xs` while iterating over it. However,
there are specialized iterators to e.g. remove elements in-place (`Drain`).

### Needless `collect()`

Also regarding iterators, if you use `FromIterator::collect()`, be aware that
you incur allocation for a new collection and force evaluation for each
iteration step. So before writing out that *collect*, ask yourself if you
*really* need it. This:

```rust
let nopes : Vec<_> = bleeps.iter().map(boop).collect();
let frungies : Vec<_> = nopes.iter().filter(|x| x > MIN_THRESHOLD).collect();
```

Could most probably be this:

```rust
let frungies : Vec<_> = bleeps.iter()
                              .map(boop)
                              .filter(|x| x > MIN_THRESHOLD)
                              .collect();
```

Depending on what we do with our frungies, we may even get rid of the second
`collect()`. Watch out for side effects, which might be relevant and change
evaluation order if a `collect()` is removed.

### Avoid needless allocation

Rust gives us a good number of tools to avoid needless allocations. It however
also gives us tools to do them if we're so inclined. Especially when starting
out and getting into the first disagreements with the borrow checker, often a
simple fix is to call `.to_owned()` or `.clone()`. However, while this keeps
the code simple, performance may suffer.

As there are multiple techniques to avoiding allocation, I'll only list those
that I deem both useful and sufficiently easy to implement:

* Use `&str` instead of `&String` (or even `String` unless you explicitly want
to consume it)
* Similarly, use `&[T]` (slices) instead of `&Vec<T>` (or even `Vec<T>`). Also
use `&mut [T]` instead of `&mut Vec<T>` if you have no resizing operation.
* For static values, you can often get away with arrays instead of `Vec`s. Just
reference them to get a slice.
* If you cannot replace your owned value with a borrow, consider using a `Cow`.
For example, it is often possible to replace `String` with `Cow<'static, str>`
and `Vec<T>` with `Cow<'a, [T]>` if you can borrow in some but not all cases.
* Sometimes, when changing an enum, we want to keep parts of the old value. Use
[`mem::replace` to avoid needless clones](https://github.com/rust-unofficial/patterns/blob/master/idioms/mem-replace.md).

### Avoid other needless work

Sometimes, adding a bit more lazyness can make a positive difference. For
example, in the name of readability, people may convert this:

```rust
match my_option {
    Some(foo) -> frobnicate(foo),
    None -> calculate_default_frob(),
}
```

to `my_option.map_or(calculate_default_frob(), frobnicate)`, which calculates
the default frob even when there is a `foo`. Using
`my_option.map_or_else(calculate_default_frob, frobnicate)` solves this
particular case. Note that while I wrote the `fn` names directly, sometime you
need a closure to capture some arguments (e.g. on `Result::ok_or_else(..)`) or
to auto-dereference them (e.g. `&Box`es will be dereferenced to refs to their
contents). The easiest way then is to always use a closure and let
[clippy](https://github.com/Manishearth/rust-clippy) tell you when to remove
it.

Also Rust uses LLVM, which performs dead-store analysis, which *may*
optimize away the needless default calculation. However, this analysis doesn't
always work, so as always measure the effect.

### More?

That's all I have for now. Do you have navigated other pitfalls? Discuss on
[/r/rust] or [rust-users]!

[/r/rust]: https://www.reddit.com/r/rust/comments/6ep1ao/blog_rust_performance_pitfalls/
[rust-users]: https://users.rust-lang.org/t/blog-rust-performance-pitfalls/11163?u=llogiq
