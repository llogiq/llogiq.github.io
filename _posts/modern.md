---
title: Yeah, but what *is* "modern" programming?
---

Recently Prof. D. Lemire wrote a
[blog post](http://lemire.me/blog/2017/07/15/what-is-modern-programming/) about
his programming history and how – in his opinion – "modern" programming is
different than what we did in ye olde days. I really like his writing, and
completely agree with the article. Still, I have some additional ideas on what
constitutes "modern" programming I want to share here.

Like Prof. Lemire, I started out with BASIC and assembly (what a coincidence),
then settled on Turbo Pascal (before being shanghaied into Java during my CS
degree).

My first computer had a Z80 CPU and 32 kilobytes of RAM. My first PC had one
whopping Megabyte of RAM (of which 640 kilobytes were usable directly and the
rest by playing some tricks with extended/expanded memory).

Current CPUs are not only much faster, but also a lot more complex than the
8MHz 80186 I wrote my first Pascal code on. Where we had a few scarce 16-bit
registers, we now have 64-bit wide general purpose registers, plus a number of
even wider SIMD ones, complete with their own opcodes to do operations in
parallel. Where ye olde 16-bit CPU took 4 cycles for an `add` instruction (and
many more for a `mul` or even `div`, current CPUs can sometimes do more than
one instructions per cycle, thanks to pipelining. I also have two cores that
can each execute two threads in my CPU, so I can do four things in parallel, on
a low-end CPU. High-end desktop CPUs now have eight, ten or even more cores.

Even better, our computers now include GPUs that offer even broader
parallelization opportunities (for those able to program them), so our code can
do massive computations that would have been infeasible even on early 90's
supercomputers.

For many of us, that doesn't matter much, because a good portion of their
time they don't really use a desktop or notebook, but a smaller, mobile
personal device called Smartphone. Those now have CPUs and RAM that rival the
contemporary notebook specs. Heat and power draw are the chief limiting
factors. But I digress.

Turbo Pascal really was a wonderful language and a great development
environment. Though I rarely used the debugger, I liked using the IDE a lot.
Yet the other IDE features we take for granted (syntax highlighting,
context-sensitive content assist, call hierarchy, refactorings, quick fixes to
name a few) were missing. TP also had very little in the way of optimizations,
making one go down to assembly level (which was available via `asm { .. }`
syntax) for maximum performance.

Pascal didn't offer those features because they wouldn't have been usable with
early '90s CPUs and memory – either too slow due to scarcity of CPU cycles per
second or even infeasible to implement due to scarcity of bits in RAM.

Contrast with a contemporary optimizing compiler, which – even if it would have
run at all – would take hours, nay, days to compile a medium-size project (I
have no hard numbers on how much optimizations benefit runtime, but a Rust
program compiled with `cargo build --release` usually runs one or two orders of
magnitude faster than the unoptimized version). The compilers can make use of
the increased complexity to make our code run faster.

Not only are our compilers more complex, our langauges are, too (well, with the
possible exception of Go, but that's intentional). Even Java now has some form
of lambdas and streams, so partial functional programming should now be
considered mainstream (hint: it wasn't in the days of LISP machines). If I
choose a VM environment, I can get a garbage collector to deal with the problem
of cleaning up memory after my program is done with it.

Many of our programming language use their powerful type systems to allow us to
reuse code across different types while checking a good number of invariants at
compile time. All without requiring us to write a proof of those invariants –
it's all implicit.

And if something goes amiss, the error messages we get are fabulous! Look into
Elm or Rust for the best examples, but even gcc nowadays has some good
examples.

Not only can we unit-test, we write documentation (well, the better of us do)
that includes examples that are actually tested during our build (for those of
us that use python or Rust, or Java with one of the javadoc extensions, I also
wrote a doctest.lua at one point). With Rust documentation, the examples even
include a playground link, so we can execute them online!

In our unit tests, we can make use of properties-based testing methods like
[quickcheck](https://github.com/BurntSushi/quickcheck). We even invented
techniques to test coupled instances (stubbing and/or mocking), although to be
fair, many consider those a code smell.

When our early 90's code crashed, we got strange patterns on the screen, or
maybe the occasional corrupted file. Nowadays, we get DDoS botnets,
crypto-trojans, banking scams and all sorts of nasty things. To paraphrase Neal
Stephenson's "Snow Crash", this is no longer a safe place.

To counter this, we have built bespoke static code analysis tools. Code deemed
security-relevant is also now heavily fuzz-tested, a technique that has only
recently become feasible thanks to the explosion of available CPU cycles we can
throw at the problem.
