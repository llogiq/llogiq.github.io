---
title: Rust Faster!
---

A staple of all performance discussion is the Great Programming Language 
Benchmarks Game, a.k.a. the "Shootout", currently maintained by Isaac Guoy.
Despite the fact that the benchmarks say more about the resourcefulness of
certain members of the respective programming language communities than the
languages themselves, this site is often cited in discussions of the relative
merits of programming languages.

Since one of the core tenets of Rust is "Fast", we have some motivation to 
score well on those benchmarks. Unfortunately, we're lagging behind C/C++ in
some of them as of yet. teXitoi, Veedrac and yours truly set out 
semi-independently to rectify the situation. The following discusses the
performance tricks we pulled.

### fasta and fasta-redux

I am actually the one responsible for the slow fasta-redux implementation. It
was submitted as a baseline to benchmark against, but I didn't get around to
actually optimize it. However, it bugged me, so I at least inserted a 
`BufWriter` to at least have faster IO.

Meanwhile, Veedrac had a few insights on how to optimize the *fasta* benchmark. 
The PR is [here](https://github.com/TeXitoi/benchmarksgame-rs/pull/20).

The biggest trick this program pulls is the full parallelization of the random
number generator.

TODO: Describe the powmod trick

Another trick nicked from a haskell version is to create a lookup table of size
`MODULUS` so that we can get the respective proteine for the random number by a
simple lookup. Since the `MODULUS` isn't too big, and we only need one byte per
value, this doesn't eat too much RAM and lets the cores concentrate on random
number generation.

For the repeated text, there's another trick stolen from the Haskell version:
Repeat the string so that every line is some index in it, then just insert the
newline and print the slice for each line.

Finally, the random fasta has an optimized writer that uses a mutex + count to
efficiently allow the threads to synchronize their output. One would think that
it should be possible to do better with atomics, but that would require unsafe
and the work packets are so big that there is no contention, so a Mutex works
nicely. The code to output the random lines pre-fills the output array with 
newlines so that it can just skip the positions of the line breaks.

Veedrac's implementation just smokes everything else on four cores, and is
still faster than speed limit on one core.

I just adapted his code for the *fasta-redux* benchmark; in the this case the
lookup table is predefined by the benchmark rules (and we need some constant
factor to index into the table). Apart from that the code is exactly the same.

### spectralnorm

Rust didn't do as good as it could because the benchmarksgame site uses stable
which cannot use unstable APIs, notably this means we cannot use SIMD directly.
teXitoi had the idea to rewrite the hottest code in a way that lets LLVM
auto-vectorize it. He wrote about it in an 
[issue](https://github.com/TeXitoi/benchmarksgame-rs/issues/9) on his repo and
I [wrote](https://github.com/TeXitoi/benchmarksgame-rs/pull/22) the 
implementation.

The hot function in question:

```Rust
fn A(i: usize, j: usize) -> f64 {
    ((i + j) * (i + j + 1) / 2 + i + 1) as f64
}
```

And here is the version that uses custom autovectorizable `usizex2` and `f64x2`
types:

```Rust
fn Ax2(i: usizex2, j: usizex2) -> f64x2 {
    ((i + j) * (i + j + usizex2(1, 1)) / usizex2(2, 2) + i + usizex2(1, 1)).into()
}
```

On the first glimpse, this looks more complex than the original, but it also
does twice the work. The implementations of `usizex2` and `f64x2` are actually
pretty boring; they just distribute the operations to their contents, e.g.

```Rust
struct usizex2(usize, usize);
impl std::ops::Add for usizex2 {
    type Output = Self;
    fn add(self, rhs: Self) -> Self {
        usizex2(self.0 + rhs.0, self.1 + rhs.1)
    }
}
//... similar implementations for Div and Mul
```

With this. calling the `A`-function changes from:

```Rust
let bot = f64x2(a(i, j), a(i, j + 1));
```

to

```Rust
let bot = a(usizex2(i, i), usizex2(j, j+1));
```
for a nice 20% speedup. Note that the official Rust version *can* use nightly
and thus SIMD. In a few weeks this interim version can be replaced again,
yielding to the faster official version.

### k_nucleotide

Veedrac did a
[full rewrite](https://github.com/TeXitoi/benchmarksgame-rs/pull/21) with:

* an improved hash table, using flat Robin Hood Hashing,
* pre-compacted input,
* better load balanced threading,
* really cool pack_byte implementation.

The `pack_byte` function should pack DNA sequences into two bits each. The
trick behind Veedrac's implementation is that "A", "C", "G" and "T" have all
different ASCII representations in the second-to-lowest two bits. So the 
function just shifts the ASCII bytes by one to the right and masks the lowest
two bits. Look ma, no lookup!

Choice quote from the source:

```Rust
        // Yes, this is hillarious
        pub fn eat(&mut self, cereal: &[u8]) { ...
```

TODO

### thread_ring

This one isn't even in the list of comparable benchmarks, because it divides
languages using native threads from languages that have some abstraction. 
Rust of course falls in the former category, and those tend to do worse on
this benchmark.

Veedrac has a superb mixed mutex/spinlock implementation that beats all
other native-thread based implementations in terms of speed, however is
subject to possible pathological behavior. Luckily this wasn't triggered
in our benchmarks so far.

TODO

### chameneos-redux

Now Veedrac was clearly on a roll. His chameneos-redux is so fast it's no longer
funny. His implementation is faster than all others by an order of magnitude.

TODO
