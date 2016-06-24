---
title: Announcing Overflower
---

Integer overflow handling is a problem in many languages. Some, like Java, 
don't even have specialized methods to check or cope with overflow, so you're 
left on your own. I work with statistic computations that sometimes need higher 
precision than [IEE754](https://en.wikipedia.org/wiki/IEEE754) double-width can 
give us, so using 64 bit integers is an appealing solution (that unlike bignums 
doesn't kill performance). However, integers usually wrap around on overflow, 
which is a bad thing to happen if you try to optimize a value.

In Rust, the story is much better, with a compiler option to enable/disable 
overflow checks globally (and the default to enable them in debug builds, but 
disable them in release builds), checked/wrapping and saturating methods on all 
integer types and even wrapper types that use the specialized methods in 
arithmetic operations. However, I felt that this puzzle was missing a piece. 
Ideally, I'd want to just tell the compiler: "Within this method/module/crate, 
use wrapping operations" or something like that. Cheap to put in, cheap to 
delete again, and neither wrapper types nor long method calls instead of 
arithmetic operators.

As I recently gained some confidence in [writing](/2016/05/17/flamer.html) 
procedural macros, I thought "how hard could this be"? So I set out to build a 
crate to do exactly this. As a cute pun, I named it 
"[overflower](https://github.com/llogiq/overflower)".

### Use Cases

Apart from my rather specific use case (I want `--release` all the way, and get 
checked overflow every now and then, but only in certain modules or even 
methods, to be easily switched in and out), there are other uses for such a 
crate. For example, specifying `#[overflow(wrap)]` will ensure that arithmetic 
operations won't cause inadvertent panics even in debugging mode, which appears 
to be quite valuable in panic handlers (because double-panics suck) or 
interrupt handlers (where you may not panic at all).

Also it lends itself to experiments like "what if that code used saturating 
arithmetic?", which are a fun way to spend the time, for some admittedly rather 
improbable definition of fun. :-)

### Implementation

The first thing to note is that Rust uses traits to define arithmetic 
operators. Combine this with type inference, and you'll get a number of 
problems if you try to replace e.g. `x + y` with `x.add_or_panic(y)`. Even 
without implementing `std::ops::Add` for your own types, what if `x` is a 
`String` and `y` is a `&str`? Your code no longer compiles, that's what!

Doing type inference on an AST is certainly possible, but I did not want to go 
that route, because it's complex, cumbersome, error-prone and subject to change 
with future additions to Rust's type system (for example, I hear `impl trait` 
is going to be implemented before the end of the year). My time's better 
invested in writing code or blog copy than trying to follow rustc's type 
system.

As luck has it, just recently support for 
[specialization](https://github.com/rust-lang/rfcs/blob/master/text/1210-impl-specialization.md)
landed in nightly. Since a procedural macro is a compiler plugin and thus has
to use nightly anyway, why not use it to specialize our own traits for integer
types and delegate to the `std::ops` traits for everyone else? A quick 
proof-of-concept turned out to work, so I quickly implemented the beginnings of 
[overflower_support](https://crates.io/crates/overflower_support). This crate 
contains the traits that get called instead of those in `std::ops`, and are 
specialized for integral types to handle overflow in a specific way.

Now all overflower has to do is to fold the code and replace binary operations 
with the corresponding method calls (actually the current version uses calls, 
which thanks to unified function call syntax works beautifully, and those don't 
require the traits to be in scope). I also had to [expand 
macros](/2016/06/11/expand.html) so I could deal with overflow within macro 
expansions (I hear that this is going to change in the future, so procedural 
macros can work on pre-expanded code directly).

I note that the same technique could be used to dramatically simplify 
[mutation testing](/2016/03/24/mutest.html).

### Conclusion

I'm quite happy with the current state of overflower. There are some things to 
be done, notably I'll need a lot more tests (and I'd be glad to merge your 
contributions!), but all in all, I feel this piece fits the overflow handling 
puzzle quite well. It's a pity that I cannot use this in Java.

Or stable Rust for that matter. If the standard library could at least make the 
traits of the `overflower_support` crate available, rewriting the plugin to use 
syntex would enable using this from within stable, but I'm not a core dev and 
don't know whether they'd like the idea. This state of affairs is temporary 
anyway, because sooner or later specialization will land in stable and then we 
will, too.

So do you think this belongs in core? Discuss on 
[rust-users](https://users.rust-lang.org/t/blog-announcing-overflower/6321) or 
[/r/rust](https://www.reddit.com/r/rust/comments/4poxuu/blog_announcing_overflower/).

⚘
