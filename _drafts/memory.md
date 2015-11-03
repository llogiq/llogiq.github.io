---
title: Memory Profiling Rust Applications
---

_(Rust 1.3.0 stable and 1.5.0 nightly were used for this article)_

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

```Rust
use std::iter::Iterator;

fn main() {
	let x : Vec<usize> = (1..100000).collect();
}
```
_Listing 1: An application which acquires some memory_

### `jemalloc`

The first thing we can do is ask jemalloc how much memory it has claimed. To
do this, we extend our method, calling jemalloc's `...` as a C function.

```Rust
use std::iter::Iterator;

extern {
	//TODO je_print_stats() invocation
}

fn safe_print_stats() {
	unsafe { /* TODO */ }
}

fn main() {
	safe_print_stats();
	let x : Vec<usize> = (1.100000).collect();
	safe_print_stats();
}
```
_Listing 2: Added allocation statistics before and after our code

Let us compile and run it.

```
$ rustc -O memprof2.rs

```

This is workable for our very small example, but in most programs, we have more
than one line of code responsible for allocation. Luckily, we do have some 
tools that allow us to get a more complete picture. Enter valgrind:

### Valgrind

Valgrind simulates a computer in order to find out certain things about the 
runtime of the program. While the `memcheck" memory analyzer is probably its 
best-known application, it also comes with the "massif" memory profiler tool.

So we re-run our original program (see Listing 1) under massif:

```
$ rustc alloctest.rs -O -g
valgrind --tool=massif ./alloctest
