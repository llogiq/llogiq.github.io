
---
title: A Java Inheritance Puzzler
---

Recently, [Aaron Turon](https://github.com/aturon) submitted RFC PR 1210 
([rendered](https://github.com/aturon/rfcs/blob/impl-specialization/text/0000-impl-specialization.md)),
which allows overlapping trait `impl`ementations. This would offer many of the
benefits of an inheritance-based model, but alas! also shares some of the
downsides.

The discussion prompted me to 
[warn](https://github.com/rust-lang/rfcs/pull/1210#issuecomment-121143016) that 
people will misuse it to create code they can no longer understand. So I fully
agree that having this feature would be great, but we should discourage people
from using it unless they have good reason to.

I cited an example of a code snippet using inheritance that, despite being 
quite small, is hard to understand. Unfortunately, legal reasons forbid me from 
quoting it, so I created a slightly simplified version that should still 
demonstrate the problem.

Consider the following Java code:

```Java
class A {
	int x = 3;
	int a() { x++; return x + c(); } 
	int b() { return a(); }
	int c() { return x; }
}

class B extends A {
	@Override int a() { return super.a() + c(); }
}

class C extends B {
	@Override int b() { return a() * 3; }
	@Override int c() { return x + 1; }
}
```

This is only 15 lines of code, 2 of which are empty. The example has but 275
characters, of which 33 are annotations the code would compile without. Still, 
it takes some thought to reason about the code.

Can you tell what the result of `new C().b()` will be without running the code? 
How sure are you about the result? Note that while we do have some mutable 
state here, the example could also have been written using method arguments to 
the same effect.

As an aside, the original version of this was an exam question I once gave to 
more than 600 students (with 22 lines of code), of which *only one* was able to 
determine the correct result.
