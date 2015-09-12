---
title: Memory Profiling Rust Applications
---

When I wrote about 
[Profiling Rust Applications on Linux](/2015/07/15/profiling.html), I
concentrated on getting a CPU profile (and only incidentially touched other
metrics). However, I left out one very important aspect: How to obtain the
memory profile of a Rust application.

When optimizing code, reducing the amount of allocated memory often covers a
good number of low hanging fruit that increase performance with relatively
little effort, so this is what the experienced performance engineer will look
at first.

Now let's take an application that actually uses memory, and see what we can
find out.

