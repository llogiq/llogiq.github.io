---
title: Ideas for Rust Meetups
---

Since I'm co-organizing the Rhein-Main Rust meetup (and am probably the main driving force behind
it), I thought, it might be useful to share a few ideas we have that we have either already done, or
plan doing – perhaps other meetup organizers can benefit from this. Note that our meetups usually
run 2-4 hours, but some attendees may have to join late or leave early so the format has to take
this into account.

* The usual – some talks if any, sharing experiences
* Discuss a Feature – Select a language feature or std API. Have everyone tell their experience
with it, discuss how it interacts with other features, what idioms it enables, what pitfalls it
enatils, etc.
* Code Review – Everyone can bring a piece of code, the whole meetup reviews it
* Mentoring round: Split the attendees into newbies and mentors. Have the mentors help the newbies
with their current projects (or help them find one if they don't already have one)
* [Clippy lint workshop] – meet up and code clippy lints. It's fun, but needs some preparation
(select lint issues to work on, have a plan how to do it)
* embedded Rust workshop – bring your own (or share) hardware and try to make
the LEDs blink
* [crate polishing workshop] – select a crate, find tasks (e.g. Cargo.toml attributes, README, code
blocks / documentation) and split off small teams to work on them. This works best if the project
maintainer is in the room to juggle the PRs
* Rustc polishing workshop – same as crate polishing workshop, but for some part of the compiler.
Needs to limit scope to a single compiler crate or even a subset of one, find the areas to work on
before starting
* Rust Docs workshop – same thing, but focused on docs
* Similar, we could try barn-raising [rust patterns] or idioms
* Micro Game Jam – Usually one has far more time to create games, so one needs a template project
that has enough structure to work from – perhaps even allow folks to choose between multiple
templates for different game styles
* RFC sniping – Vote on an RFC to 'snipe', read it together and see if you can come up with
improvements ranging from fixing typos to better examples, prior art etc.

----

Have you tried something like this – or something different – at your meetup?
Or do you have other ideas? Discuss on [reddit], [twitter] or [rust-users]!

[Clippy lint workshop]: https://llogiq.github.io/2016/04/16/apology.html
[crate polishing workshop]: https://llogiq.github.io/2017/02/27/cpw.html
[rust patterns]: https://github.com/rust-unofficial/patterns
[reddit]: https://www.reddit.com/r/rust/comments/95ksrz/blog_ideas_for_rust_meetups
[twitter]: https://mobile.twitter.com/llogiq/status/1027137686941954048
[rust-users]: https://users.rust-lang.org/t/blog-ideas-for-rust-meetups/19423?u=llogiq
