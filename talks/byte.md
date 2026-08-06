---
marp: true
author: Andre 'llogiq' Bogus
title: How Fast Can You Count?
description: Nerdsniped by a comment in Raph Levien's source code to count bytes faster than anyone else
class: invert
---
<style>
:root {
  background: #000;
}
.no { border: 0px; background: #000; }
</style>
# How Fast Can You Count?

<!--
There are some patterns in my code
where I'm not proud of what I wrote
Don't worry – run clippy

It warns on needless things and such
And also where I clone too much…

Uhh-uuh… Don't worry – run clippy

I know that clippy has my back
It's here to help, not to attack me…

It won't just fix my own mistakes
But also those my colleague makes…

Uhh-uuh… Don't worry – run clippy

I will sometimes get lints I wrote
so clippy still improves my code…

Calling it with dash-dash fix
will happily your code remix…

Uhh-uuh… Don't worry – run clippy
-->

---
## A nerd snipe

In xi editor's rope crate, I found [this](https://github.com/xi-editor/xi-editor/blob/62fe30820aebc568086e4074117549a1d254f8de/rust/rope/src/rope.rs#L187-L191):

```rust
// TODO: explore ways to make this faster - SIMD would be a big win
// memchr is probably best for now
fn count_newlines(s: &str) -> usize {
    s.as_bytes().iter().filter(|&&c| c == b'\n').count()
}
```

<!--
Back in 2016, I checked out a lot of Rust code bases out there to search for patterns to lint with clippy. And in Raph Levien's xi-editor code I found this.

A normal person would probably have chuckled and moved on. But apparently I am not a normal person. I should say I'm not alone in this, because for what followed I teamed up with Joshua Landau, so shoutout to him.

But I get ahead of myself.
-->

---
## How does counting bytes work?

`input.as_bytes().iter().filter(|&&c| c == b'\n').count()`

- The naive version: 1 byte at a time
- Per iteration: Compare one byte with the given `'\n'`, increment `count` for each matching byte
- On intel, takes one `cmpb`, one `sete` and one `addq`
- That amounts to 3 instructions per byte (plus the byte read + loop)
- Note that each instruction requires knowledge of the result of the prior one

---
## How does counting bytes work?

Actually, this used to be a branch + increment until I made it branchless in [#39107](https://github.com/rust-lang/rust/pull/39107):

```rust
impl<I: Iterator, P> Iterator for Filter<I, P> where P: FnMut(&I::Item) -> bool {
    #[inline]
    fn count(mut self) -> usize {
        let mut count = 0;
        for x in &mut self.iter {
            count += (self.predicate)(&x) as usize;
        }
        count
    }
}
```

<!--
While optimizing byte counting, I found that Rust's standard library's `Filter::count` would compile down to a branch + increment. I will explain later why this is slower, but for now let's just say I made it slightly harder to beat the baseline.

Which leads to the question: How do we find out what is actually faster?
-->

---
## The First Rule of Optimization

> Measure, don't guess

– Kirk Pepperdine

<!--
I'd like to pause here to highlight this most imprortant rule of optimization. Everyone who's done perf work has found one or more hypotheses disproved by benchmarks. This is a most humbling experience. And it's pretty easy to deceive oneself with benchmarks, too, unless one takes care to really understand what's happening. Tread carefully.
-->
---
## Measurement

#### Benchmarking is hard. Always ensure you understand what you measure.

- Compiler may optimize out all of your benchmark code. `black_box` is your friend
- Your CPU's branch prediction may work on a benchmark with synthetic inputs, but fail in prod
- Hot code will be in caches, so your benchmark may need to "warm up" before measurement
- Sometimes speeding up a particular part of the system may slow down the whole!

<!--
Benchmarks should ideally be as close to the production workload as possible. However, that's usually a faraway dream for multiple reasons such as cost, deta security and privacy considerations, etc.

So what can we do to ensure our benchmarks actually measure what we want to measure?

* Don't just use `start.elapsed()` in your code to measure, use an actual benchmark crate
* Use `black_box` on all inputs. When in doubt, look at the assembly. When not in doubt, you shouldn't be trusted to benchmark
* make inputs as random as live data as much as possible
* if you run parallel / concurrent workloads, do causal profiling. There is an amazing tool called `coz` for that
-->
---
## Benchmarking crates

- bencher: The previously nightly only benchmarking harness of the compiler
- criterion: Ups the statistics, bells & whistles. Trades benchmarking speed for accuracy precision
- divan: A good tradeoff of benchmarking speed, ease of use and result fidelity

<!--
I started with bencher in early 2016 until I found criterion. bytecount's benchmarks use criterion. Nowadays I mostly use divan unless I need the statistical power of criterion.
-->
---
## Let's benchmark our first optimization

```rust
static BYTES: &[u8] = &[0; 1024];

#[divan::bench]
fn branch() -> usize {
    let mut count = 0;
    for &b in BYTES {
        if b == b'\n' {
            count += 1;
        }
    }
    count
}
```
<!--
With divan, we simply annotate a function as `divan::bench`. We can also add inputs and some configuration as part of the annotation, although we don't need that here.
-->

---
## Let's benchmark our first optimization

```rust
#[divan::bench]
fn sete() -> usize {
    let mut count = 0;
    for &b in BYTES {
        count += (b == b'\n') as usize;
    }
    count
}

fn main() {
    divan::main();
}
```
<!--
Now we have both variants as benchmark functions, we add a `main` function to run the benchmarks for us.
-->

---
## Let's benchmark our first optimization

```toml
[dev-dependencies]
divan = "0.1.21"

[[bench]]
name = "count"
harness = false
```

<!--
In Cargo.toml, we need to add our benchmark crate as a dev dependency and also add the benchark.
The `harness = false` is needed to avoid the unstable bencher harness that is only available on nightly.
-->
---
## Let's benchmark our first optimization

```
Timer precision: 41 ns
count      fastest  │ slowest  │ median   │ mean     │ samples │ iters
├─ branch  0.01 ns  │ 0.516 ns │ 0.02 ns  │ 0.025 ns │ 100     │ 1638400
╰─ sete    0.007 ns │ 0.079 ns │ 0.013 ns │ 0.015 ns │ 100     │ 1638400
```

<!--
Running this gives us a neatly formatted table with some information. Both the mean and median shows that `sete` is faster, so we're done, right?

(rhetorical pause)
-->

---
## Spot the error

What's wrong?

1. The timer precision is too low to get a valid result.
2. BYTES is too easy to guess for the branch predictor.
3. The `sete`-based function gets optimized into `0`.
4. I misspelled `brunch`.

<!--
All but 4 are correct
-->

---
## Let's fix our first optimization benchmark

```rust
#[divan::bench]
fn sete() -> usize {
    let mut count = 0;
    for &b in std::hint::black_box(BYTES) {
        count += (b == b'\n') as usize;
    }
    count
}
```

---
## Let's fix our first optimization benchmark

```
Timer precision: 41 ns
count      fastest │ slowest  │ median   │ mean     │ samples │ iters
├─ branch  165 ns  │ 465.8 ns │ 167.6 ns │ 181.2 ns │ 100     │ 3200
╰─ sete    165 ns  │ 313.5 ns │ 167.6 ns │ 172.6 ns │ 100     │ 3200
```

<!--
The difference is striking: Our code is now actually executed
-->
---
## What's in the `black_box`?

`std::hint::black_box`, unlike `std::convert::identity` tells the compiler not to assume anything about its input.

Thus, it precludes a good number of optimizations with minimal overhead.


---
## Interlude: Ways To Go Fast

- Avoid work
- Amortize work better
- Beware of unpredictable branches
- Do work in parallel
- Spread out work to reduce data dependencies
- Reduce the amount of memory needed

<!--
In general, when doing performance work, there are a number of ways to improve performance.

* Avoiding work can be done by finding stuff that is actually not needed, or by caching and reusing often used intermediate results. In our byte counting case, this is alas impossible.
* Amortizing work better is what we will mostly be talking about, because that is what applies the most.
* I will get to the other points later. (I'm trying to do a spaced repetition with you so you'll remember it more easily)
* There's also a point I'm not mentioning at all, but sometimes you can trade accuracy for performance. For example in games this is done a lot.
-->

---
## Back to counting bytes

Now our algorithm is basically:

<table>
<tr><td>b1=='\n'</td><td class="no">+</td><td>b2=='\n'</td><td class="no">+</td><td>b3=='\n'</td><td class="no">+</td><td>b4=='\n'</td><td class="no">+</td><td>b5=='\n'</td><td class="no">+</td><td>…</td></tr>
</table>

(Well, there's a `0 +` at the start, but bear with me)

---
## Amortizing work better: SIMD

1. add multiple comparisons together

| b1=='\n'|  b2… | b3   | b4   |
|---------|------|------|------|
| +       |  +   |  +   |  +   |
|**b5**   |**b6**|**b7**|**b8**|
| +       |  +   |  +   |  +   |
| :       |  :   |  :   |  :   |

2. do a horizontal addition to complete the count

<table>
<tr><th>b1+b5</th><td class="no">+</td><th>b2+b6</th><td class="no">+</td><th>b3+b7</th><td class="no">+</td><th>b4+b8</th></tr>
</table>

<!--
Our algorithm gets a little more complex by first taking the equality and doing a vector addition on multiple bytes in parallel.
Then we need a second step to horizontally sum the results. Here in the example illustration we take 4 values at once. In practice it will usually be 16, 32 or even 64, depending on architecture and target feature.
-->
---
## Amortizing work better: SIMD

SIMD = Single Instruction, Multiple Data

E.g. One comparison of four values.

In practice, there are various ways to get there

- Autovectorization
- SIMD crates (e.g. faster, fearless_simd)
- std::simd (nightly only for now)
- std::arch

---
## Autovectorization

If the code aligns to a very simple model of computation,
the compiler will replace slow scalar (one instruction, one data)
code with fast SIMD code.

\+ no unsafe code, easily readable simple Rust code
\- the compiler does not guarantee the optimization

<!--
Autovectorization is both awesome and nearly unusable, because while you get to write normal looking, in fact deceivingly simple Rust code, you need to check the generated assembly on each architecture / operating system / LLVM version / compiler version to ensure it's actually vectorized.

There are tools to automate this somewhat, but they're brittle.
-->
---
## SIMD library

A unified API on top of either the intrinsics or some custom codegen.

\+ One implementation will give good enough perf for many cases
\- Tradeoff between API elegance and perf

<!--
bytecount uses the `faster` library behind a feature flag. So while it usually will get better perf than scalar code, it will every now and then leave some perf on the table.
-->
---
## `std::simd`

Experimental, mostly like the given SIMD libraries

<!--
Like with the SIMD libraries I mentioned, the awesome thing is that you can write it once and have SIMD code generated for multiple architectures. The not so awesome thing is that you'll sometimes leave some perf on the table, and the whole "unstable" thing is also putting folks off.
-->
---
## `std::arch`

\+ Full control over the emitted assembly allows for maximum performance
\- needs a code path per architecture (usually x86/AVX/AVX512/NEON/WASM/...)
\- somewhat finicky (although things are improving)

<!--
This is what bytecount uses (with codepaths for SSE2, AVX2, ARM/NEON and WebAssembly, …
-->

---
## A non-"SIMD" SIMD option

Some bit twiddling can be done using normal `usize`s. For example:

```rust
fn bytewise_equal(lhs: usize, rhs: usize) -> usize {
    let lo = usize::MAX / 0xFF;
    let hi = lo << 7;

    let x = lhs ^ rhs;
    !((((x & !hi) + !hi) | x) >> 7) & lo
}
```

<!--
…as well as a generic bit-twiddling based integer-pseudo-SIMD. Why `usize`? Because it's usually not worth doing 64-bit arithmetic on 32 bit systems.

In this example, we get 4 byte equality comparisons for the price of 5 instructions. Even on 32 bits, that's less than the 4 `cmpb`s and `sete`s (discounting some shifting and or-ing to collect the results). It's the same 5 instructions on 64 bits, btw.
-->

---
## CPUs are complex beasts

* Caches
* Pipelining
* Branch prediction

<!--
Up until the late 90s, CPUs executed the machine code more or less directly. Nowadays, CPUs compile the machine code on the fly into microcode instructions. Also back then, multicore systems were in the realm of supercomputers. Now we all have those supercomputers in our pockets.

But that's not the complexity I'm talking about.
-->

---
## Caches

Since the 80s, CPUs got far faster than memory. They can run some 10s-100s of instructions while waiting for one memory access.

Caches reduce that problem: Cache access is far faster. However, they are also limited.

Caches are a *hierarchy*. Usually there's L1 (fastest/smallest), L2 and L3 (slowest/largest).\
There's also the instruction cache, because the CPU also needs code to execute.

<!--
Physics has a weird way of not cooperating: Ideally we'd want memory as fast as CPU, and a lot of memory, too. Unfortunately electrons travel at half light speed, which at a few milimetres distance between core and memory will become a bottleneck. Also there's the difference between SRAM and DRAM.

SRAM (static RAM) faster, less power hungry, 6 transistors per bit
DRAM (dynamic RAM) 1 Transistor+Capacitor per bit, but slower & more power hungry

Therefore SRAM is mostly used in caches, where the proximity to the core adds to the read/write speed.

With that said, the smaller size of caches means that every now and then, they need to flush, and then RAM slowness comes back to bite us. Perf tools give us ways to measure cache misses. If your algorithm causes a lot of those, think about how to reduce the memory usage. For example, for a statistical data processing tool I once devised a compression algorithm that blended in with my operation, so I could work directly on the compressed data, reducing RAM usage by a factor of 8. The cache miss reduction alone easily payed for the additional compute to deal with compressed data, and then some.

Caches work by the MESI protocal. The acronym stands for the four states a cache line can be in: Modified, Exclusive, Shared or Invalid. For example, a change on one core will mark the cache line as modified and invalidate all copies of that cache line on all other cores that have it. This also means that it's a good idea to seperate read-only and modifiable data, because if those inhabit different cache lines, the read-only ones can be shared without risking invalidation.
-->
---
## Caches

On nightly, we can help the CPU by issuing a suitable "prefetch" instruction, telling the cache to expect some read or write.

```rust
#![feature(hint_prefetch)]
use std::hint::{Locality, prefetch_read};

prefetch_read(ptr, Locality::L1);
prefetch_read_non_temporal(prt, Locality::L2):
prefetch_read_instruction(*const my_function, Locality::L1);
prefetch_write(ptr, Locality::L1);
prefetch_write_non_temporal(prt, Locality::L2):
```

<!--
read/write = get the cache line even at the cost of evicting something likely to be needed
read/write_non_temporal = get the cache line, unless there are too many hot things in cache
read_instruction = put the given code into the instruction cache, we're going to run it

The read/write difference is because of the cache protocol. Writes will invalidate all other copies of the cache and may trigger a sync between cores.

Btw. in our tests, prefetch made no difference, because we already read the memory linearly and todays CPUs absolutely love to churn through such stuff.
-->
---
## Pipelining

Remember what I said about doing work in parallel? CPU cores already do that.

Instead of doing instructions one by one:

<table>
<tr><td>insn1</td><td>insn2</td><td>insn3</td><td>insn4</td><td>…</td></tr>
</table>

They start the next few instructions in advance:

<table>
<tr><td colspan="3">insn1</td><td></td><td></td><td></td></tr>
<tr><td></td><td colspan="3">insn2</td><td></td><td></td></tr>
<tr><td></td><td></td><td colspan="3">insn3</td><td></td></tr>
<tr><td></td><td></td><td></td><td colspan="3">insn4</td></tr>
</table>

---
## Pipelining

However there are two catches.

---
## Pipelining Pitfall #1: Data dependencies

If insn2 depends on the result of insn1, we're serial again

Easiest case: `(a + b) + (c + d)` vs. `((a + b) + c) + d`

<!--
What do you think is slower or faster? Or are both the same?
-->
---
## Pipelining Pitfall #1: Data dependencies

dense:  `(a + b) + (c + d)` / flat: `((a + b) + c) + d`
```
Timer precision: 41 ns
add       fastest  │ slowest  │ median   │ mean    │ samples │ iters
├─ dense  0.189 ns │ 0.741 ns │ 0.208 ns │ 0.29 ns │ 100     │ 1638400
╰─ flat   0.192 ns │ 1.168 ns │ 0.685 ns │ 0.57 ns │ 100     │ 819200
```

<!--
So as we can see, the tree-based "dense" variant is slightly faster.
-->
---
## Interlude: Benchmarking Pitfall #1: Misleading yourself

```asm
add4_dense:
	addq	%rdi, %rsi
	leaq	(%rdx,%rcx), %rax
	addq	%rsi, %rax
	retq
add4_flat = add4_dense
```

<!--
However, I'm actually misleading you here. Again, divan is not suited for this type of benchmark, and LLVM can nowadays see through our code and emit the dense code anyway, so any changes that appear will come down to code alignment. With float ops, you can actually see a difference, because float addition, unlike integer addition is *not* associative.

Anyway, back to our main program.
-->

---
## Pipelining Pitfall #1: Data dependencies

In our byte counting algorithm, we can keep two counters and count alternating SIMD words, adding the counters at the end to arrive at our final result.

Beware that this might consume all of our registers, leading to spilling.

<!--
The solution is to Interleave calculations to spread out data dependencies over multiple instructions. This means at least instructions won't depend on the results of the previous ones (but on the previous-to-previous one).
-->

---
## Pipelining Pitfall #2: Branches

If the CPU *doesn't* know where the next instruction is, it has to
- scrap the computation done in advance
- reset the state to what it would have been just before the branch
- do the branch

<!--
Yup, we're back to unpredictable branches. And we already have the benchmarks to prove it.

Note the "If" at the beginning, which brings us to *Branch Prediction*
-->

---
## Branch Prediction

To alleviate that, the CPU will attempt to predict whether a branch will be taken.

The ways to do that have become increasingly complex.

We can measure branch misprediction and try to help the CPU:

```rust
#[cold]
fn not_on_hot_path() { }

if is_cold {
    std::hint::cold_path();
}

// doesn't branch if `sete` or equivalent available
std::hint::select_unpredictable(cond, t, f);
```

<!--
Early branch prediction was like "always take jumps back, never take forward branches". Now we have AI in our CPU.

We can mark a function or branch as cold, so the compiler will generate code in a way that tells the CPU's branch predictor not to expext that branch being taken in hot code.

Or we can use `select_unpredictable` which will compile down to a conditional set instruction if the architecture has one, removing a branch entirely.

Note that this requires calculating both values, so if one of them is expensive and not often taken, it can still be slower than using a branch, sometimes even if badly predicted.

Sometimes making branches easier to predict beats removing them outright
-->
---
## But I digress

Most of those things don't apply to counting bytes (but may well apply to parsing).

However, I have not yet fully explained the amortization angle.

The simple data-parallel count does 255 SIMD equalities and additions before horizontally summing the results.

We can do better.
<!--
So far I haven't showed you the actual algorithm behind bytecount
-->

---
## Hyperscreamingcount

The idea is to amortize the horizontal sum (hsum) better. Instead of doing a full hsum, we do a *widening addition*:

<table>
<tr><th>byte</th><th>0</th><th>1</th><th>2</th><th>3</th><th>4</th><th>5</th><th>6</th><th>7</th></tr>
<tr><th>bytecounts</th><td>c1</td><td>c2</td><td>c3</td><td>c4</td><td>c5</td><td>c6</td><td>c7</td><td>c8</td></tr>
<tr><th>hi masked &gt;&gt; 8</th><td>0</td><td>c1</td><td>0</td><td>c3</td><td>0</td><td>c5</td><td>0</td><td>c7</td></tr>
<tr><th>lo masked</th><td>0</td><td>c2</td><td>0</td><td>c4</td><td>0</td><td>c6</td><td>0</td><td>c8</td></tr>
<tr><th>added</th><td colspan="2">c1+c2</td><td colspan="2">c3+c4</td><td colspan="2">c5+c6</td><td colspan="2">c7+c8</td></tr>
</table>

---
## Hyperscreamingcount

We can then add 256 of those N·u16 together before doing a 16-bit horizontal sum, thus amortizing the sum over 65024 bytes.

Why does this make a difference? The horizontal sum is hard to parallelize:

<table>
<tr><th>byte</th><th>0</th><th>1</th><th>2</th><th>3</th><th>4</th><th>5</th><th>6</th><th>7</th></tr>
<tr><th>8->16</th><td colspan="2">0+1</td><td colspan="2">2+3</td><td colspan="2">4+5</td><td colspan="2">6+7</td></tr>
<tr><th>16->32</th><td colspan="4">0+1+2+3</td><td colspan="4">4+5+6+7</td></tr>
<tr><th>32->64</th><td colspan="8">0+1+2+3+4+5+6+7</td></tr>
</table>

---
## Hyperscreamingcount

In contrast, the widening addition is merely the first step of a full horizontal sum:

<table>
<tr><th>byte</th><th>0</th><th>1</th><th>2</th><th>3</th><th>4</th><th>5</th><th>6</th><th>7</th></tr>
<tr><th>8->16</th><td colspan="2">0+1</td><td colspan="2">2+3</td><td colspan="2">4+5</td><td colspan="2">6+7</td></tr>
</table>

So it can be three times faster.

As the horizontal sum is 3/255th of the whole operation, this makes it up to 2% faster than the simple 255-byte horizontal sum*.

<!--
* Provided we're not waiting on memory
-->
---
## Hyperscreamingcount

In theory, we could do another widening step before doing the full sum, but that would only amortize the counts on strings larger than ~2GB (and at that point, register pressure might make the whole thing slower again).

---
## Are we fast?

```
counts/naive/10         time:   [2.5873 ns 2.5932 ns 2.6003 ns]
                        thrpt:  [3.5816 GiB/s 3.5914 GiB/s 3.5996 GiB/s]
counts/naive_32/10      time:   [2.1744 ns 2.1812 ns 2.1895 ns]
                        thrpt:  [4.2536 GiB/s 4.2698 GiB/s 4.2830 GiB/s]
counts/hyper/10         time:   [1.7541 ns 1.7566 ns 1.7594 ns]
                        thrpt:  [5.2934 GiB/s 5.3017 GiB/s 5.3094 GiB/s]
```

---
## Are we fast?

```
counts/naive/1000       time:   [135.15 ns 135.34 ns 135.56 ns]
                        thrpt:  [6.8704 GiB/s 6.8815 GiB/s 6.8910 GiB/s]
counts/naive_32/1000    time:   [66.321 ns 66.495 ns 66.702 ns]
                        thrpt:  [13.962 GiB/s 14.006 GiB/s 14.043 GiB/s]
counts/hyper/1000       time:   [10.733 ns 10.743 ns 10.754 ns]
                        thrpt:  [86.603 GiB/s 86.691 GiB/s 86.771 GiB/s]
```
---
## Are we fast?
  
```
counts/naive/100000     time:   [13.333 µs 13.355 µs 13.377 µs]
                        thrpt:  [6.9619 GiB/s 6.9737 GiB/s 6.9850 GiB/s]
counts/naive_32/100000  time:   [6.4579 µs 6.4790 µs 6.5049 µs]
                        thrpt:  [14.317 GiB/s 14.375 GiB/s 14.422 GiB/s]
counts/hyper/100000     time:   [932.59 ns 933.51 ns 934.47 ns]
                        thrpt:  [99.663 GiB/s 99.766 GiB/s 99.864 GiB/s]
```
---
## Are we fast?
  
```
counts/naive/10M        time:   [1.3317 ms 1.3328 ms 1.3341 ms]
                        thrpt:  [6.9811 GiB/s 6.9876 GiB/s 6.9932 GiB/s]
counts/naive_32/10M     time:   [642.94 µs 643.63 µs 644.33 µs]
                        thrpt:  [14.454 GiB/s 14.470 GiB/s 14.485 GiB/s]
counts/hyper/10M        time:   [111.68 µs 111.79 µs 111.92 µs]
                        thrpt:  [83.214 GiB/s 83.311 GiB/s 83.390 GiB/s]
```
---
## Further considerations

SIMD equality and vector-to-bitmask operations are a good starting point for parsers.

e.g. `(x >> 1 | carry << 63) & y` combines an "xy" search in 64 bits.

Also sometimes it's useful to create a "latch" (set all following bits to 1 or 0 respectively):

```rust
fn latch(mut u: u64) -> u64 {
    u ^= u << 32;
    u ^= u << 16;
    u ^= u << 8;
    u ^= u << 4;
    u ^= u << 2;
    u ^ u << 1
}
```

*and* / *or* / *xor* the resulting masks for maximum effect.

---
## Similarities between architectures

| Operation    | x86             | ARM    | WASM        |
|--------------|-----------------|--------|-------------|
|bytewise equal|_mm_cmpeq_epi8   |vceqq_u8|u8x16_eq     |
|bytewise add  |_mm_add_epi8     |vaddq_u8|u8x16_add    |
|bytewise and  |_mm_and_epi8     |vandq_u8|v128_and     |
|bytewise or   |_mm_or_epi8      |vorq_u8 |v128_or      |
|create bitmask|_mm_movemask_epi8|*       |u8x16_bitmask|

\* there's a combination of narrowing reductions

<!--
So as you can see, mostly the naming schemes are different, but moxt operations come out the same. I've used SSE2 instructions for x86, because those are most widely available. If you want the corresponding AVX2 instructions working on the double width SIMD vectors, you need to put a 256 after the _mm. And if you want AVX512, which has even double that, replace the 256 with 512.
-->

---
## Differences between architectures

|              |SSE2|AVX2|AVX512|AARCH64| WASM   |
|--------------|----|----|------|-------|--------|
|Byte Lanes    |16  |32  |64    |16     |16      |
|SIMD registers|8   |16  |32/64 |32     |implicit|

<!--
The really notable differences are the SIMD vector width and the number of registers.
-->

---
## Perf Pitfall: `#[target_feature(..)]`

E.g. for AVX2 code, set #[target_feature(enable = "avx2")] *and* guard your code with `#[cfg(target_feature = "avx2")]`.`

For x86, there's the option of compiling for the native architecture / target features, which may mean different versions for sse2, ssse4, avx and avx512, or compile all of those into one binary and do runtime detection.

```rust
if is_x86_feature_detected!("avx2") {
    unsafe { avx2::code_path(input) }
} else {
    unsafe { sse2::code_path(input) }
}
```

---
## Final Words

- Don't forget to compiler for `--release`
- Check whether PGO gives you benefits
- Always code a simple baseline version first
- Know when to stop
- Have fun optimizing!

---
# That's all, folks!
