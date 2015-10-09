---
title: Lints That Collect Data Per Crate
---

Here's a short description of a technique I encountered while trying to write by [Cow lint](TODO). When we have some more complex things to lint, we may sometimes require visiting the whole crate before reporting something – be it to weed out false positives or to add other related parts of the code in the report.

The technique has two parts: Data collection and deferred reporting. Since the data collection part will vary by lint, I will assume that you already know how to store data in a `struct`.

The trick is to use a `Visitor` implementation for walking the crate instead of walking it within the `*LintPass`. This looks like the following (I use a `LateLintPass here, but the `EarlyLintPass` works just the same):

```Rust
// our lint pass is but a shell to call the visitor
struct MyLintPass;

impl LintPass for MyLintPass {
    fn get_lints(&self) -> LintArray {
        lint_array!(MY_LINT)
    }
}

impl LateLintPass for MyLintPass {
    /// we only override `check_crate(..)` to get called once per crate
    fn check_crate(&mut self, cx: &LateContext, krate: &Crate) {
        let mut cv = MyVisitor::new(cx);
        cv.walk_crate(krate); // use the Visitor to walk the crate
        cv.report(krate); // report now we have all data
    }
}

struct MyVisitor<'v, 't: 'v> {
    cx: &'v Context<'v, 't>,
    .. // data storage here
}

impl MyVisitor {
    fn report(self, krate: &Crate) {
        .. // whatever you want to report here
    }
}

impl Visitor for MyVisitor {
    .. // collect data here
}
```

The `rustc_front::visit::Visitor` trait is quite similar to `LateLintPass` (apart from missing the `Context`, but we compensate by storing a reference to it within our visitor). One thing we need to keep in mind is that whenever we override a method, we need to call the corresponding `walk_..` method from the `rustc_front::visit` module, otherwise our Visitor will ignore whatever is in the thing we visit with the method. So if we implement `visit_fn(..)` and forget to call `walk_fn(self, fnkind, fndecl, block, span)`, our visitor will not even look into the innards of any function. While this may well be acceptable for some lints, it's something to keep in mind.
