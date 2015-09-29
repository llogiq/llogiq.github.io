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

### Fasta and Fasta-redux

Veedrac had a few insights on how to optimize the *fasta* benchmark. The PR is
[here](https://github.com/TeXitoi/benchmarksgame-rs/pull/20).

The biggest trick this program pulls is the full parallelization of the random
number generator.

TODO: ...

I just adapted his code for the *fasta-redux* benchmark; in the this case the
lookup table is predefined by the benchmark rules. Apart from that the code is
exactly the same.

### Spectralnorm

Rust didn't do as good as it could because the benchmarksgame site uses stable
which cannot use unstable APIs, notably this means we cannot use SIMD directly.
teXitoi had the idea to rewrite the hottest code in a way that let LLVM
auto-vectorize it. He wrote about it in an 
[issue](https://github.com/TeXitoi/benchmarksgame-rs/issues/9) on his repo and
I [provided](https://github.com/TeXitoi/benchmarksgame-rs/pull/22) the 
implementation.

The hot function in question:

```Rust
fn A(i: usize, j: usize) -> f64 {
    ((i + j) * (i + j + 1) / 2 + i + 1) as f64
}
```

And here is the version that uses custom autovectorizable usizex2 and f64x2
types: 

```Rust
fn Ax2(i: usizex2, j: usizex2) -> f64x2 {
    ((i + j) * (i + j + usizex2(1, 1)) / usizex2(2, 2) + i + usizex2(1, 1)).into()
}
```

Calling this function changes from:

```Rust
let bot = f64x2(a(i, j), a(i, j + 1));
```

to

```Rust
let bot = a(usizex2(i, i), usizex2(j, j+1));
```
for a nice 20% speedup.

### k_nucleotide

Veedrac did a
[full rewrite](https://github.com/TeXitoi/benchmarksgame-rs/pull/21) with:

* an improved hash table, using flat Robin Hood Hashing,
* pre-compacted input,
* better load balanced threading,
* really cool pack_byte implementation.

The `pack_byte` function should pack DNA sequences into two bytes each. The
trick behind Veedrac's implementation is that "A", "C", "G" and "T" have all
different ASCII representations in the second-to-lowest two bits. So the 
function just shifts the ASCII bytes by one to the right and masks the lowest
two bits. Look ma, no lookup!

TODO

### 
