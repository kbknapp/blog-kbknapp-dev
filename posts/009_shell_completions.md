+++
title = "CLI Shell Completions in Rust"
slug = "shell-completions"
draft = false
template = "page.html"
date = 2021-01-08
in_search_index = true
[taxonomies]
categories = ["programming", "rust", "clap", "blog"]
tags = ["programming", "rust", "cli", "shell"]
+++

Taking a look at adding shell completions to CLI programs using Rust. 

In this post we add shell completions to the XKCD CLI utility [we made earlier](../rust-cli/index.html).
We'll show how easy it is to support multiple shells, and generate the
completions both at compile time and/or run time.

<!-- more -->

**Updates: Updated 2021-10-09 to clap 3.0-beta.4**

# Why Completion Scripts

Perhaps my biggest daily, "Ugh.." moment is working on the command line and
typing `<TAB><TAB>` and getting no shell completion help. Even when I'm using a
tool that I'm extremely familiar with, I still end up using shell completions
(also called *tab* completions) to discover new options or explore the flags further.

{% fk(after=true, emoji="(눈_눈)") %}

Sometimes I simply forgot the exact syntax..."is it `--remove` or
`--delete`...Oh right, on this tool it's `-rm`"

{% end %}

I think we can agree shell completions are useful. So why don't all utilities
provide shell completions? Probably because they're a giant pain to create, they
must be kept in sync with the actual CLI flags/options, and there are a ton of
shells to support!

Imagine typing `foo --<tab><tab>` and seeing `--directory` was a valid option,
only to hit `<return>` and have the CLI tell you, `error: --directory option not
found` or similar. Turns out the CLI updated and is now using `--dir`. Out of
sync completions are terrible.

## `clap` to the rescue!

I've got great news. Adding/supporting shell completions with your Rust based
CLI is very easy! Using the [`clap`](https://crates.io/crates/clap) crate, we
can generate shell completion scripts either at compile time, or provide a
special option to allow users to generate shell completion scripts at run time on
the fly. 

{% fk(after=true, emoji="(¬,‿,¬)") %}

Or both.

{% end %}

Using generated completion scripts we get the following benefits:

- Our completion scripts will always be in sync with our actual CLI options
- We can support Bash, Zsh, Fish, PowerShell, and Elvish using about five lines
  of code
- Using both compile time and run time gives our users options for how/where to
  use these scripts

## Whats so hard?

Before I said that creating shell completion scripts was hard, but how hard? A
popular command line tool [ripgrep](https://github.com/BurntSushi/ripgrep) which
has a moderately large CLI space (not huge, but not trivial either) has a Bash
completion script that is 213 lines lone (as of v12.1.1)! That's *just* Bash.
Other shells have similarly sized scripts. `rustup` which has a fairly large CLI
space has a 1,110 line Bash script.

Enough preamble, get to the code!

# Overview 

We'll be updating our [`grab-xkcd` program from a previous
post](../rust-cli/index.html) with the new code, but in order to clearly
demonstrate the different methods and not conflict with the original article I
will use two new branches
[`completions-ct`](https://github.com/kbknapp/grab-xkcd/tree/completions-ct) for
compile time completions, and
[`completions-rt`](https://github.com/kbknapp/grab-xkcd/tree/completions-rt) for
run time completions. If the reader is feeling froggy, you can combine the two!

Let's start with compile time.

# Compile Time Completions

I tend prefer compile time completions because those completions scripts can be
checked into version control, or even tweaked further from the auto-generated
script. Originally, the auto-generated scripts were meant to be a starting
point, but they turned out to work so well that almost no one takes the time to
actually tweak them further.

The downsides to the compile time method are that it requires two new "build
dependencies" (meaning compile time), and can slow down compile times.
Generally, this isn't an issue in practice as generating completion scripts is
crazy fast. But, if you're very cautious with build times, or are trying to
avoid compile time dependencies for some reason, the run time method may be
better.

Also, these scripts will still need to be consumed/installed by the user
somehow. Normally this is done in a packaging format (deb, rpm, msi, exe, dmg,
etc.). But if the CLI tool you're building does not have any packaging, the user
will need to install these scripts manually based on the shell they're using.

## Build Scripts

The process of telling `clap` to generate a completion script at compile time
happens in a [`cargo` build
script](https://doc.rust-lang.org/cargo/reference/build-scripts.html).

The way completion scripts are generated changed between `clap` v2 and the
upcoming v3. In v3 (which we used in the previous article) moved the completion
code to a separate crate to avoid code bloat. Since we're still using v3 we need
to add that extra crate.

## Build Dependencies

We also need to tell `cargo` that `clap_generate` (and also `clap`) is a "build
dependency" (meaning required at compile time). `clap` proper will also remain a
regular dependency.

```
$ cargo add clap clap_generate --allow-prerelease --build
    Updating 'https://github.com/rust-lang/crates.io-index' index
      Adding clap_generate v3.0.0-beta.4 to build-dependencies
      Adding clap v3.0.0-beta.4 to build-dependencies
```

If we were to look at our `Cargo.toml` we'd now see two `dependencies` tables,
with the new one being `[build-dependencies]` listing the two crates above.
[`[dependencies]`] is unchanged, and still includes just normal `clap` since
that's still required to parse run time arguments.

Next, since `clap_generate` needs a
[`clap::App`](https://docs.rs/clap/3.0.0-beta.4/clap/struct.App.html) instance
in order to be able to walk our CLI and generate everything, we'll need to use
the `into_app()` method that was `#[derive]`d on our `Args` struct in the
previous article.

## Stubbing

Let's first, stub out the `build.rs` file in our project root directory:

```rust
// in build.rs

use clap::{IntoApp, Clap};

include!("src/cli.rs");

fn main() {
    let mut app = Args::into_app();

    todo!("generate the completion scripts!");
}
```

We built `grab-xkcd` as a binary, and not a library. So we need some way to
include our `Args` struct in this build script. We could have re-factored out the
core logic into a library, and then consumed that library in our `main.rs`
binary...but that's overkill for this example. So we simply `include!()` the
source file directly, which is like a copy/paste.

## Generate Bash

Now that we have an `App` struct, we can pass that to `clap_generate` and see
the magic!

Let's add a Bash completion script:

```rust
// in build.rs
use clap_generate::{generators::Bash, generate_to};
use clap::{IntoApp, Clap};

include!("src/cli.rs");

fn main() {
    let mut app = Args::into_app();
    app.set_bin_name("grab-xkcd");

    let outdir = env!("CARGO_MANIFEST_DIR");
    generate_to::<Bash, _, _>(&mut app, "grab-xkcd", outdir);
}
```

There is a paper-cut that we have to call `set_bin_name()` due to some
complexities about how the derive magic works, and how it interacts with what
`generate_to` expects. The short version is `generate_to` is written to handle a
large swath of CLI types, some of which may have different binary names, than
program names (i.e. ripgrep-`rg`). Completion scripts are normally based off the
binary name (which is how shells find which scripts to call). The `derive` magic
in `clap` has no way of knowing the binary name, as our binary is certainly not
called `Args`. We could have set the binary name as part of our `Args`
declaration, which is probably the right way to do it in the long run, but for
this demo it's overkill. So we just set it manually on the instance of the `App`
struct and move on.

Walking through the rest of the above:

- We import a single
  [`Generator`](https://docs.rs/clap_generate/3.0.0-beta.4/clap_generate/trait.Generator.html),
  `Bash` which will walk our `App` struct and generate all options.
- We get the value of `cargo`'s environmental variable `CARGO_MANIFEST_DIR`
  which is our source root and where we will be saving the script to.
- In `generate_to` we must provide some generic arguments. The first is the most
  important, and is the `Generator` that will be used to make the completion
  script. The other two are related to the other two final function arguments
  and just convenience conversion traits. Rust can infer them, hence the `_`.
- The arguments are:
  - The `App` struct to walk over
  - The binary name to use throughout the script (which may differ from the real
    binary name in some special circumstances)
  - A path directory to save the generated file to

That's it! If we look in our project root, you should see a `grab-xkcd.bash`!
Even our super small CLI has generated a 65 line script:

```
$ wc -l grab-xkcd.bash
65 grab-xkcd.bash
```

Looking at the head you can see some of the boilerplate already:

```
$ head grab-xkcd.bash
_grab-xkcd() {
    local i cur prev opts cmds
    COMPREPLY=()
    cur="${COMP_WORDS[COMP_CWORD]}"
    prev="${COMP_WORDS[COMP_CWORD-1]}"
    cmd=""
    opts=""

    for i in ${COMP_WORDS[@]}
    do

...
```

## Testing

So let's try it out! We can test it by just `source`ing it, if you're using
`bash` as your shell:

![](../imgs/shell_completions_ct.gif)

It works!

## Project Cleanup

If we wanted to add additional shells it's just more calls to `generate_to`.
We'll actually, place all the scripts in a new `completions/` dir in our project
root, so as not to get unwieldy.

```
$ mkdir completions
```

And we update our build script:

```rust
// in build.rs
use clap_generate::{generators::*, generate_to};

include!("src/cli.rs");

fn main() {
    let mut app = Args::into_app();
    app.set_bin_name("grab-xkcd");

    let outdir = std::path::Path::new(env!("CARGO_MANIFEST_DIR")).join("completions/");
    generate_to::<Bash, _, _>(&mut app, "grab-xkcd", &outdir);
    generate_to::<Fish, _, _>(&mut app, "grab-xkcd", &outdir);
    generate_to::<Zsh, _, _>(&mut app, "grab-xkcd", &outdir);
    generate_to::<PowerShell, _, _>(&mut app, "grab-xkcd", &outdir);
    generate_to::<Elvish, _, _>(&mut app, "grab-xkcd", &outdir);
}
```

We had to create a new `Path` and `join` it to our new directory, but that's not
too bad. We also had to take a reference to `outdir` in each call to
`generate_to` since we're not passing a `PathBuf` and not a `&'static str` any
longer. But hey, notice we didn't need to do anything else special and
`generate_to` could handle those two totally different types with ease (thanks
to the second `_` generic parameter which is actually a `T: Into<OsString>` for
those that care).

If we look in our `completions/` dir, yup, we see a bunch of new completion
scripts!

# Run time 

Ok, so compile time was pretty easy. Maybe a little fuss around adding
build-dependencies and making a build script, but overall not too bad.

But let's say you're just making a small CLI, and don't want to have to worry
about packaging or installing completions scripts with your program, etc.
Luckily, we can have our binary just spit out completion scripts on demand to
stdout! Then the user can source the output if they want, or redirect it to a
file if they want to "install" it.

A traditional way of doing this may have included inserting the completion
scripts themselves into your binary file statically...which would probably be
terrible (even though they compress well). This still runs the risk of getting
out of sync with your CLI, etc.

Let's use what we learned with the compile time completion scripts, but just
do it at run time instead.

## Extend the CLI

In order to do this, we'll actually need to extend our CLI a little bit to
include a new option/flag for generating these completions. I normally advocate
for using subcommands instead of options/flags, but since our CLI is so small
and we don't have any other subcommands we'll forgo that idea. Instead we'll add
a `-c/--completions` option which accepts any of the supported shells and spits
out the completion script to stdout.

```rust
/// A utility to grab XKCD comics
#[derive(Clap)]
pub struct Args {
    // .. Same as before

    /// Generate a SHELL completion script and print to stdout
    #[clap(long, short, arg_enum, value_name="SHELL")]
    pub completions: Option<Shell>,
}

#[derive(ArgEnum, Copy, Clone)]
pub enum Shell {
    Bash,
    Zsh,
    Fish,
    PowerShell,
    Elvish
}
```

This should look pretty familiar to you from the previous article, however let's
step through it really quickly:

- With `short, long` we tell `clap` to generate a short `-c` and long
  `--completions` automatically from the name of the field
- `arg_enum` tells `clap` the enum we are using should be used as the only
  allowed value variants
- `value_name="SHELL"` tells `clap` to replace the default placeholder in the
  `--help` message from `--completions <completions>` which is just derived
  from the field name, to `--completions <SHELL>`. It's not required, but I
  think it helps readability.
- We create our enum with the variants we wish to support.

## Using `--completions`

With that out of the way we can now look at using this new argument. In our
`main()` function we will check if that argument was used, and if so generate
the script and print to stdout and exit.

Since we'll need to know which shell to generate our completions for, we can
push that logic to the enum itself and keep our `main()` function clean.

```rust
fn main() -> Result<()> {
    let args = cli::Args::parse();
    
    if let Some(shell) = args.completions {
        shell.generate();
        std::process::exit(0);
    }

    let client = client::XkcdClient::new(args);
    client.run()
}
```

This could be condensed into a single `args.completions.map(..)`, but for this
demo we'll leave it as is since the function is so short and concise anyways. I
also like that the `std::process::exit` function is visible so we know this
could end execution of our program just from looking at `main`. If we were to
reduce this to a `map` we'd probably also rename the enum's `generate` method to
something like `generate_and_exit` to make it more clear as well. But I digress.

## `clap_generate::generate`

Instead of the `clap_generate::generate_to` that was used in compile time, we'll
be using `clap_generate::generate` inside our `Shell::generate` method. First
let's add `clap_generate` to our normal run time dependencies:

```
$ cargo add clap_generate --allow-prerelease
    Updating 'https://github.com/rust-lang/crates.io-index' index
      Adding clap_generate v3.0.0-beta.4 to dependencies
```

Now we can actually implement `Shell::generate`:

```rust
// in src/cli.rs
use std::io::stdout;

use clap::{Clap, IntoApp, ArgEnum};
use clap_generate::{generators::*, generate_to};

impl Shell {
    fn generate(&self) {
        let mut app = Args::into_app();
        let mut fd = std::io::stdout();
        match self {
            Shell::Bash => generate::<Bash, _>(&mut app, "grab-xkcd", &mut fd),
            Shell::Zsh => generate::<Zsh, _>(&mut app, "grab-xkcd", &mut fd),
            Shell::Fish => generate::<Fish, _>(&mut app, "grab-xkcd", &mut fd),
            Shell::PowerShell => generate::<PowerShell, _>(&mut app, "grab-xkcd", &mut fd),
            Shell::Elvish => generate::<Elvish, _>(&mut app, "grab-xkcd", &mut fd),
        }
    }
}
```

A mouthful of a `match` but on closer inspection isn't too bad.

- We create the `clap::App` struct from our CLI just like in the compile time
  version
- We get a reference to the stdout buffer which implements the `std::io::Write`
  trait that `clap_generate::generate` is expecting.
- We match on our shell type, and then pass on to the real `generate` function

## Testing them out

With that all wired up, it's time for a test!

```
$ grab-xkcd --completions bash | head
_grab-xkcd() {
    local i cur prev opts cmds
    COMPREPLY=()
    cur="${COMP_WORDS[COMP_CWORD]}"
    prev="${COMP_WORDS[COMP_CWORD-1]}"
    cmd=""
    opts=""

    for i in ${COMP_WORDS[@]}
    do
```

Looks the same as before! I bet we can `source`/`eval` it.

![](../imgs/shell_completions_rt.gif)

Yay!

# Wrap Up

In this article we've seen how we can add shell completion scripts for various
shells with relative ease. We've seen both completion scripts generated at
compile time, and run time.

It's totally possible to implement both strategies as well.

In all honesty, one could mock existing CLIs *not* written in Rust, and simply
generate completion scripts for them using this method.

Again, the complete code from this article can be found on the two branches: 

- Compile time: [`completions-ct`](https://github.com/kbknapp/grab-xkcd/tree/completions-ct) 
- Run time: [`completions-rt`](https://github.com/kbknapp/grab-xkcd/tree/completions-rt)

Discussed this post on [DEV.to](https://dev.to/kbknapp/cli-shell-completions-in-rust-37g1)!
