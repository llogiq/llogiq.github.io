---
title: Plain Text Reigns Supreme
---

Now and then, I stumble over people lamenting that our programs are still restricted by plain text when other means would be better. As a reference for future discussions, I'm writing down my argument why this is wrong here.

# Inertia

I'm bringing this first because it is the strawman every proponent of non-textual format drags out to be shot. *Surely once we start doing something to promote better formats, streams of bytes will take some time to vanish, but vanish it will*... well, no, but nice try.

# Tools / Interoperability

This goes beyond plain inertia: Plain text keeps things interoperable. We can have tools that work regardless of language, because the medium they all rely on is plain text. While some tools are available that enrich the underlying plain text with additional information (e.g. types, scopes, control flow, etc.) to provide functionality, note that this information is *implicit* in the text – it can be recomputed any time. Tools that work on plain text tend to work well together – see the usual UNIX suspects for a demonstration of the priniciple.

# Robustness

<a name="Robustness"></a>It's no surprise the internet runs on plain text protocols. Not only because to transfer stuff over the network, you have to serialize it into a stream of bytes.

Plain text formats are extremely robust. With a non-textual format, when your file gets corrupted, you lose the whole thing. With plain text, you can look into it to see what's wrong, and probably fix it. I'd like to stress this point, because it's actually happening with some *solutions* that embed programs in databases or proprietary binary formats. Flip one bit, lose your whole program.

# Keyboards vs. mouse

[Fitt's Law](http://en.wikipedia.org/wiki/Fitt's_Law) postulates a lower bound for the time to target a UI element using a pointing device, no matter if you are using a mouse, touchpad, trackpad, trackball or other device. On the other hand, trained touch typists (and good programmers usually are) type words without paying attention to the act of typing due to [muscle memory](https://en.wikipedia.org/wiki/Muscle_memory). This also means that given enough training it is possible to type code *faster than thinking it up*.

In short, the keyboard is still the fastest way to transfer thoughts from the brain to the computer. Even though the usual keyboard layout [was created to slow down typing](https://en.wikipedia.org/wiki/Qwerty) (and some programmers are moving to alternatives like Dvorak), there hasn't been any sufficiently strong contender during the last decades (and it wasn't for lack of trying [citation needed]).

However, typing will at least restrict your *input* to streams of characters. Thus it makes sense to use this input for programming.

Case in point: There is an identity management solution called Oracle Waveset, which contains a graphical scripting language that stores its scripts in XML. The resulting code is ugly and so verbose that Java looks like a code golfing-capable language in comparison, but everyone I know using it writes the XML code by hand, because even with all the verbosity, that's still faster and more productive than clicking together the graphical representation.

# Abstraction and Structure

Some people even argue that there is a mismatch between the structured ASTs and their unstructured representation as sequences of bytes. But this reasoning is at least very questionable: The syntax is what gives the programs structure. Parsing, while perhaps not being the solved problem that some see it as, is certainly feasible for a whole lot of languages (then there is Perl), and doesn't add too much complexity; also as in the example above, even if you define your language to embed the AST in some medium format so that parsing is positively trivial, text wins out.

Now suppose *X* years from now on, we all program on 3D visual representations (which may look like what Hugh Jackman did in Password Swordfish) – this certainly looks cool, but good looks aren't getting the job done. We'd still have a textual language below: The things you type that change the representation on screen. Thus the future-vim-script you're typing becomes the de-facto programming language, which is arguably superior to the leaky snazzy animated three-dimensional abstraction on your screen.

Now some people discussing this go the other route and argue that we should store the ASTs on disk and modify them in a structured way. Depending on *how* you store it, you'll either come up with S-Expressions (Congratulations! You reinvented Lisp!), JSON or XML (You reinvented Lisp, but worse) or any binary format (Nothing to congratulate here, see [above](#Robustness)).

# Unfinished Works

The idea of creating programs by starting out with an empty (but valid) AST and apply modifications that ensure the validity of the modified AST isn't new. But in practice, this doesn't work very well. The problem is our brains: We humans are quite messy creatures. We don't think our concepts fully formed, and a lot of programming is more exploration than creation. Our programs are in an invalid state more than half of the time we work on them.

On the flip side, this messyness gives rise to our creativity, so any language that attempts to stifle the former will necessarily suppress the latter. The right solution to syntax errors is smarter error messages, not trying to make syntax errors impossible (a moot point anyway, because the *really* bad errors are semantic).

# Thinking in Text

Finally, every graphical language I've seen so far still employs text (or sometimes mathematical symbols where applicable) to represent most things – whether it's in boxes, circles or what have you. The simple reason is that we humans tend to *think in language*. After all, language is *the* medium we express ourselves in (unless you are a mute who does expressive dancing). While *naming things* is one of the really hard problems programmers face, it's also necessary to make our programs understandable.

For example, suppose I have a variable, and because this is a graphical language, I call it `☠` (for those without unicode display, this is &amp;#x2620;: Skull and Crossbones). Now from that "name", could you understand what this variable *means*? I certainly don't.
