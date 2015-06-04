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
[#19](https://github.com/Manishearth/rust-clippy/issues/19): I want to 
match uses of `Option::and_then(..)` where the closure only ever 
returns `Some(..)` and suggest to use `Option::map(..)` instead. So I 
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

There are two things at play here: First I actually deny our (yet
unwritten) lint, then I mark the line where the lint will show the 
error with "`//~ERROR`" and the start of the error message.

Next, I actually define the lint. I'm going to create a new file,
because I think I'll be adding more variations of this in the future.

I usually use `syntax::ast::*` directly. Though it has bitten me once 
when writing the 
[`len_zero`](https://github.com/Manishearth/rust-clippy/blob/master/src/len_zero.rs) 
lint (where I inadvertantly used a function that was named the same as the 
one in `rustc::middle::ty` which I actually wanted to use), I find that 
littering my code with `ast::this` and `ast::that` makes it so noisy 
that the blanket import is warranted here.

So without further ado, I declare a lint (in `src/options.rs`):

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
lints like this in the future. Also note that I want to check 
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

Now before I actually write the check, I want to introduce it in 
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
compiles (it does) and (b) my test fails (it does -- I get 
`error: compile-fail test compiled successfully!`). I will also get a
few warnings about unused arguments in `check_expr`, which I ignore
for now.

Actually, I don't need to run the whole test suite (though it's nice
to do now and then, especially after a rust update), I can specify
which test to run. The command line to do so is:

```
TESTNAME=foo cargo test --name compile-test
```

rust-clippy uses the very clever 
[compiletest_rs](https://crates.io/crates/compiletest_rs) crate written
by Thomas Bracht Laumann Jespersen and Manish Goregaokar, among others
to allow us to check that rustc really fails at the correct lines of
code. The test file to start it is `tests/compile-test.rs` (therefore
the `--name`) and the compile_test machinery actually looks for the
`TESTNAME` environment variable to figure out which file to run.

Next, I want to actually implement the lint. But before I do this,
wouldn't it be great to be able to see how to match the expression?
Luckily, all `syntax::ast` types implement Debug, so I can use this
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
exact representation of the thing I'm trying to (not) match.

Note: After I had written the above, 
[Manish Goregaokar](http://manishearth.github.io/) made a suggestion 
that makes this `check_item` business pretty moot: You can just comment
out the plugin lines (because else rustc will hiccup) and run the 
following command to get a complete JSON representation of the abstract
syntax tree:

`rustc tests/compile-fail/options.rs -Z ast-json`

If this looks fairly unreadable, you may want to pipe it through some
JSON pretty printer. I use `aeson-pretty -i 2` to that effect.

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
which is to be expected, considering I have not yet checked the
arguments of the call if None is ever returned. To complete the lint, 
I have to actually check the arguments.

Note that there are two arguments: The first one is a self argument and
should be of type `Option` -- I better check that this is the case, in
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

Yes, this is a very wordy representation, but the gist is that I
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

Now I need to implement our additional checks (after our `impl`), but
I'll first use our debug output trick again:

```rust
fn is_option(cx: &Context, expr: &Expr) -> bool {
	let ty = &walk_ty(&ty::expr_ty(cx.tcx, expr));
	// here I just output our type:
	cx.sess().span_note(expr.span, &format!("{:?}", ty));
	true
}

fn is_some(expr: &Expr) -> bool {
	true // just return true for the moment
}
```

Now there is something I need to explain: `expr_ty` is a function in 
`rustc::middle::ty` I just imported which returns the type of a given 
expression as `rustc::middle::ty::Ty`. The `walk_ty` function is part 
of rust-clippy, and resides in 
[`types.rs`](https://github.com/Manishearth/rust-clippy/blob/master/src/misc.rs#L13).
This function is used to get rid of all `&`-references the type may
have and gets us to the concrete type definition. 

From that, I learn what our type is:

`tests/compile-fail/options.rs:13:2: 13:3 note: TyS { sty: ty_enum(DefId 
{ krate: 2, node: 117199 }, Substs { types: VecPerParamSpace 
{TypeSpace: [TyS { sty: ty_int(i32), flags: 0, region_depth: 0 }], 
SelfSpace: [], FnSpace: [], }, regions: 
NonerasedRegions(VecPerParamSpace {TypeSpace: [], SelfSpace: [], 
FnSpace: [], }) }), flags: 0, region_depth: 0 }`

Now I can match `ty.sty` as `ty_enum`, retrieve the `DefId` and from
there (somehow) get the type definition.

Before I do that, I'm going to give a broad overview over the services
rustc affords us lint writers:

* the `Context`: It is given to all `check_*` functions and besides
having methods to actually report a finding, it has two members of
interest: the `tcx` (which is of type `ctx` – go figure), which has 
just about all context information the compiler is using available, and
the Session, which can be retrieved by the `cx.sess()` method, and
which has other methods to make our lives easier, notably the 
`codemap()`, which has a `span_to_snippet(span: Span) -> Option<&str>`
method to get the actual code marked by a `Span`, which is available on
many AST types. The codemap also has a 
`with_expn_info(id: ExpnId, fn: F) -> T where F: FnOnce(Option<&ExpnInfo>) -> T`
method that can be used with the `expn_id` of a `Span` to find out if
the macro that expanded to this piece of code, if any. The 
[`in_macro`](https://github.com/Manishearth/rust-clippy/blob/master/src/utils.rs)
function can take this `Option<&ExpnInfo>` to determine if the macro
that expanded the code was in the current crate. We use that in some
lints to reduce false positives.
* the `rustc::middle::ty` crate has an awesome lot of functions to 
slice, dice and do everything nice to types as the type checker sees
or infers them. Notable uses within rust-clippy are: 
[`len_zero`](https://github.com/Manishearth/rust-clippy/blob/master/src/len_zero.rs)
uses it to actually look up methods in a type of an expression, 
[`mut_mut`](https://github.com/Manishearth/rust-clippy/blob/master/src/mut_mut.rs)
uses the same `expr_ty` function we use above to find mutable
references to mutable references which can be matched as `ty_ptr` or
`ty_rptr` depending on whether there's a lifetime bound, 
* finally, the AST itself contains many helpful and valuable functions
one can use.

Stay tuned, I'm going to extend this post once I get around to it.
