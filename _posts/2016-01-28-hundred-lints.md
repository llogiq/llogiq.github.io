---
title: A hundred lints
---

[clippy](https://github.com/Manishearth/rust-clippy) just reached a hundred
lints. Besides the obvious
[partying](https://www.youtube.com/watch?v=3RbnBvQ1bC0), this is a good time to
look at the lints, and see what we've come up with so far.

When Manish started clippy, he had about 5 lints, most of them fairly trivial.
I added my own [`eq_op`](https://github.com/Manishearth/rust-clippy/wiki#eq_op) 
lint, which was a bit more complex (and, as it turned out later, wrong), and we
both followed up with a heap of lints each. Others joined us, we got automated
testing, a CONTRIBUTION.md and even more heaps of lints. But what do we lint 
*against*? What do we lint *for*?

# Reasons to lint

There are a few overarching themes that most of the lints follow:

* Likely Errors – some patterns are usually programmer errors, e.g.
[`bad_bit_mask`](https://github.com/Manishearth/rust-clippy/wiki#bad_bit_mask),
[`cmp_nan`](https://github.com/Manishearth/rust-clippy/wiki#cmp_nan),
[`eq_op`](https://github.com/Manishearth/rust-clippy/wiki#eq_op),
[`empty_loop`](https://github.com/Manishearth/rust-clippy/wiki#empty_loop),
[`match_overlapping_arm`](https://github.com/Manishearth/rust-clippy/wiki#match_overlapping_arm),
[`ineffective_bit_mask`](https://github.com/Manishearth/rust-clippy/wiki#ineffective_bit_mask),
[`min_max`](https://github.com/Manishearth/rust-clippy/wiki#min_max),
[`modulo_one`](https://github.com/Manishearth/rust-clippy/wiki#modulo_one),
[`nonsensical_open_options`](https://github.com/Manishearth/rust-clippy/wiki#nonsensical_open_options),
[`out_of_bounds_indexing`](https://github.com/Manishearth/rust-clippy/wiki#out_of_bounds_indexing),
[`range_step_by_zero`](https://github.com/Manishearth/rust-clippy/wiki#range_step_by_zero),
[`unit_cmp`](https://github.com/Manishearth/rust-clippy/wiki#unit_cmp) and others
* Readability – a good many lints suggest readability improvements, such as
[`approx_constant`](https://github.com/Manishearth/rust-clippy/wiki#approx_constant),
[`block_in_if_condition_stmt`](https://github.com/Manishearth/rust-clippy/wiki#block_in_if_condition_stmt),
[`collapsible_if`](https://github.com/Manishearth/rust-clippy/wiki#collapsible_if),
[`cyclomatic_complexity`](https://github.com/Manishearth/rust-clippy/wiki#cyclomatic_complexity),
[`filter_next`](https://github.com/Manishearth/rust-clippy/wiki#filter_next),
[`len_zero`](https://github.com/Manishearth/rust-clippy/wiki#len_zero),
[`let_and_return`](https://github.com/Manishearth/rust-clippy/wiki#let_and_return),
[`needless_return`](https://github.com/Manishearth/rust-clippy/wiki#needless_return),
[`no_effect`](https://github.com/Manishearth/rust-clippy/wiki#no_effect),
[`temporary_assignment`](https://github.com/Manishearth/rust-clippy/wiki#temporary_assignment),
[`type_complexity`](https://github.com/Manishearth/rust-clippy/wiki#type_complexity),
[`unused_lifetimes`](https://github.com/Manishearth/rust-clippy/wiki#unused_lifetimes)
* Performance – As Rust is touted as a systems language, we have some lints to
avoid patterns that produce suboptimal code. Examples are
[`box_vec`](https://github.com/Manishearth/rust-clippy/wiki#box_vec),
[`boxed_local`](https://github.com/Manishearth/rust-clippy/wiki#boxed_local),
[`cmp_owned`](https://github.com/Manishearth/rust-clippy/wiki#cmp_owned),
[`extend_from_slice`](https://github.com/Manishearth/rust-clippy/wiki#extend_from_slice),
[`linkedlist`](https://github.com/Manishearth/rust-clippy/wiki#linkedlist),
[`mutex_atomic`](https://github.com/Manishearth/rust-clippy/wiki#mutex_atomic),
[`or_fun_call`](https://github.com/Manishearth/rust-clippy/wiki#or_fun_call),
[`redundant_closure`](https://github.com/Manishearth/rust-clippy/wiki#redundant_closure),
[`str_to_string`](https://github.com/Manishearth/rust-clippy/wiki#str_to_string),
[`string_to_string`](https://github.com/Manishearth/rust-clippy/wiki#string_to_string)
* Being a good citizen, this is enforced by
[`unstable_as_mut_slice`](https://github.com/Manishearth/rust-clippy/wiki#unstable_as_mut_slice),
[`zero_width_space`](https://github.com/Manishearth/rust-clippy/wiki#zero_width_space)
and others.

There are a few things to note here.

First, the reasons may at times be contrary. For example, `extend` is certainly
shorter and thus probably more readable than `extend_from_slice`, but the
latter can be faster.

Second, I'd like to repeat my statement from my November 2015
[talk](https://llogiq.github.io/talks/clippy.html) that we're pretty stable
for a Rust compiler plugin – we haven't had a rustup which broke the build for
at least five days now, and usually only need to change stuff every two weeks
or so. Most of my
[June blog entry](http://llogiq.github.io/2015/06/04/workflows.html) is still
valid, although some things moved around a bit.

Third, most lints are pretty good in that they have had no reported false
positives for some time. This is mostly because most of our lints *are* pretty
simple, so there is little room for error. This is great because picking the
low-hanging fruit gives us a lot of bang for very little buck.

Fourth, while most lints (81 to be exact) `warn` by default, three `deny` by
default and sixteen are `allow`ed. The reasons may be different in each case,
but the broad rule of thumb is that we welcome any `allow` lint, but to `warn`
we want to be fairly sure that there is a good benefit to warning and every
coder should at least think twice if the thing linted doesn't apply to their
code. Ẁe only `deny` stuff that either cannot be possibly correct, e.g. bad bit
masks that make the comparison useless or that is actively misleading like a
zero-width space in the code – whoever does this is just plain evil.

Regarding the `allow`ed lints, those are either lints for special cases, e.g.
against *all forms* of numerical truncation, against shadowing (I found too
many matches in the wild which would have swamped the compiler output, so it
stays on `allow`), against String addition, needless mutexes etc. Some of them
simply aren't tested sufficiently to make them `warn` because they could still
have false positives.

# Now what?

We're not done yet. Clippy will get more lints and the lints we have will
become better with time (and effort – so I hereby invite everyone to join us!).
It will also probably become even more opinionated (as we already invariably
are), which may or may not be a good thing. To see if we are on the right
track, Manish has opened a
["RFC" issue](https://github.com/Manishearth/rust-clippy/issues/533), so if you
think we do something wrong, or could do something better, this is the place to
chime in.

Finally, since 2016 was touted as the year of IDE integration for Rust, I'd
really like to explore how clippy could fit into this – especially given that
we often have suggestions that could be automatically (or semi-automatically)
applied.
