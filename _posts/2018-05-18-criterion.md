---
title: Porting Rust Benchmarks To Criterion
---

A few weeks ago, I set out to convert [bytecount](https://github.com/llogiq/bytecount)'s benchmarks
to [criterion](https://docs.rs/criterion), a statistics-driven benchmarking framework started by
Jorge Aparicio and maintained by Brook Heisler.

Before, bytecount used [bencher](https://docs.rs/bencher) for its benchmarks, which is a straight
port of the unstable, nightly-only `std::test` benchmark framework, extended to work with stable
Rust. This was a great benefit compared to `std::test`, because now we could benchmark on all Rust
versions (stable, beta, nightly, some specific version) without needing to fear regressions.

Being a data engineer, I always wanted to try criterion since I first heard of it, so when Brook
took over maintenance of the project and made some critical usability improvements, I finally dug
in and converted our benchmarks.

First, the good news: The usability benefits translate into the removal of a lot of macros. Where
we needed one benchmark function per benchmark size, we can now specify a suite of functions and
a slice of sizes, and Criterion will take care of the rest. Neat!

The statistics part means that criterion is generally somewhat slower than bencher, but the results
are very likely more reliable. And we can get pretty graphs – who doesn't love those?

Second, the bad news: I had a hard time convincing both Travis and AppVeyor to run our Criterion.rs
benchmarks. Especially with larger byte slices, we get in the realm of a macrobenchmark, so the
statistical rigor isn't really needed here, but makes us run against various build timeouts.

Luckily, since we no longer need to know the benchmark sizes at compile time, we can supply them
via a commandline argument at runtime – this is also a win for quick tests. Also the defaults are
pretty solid for running your own benchmarks, but too heavy for a CI, so I had to reduce warmup and
measurement time and the number of resamples. I also disabled graph generation on CI, because we
don't do anything with the results anyway.

So how does one convert Rust benchmarks to criterion?

First, you no longer need `#[bench]` attributes on functions. Those we had already removed when
switching to the `bencher` crate. Instead, you need to declare `criterion_group!(..)`s and one
`criterion_main!(..)` with all group names at the end of your benchmark file. The latter has but
one syntax, it just takes a list of group names. The former has two syntax variants. In bytecount,
we use the more complex one, that allows us to configure the Criterion runner for this group, but
for most benchmarks the first form will be sufficient. The benchmark functions get the Criterion
object and can set up and run the benchmarks programmatically:

```
fn target(criterion: Criterion) {
    criterion.bench("target",
        Benchmark::new("foo", ..));
}

criterion_group!(group_name, target);
criterion_main!(group_name);
```

This is pretty similar to Bencher, apart from the indirection in the target, which allows us to
introduce multiple input values, comparisons, etc.

The verdict: Criterion is a tad more complex than Bencher. It certainly has the better statistics
(though it could in theory improve even more), and allows for finer-grained control, but the
complexity still comes at a cost – in this case runtime, especially on larger/slower benchmarks.
Perhaps Brook should introduce a compatibility layer that allows bencher-benchmarks to run
with criterion unchanged – this would allow folks to cheaply test before switching.
