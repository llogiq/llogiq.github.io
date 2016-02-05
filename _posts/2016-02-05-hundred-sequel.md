
---
title: A hundred lints: The sequel
---

My last blog entry got some nice feedback, and here I'm trying to give
my thoughts to some of it:

> Rust-clippy is great, but I always had something like Ada's
restrictions in mind too
(https://en.wikibooks.org/wiki/Ada_Programming/Pragmas/Restrictions …)
.. anyone for a rust-dogme?

— Graydon Hoare on
[twitter](https://twitter.com/graydon_pub/status/695048766622011392)

@Graydon: You are hereby cordially invited to submit an
[issue](https://github.com/Manishearth/rust-clippy/issues/new) to
discuss this further. :-)

With that said, we should bear in mind that some things are still hard
to ensure (or at least check) in Rust:

* That an arbitrary function within an external library will not diverge (i.e.
check for `panic!()`)
* That a function is pure ([mcarton](https://github.com/mcarton) recently
suggested viewing functions with immutable arguments as pure, but this fails
to account for interior mutability, alas. Still it may be a useful heuristic
nonetheless, depending on the level of "purity" required)
* That a function won't recurse (endlessly or at all)
* That a function won't do IO
* That a function won't allocate
* That a function does not include undefined behavior
* That a function does not call into FFI code
* That a function stays within a particular stack size

The last one is currenly a real problem, as it causes people's code to
segfault due to stack overflow. Some of those people subsequently go on
reddit to complain that their "safe" language let them crash their
program. :-P I think the next-to-last one is *very* interesting – we
obviously have less UB in Rust than in other languages, but you never
know if that function you call introduces some of it.

Why is this hard? I blame the crate barrier. Note that I don't want to
break it down and recompile everything all the time. But we may want to
think about upping rustc's compiled-code introspection capabilities so
we can answer those questions with some certainty. Apart from allowing
us to reason about the aforementioned questions, it could make some of
our current lints better, which now have to take a rather conservative
stance.

Now how would this be done? I figure we'd need to make some crate
metadata that we should be able to get accessible to lints (and the
rest of the compiler). The call graph of a function, for example (a
list of functions that each function calls would be sufficient, though
this obviously stops at the FFI boundary, so there may be some false
negatives). If a function accesses any global state (which is locally
discoverable). Note that we have all this information crate-locally
while we're compiling a crate. We just don't store it in the rlib, as
far as I know.

----

Some have expressed their belief that many lints of clippy belong in
rustc. I'm personally at best lukewarm on this for various reasons: We
can develop lints better if we don't have to wait half an hour for the
thing to compile :-P. Also we can try out more stuff without slowing
the rustc people down with our follies – the resulting communication
fallout from a lint-breaking change (which, while not too frequent,
still occurs every now and then) would take up time that both teams can
put to better purposes. Finally I think that we shouldn't think about
the *compiler* too much, but focus on the development environment and
focus on improving integration there.

----

Since I wrote the last post, we have gained a good number of new lints
(and are at 110 at the time of this writing, a mere 8 days after the post).
Some bugs have been fixed, yet others remain. I'm personally very happy
with how the project advances, Manish's formidable leadership and our
focus on being inclusive has given us fertile soil for many more
helpful improvements, which will hopefully benefit a good part of the
Rust ecosystem.

I've also just started writing library-specific lints (for [Andrew
Gallant](https://github.com/BurntSushi)'s glorious
[regex](https://crates.io/crates/regex) crate) – we have not yet
decided if those will stay in clippy or become their own `regex-lints`
crate. So library crate authors: If you find that your library API can
be misused, but lack a zero-overhead way of preventing that misuse,
perhaps a lint could at least flag the problematic combinations? I'd
like to hear from security-relevant crates in particular.

Discuss this on [r/rust](https://reddit.com/r/rust) or
[rust-users](https://users.rust-lang.org)!
