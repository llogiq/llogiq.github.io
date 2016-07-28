---
title: Another happy quickcheck customer
---

Even when I announced [overflower](https://github.com/llogiq/overflower), I 
confessed to lacking tests for the support library. Since the code is fairly 
simple, I am of acceptable confidence that it'll work, but I'd sleep more 
soundly if this stuff really was tested.

I *could* of course craft known-good inputs to test the functions, but then I'd 
miss out on the unknown inputs, good or bad. I also always wanted to try 
[quickcheck](https://github.com/BurntSushi/quickcheck), and this is the perfect 
opportunity.

To sum it up, quickcheck rocks. And `overflow_support` does what I want. If 
that's all you wanted to know, you can stop reading here.

Still here? Cool.

Quickcheck contains *a lot* of cleverness so you can simply write a predicate 
function for some input type which has an `Arbitrary` implementation (which 
already covers the usual suspect types and is easy to extend), and call 
`quickcheck::quickcheck(_)` on that function (to be fair, you'll need to 
explicitly cast to the `fn(..) -> _` type for that to work). There's even a 
plugin that lets you annotate your predicate functions with `#[quickcheck]` 
directly which turns them into quickcheck-using tests (I didn't test it though, 
because the direct way worked good enough for me).

However, there was one slight wrinkle: Since some of my methods under test 
panic, I needed to deal with that. Luckily, `std::panic::catch_unwind(_)` can 
turn panics into results (with earlier Rust versions, I could have used a 
thread). Unfortunately, the *panic handler* which prints a message (and 
optionally a stack trace) gets called regardless.

So I tried to silence the panics (cue 
[odyssey](https://stackoverflow.com/questions/38514554/how-can-i-silently-catch-panics-in-quickcheck-tests)). 
This is just a bit more complicated because the tests run in parallel, and you 
cannot depend on execution order to reliably distinguish quickcheck panics 
(which contain the reduced test inputs, thus we want them on standard output) 
and caught panics from my tests.

In the end, I relied on the fact that my tests only threw errors from two 
locations (in `src/lib.rs` and somewhere in `num` for overflow). Whie this is a 
brittle and very specific solution, it works well enough for me.

Quickcheck is plenty fast, given that it generates 100 random inputs and runs 
your test code with each of them. Even with the 100 random executions, 
quickcheck may sometimes miss an error in the tests. However, over a few runs, 
I'm fairly confident that finding more errors by chance is *very* unlikely.

So if you have any non-trivial project that could benefit from more test 
coverage, give quickcheck a try. Even with fairly uncommon scenarios like mine, 
using quickcheck really is quick – writing all those tests by hand would've 
been prohibitively time-consuming.
