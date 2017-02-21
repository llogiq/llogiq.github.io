---
title: The System Programming Enclosure Movement
---

When Systems Programming was conceived, programmers either toggled switches to
enter code or even manually *wired* the systems to get the desired effect.
Programmers did not know good from evil, all they knew was they were forbidden
to eat from the tree of abstraction.

Suffice to say, the toggling and wiring was not very ergonomic, so a few
[bites] later we got assembly. A deluge of assemblers later, the survivors had
primitive macros, so they could build structures on top of another.

People begin building a tower to the heavens out of assembly code, which
angered some deity, so we get different programming languages. Lots of them.
And people coding in language A could not understand people coding in language
B and vice versa. The tower project was cancelled.

Still, the new languages had little in the way of managing memory, because when
all you've got is a hundred bytes (yes, *bytes*, without any prefix), keeping
track of what's in there is not only vital, but also manageable.

When memory got more plentiful, we got RAII, reference counting and garbage
collection. While that made some things slower – sometimes dramatically so – it
allowed for other things that were previously impossible. Things in memory
started to be *owned*, but most things belonged to no one in particular.

Fast forward to 2015. We now have systems with multiple gigabytes of memory. In
fact, I personally ran a job that year that used a *90GB* heap. And this was
all in RAM – we had deactivated swap before to avoid trashing.

During that year, [Rust] reaches 1.0. This is kind of a big deal, because
where people didn't really mind their stuff in memory and left garbage
collectors to deal with it, we now have a language where just about everything
is owned. By a *single* owner.

Imagine that. Where other languages could have parishes, or clubs, or other
associations (or just the whole town) that could own things collectively, in
Rust they must use appoint an `Rc` (or `Arc` if they live on different threads)
to own stuff, or declare one of them the canonical owner.

The argument in favor of this [enclosure] movement is that commonly owned
memory tends to become a garbage heap. Since Rust as a systems language wants
to avoid paying garbagepeople, they needed a single owner to take the
responsibility. Rustaceans often also argue that it leads to better code. On
the other hand, canonical ownership *is* restrictive, and some algorithms are
even deemed infeasible without garbage collection.

Ownership does not imply exclusive usage – Rust has *borrowing* to let multiple
entities work with stuff. The people who did not own stuff (because until now
they could always access everything they needed) found themselves talking the
owner of stuff into giving them a reference – many found out they only wanted
to *look at* stuff, so they only needed an immutable reference; which optimized
memory usage further, because many folks could look at the same thing at the
same time.

Elsewhere, people wanting to *change* stuff needed to secure a *mutable
reference* from the owner. For fear of interference, no one was allowed to look
at stuff during the change (for those who balk at this rule, applying a similar
rule to car crash sites would certainly reduce further fatalities). Since
owners are fairly liberal about lending out stuff, this system works remarkably
well.

Also there are some types designed to skirt the law, allowing to change things
from an immutable borrow. They are all based on a type called `UnsafeCell`,
which offers a vivid description of where those who misuse this particular
device will end up.

Still, this "enclosure" is not without detractors. Some fear that the owners
might turn into misers, only lending out scraps or even nothing. So far, those
fears have been unfounded, but looking into the future is always a risky
business.

Others decry the complexity that has been strewn across their code; with
lifetime generics and additional blocks to appease the borrow checker. However,
they essentially compare Rust to places where the garbagemen pick up all their
litter every Thursday, and they don't want to pay for the service either.
*This* is the real tragedy of the commons.

[bites]: https://en.wikipedia.org/wiki/Grace_Hopper
[Rust]: https://rust-lang.org
[enclosure]: https://en.wikipedia.org/wiki/Enclosure
