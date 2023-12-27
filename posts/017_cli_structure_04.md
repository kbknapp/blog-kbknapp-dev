+++
title = "CLI Structure in Rust - Part 4"
slug = "cli-structure-04"
draft = false
template = "page.html"
date = 2023-12-26
in_search_index = true
[taxonomies]
categories = ["programming", "rust", "blog"]
tags = ["rust", "cli", "clap"]
+++

In Part 4 of this series we go back to the beginning and look at Derive based
CLIs, and how we can structure our program with the tradeoffs that a Derive
based CLI brings.

<!-- more -->

## Series Contents

* [Part 1 - Contexts][part_i]
* [Part 2 - Builder Based CLIs][part_ii]
* [Part 3 - Using Traits in Builder Based CLIs][part_iii]
* Part 4 - Derive Based CLIs (you are here)
* [Part 5 - Traits with Derive Based CLIs][part_v]

## Previously On...

Up until this point we'd been using `clap`'s Builder approach to making a CLI
and then defining a trait to help with some of the strict enforcement of how we
want commands to run.

Builder CLIs are great in that they're the most flexible and fastest to
compile, however they're more verbose to create and use at runtime by a
significant margin.

This can make them more painful to maintain and use throughout a program and
can even introduce subtle bugs when converting from the `clap::ArgMatches` to
your `Ctx` structs.

Now we go back to the beginning and start anew using `clap`'s Derive mode.

## Going Back in Time

First, let's check out the main (empty) branch of our repository and create a
new `derive` branch; we'll then add the dependencies we need:

```text
$ git switch main
$ git swwitch -c derive
$ cargo add anyhow
$ cargo add clap -F derive
```

Notice we have to activate `clap`'s `derive` feature to enable the derive
macros. This is because they pull in additional dependencies such as `syn` and
`quote` and will increase the compile time to some extent.

Like before, here's a quick code dump to get us started:

> **NOTE**
> Just like in our Builder example, we're enot attempting to show all the cool
> things you can do with `clap`.

```rust
// src/main.rs
mod cli;

use clap::Parser;

use crate::cli::Bustup;

fn main() -> anyhow::Result<()> {
    let args = Bustup::parse();

    todo!("Run the program!");

    Ok(())
}
```

And now the actual CLI:

> **NOTE**
> I like to break up the commands into individual files, however for a small
> toy example like this that doesn't normally make sense and adds more noise
> than necessary.
> As such I will not be showing files that just contain module definintions or
> re-exports. Likewise, you're also free to keep everything in a single file if
> you're following along and wish to do so.

```rust
// src/cli.rs
mod cmds;

use clap::{Parser, Subcommand};

use crate::cli::cmds::*;

/// not rustup
#[derive(Parser)]
pub struct Bustup {
    #[command(subcommand)]
    pub cmd: BustupCmd,
}

#[derive(Clone, Subcommand)]
pub enum BustupCmd {
    Update(update::BustupUpdate),
    Target(target::BustupTarget),
}
```

```rust
// src/cli/cmds/update.rs
use clap::Args;

/// update toolchains
#[derive(Clone, Args)]
pub struct BustupUpdate {
    /// toolchain to update
    #[arg(default_value = "default")]
    pub toolchain: String,

    /// forcibly update toolchain
    #[arg(short, long)]
    pub force: bool,
}
```

```rust
// src/cli/cmds/target.rs
mod add;
mod list;
mod remove;

use clap::{Args, Subcommand};

/// manage targets
#[derive(Clone, Args)]
pub struct BustupTarget {
    /// toolchain to update
    #[arg(short, long, default_value = "default")]
    pub toolchain: String,

    #[command(subcommand)]
    pub cmd: BustupTargetCmd,
}

#[derive(Clone, Subcommand)]
pub enum BustupTargetCmd {
    Add(add::BustupTargetAdd),
    List(list::BustupTargetList),
    Remove(remove::BustupTargetRemove),
}
```

```rust
// src/cli/cmds/target/add.rs
use clap::Args;

/// list targets
#[derive(Clone, Args)]
pub struct BustupTargetAdd {
    /// target to add
    #[arg(default_value = "default")]
    pub target: String,
}
```

```rust
// src/cli/cmds/target/list.rs
use clap::Args;

/// list targets
#[derive(Clone, Args)]
pub struct BustupTargetList {
    /// only list installed targets
    #[arg(short, long)]
    pub installed: bool,
}
```

```rust
// src/cli/cmds/target/remove.rs
use clap::Args;

/// remove a target
#[derive(Clone, Args)]
pub struct BustupTargetRemove {
    /// target to remove
    #[arg(default_value = "default")]
    pub target: String,
}
```

We can see that the CLI build properly by passing the `--help` flag to the
various commands:

```text
$ cargo run -q -- --help
Not rustup

Usage: bustup [COMMAND]

Commands:
  update  update toolchains
  target  manage targets
  help    Print this message or the help of the given subcommand(s)

Options:
  -h, --help  Print help
  
$ cargo run -q -- update --help
update toolchains

Usage: bustup update [OPTIONS] [toolchain]

Arguments:
  [toolchain]  toolchain to update

Options:
  -f, --force    Forcibly update
  -h, --help     Print help
  
$ cargo run -q -- target --help
manage targets

Usage: bustup target [OPTIONS] [COMMAND]

Commands:
  add     add a target
  list    list targets
  remove  remove a target
  help    Print this message or the help of the given subcommand(s)

Options:
  -t, --toolchain <toolchain>  toolchain to use [default: default]
  -h, --help                   Print help
  
$ cargo run -q -- target list --help
list targets

Usage: bustup target list [OPTIONS]

Options:
  -i, --installed              Only list installed targets
  -t, --toolchain <toolchain>  toolchain to use [default: default]
  -h, --help                   Print help
```

However, if we try to run it, we get a panic due to our `todo!()`:

```text
$ cargo run
thread 'main' panicked at src/main.rs:6:5:
not yet implemented: implement Bustup!
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

Let's commit this as our starting point.

```text
$ git commit -am "starting point"
```

## Running the Program

So we have the basic CLI structure, and unlike the Builder method we already
kind of have the structure we want at least file system/module wise.

It's also so much easier to just `match` our way into the code path we want
since we have well defined enums that *already appear* to have their own
self-contained context.

> **WARNING**
> Don't be fooled into believing the CLI structs are context structs!

### Naive Matching and no `Ctx`

The naive method is to match on a particular subcommand, and dispatch to some
`run`-like function that takes the CLI struct by reference as context. This is
a common approach, but there are downsides. Let's implement this method for a
single command `bustup update` just so we can contrast it later.

```rust
// src/cli/cmds/update.rs
use anyhow::Result;

impl BustupUpdate {
    pub fn run(&self) -> Result<()> {
        println!("updating toolchain...{}", self.toolchain);
        Ok(())
    }
}
```

And our `main.rs`:

```rust
// src/main.rs

fn main() -> anyhow::Result<()> {
    let args = Bustup::parse();

    match args.cmd {
        cli::BustupCmd::Update(update) => update.run(),
        _ => todo!("implement other subcommands"),
    }
}
```

We can see that it works by running the update command:

```text
$ cargo run -- update
updating toolchain...default

$ cargo run -- update footoolchain
updating toolchain...footoolchain
```

This is already way less verbose than the Builder method, and we no longer have
to worry about "stringly-typed" bugs.

> **NOTE**
> See the [Appendix A][appendix_a] to this post about how `clap` actually
> allows removing a few more layers of indirection that we're using in this
> example.

For extremely simple CLIs this can work. However, even simple CLIs can suffer
by forgoing proper context structs.

## A second talk about `Ctx`

It's so tempting to just use our CLI struct as the passed in context like
we did above. And for a simple CLI, it'd probably be fine. But we're pretending
to build a large and complex CLI.

The CLI structs we defined are *only* convenient CLI structs. **They are not
context!**

Just like how we mentioned previously a `--color` and `--no-color` flag, in a
CLI struct that may look something like:

```rust
#[derive(Parser)]
struct Args {
    #[arg(long, default_value = "auto", value_enum)]
    color: ColorChoice,

    #[arg(long)]
    no_color: bool,
}

#[derive(ValueEnum)]
enum ColorChoice {
    Auto,
    Always,
    Never,
}
```

Not only will *all* code that wants to determine if it should color something
deconflict both `Args::color` and `Args::no_color`, but remember there are also
environment variables and configuration files to handle!

Since we already went over a run-of-the-mill context struct being threaded
through the program we will omit that exercise in the Derive based method
because there is zero different between it and the Builder version.

This get interesting though when we use a context struct and try to use a trait
to define some structure.

## Next Time

In the [next post][part_v] we'll see how to modify our trait that we used in
the Builder method for our current Derive method and discuss some more
tradeoffs.

## Appenix A: Removing More Layers

If we trim our example down to just the absolute essentials for a single
`bustup target add` command, it would currently look like this (moving
everything into a single file for clarity):

```
/// not rustup
#[derive(Parser)]
pub struct Bustup {
    #[command(subcommand)]
    pub cmd: BustupCmd,
}

#[derive(Clone, Subcommand)]
pub enum BustupCmd {
    Target(target::BustupTarget),
}

/// manage targets
#[derive(Clone, Args)]
pub struct BustupTarget {
    /// toolchain to update
    #[arg(short, long, default_value = "default")]
    pub toolchain: String,

    #[command(subcommand)]
    pub cmd: BustupTargetCmd,
}

#[derive(Clone, Subcommand)]
pub enum BustupTargetCmd {
    Add(add::BustupTargetAdd),
}

/// list targets
#[derive(Clone, Args)]
pub struct BustupTargetAdd {
    /// target to add
    #[arg(default_value = "default")]
    pub target: String,
}
```

Because `bustup` itself has no arguments or state, we can actually implement
`clap::Parser` directly on an enum, and then use struct-variants for our enum
which condenses it down even further. This would make the example:

```
/// not rustup
#[derive(Parser)]
pub enum Bustup {
    #[command(subcommand)]
    Target {
        /// toolchain to update
        #[arg(short, long, default_value = "default")]
        pub toolchain: String,

        /// manage targets
        #[command(subcommand)]
        pub cmd: BustupTargetCmd,
    },
}

#[derive(Subcommand)]
pub enum BustupTargetCmd {
    /// add a target
    Add {
        /// target to add
        #[arg(default_value = "default")]
        pub target: String,
    }
}
```

Even though the latter is far less verbose, I find that it gets cluttered when
the CLI is of any reasonable size. But this is purely subjective, and other find
the condensed version easier to grok.

Even if your top level command (i.e. `bustup` itself in our example) has actual
arguments or state, you're still able to use enum struct-variants to remove a
layer of indirection if you wish.

If you're interested in this style of de-duplication I'd also suggest looking
at the `clap` documentation for [`flatten`][flatten] and more generally using
the [`Args`][args_trait] trait to define structs of common arguments for
multiple command structs.


[appendix_a]: https://kbknapp.dev/cli-structure-04/#appendix-a-removing-more-layers
[part_i]: https://kbknapp.dev/cli-structure-01/
[part_ii]: https://kbknapp.dev/cli-structure-02/
[part_iii]: https://kbknapp.dev/cli-structure-03/
[part_v]: https://kbknapp.dev/cli-structure-05/
[flatten]: https://docs.rs/clap/latest/clap/_derive/index.html#flattening-hand-implemented-args-into-a-derived-application
[args_trait]: https://docs.rs/clap/latest/clap/trait.Args.html
