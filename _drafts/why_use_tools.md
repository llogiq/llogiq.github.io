---
title: (Why) Should We Use Tools On Our Code?
---

As some of you will already know, I'm a big proponent of tools that do things to my code. Whether it's [clippy](https://github.com/Manishearth/rust-clippy), [FindBugs](http://findbugs.sourceforge.net), [rustfmt](http://github.com/nrc/rustfmt), [sloccount](http://www.dwheeler.com/sloccount), [doxygen](http://www.stack.nl/~dimitri/doxygen), [infinitest](http://infinitest.github.io), [doctest](https://docs.python.org/2/library/doctest.html), [afl](http://lcamtuf.coredump.cx/afl), gcov or whatever, I'll run it on my code (where applicable of course), unless it's too much of a hassle. I even spell-check my comments.

If you now say "Yeah, me too", you can stop reading here.

Ah, you're still here. You probably also wonder why I use all this stuff. Is it worth it? Why should *I* do this? Let's first identify some tool categories and then look at the pros and cons for each of them. The tools I mentioned above fall into one or more of the following categories: [Static Program Analysis](https://en.wikipedia.org/wiki/Static_program_analysis)/Lints, Metrics, Formatting, Documentation and Testing.

### Static Program Analysis

I'm incredibly biased here. Having written tools in that space for multiple languages, I'm confident that such tools improve both the code and the mental model of the programmer. With that out of the way, why should you use them? 

Often, such tools may highlight issues or show avenues for optimization the programmer would not have even considered themselves. People aren't that good at detecting subtle patterns in code, especially if those patterns have low locality. Static analysis can help because it has all the meta data, won't overlook things or be blinded by appearance.

*But*, I hear some of you say, *I'm already an experienced coder. My code is* good. Great! Still that doesn't exempt you from the fallibility that makes us human (unless you are already a [bot](https://reddit.com/r/botsrights)). When we started to run clippy on [rustc](https://github.com/rust-lang/rust) itself, we got more than a hundred valid suggestions for improvement (which I subsequently implemented). Are the rust programmers newbies? Are they bad programmers? Nothing could be further from the truth. They just didn't have access to the tool before.

Improving our code based on a lint's suggestion has a profound consequence: You'll be more likely to detect those patterns in the future yourself. Believe it or not, it will make you a better programmer. I experienced this when I started running FindBugs on my Java code. Soon I noticed problems with my code that I would have overlooked just weeks ago. Nowadays, my code is FindBugs clean except in very rare circumstances. However, I still run FindBugs on my code, because it's cheap, and it still catches the occassional mishap.

### Metrics

**Metrics are useless**. We simply don't have any *meaningful* metric for source code. To gauge a programmers output by lines of code is the same as measuring an aircraft engineer's progress by weight and all that jazz. I'll argue that there still is some value in those metrics, if you are alright with having a good helping of salt when interpreting them.

For example, I recently looked at an estimate of the numbers of lines of code I had written in Rust, and concluded that I may not call myself a newbie anymore. When I look at this post, I'll look at the word count to ensure it's not boring you too much. I also try to keep my functions below about 30 lines of code, since that's what the source window of my editor will show at once.

### Formatting

Consistency is vital in reading comprehension, so using a good formatting tool to ensure it should be a no-brainer. Of course, there are instances where you can do better than the tool and there should be a way to override it (e.g. via an annotation or comment). Also when everyone on a project has the same formatting, diffs between versions will be reduced to the actual changes instead of mixing real changes with unrelated formatting changes. Again, computers are better at being consistent than us humans, so why not enlist them to help?

### Documentation

Here my experience has been more mixed. While the tools are usually good enough to create something decent, sometimes they engender bad practices, as the proverbial content-free javadoc comment:

```java
    /**
     * frob the bar
     * @param bar the bar
     * @return the frobbed bar
     */
    public Bar frob(Bar bar) { ... }
```

That said, documentation tools, when used correctly can be amazingly useful, especially for libraries, which aren't complete without good documentation. Since documentation is a creative process (like writing a book), the tools cannot protect you from doing a bad job of it. Projecting failure to write meaningful documentation on the tools used to generate a browseable (or printable, or ...) version is just being dishonest with oneself.

### Testing

Apart from static analysis (with its very limited knowledge about your problem), testing is still the cheapest way to find problems in code. Tools to make testing easier makes this even cheaper. Now when I write "testing", a good number of people will reflexively prepend "unit", and think frameworks like JUnit, Spock, py.test, test::more of whatever your language has, would be the one and only tool needed to test everything.

This of course misses out on other tools:
* [doctest](https://docs.python.org/2/library/doctest.html) runs examples in documentation as tests. This is great because it lets your documentation examples serve double duty: As documentation (good docs contain examples!) and unit-tests. This also means your tests will show you if your docs go stale. There are doctest tools for many languages (full disclosure: I wrote one for lua), others have them built in.
* [american fuzzy lop](http://lcamtuf.coredump.cx/afl) is a code-directed fuzz testing tool. A fuzz testing tool will generate inputs for your program to make it crash. This one uses code instrumentation to help guide its input generation, so it will find problematic inputs relatively fast with very little configuration. If you have some idle CPU, this is a great tool to find out surprising things about your program, and become a little more paranoid in the process.
* There are of course tools that enhance unit testing, like [QuickCheck](https://en.wikipedia.org/wiki/QuickCheck), which will pseudorandomly autogenerate inputs for your functions to test), and [infinitest](http://infinitest.github.io), which whenever you save a file will automatically run all unit tests that have a dependency on that file.

Again, those tools will help you find problems with your code that you either cannot find or would need much more effort to find. Of course *my code is perfect, bugs only exist in other people's code*, but even then wouldn't it be good to know about them? I'll leave the choice to you.
