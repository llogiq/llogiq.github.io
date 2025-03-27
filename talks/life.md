---
marp: true
author: Andre 'llogiq' Bogus
title: Rust Life Improvements
description: A series of hints to Rust faster, better, stronger...
class: invert
---
<style>
:root {
  background: #000;
}
</style>
# Rust Life Improvements
<!--
You say that Rust is like a religion
the community is toxic
and you rather stay apart.
You say that C can be coded safely
that it is just a skill issue
still I know you just don't care.

R: And I say "what about mem'ry unsafety?"
You say "I think I read something about it
and I recall I think that hackers quite like it"
And I say "well that's one thing you got!"

In C you are tasked with managing mem'ry
no help from the compiler
there's so much that can go wrong?
So what now? The hackers are all over
your systems, running over
with CVEs galore.

R: ...

You say that Rust is a woke mind virus,
rustaceans are all queer furries
...

And now let's jump right in!
-->
---
## rustup

Try the beta channel!

- 6 weeks perf improvements compared to stable
- good stability track record
	- despite 34 point releases up to 1.84.1
	- *if* you encounter a bug, you can help the ecosystem
- no daily recompiles as with current nightly

---
## Beta

![w:20em](life/days_to_minor_version.svg)

<!-- Sorry that the graph is from right to left. I didn't find the time to
fix that. The median point release - if there was one - can out 15 days after
the point zero release. So if you can wait a bit more than two weeks to update,
you'll still have many of the benefits with only half the point updates.

Note that every problem found in beta will also create a new beta backport.

Let's move on to cargo -->

---
## Tooling: `cargo run --example`

```sh
$ ls examples
foo.rs bar.rs

$ cargo run --example foo
```

<!-- perhaps not all of you know that cargo can run examples like this? -->

---
## Tooling: cargo shortcuts

```sh
$ cargo b # build
$ cargo c # check
$ cargo d # doc
$ cargo d --open # opens docs in browser
$ cargo t # test
$ cargo r # run
$ cargo rm $CRATE # remove
```

<!-- or that cargo has built-in shortcuts? Reduce your RSI! -->

---
## Cargo.toml: strip

strip your release binaries

```toml
[profile.release]
strip=true
```

<!-- To counter the meme that rust binaries are large, strip your release
binaries, unless you need the debuginfo for something -->

---
## Tooling: cargo config

project-wide: `<my_project>/.cargo/config.toml`

user-wide: `$CARGO_HOME/config.toml`

- Unix: `$HOME/.cargo/config.toml`
- Windows: `%USERPROFILE%\.cargo\config.toml`

<!-- cargo has a project- and user wide configuration file. Here's where to
put it. And now let's see what we can do with it: -->

---
## Tooling: cargo config

More shortcuts!

```toml
[alias]
c = "clippy"
do = "doc --open"
ex = "run --example"
rr = "run --release"
bl = "bless"
s = "semver-checks"
```

<!-- note that clippy will also check all lints that cargo check does, and
then some. bless is mostly used with compiletest, a tool that checks if we
get the right compiler errors. So if you do type shenanigans to ensure your
users' safety, perhaps use that, too. -->

---
## Tooling: cargo config

```toml
[build]
rustflags = [
# compile for the current CPU
  "-C", "target-cpu=native",
# compile into a zero-install relocatable static binary
  "-C", "target-feature=+crt-static"
]
# shared target folder
target = "/home/<user>/.cargo/target"
````

<!-- But wait, there's more! You can have cargo compile to use all your CPU's
bells and SIMD whistles (especially if it has AVX512 capabilities)

Also consider using a shared target folder to reduce duplication of build
artifacts on your disk.
-->

---
## Tooling: cargo config

Configure active lints:

```toml
[lints.rust]
missing_docs = "deny"
unsafe_code = "forbid"

[lints.clippy]
dbg_macro = "warn"
```

<!-- you can also configure lints here, so it won't be needed in your code -->

---
## Tooling: clippy

Clippy lints to try

- `missing_panics_doc`, `missing_errors_doc`, `missing_safety_doc`
- `unnecessary_safety_doc`
- `multiple_crate_versions`
- `non_std_lazy_statics`
- `ref_option`, `ref_option_ref`
- `same_name_method`
- `fn_params_excessive_bools`

---
## Tooling: clippy

`missing_panics_doc`, `missing_errors_doc`, `missing_safety_doc`

```rust
pub unsafe fn unsafe_panicky_result(foo: Foo) -> Result<Bar, Error> {
    match unsafe { frobnicate(&foo) } {
        Foo::Amajig(bar) => Ok(bar),
        Foo::Fighters(_) => panic!("at the concert");
        Foo::FieFoFum => Err(Error::GiantApproaching),
    }
}`
```

---
## Tooling: clippy

`missing_panics_doc`, `missing_errors_doc`, `missing_safety_doc`

```rust
/// # Errors
/// This function returns a `GiantApproaching` error on detecting giant noises
///
/// # Panics
/// The function might panic when called at a Foo Fighters concert
///
/// # Safety
/// Callers must uphold [`frobnicate`]'s invariants'
```

---
## Tooling: clippy

`unnecessary_safety_doc`

```rust
/// # Safety
///
/// This function is actually completely safe`
pub fn actually_safe_fn() { todo!() }
```

---
## Tooling: clippy

`multiple_crate_versions`

- mycrate 0.1.0
  - rand 0.9.0
  - quickcheck 1.0.0
	  - rand 0.8.0

<!-- The lint will report multiple crate versions, in this case `rand`.
The downside is that both versions need to be compiled, and may even
lead to incompatible types later down the road.

With that said, it's often not really an issue.
-->

---
## Tooling: clippy

- `non_std_lazy_statics`

```rust
// old school lazy statics
lazy_static! { static ref FOO: Foo = Foo::new(); }
static BAR: once_cell::sync::Lazy<Foo> = once_cell::sync::Lazy::new(Foo::new);

// now in the standard library
static BAZ: std::sync::LazyLock<Foo> = std::sync::LazyLock::new(Foo::new);
```

---
## Tooling: clippy

- `ref_option`, `ref_option_ref`

```rust
fn foo(opt_bar: &Option<Bar>) { todo!() }
fn bar(foo: &Foo) -> &Option<&Bar> { todo!() }

// use instead
fn foo(opt_bar: Option<&Bar>) { todo!() }
fn bar(foo: &Foo) -> Option<&Bar> { todo!() }
```

---
## Tooling: clippy

- `same_name_method`

```rust
struct I;
impl I {
    fn into_iter(self) -> Iter { Iter }
}
impl IntoIterator for I {
    fn into_iter(self) -> Iter { Iter }
    // ...
}
```

<!-- avoid ambiguity that would later need a turbo fish to resolve -->

---
## Tooling: clippy

- `fn_params_excessive_bools`

```rust
fn frobnicate(is_foo: bool, is_bar: bool) { ... }

// use types to avoid confusion
enum Fooish {
    Foo
    NotFoo
}
```

---
## Tooling: clippy

Configuration

```toml
# for non-library or unstable API projects
avoid-breaking-exported-api = false
# let's allow even less bools
max-fn-params-bools = 2

# allow certain things in tests
# (if you activate the restriction lints)
allow-dbg-in-tests = true
allow-expect-in-tests = true
allow-indexing-slicing-in-tests = true
allow-panic-in-tests = true
allow-unwrap-in-tests = true
allow-print-in-tests = true
allow-useless-vec-in-tests = true
```

---
## Tooling: cargo semver-checks

If you have a library, please use cargo semver-checks before cutting a release.

```sh
$ cargo semver-checks
    Building optional v0.5.0 (current)
       Built [   1.586s] (current)
     Parsing optional v0.5.0 (current)
      Parsed [   0.004s] (current)
    Building optional v0.5.0 (baseline)
       Built [   0.306s] (baseline)
     Parsing optional v0.5.0 (baseline)
      Parsed [   0.003s] (baseline)
    Checking optional v0.5.0 -> v0.5.0 (no change)
     Checked [   0.005s] 148 checks: 148 pass, 0 skip
     Summary no semver update required
    Finished [  10.641s] optional
```

---
## Tooling: Tests

Doctests are fast now (apart from `compile_fail` ones)

---
## Tooling: Tests

Want to `#[test]` stuff in a binary crate? Use a mixed crate!

```toml
[lib]
name = "my_lib"
path = "src/lib.rs"

[[bin]]
name = "my_bin"
path = "src/main.rs"
```

---
## Tooling: Tests

snapshot tests with insta

```rust
#[test]
fn snapshot_test() {
    insta::assert_debug_snapshot!(my_function());
}
```

<!-- insta will use either the debug representation or a serialization in
various formats as "snapshot" that you check once, then reuse. -->

---
## Tooling: Tests

snapshot tests with insta: redactions

```rust
#[test]
fn snapshot_test() {
    insta::assert_json_snapshot!(
        my_function(),
        { ".id" => "[id]" }
    );
}
```

---
## Tooling: Tests

Test your tests with mutation testing

```sh
$ cargo mutants
Found 309 mutants to test
ok       Unmutated baseline in 3.0s build + 2.1s test
 INFO Auto-set test timeout to 20s
MISSED   src/lib.rs:1448:9: replace <impl Deserialize for Optioned<T>>::deserialize -> Result<Optioned<T>,
 D::Error> with Ok(Optioned::from_iter([Default::default()])) in 0.3s build + 2.1s test
MISSED   src/lib.rs:1425:9: replace <impl Hash for Optioned<T>>::hash with () in 0.3s build + 2.1s test
MISSED   src/lib.rs:1202:9: replace <impl OptEq for u64>::opt_eq -> bool with false in 0.3s build + 2.1s test
TIMEOUT  src/lib.rs:972:9: replace <impl From for Option<bool>>::from -> Option<bool> with Some(false) in 0.4s
 build + 20.0s test
MISSED   src/lib.rs:1139:9: replace <impl Noned for isize>::get_none -> isize with 0 in 0.4s build + 2.3s test
MISSED   src/lib.rs:1228:14: replace == with != in <impl OptEq for i64>::opt_eq in 0.3s build + 2.1s test
MISSED   src/lib.rs:1218:9: replace <impl OptEq for i16>::opt_eq -> bool with false in 0.3s build + 2.1s test
MISSED   src/lib.rs:1248:9: replace <impl OptEq for f64>::opt_eq -> bool with true in 0.4s build + 2.1s test
MISSED   src/lib.rs:1239:9: replace <impl OptEq for f32>::opt_eq -> bool with false in 0.4s build + 2.1s test
...
309 mutants tested in 9m 26s: 69 missed, 122 caught, 112 unviable, 6 timeouts
```

---
## Tooling: Tests

If you still use `cargo nextest`, you can go back to `cargo test` now.

It's gotten much faster.

---
## Tooling: Rust-Analyzer

```toml
# need to install the rust-src component with rustup
rust-analyzer.rustc.source = "discover" 
# on auto-import, prefer importing from `prelude`
rust-analyzer.imports.preferPrelude = true
# don't look at references from tests
rust-analyzer.references.excludeTests = true
```

---
## Tooling: `cargo sweep`

The problem:
```sh
$ du -sh target
37.6G
```

---
## Tooling: `cargo sweep`

The solution:
```sh
$ cargo sweep --time 14 # remove build artifacts older than 2 weeks
$ cargo sweep --installed # remove build artifacts from old rustcs
```

Pro Tip: Add a cronjob (for example every Friday on 10 AM):

```crontab
0 10 * * fri sh -c "rustup update && cargo sweep --installed"
```

---
## Tooling: `cargo wizard`

![w: 75%](life/wizard-demo.gif)

---
## Tooling: `cargo-pgo`

Profile Guided Optimization can improve perf with little effort

Listen to Aliaksandr Zaitsau's talk for more info

---
## Tooling: `cargo component`

- Run your code in `wasm32-wasip1` (or later)
- the typical subcommands (`test`, `run`, etc.) work as usual
- can use a target runner:

```toml
[target.wasm32-wasip1]
runner = ["wasmtime", "--dir=."]
```
<!-- Run your code locally under a WASM runtime, the argument is used to
allow accessing the current directory. You can also use different directories
-->

---
## bacon

compiles and runs tests on changes

great to have in a sidebar terminal

---
## Language: Pattern matching

Destructuring tuples, slices and matching integer ranges

```rust
match (foo, bar) {
  (1, [a, b, ..]) => todo!(),
  (2 ..= 4, x) if predicate(x) => frobnicate(x),
  (5..8, _) => todo!(),
  _ => ()
}
```

---
## Language: Pattern matching

Or-combine patterns with pipe, even within other patterns

```rust
if let Some(1 | 23) | None = x { todo!() }

match foo {
  | Foo::Bar
  | Foo::Baz(Baz::Blorp | Baz::Blapp)
  | Foo::Boing(_)
  | Foo::Blammo(..) => todo!(),
  _ => ()
}

matches!(foo, Foo::Bar)
```

<!--
Note that the bindings of those patterns must have the same types
-->

---
## Language: Pattern matching

function signatures are patterns

```rust
fn frobnicate(Bar { baz, blorp }: Bar) {
  let closure = |Blorp(flip, flop)| blorp(flip, flop);
}
```
<!-- and closures are functions, too -->

---
## Language: Pattern matching

`let` and assignments

```rust
let (a, mut b, mut x) = (c, d, z);
let Some((e, f)) = foo else { return; };
(b, x) = (e, f);
```

---
### Language: Annotations

use `#[expect(..)]` instead of `#[allow(..)]`

```rust
#[expect(clippy::collapsible_if)
fn foo(b: bool, c: u8) [
    if b {
        if c < 25 {
            todo!();
        }
    }
}
```

<!--
If you use "allow", the annotation will just stay there even if the code
changes so the lint no longer applies. On the other hand, using expect gives
an error in that case, so you don't forget to remove the annotation.
-->

---
### Language: Annotations

Add `#[must_use]` judiciously

```rust
#[must_use]
fn we_care_for_the_result() -> Foo { todo!() }

#[must_use]
enum MyResult<T> { Ok(T), Err(crate::Error), SomethingElse }

we_care_for_the_result(); // Err: unused_must_use
returns_my_result(); // Err: unused_must_use
```

<!--
If you write a library, help your users avoid mistakes.
You can annotate functions that are only useful for their results with
`must_use` so the `unused_must_use` lint will pick up calls that don't.

You can also annotate types that should always be used when returned from
functions.
-->

---
### Language: Annotations

Traits sometimes need special handling. Tell your users what to do:

```rust
#[diagnostic::on_unimplemented(
    message = "Don't `impl Fooable<{T}>` directly, `#[derive(Bar)]` on `{Self}` instead",
    label = "This is the {Self}"
    note = "additional context"
)]
trait Fooable<T> { .. }
```

---
### Language: Annotations

Sometimes, you want internals to stay out of the compiler's error messages:

```rust
#[diagnostic::do_not_recommend]
impl Fooable for FooInner { .. }
```

<!--
Sometimes you have internal types, so that when a trait implementation is
required, type inference should not recommend them. In that case, add the
annotation to the impl, not the type.
-->

---
## library: `Box::leak`

For `&'static`, once-initialized things that don't need to be `drop`ped

```rust
let config: &'static Configuration = Box::leak(create_config());
main_entry_point(config);
```

---
# That's all, folks!
<!--
Thank you for being such a nice audience.

To live-rebuild:
while ! inotifywait -e modify life.md; do marp life.md; done
-->
