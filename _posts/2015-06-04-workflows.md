---
title: My lint-writing workflow
---

Recently, I wrote a good number of lints for 
[rust-clippy](https://github.com/Manishearth/rust-clippy). In order to 
organize my thoughts and to share my findings, I'm going to write a bit 
about my workflow (or how I'd like it to be; reality is more often than 
not more messy than theory). I hope that this will be useful to 
someone; perhaps even as a guide for people who wish to contribute to 
clippy.

So let's go through the motions of creating a lint: It all starts out 
with an idea *("Hey, we could warn when someone does this thing...")*. 
I usually create an issue on the project so people have a chance to 
discuss (and sometimes bring useful viewpoints into the fray). This 
will also include at least one example (and possibly counterexamples if 
I fear there may be false positives).

To make this post actually useful, let's look at a specific example, 
similar to issue 
[#19](https://github.com/Manishearth/rust-clippy/issues/19): We want to 
match uses of `Option::and_then(..)` where the closure only ever 
returns `Some(..)` and suggest to use `Option::map(..)` instead. So we 
write up the most easy example (and one counter-example) in 
`tests/compile-fail/options.rs`:

```rust
#![feature(plugin)]
#![plugin(clippy)]

#![deny(option_and_then_some)]

// the easiest case
fn and_then_should_be_map(x: Option<i32>) -> Option<i32> {
    x.and_then(Some) //~ERROR Consider using _.map(_)
}

// and an easy counter-example
fn really_needs_and_then(x: Option<i32>) -> Option<i32> {
    x.and_then(|o| if o < 32 { Some(o) } else { None })
}

// need a main anyway, use it get rid of unused warnings too
fn main() {
    assert!(and_then_should_be_map(None).is_none());
    assert!(really_needs_and_then(Some(32)).is_none());
}
```

There are two things at play here: First we actually deny our (yet
unwritten) lint, then we mark the line where the lint will show the 
error with "`//~ERROR`" and the start of the error message.

Next, we actually define the lint. I'm going to create a new file,
because I think we'll be adding more variations of this in the future.

I usually use `syntax::ast::*` directly. Though it has bitten me once 
when writing the 
[`len_zero`](https://github.com/Manishearth/rust-clippy/blob/master/src/len_zero.rs) 
lint (where I inadvertantly used a function that was named the same as the 
one in `rustc::middle::ty` which I actually wanted to use), I find that 
littering my code with `ast::this` and `ast::that` makes it so noisy 
that the blanket import is warranted here.

So without further ado, we declare a lint (in `src/options.rs`):

```rust
use syntax::ast::*
use rustc::lint::{Context, LintArray, LintPass};

declare_lint! { 
    pub OPTION_AND_THEN_SOME, Warn,
    "Warn on uses of '_.and_then(..)' where the contained closure is \
     guaranteed to return Some(_)"
}
```

And create a `struct` and `impl` to actually implement the `LintPass`.
Again, I'm going to name it `Options`, because I suspect we will add more 
lints like this in the future. Also note that we want to check 
method calls, which are `Expr`s, so we implement the `check_expr`
method:

```rust
#[derive(Copy,Clone)]
pub struct Options;

impl LintPass for Options {
    fn get_lints(&self) -> LintArray {
        lint_array!(OPTION_AND_THEN_SOME)
    }

    fn check_expr(&mut self, cx: &Context, expr: &Expr) {
        // insert check here.
    }
}
```

Now before we actually write the check, we want to introduce it in 
[`src/lib.rs`](https://github.com/Manishearth/rust-clippy/blob/master/src/lib.rs):

```rust
// declare the module
pub mod options;

// first, register the Lint
    reg.register_lint_pass(box options::Options as LintPassObject);
    
    // also add it to the clippy lint group:
    reg.register_lint_group("clippy", vec![
    //...
    options::OPTION_AND_THEN_SOME,
//...
]);
```

Now it is time to run `cargo test` so I can see that (a) my code
compiles (it does) and (b) my test fails (it does -- we get 
`error: compile-fail test compiled successfully!`). I will also get a
few warnings about unused arguments in `check_expr`, which I ignore
for now.

Next, I want to actually implement the lint. But before I do this,
wouldn't it be great to be able to see how to match the expression?
Luckily, all `syntax::ast` types implement Debug, so we can use this
to have a look at them. Note that our `fn ...`s are `syntax::ast::Item`s
and can be checked using `check_item`. So I add this function, match
our tested `fn`s by name, and `note` their Debug representation:

```rust
fn check_item(&mut self, cx: &Context, item: &Item) {
    if item.ident.as_str().contains("and_then") {
        cx.sess().note(&format!("{:?}", item))
    }
}
```

Now I am presented with two walls of text that represent the two `fn`s
in my compile-test. The bad thing is that it's very easy to get lost in
the maze of different brackets, but the good thing is that this is an
exact representation of the thing we're trying to (not) match.

(Aside: Perhaps one day, we can create a program that takes a few 
examples and counter-examples creates a lint stub that matches the 
former and ignores the latter. But until then, we have the joy and 
frustration of doing it by hand)

Both our `Item`s have a node of `ItemFn(..)`, and contain a `Block` with 
empty `stmt`s and an `expr: Some(Expr {..})`. This is the expression I 
want to match. So I retire my `check_item` for now (to be resurrected 
when writing another lint) and focus my attention on the `check_expr` 
function. I also copy the output into a random text file for later 
reference.

Our expression has a node of 
`ExprMethodCall(Spanned { node: and_then#3, .. }, ..)`, so I start by
matching this. The `and_then` is actually an Ident, by the way. Now my
`check_expr` function looks like this:

```rust
fn check_expr(&mut self, cx: &Context, expr: &Expr) {
    if let ExprMethodCall(Spanned { node: ref ident, .. }, _, 
            ref args) = expr.node {
        if ident.as_str() == "and_then" {
            cx.span_lint(OPTION_AND_THEN_SOME, expr.span,
                "Consider using _.map(_) instead of _.and_then(_) \
                 if the argument only ever returns Some(_)")
        }
    }
}
```

Now `cargo test` shows that we match on both instances of `and_then`,
which is to be expected, considering we have not yet checked the
arguments of the call if None is ever returned. To complete the lint, 
we have to actually check the arguments.

Note that there are two arguments: The first one is a self argument and
should be of type `Option` -- we better check that this is the case, in
order get rid of false positives should other types define an `and_then`
method.

The second argument is our function or closure. For our first test
example, the Debug output reduced to the representation of the second
argument was:

`Expr { id: 15, node: ExprPath(None, Path { span: Span { lo: 
BytePos(161), hi: BytePos(165), expn_id: ExpnId(4294967295) }, global: 
false, segments: [PathSegment { identifier: Some#2, parameters: 
AngleBracketedParameters(AngleBracketedParameterData { lifetimes: [], 
types: [], bindings: [] }) }] }), span: Span { lo: BytePos(161), hi: 
BytePos(165), expn_id: ExpnId(4294967295) } }`

Yes, this is a very wordy representation, but the gist is that we
have an `ExprPath` whose `Path` evaluates to `Some`.

Add both checks and our `check_item` becomes:

```rust
fn check_expr(&mut self, cx: &Context, expr: &Expr) {
    if let ExprMethodCall(Spanned { node: ref ident, .. }, _, 
            ref args) = expr.node {
        if ident.as_str() == "and_then" && args.len() == 2 &&
                is_option(&args[0]) && is_some(&args[1]) {
            cx.span_lint(OPTION_AND_THEN_SOME, expr.span,
                "Consider using _.map(_) instead of _.and_then(_) \
                 if the argument only ever returns Some(_)")
        }
    }
}
```

Now we need to implement our additional checks (after our `impl`), but
we'll first use our debug output trick again:

```rust
fn is_option(cx: &Context, expr: &Expr) -> bool {
	let ty = &walk_ty(&ty::expr_ty(cx.tcx, expr));
	// here we just output our type:
	cx.sess().span_note(expr.span, &format!("{:?}", ty));
	true
}

fn is_some(expr: &Expr) -> bool {
	true // just return true for the moment
}
```

From that, we learn what our type is:

`tests/compile-fail/options.rs:13:2: 13:3 note: TyS { sty: ty_enum(DefId 
{ krate: 2, node: 117199 }, Substs { types: VecPerParamSpace 
{TypeSpace: [TyS { sty: ty_int(i32), flags: 0, region_depth: 0 }], 
SelfSpace: [], FnSpace: [], }, regions: 
NonerasedRegions(VecPerParamSpace {TypeSpace: [], SelfSpace: [], 
FnSpace: [], }) }), flags: 0, region_depth: 0 }`

Now we can match `ty.sty` as `ty_enum`, retrieve the `DefId` and from
there (somehow) get the type definition.

Stay tuned, I'm going to extend this post once I get around to it.
