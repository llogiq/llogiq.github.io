---
marp: true
author: Andre 'llogiq' Bogus
title: Easy Mode Rust
description: Don't Learn Rust, Just Be Productive.
class: invert
---
<style>
@font-face {
	font-family:"Formula Condensed";
	src: url(PPFormula-CondensedBold.woff2);
}
h1,h2 {
	font-family:"Formula Condensed",sans-serif;
	color: #fdd92a;
}
h1 { font-size: 5em; }
h2 { font-size: 3em; padding-bottom:0px; }
pre { font-size: 1.2em; }
:root {
	--color-fg-default: #fff;
	background-image: url(rust-background.png);
	font-size:2.3em;
}
</style>
# Easy Mode Rust

Andre 'llogiq' Bogus

<!--
Come gather Rustaceans wherever you roam
and admit that our numbers have steadily grown.
The community's awesomeness ain't set in stone,
so if that to you is worth saving
then you better start teamin' up instead of toilin' alone
for the times, they are a-changin'.

Come bloggers and writers who tutorize with your pen
and teach those new folks, the chance won't come again!
Where there once was one newbie, there soon will be ten
and your knowledge is what they are cravin'.
Know that what you share with them is what you will gain
for the times, they are a-changin'.

Researchers and coders, please heed the call,
Without your efforts Rust would be nothin' at all
and unsafety would rise where it now meets its fall.
May C++ proponents be ravin'.
What divides them from us is but a rustup install
for the times, they are a-changin'.

Fellow moderators throughout the land,
don't you dare censor what is not meant to offend
otherwise far too soon helpful people be banned
and what's left will be angry folks ragin'.
Our first order of business is to help understand
that the times, they are a-changin'.

The line it is drawn, the type it is cast
What debug runs slow, release will run fast
as the present now will later be past
and our values be rapidly fadin'
unless we find new people who can make them last
for the times, they are a-changin'.
-->

---
## Ground Rules

* We want a *small* language subset to learn
* We may lose performance 🐢
* We may end up with more lines of code 📜
* We don't strive for good looks, 💃
  just get the job done 💪🏾

<!--
First up, let me welcome all of you who want
to avoid learning Rust!

Caution: This will counter some of the things
you would want to learn when trying to write
idiomatic or performant code.

The goal here is to make you productive without
learning the majority of Rust concepts.
-->

---
## The Sorrow Checker

```rust
for item in items.iter() {
  if predicate(item) {
    items.push(modify(item));
  }
}
```

 <!-- Oops, let me fix this real quick -->

---
## The Borrow Checker

```rust
for item in items.iter() {
  if predicate(item) {
    items.push(modify(item));
  }
}
```

<!--
Here here we have an example that you might run into,
trying to extend a collection with a filtered and
modified version of itself.

Let's try to compile it.
-->

---
## The Borrow Checker

```
  Compiling unfortunate v0.0.1
error[E0502]: cannot borrow `items` as mutable because it is also borrowed as immutable
  --> src/main.rs:16:13
   |
13 |     for item in items.iter() {
   |                 ------------
   |                 |
   |                 immutable borrow occurs here
   |                 immutable borrow later used here
...
16 |             items.push(new_item);
   |             ^^^^^^^^^^^^^^^^^^^^ mutable borrow occurs here

For more information about this error, try `rustc --explain E0502`.
```

---
## The Borrow Checker

```rust
//               vvvvvvvv
for item in items.clone().iter() {
  if predicate(item) {
    items.push(modify(item));
  }
}
```

---
## The Borrow Checker

```rust
for i in 0..items.len() {
  if predicate(&items[i]) {
    let new_item = modify(&items[i]);
    items.push(new_item);
  }
}
```

<!--
`if`, unlike `if let`, no longer borrows the
condition once the then-path is taken.

I would not suggest you use this, because the
change is more complex and has a higher risk
of borrow checking still ruining your day.
-->

---
## Macros

* *Calling* predefined macros is OK
  * as long as you roughly understand what they do
  * For example, `println!("Hi there");`
* Avoid writing your own macros

---
## Macros

```rust
// don't write a macro
macro_rules! make_foo {
  { $($a:ident),* } => { $(let $a = { "foo" };)* };
}
make_foo!(a, b);
// copy & paste instead
let a = { "foo" };
let b = { "foo" };
```

<!--
copy & paste is often OK

Yes, this is a contrived example, for brevity.
But even if you add a few tens of lines of code,
you might manage just fine.
-->

---
## Macros

Write code that writes code

```rust
fn format_code(names: &[&str]) -> String {
  let mut result = String::new();
  for name in names {
    result += format!("\nlet {name} = \"foo\";");
  }
  result
}
```

<!--
If you'd have to copy and edit more than a few
tens of lines of code, write code to generate
that instead.

And yes, I kept the contrived example. 
-->

---
## Macros

Need to keep it updated? Write a test

```rust
#[test]
fn update_code() {
  let (prefix, actual, suffix) = read_code();
  let expected = format_code(&["a", "b"]);
  if expected == actual { return; }
  write_code(prefix, expected, suffix);
  panic!("updated generated code, please commit");
}
```
(Thanks to Alexey Kladov (@matklad) for this technique)

<!--
The code here is simplified, but shows the idea:
Have some markers in the code to be able to split it
into prefix, current generated code and suffix
read and generate the piece of code and then compare
write_code will actually change the file on disk

https://matklad.github.io/2022/03/26/self-modifying-code.html
-->

---

## Traits

* Avoid if possible
* `#[derive(..)]` where possible
* Bag of if-clauses
* Some frameworks force impl ☹

<!--
You don't need to learn how to define and implement
traits in most cases. Otherwise, look if you can use
a `#[derive]` to implement a trait for you. In cases
you need to distinguish various cases, use `if` or
`match` clauses instead. Finally, if you really need
to `impl SomeTrait for MyType`, let the IDE help you. 
-->

---

## Generics

```rust
struct Foo<A, B> { .. }
fn foo<A, B>(a: A, b: B) { .. }
```

* Build a plain `FooAB` / `foo_a_b` per A-B combination instead
* Many As and Bs? Consider code generation

---
## Lifetimes

```rust
// don't:
struct Borrowed<'a>(&'a u32);
fn borrowing<'a, 'b>(a: &'a str, b: &'b str) -> &'a str { .. }
// instead, do:
struct Arced(Arc<u32>);
fn cloned(a: String, b: String) -> String { .. }
fn arced(a: Arc<String>, b: Arc<String>) -> String { .. }
```
<!--
Avoid borrows leading to unelided lifetimes
`.clone()` and `Arc` stuff aggressively instead
-->

---
## Unsafe

* Don't. 'nuff said.
<!--
Unless you do embedded development, you won't need
any unsafe. You can in 90% of cases get very acceptable
performance with safe code.
-->

---
## SIMD

Don't. 'nuff said.
<!-- As stated in the ground rules, we're not after perf here. -->

---
## Higher kinded types

Don't. 'nuff said.
<!-- No Generics? Then you also won't need HKTs -->

---
## Async

* Avoid where possible.
* Some crates require async ☹
* prepend `async` before your function signature
* `.await` every function call
  * then remove the `.await` if the compiler complains
* Avoid blocking things like e.g. `Mutex`

<!--
Async Rust is quite powerful, but we're after simple,
so avoid. If however you cannot, write as if you were
sync, append `.await` to all fn() calls, remove if
compiler complains.

Also avoid using blocking types like `Mutex`, `RwLock`,
... if you really need to, use your async runtime's types.
-->
---
## Modules & Imports

### Two extreme cases:
1. 1 100k lines file
2. 10k 10-line files, deeply nested

We want to avoid both

---
## Modules & Imports

Problem: Modules conflate code organisation with path hierarchy.

Solution: Let's split those.

---
## Modules & Imports

```rust
// lib.rs
pub fn a() {}
pub fn b() {}
```

---
## Modules & Imports: Split Code

```rust
// lib.rs
mod b;
pub use b::b; // forward visibility
pub fn a();
```
```rust
// b.rs
pub fn b() {}
```

<!--
By always coupling a module with an import,
we keep the visible path structure the same.

Note that we don't make `b` public.
-->

---
## Modules & Imports: Submodules

```rust
// lib.rs
pub mod b; // add `pub` here
//pub use b::b; // deleted
pub fn a();
```
```rust
// b.rs
pub fn b() {}
```

<!--
Now we can make a module public and remove the
`pub use` to move the code into the module path.

Callers will need to call `b::b()` instead of `b()`.
-->

---
## Syntax

* Mostly keep to `if`, `for` & `while` (`let`)
* `match` is a bit harder to learn, but crazy powerful
	* avoid pattern guards 
	`match foo { x if x > 3 => { .. }, _ => {} }`
  * avoid nested patterns (`if let Ok(Some(foo))) = ..`)

---
## Data structures

Don't implement your own

<!--
Often people decry Rust because it's hard to
build data structures in it. Don't fall for that,
you can do a lot of stuff without building your
own data structure.
-->

---
## Data structures

1. start
2. more
3. end

```rust
vec!["start", "more", "end"]
```

<!--
Yes, Rust has a number of cool data structures.
But we don't want to learn all of those. So let's
keep it simple. If you have a list of things, use
`Vec`, even if your list is at most one element.
-->

---
## Data structures

| **KEY**   | **VALUE**                  |
|-----------|----------------------------|
| recursion | please look at "recursion" |

```rust
HashMap::from([
  ("recursion", "please look at recursion"),
])
```

<!--
if you need to lookup things, use `HashMap`.
--->

---
## Custom iterators

* Don't. Just `.collect()` your things
* If you *really* need to, `Box` your iterator:

  ```rust
  let iter = foo.filter(|e| ..).map(|e| ..);
  Box::new(iter) as Box<dyn Iterator<Item = _>>
  ```

---
# That's all, folks!

* You no longer need to learn Rust
* But feel free to do that anyway

---
# Questions?