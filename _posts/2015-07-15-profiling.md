---
title: Profiling Rust applications on Linux
---

What's measured gets optimized. So if we want high-performance Rust code, we
need to measure it. Some time ago I 
[wrote about benchmarks](/2015/06/16/bench.html), and how Rust's test crate
makes it easy to set up a simple benchmark.

Now I'm going to tackle the second type of performance measurement: Profiling.
Whenever we want to optimize, we better profile our code to see where the time
is spent.

Now Rust has no gprof support, but on Linux there are a number of options
available to profile code based on the DWARF debugging information in a binary
(plus supplied source). While the `valgrind`-based tools (for our requirements 
callgrind) use a virtual CPU, `oprofile` reads the kernel performance counters
to get the actual numbers. 

Aside: [JMH](http://openjdk.java.net/projects/code-tools/jmh/) (Java 
Microbenchmark Harness), which is poised to become a part of JDK9, uses the 
same trick with the `-prof perfasm` option.

So I needed some code to test the profiler on. I took the *nbody* benchmark
from teXitoi's repository (also the C++ #8 benchmark from the benchmarksgame 
page to compare against). The benchmarksgame reports 2× runtime for the Rust
program compared to the C++ program.

Compiling and running the C++ program yielded:

```
$ /usr/bin/g++ -g -c -pipe -O3 -fomit-frame-pointer -march=native -mfpmath=sse \
  -msse3 --std=c++11 nbody.gpp-8.c++ -o nbody.gpp-8.c++.o && /usr/bin/g++ \
  nbody.gpp-8.c++.o -o nbody_gpp-8 -fopenmp
$ time ./nbody_gpp-8 50000000
-0.169075164
-0.169059907

real    0m8.417s
user    0m8.412s
sys     0m0.000s
```

So my PC is a bit faster than the benchmark server. Nice!

Next, let's have a look at the Rust version. I'm using the nightly channel and
compile with `-O`:

```
$ rustc -O nbody_rs.rs
$ time ./nbody_rs 50000000
-0.169075164
-0.169059907

real    0m12.651s
user    0m12.648s
sys     0m0.000s
```

Now this is much better than the 2× result in the benchmark, rather 1.5×C++
performance. Still there's a difference, and profiling the rust code will show
us where it likely is.

Using valgrind
--------------

Installing valgrind can be done with your system's package manager, otherwise
just download the source package; a `configure && make && make install` should
do the trick.

Valgrind uses the DWARF debugging information, so we need to recompile with 
`-g`:

```
$ rustc -O -g nbody-rs.rs
```

Using callgrind to collect the profile is simple, just prefix 
`valgrind --tool=callgrind` to the program. This will collect the statistics to
be used by `callgrind_annotate` later:

```
$ valgrind --tool=callgrind ./nbody 50000000
==11135== Callgrind, a call-graph generating cache profiler
==11135== Copyright (C) 2002-2013, and GNU GPL'd, by Josef Weidendorfer et al.
==11135== Using Valgrind-3.10.1 and LibVEX; rerun with -h for copyright info
==11135== Command: ./nbody 50000000
==11135== 
==11135== For interactive control, run 'callgrind_control -h'.
-0.169075164
-0.169059907
==11135== 
==11135== Events    : Ir
==11135== Collected : 29800561414
==11135== 
==11135== I   refs:      29,800,561,414

$ callgrind_annotate --auto=yes callgrind.out.11135
[...]
            .              for bj in b_slice.iter_mut() {
1,000,000,000                  let dx = bi.x - bj.x;
1,000,000,000                  let dy = bi.y - bj.y;
1,000,000,000                  let dz = bi.z - bj.z;
            .  
4,000,000,000                  let d2 = dx * dx + dy * dy + dz * dz;
2,000,000,002                  let mag = dt / (d2 * d2.sqrt());
            .  
  500,000,000                  let massj_mag = bj.mass * mag;
2,500,000,000                  bi.vx -= dx * massj_mag;
2,500,000,000                  bi.vy -= dy * massj_mag;
2,000,000,000                  bi.vz -= dz * massj_mag;
            .  
  500,000,000                  let massi_mag = bi.mass * mag;
1,500,000,000                  bj.vx += dx * massi_mag;
1,500,000,000                  bj.vy += dy * massi_mag;
2,000,000,000                  bj.vz += dz * massi_mag;
            .              }
1,000,000,000              bi.x += dt * bi.vx;
1,000,000,000              bi.y += dt * bi.vy;
  750,000,000              bi.z += dt * bi.vz;
            .          }
[...]
```

At least the values are all sufficiently high, but the cost model seems to be
overly simplistic. I'll discuss this on reddit, so if someone knows how to get
more usable values out of this, drop me a line.

Since this did not show us too much (besides the predictable fact that the
inner loop of our `advance()` function is the hottest code), let's have a look
at `oprofile`.

Using oprofile
--------------

To install oprofile, I simply typed:

```
$ sudo apt-get install linux-tools-generic oprofile
```

Your system may have similar options. If you cannot use a package manager, the 
oprofile manual has an 
[installation guide](http://oprofile.sourceforge.net/doc/install.html) that
tells you how to install from source.

Now let's have a look where our nbody-rs spends its CPU time. First we call 
`operf` to collect the profile, then we can use `opannotate` to show us an 
annotated source.

Note that like `callgrind_annotate`, `opannotate` uses the DWARF debugging
symbols and relies on the source code being available. Luckily we have already
compiled with `-g`.

```
$ operf ./nbody-rs 5000000
Kernel profiling is not possible with current system config.
Set /proc/sys/kernel/kptr_restrict to 0 to collect kernel samples.
operf: Profiler started
-0.169075164
-0.169059907

Profiling done.
$ opannotate --source
[... some inconsequential errors and source ...]
              :fn advance(bodies: &mut [Planet;N_BODIES], dt: f64, steps: i32) {
               :    for _ in (0..steps) {
               :        let mut b_slice: &mut [_] = bodies;
               :        loop {
               :            let bi = match shift_mut_ref(&mut b_slice) {
               :                Some(bi) => bi,
               :                None => break
               :            };
               :            for bj in b_slice.iter_mut() {
  4352  1.2693 :                let dx = bi.x - bj.x;
  1425  0.4156 :                let dy = bi.y - bj.y;
  3348  0.9765 :                let dz = bi.z - bj.z;
               :
 20010  5.8361 :                let d2 = dx * dx + dy * dy + dz * dz;
146801 42.8157 :                let mag = dt / (d2 * d2.sqrt());
               :
  3171  0.9248 :                let massj_mag = bj.mass * mag;
 53290 15.5425 :                bi.vx -= dx * massj_mag;
  7835  2.2851 :                bi.vy -= dy * massj_mag;
  5508  1.6065 :                bi.vz -= dz * massj_mag;
               :
   886  0.2584 :                let massi_mag = bi.mass * mag;
  6670  1.9454 :                bj.vx += dx * massi_mag;
  3720  1.0850 :                bj.vy += dy * massi_mag;
  5695  1.6610 :                bj.vz += dz * massi_mag;
               :            }
 36886 10.7581 :            bi.x += dt * bi.vx;
  7109  2.0734 :            bi.y += dt * bi.vy;
               :            bi.z += dt * bi.vz;
               :        }
               :    }
```

This is the hottest function, and we can see the line with the division and
square root is taking more than 40% of the runtime. If we add `--assembly` to
`opannotate`, we'll get the following:

```
[... more inconsequential stuff ...]
               :                let d2 = dx * dx + dy * dy + dz * dz;
   403  0.1175 :    58ec:       movapd %xmm3,%xmm4
               :    58f0:       mulsd  %xmm4,%xmm4
  5532  1.6135 :    58f4:       movapd %xmm1,%xmm5
   896  0.2613 :    58f8:       mulsd  %xmm5,%xmm5
  1992  0.5810 :    58fc:       addsd  %xmm4,%xmm5
  6752  1.9693 :    5900:       movapd %xmm2,%xmm4
  4435  1.2935 :    5904:       mulsd  %xmm4,%xmm4
               :    5908:       addsd  %xmm5,%xmm4
  9860  2.8758 :    590c:       xorps  %xmm5,%xmm5
               :    590f:       sqrtsd %xmm4,%xmm5
               :                let mag = dt / (d2 * d2.sqrt());
 54280 15.8312 :    5913:       mulsd  %xmm4,%xmm5
 14744  4.3002 :    5917:       movapd %xmm0,%xmm4
               :    591b:       divsd  %xmm5,%xmm4
 77777 22.6843 :    591f:       movsd  0x30(%rsi),%xmm5
```
 
Strangely, the sqrtsd has been reported before the source line, but let's not
get hung up on that detail. We can see that the `movsd` instruction that writes
the values from the SSE registers back into memory takes most of the time –
which is to be expected.

By the way, the C++ version cheats a bit by using single precision for the
square root, also it uses the `rsqrtps` instruction via compiler instrinsics,
which approximates an inverse square root. Well, the result is the same, so
I'll let it slide. Here is the relevant section from the `opannotate` output:

```
               :        dsquared = dx[0] * dx[0] + dx[1] * dx[1] + dx[2] * dx[2];
               :  400b9b:       mulpd  %xmm2,%xmm2
  1624  0.7119 :  400b9f:       movhpd -0x18(%rax),%xmm1
               :  400ba4:       mulpd  %xmm1,%xmm1
  1054  0.4620 :  400ba8:       movhpd -0x10(%rax),%xmm0
   882  0.3866 :  400bad:       mulpd  %xmm0,%xmm0
  1159  0.5081 :  400bb1:       addpd  %xmm2,%xmm1
  3263  1.4304 :  400bb5:       addpd  %xmm1,%xmm0
               :        distance = _mm_cvtps_pd(_mm_rsqrt_ps(_mm_cvtpd_ps(dsquared)));
               :
               :        for (m=0; m<2; ++m)
               :          distance = distance * _mm_set1_pd(1.5)
               :            - ((_mm_set1_pd(0.5) * dsquared) * distance)
  8152  3.5736 :  400bb9:       movapd %xmm0,%xmm2
               :}
               :
               :extern __inline __m128 __attribute__((__gnu_inline__, __always_inline__, __artificial__))
               :_mm_cvtpd_ps (__m128d __A)
               :{
               :  return (__m128)__builtin_ia32_cvtpd2ps ((__v2df) __A);
  1666  0.7303 :  400bbd:       cvtpd2ps %xmm0,%xmm3
               :}
               :
               :extern __inline __m128 __attribute__((__gnu_inline__, __always_inline__, __artificial__))
               :_mm_rsqrt_ps (__m128 __A)
               :{
               :  return (__m128) __builtin_ia32_rsqrtps ((__v4sf)__A);
  2003  0.8781 :  400bc1:       rsqrtps %xmm3,%xmm3
  4126  1.8087 :  400bc4:       mulpd  %xmm7,%xmm2
               :}
               :
               :extern __inline __m128d __attribute__((__gnu_inline__, __always_inline__, __artificial__))
               :_mm_cvtps_pd (__m128 __A)
               :{
               :  return (__m128d)__builtin_ia32_cvtps2pd ((__v4sf) __A);
  1464  0.6418 :  400bc8:       cvtps2pd %xmm3,%xmm3
  2036  0.8925 :  400bcb:       movapd %xmm3,%xmm8
               :
               :}
```

There are other metrics to look for, and oprofile has the `ophelp`  program 
that will show all performance counters you can collect. However, those are
outside the scope of this post.

For now you have a good way to CPU-profile your program with very little 
overhead and with the option of collecting other very useful statistics.
