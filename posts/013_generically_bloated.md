+++
title = "Generically Bloated"
slug = "generically-bloated"
draft = false
template = "page.html"
date = 2023-02-05
in_search_index = true
[taxonomies]
categories = ["blog", "rust", "performance"]
tags = ["rust", "size", "performance"]
+++

Rust generics are great, but they can bloat the final code, and have negative
effects on compile times. This post discusses a "trick" to help fight those
effects.

<!-- more -->

There exists a decently common pattern to combat these issues given a few
preconditions are met. This pattern is used heavily in the standard library and
popular crates. 

However, when talking to a colleague recently they were unaware
of this pattern (to be fair I wasn't fully aware of this pattern either until
just a few years ago). This conversation made me wonder if there could be
others who also haven't yet seen this pattern that can pretty big downstream
affects!

# Basics

Let's start by just showing a basic set of generics in Rust, and why some
problems pop up.

When you use generic parameters in Rust, the compiler will actually just
generate a copy of your code for each concrete type. For example:

```rust
fn genric<T>(param: T) {
  // code...
}

fn main() {
  generic(30);    // T = i32
  generic("foo"); // T = &'static str
}
```

Turns into something like:

```rust
fn genric_i32(param: i32) {
  // code...
}

fn genric_str(param: &'static str) {
  // code...
}

fn main() {
  generic(30);
  generic("foo");
}
```

While this produces extremely efficient code at runtime, there are a few
potential downsides:

* If `generic<T>()` is of any substantial size (in the compiled binary), that
  code is duplicated leading to "bloat"
* (Although I haven't gone spelunking into the compiler code to know for sure;
  I believe) all the generated code must also be compiled and optimized
  separately! This is a lot more code for `rustc` and LLVM to churn through
* `generic<T>()` cannot be fully compiled until the compiler knows all the
  concrete types that will be used

# Library Authors Beware

The above concerns are most applicable to library authors, but can affect binary
authors as well when their binary is split into a
binary-consuming-an-internal-library.

Where this shows up is downstream consumers filing issues related to code bloat
(it turned out many concrete types were used for `T`), or slow compile times
due to your library (your library heavily relies on generics that cannot be
compiled "in advance").

# Preconditions

I mentioned earlier that the pattern we're about to discuss isn't a silver
bullet and cannot be used in all circumstances. There is an important
precondition that need to be met:

> **Important**
> The generic code must have a "preferred type" which usually means the generic
> parameter is bounded and that bound represents some kind of conversion

This should become more clear as we go along.

# Immediate Dispatch to the Rescue

The solution is to immediately dispatch to a known type. In hindsight this may
seem somewhat obvious, however if it doesn't that's OK! We're about to see this
in action!

> **Note**
> This example will be extremely contrived in order to demonstrate the effects

## In Action

First, lets define a trait that will be used as a generic bound.

```rust
trait Speak {
  fn speak(&self) -> String;
}
```

Next, we define a generic speaking function that can be used with any type
that implements the `Speak` trait.

```rust
fn generic_speak<T: Speak>(param: &T) {
  println!("It says: {}", param.speak());
}
```

Then we can define two concrete types that implement said trait as example:

```rust
struct Cat;
impl Speak for Cat {
  fn speak(&self) -> String {
    "meow".into()
  }
}

struct Dog;
impl Speak for Dog {
  fn speak(&self) -> String {
    "woof".into()
  }
}
```

Finally, we use that generic function with both a `Cat` and a `Dog`:

> **Note**
> We're creating a binary because it's easier to demonstrate :)

```rust
fn main() {
  let whiskers = Cat;
  let spot = Dog;
  generic_speak(&whiskers);
  generic_speak(&spot);
}
```

If we run it, we get what you probably expect:

```
$ cargo run --quiet
It says: meow
It says: woof
```

## Backing Up

Lets first back up and prove the claims I made at the beginning, that our
generic function actually generates two concrete functions. To see that we can
either [`cargo-llvm-lines`][cargo-llvm-lines] or [`cargo-bloat`][cargo-bloat].
I use both pretty extensively, so just to compare the output of both lets look
at them both for fun!

### `cargo-llvm-lines`

First, we'll use `cargo-llvm-lines` to see the amount of LLVM IR generated:

> **Note**
> Both tools can generate *a lot* of output, so we could be `grep`ing it down to
> size but I'll make some readability edits instead.

```
$ cargo llvm-lines
Lines               Copies            Function name
-----               ------            -------------
[ .. snip .. ]
    80 (4.9%, 50.3%)   2 (4.9%, 17.1%)  generic_speak
     5 (0.3%, 98.3%)   1 (2.4%, 82.9%)  <Cat as Speak>::speak
     5 (0.3%, 98.6%)   1 (2.4%, 85.4%)  <Dog as Speak>::speak
[ .. snip .. ]

```

Notice we do, in fact, have two copies of `blog_demo::generic_speak` (as can be
seen by the `Copies` column) and one implementation function each for the
concrete types (e.g. `<Cat as Speak>::speak` which is the code from where we
did `impl Speak for Cat`).

### `cargo-bloat`

Contrasting the above with `cargo-bloat` which shows the binary size of each
function:

> **Note**
> `cargo-bloat` by default only shows the largest 99 functions, but our example
> is to tiny we tell it show us the largest 999 functions so we can be sure our
> functions show up in the output.

```
$ cargo bloat -n 999
Analyzing target/debug/blog_demo

File  .text     Size     Crate Name
[ .. snip .. ]
0.0%   0.1%     193B blog_demo generic_speak
0.0%   0.1%     193B blog_demo generic_speak
0.0%   0.0%      44B blog_demo <Dog as Speak>::speak
0.0%   0.0%      44B blog_demo <Cat as Speak>::speak
[ .. snip .. ]
```

> **Note**
> As the name implies I tend to prefer `cargo-bloat` when working on bloat
> issues, because it looks at the final compiled binary size as opposed to just
> the LLVM IR with `cargo-llvm-lines`. Although I prefer `cargo-llvm-lines`
> when working on compile times.

Here we see that in the final binary we still have two copies of
`generic_speak`, each 193 bytes (for a total of 386 bytes) as well as the
concrete implementation function.

## Bloat 

Although contrived, one thing I like about this example is by pure lines, the
`generic_speak` function looks like almost no code! But this brings up another
great source of bloat (*especially* when combined with the issue described in
this post!): macros!

Using an LSP expand function in my editor (although similar could be done with
something like [`cargo-expand`][cargo-expand]) we see `generic_speak` actually
expands to something like this:

> **Note**
> You can't run this directly as it uses private rust internals, but it gives a
> good view into the kind things macros expand to

```rust
fn generic_speak<T: Speak>(param: &T) {
{
  {
    std::io::_print(std::fmt::Arguments::new_v1(
      &["It says: "],
      &[std::fmt::ArgumentV1::new(
        &(param.speak()),
        std::fmt::Display::fmt,
      )],
    ));
  }
}
```

## Trying Immediate Dispatch

If we take a step back we see that `Speak::speak` just produces a `String` and
all the code inside `generic_speak` really only needs that `String`. So we can
add a private internal function that accepts a `String` instead of a generic
parameter.

That would mean `generic_speak` now looks like this:

```rust
fn generic_speak<T: Speak>(param: &T) {
  fn generic_speak_string(param: String) {
    println!("It says: {param}");
  }
  generic_speak_string(param.speak());
}
```

> **Note**
> The function could be external as well, it doesn't need to be defined within
> the outer function. Although if it's not used anywhere else it makes sense to
> define it within that scope.

If we re-run `cargo-bloat` we now see:

> **Note**
> I'm going to omit the actual trait implementations (`<Dog as Speak>::speak`)
> for brevity, because those don't change, we still implement those traits with
> actual code!

```
$ cargo bloat -n 999
Analyzing target/debug/blog_demo

File  .text     Size     Crate Name
[ .. snip .. ]
0.0%   0.1%     160B blog_demo generic_speak::generic_speak_string
0.0%   0.0%      37B blog_demo generic_speak
0.0%   0.0%      37B blog_demo generic_speak
[ .. snip .. ]
```

We still have two concrete implementations of `generic_speak` since we still
have a generic function, however notice the actual code inside has gone down
from 193 bytes to just 37 bytes (essentially enough to dispatch the other
function). Now, all our "real code" lives in non-generic (and thus not
duplicated) `generic_speak_string` internal function (160 bytes).

If do the quick math of our duplicate generic functions + the new internal
function we get 234 bytes versus the original 386 bytes!

This is just a contrived example, but imagine a real library with multiple
generics and actually substantial sized functions!

Additionally, that inner function (`generic_speak_string`) is able to be fully
compiled right away since all it's types are fully known.

## A Note on Inlining and Release Builds

You may have noticed if you compiled the examples in release mode, the
functions don't appear at all!

```
$ cargo bloat -n 999 --release | grep speak
$
```

This is because the code is pretty trivial and Rust/LLVM are able to inline all
the code and optimize this away. However, in a real world library that's not
always possible.

## Why not just use a `String` as the parameter instead of the generic?

Because this was a contrived example. 

Also there are times you'll do something almost exactly like this purely for
ergonomics. For example:

```rust
// Accepts anything that can be converted to &str cheaply
fn takes_stringish<S: AsRef<str>>(param: S) { todo!("...") }
```

Sure, we could just take `&str` as the parameter, but some types may be cheaply
yet not *ergonomically* converted to a `&str`.

### More Contrived Example

At the risk of making this post too long, to show another example of one where
it's less ergonomic to ask the user for exactly what we want.

Cases often comes up around type generics, e.g. `impl Iterator<Item =
AsRef<str>>` versus trying force consumers to produce a `Vec<&str>` or similar
when they may have a collection that is not even close to a `Vec<&str>` yet
*does* implement `Iterator<Item = AsRef<str>>`.

Let's do exactly that, but instead of `AsRef<str>` we'll use our `Speak` as a
bound.

If we change our `generic_speak` and `main` functions:

```rust
fn generic_speak<T: Speak>(param: impl Iterator<Item = T>) {
  for item in param {
    println!("It says: {}", item.speak());
  }
}

fn main() {
  use std::collections::HashSet;

  let whiskers = vec![Cat, Cat];
  let mut spots = HashSet::new();
  spots.insert(Dog);
  generic_speak(whiskers.into_iter());
  generic_speak(spots.into_iter());
}
```

Notice we're taking two totally different collection types, a `Vec<Cat>` and
`HashSet<Dog>`.

Re-running our example gives:

```
$ cargo run --quiet
It says: meow
It says: meow
It says: woof
```

And `cargo-bloat` now shows:

```
$ cargo bloat -n 999
Analyzing target/debug/blog_demo

[ .. snip .. ]
0.0%   0.2%     440B  blog_demo generic_speak
0.0%   0.1%     415B  blog_demo generic_speak
[ .. snip .. ]
```

A total of 855 bytes.

Since you already know the drill, we can do the inner function dispatch thing
(with a caveat listed below):

```rust
fn generic_speak<T: Speak>(param: impl Iterator<Item = T>) {
  fn generic_speak_strings(params: Vec<String>) {
    for item in params {
      println!("It says: {item}");
    }
  }
  generic_speak_strings(param.map(|s| s.speak()).collect());
}
```

To which `cargo-bloat` reports:

```
$ cargo bloat -n 999
nalyzing target/debug/blog_demo

[ .. snip .. ]
0.0%   0.1%     405B  blog_demo generic_speak::generic_speak_strings
0.0%   0.0%      80B  blog_demo generic_speak
0.0%   0.0%      69B  blog_demo generic_speak
0.0%   0.0%      65B  blog_demo generic_speak::{{closure}}
0.0%   0.0%      65B  blog_demo generic_speak::{{closure}}
[ .. snip .. ]
```

Even though we have these closures, it's still a total of 684 bytes. Again, in
this contrived example it's not that dramatic, but in the real world it is
often quite dramatic!

#### The Caveat!

I mentioned there is a caveat to the above. We created a whole new
`Vec<String>` to pass to the inner function which is another allocation.
However, perhaps this is a case were an extra allocation like that is
acceptable compared to the code bloat and compile times. You'll have to be the
judge in your specific case.

This is also partially only due to the strange contrived API I used for this
example as well!

[cargo-llvm-lines]: https://crates.io/crates/cargo-llvm-lines
[cargo-bloat]: https://crates.io/crates/cargo-bloat
[cargo-expand]: https://crates.io/crates/cargo-expand
