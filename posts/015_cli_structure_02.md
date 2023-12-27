+++
title = "CLI Structure in Rust - Part 2"
slug = "cli-structure-02"
draft = false
template = "page.html"
date = 2023-12-20
in_search_index = true
[taxonomies]
categories = ["programming", "rust", "blog"]
tags = ["rust", "cli", "clap"]
+++

In Part 2 we dive into some actual code to see what it would look like to use
these patterns for CLIs that utilize the `clap` Builder method.

<!-- more -->

## Series Contents

* [Part 1 - Contexts][part_i]
* Part 2 - Builder Based CLIs (You are here)
* [Part 3 - Using Traits in Builder Based CLIs][part_iii]
* [Part 4 - Derive Based CLIs][part_iv]
* [Part 5 - Traits with Derive Based CLIs][part_v]

## Previously On...

Reminder I said, the commands and arguments of our `bustup` will only print
messages to the terminal (without color...perhaps I'll fully explore `Ctx`
initialization and well defined output coloring in another post).

Also another reminder, we don't care about the tool itself here, so forgive the
brevity and code dump.

## Let's gooooo

First, some setup:

```text
$ cargo new bustup
$ cd bustup
$ git add .
$ git commit -m "Initial Commit"
$ git switch -c builder
$ cargo add clap anyhow
```

And now the code:

> **NOTE**
> This post is not attempting to show all the cool things you can do with
> `clap`, or even trying to use any of the developer niceties. That would get
> in the way of what we're trying to demo, so the code is naturally a little
> terse to keep the post ~~shorter~~ bearable.

```rust
// src/main.rs
mod cli;

fn main() -> anyhow::Result<()> {
    let args = cli::build().get_matches();

    todo!("Run the program!");

    Ok(())
}
```

And now the actual CLI:

```rust
// src/cli.rs
use clap::{Arg, ArgAction, Command};

pub fn build() -> Command {
    Command::new("bustup")
        .about("Not rustup")
        .subcommand(
            Command::new("update")
                .about("update toolchains")
                .arg(
                    Arg::new("toolchain")
                        .help("toolchain to update")
                        .action(ArgAction::Set)
                        .default_value("default"),
                )
                .arg(
                    Arg::new("force")
                        .short('f')
                        .long("force")
                        .help("Forcibly update")
                        .action(ArgAction::SetTrue),
                ),
        )
        .subcommand(
            Command::new("target")
                .about("manage targets")
                .arg(
                    Arg::new("toolchain")
                        .help("toolchain to use")
                        .long("toolchain")
                        .short('t')
                        .action(ArgAction::Set)
                        .default_value("default"),
                        .global(true),
                )
                .subcommand(
                    Command::new("add")
                        .about("add a target")
                        .arg(
                            Arg::new("target")
                                .help("The target to add")
                                .action(ArgAction::Set)
                            ),
                )
                .subcommand(
                    Command::new("list").about("list targets").arg(
                        Arg::new("installed")
                            .help("Only list installed targets")
                            .long("installed")
                            .short('i')
                            .action(ArgAction::SetTrue),
                    ),
                )
                .subcommand(
                    Command::new("remove")
                        .about("remove a target")
                        .arg(
                            Arg::new("target")
                                .help("The target to remove")
                                .action(ArgAction::Set)
                                .default_value("default"),
                        ),
                ),
        )
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
not yet implemented: Run the program!
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

Let's commit this as our starting point.

```text
$ git commit -am "starting point"
```

### Running the Program

So we have the basic CLI structure, now how should we structure our program?

#### Naive Matching and no `Ctx`

The naive method is to match on a particular subcommand, and dispatch to some
`run`-like function that takes a `clap::ArgMatches` as a context. This is a
common approach, but there are downsides. Let's implement this method for a
single command `bustup update` just so we can contrast it later.

> **NOTE**
> I'm going to omit code examples that only contain things like declaring
> module structure for brevity. The full code is located in the repository if
> interested.

```rust
// src/cli/cmds/update.rs
use anyhow::Result;
use clap::ArgMatches;

pub fn run(args: &ArgMatches) -> Result<()> {
    println!(
        "updating toolchain...{}",
        args.get_one::<String>("toolchain")
    );
    Ok(())
}
```

And finally, our `main.rs`:

```rust
// src/main.rs
use crate::cli::cmds::update;

fn main() -> anyhow::Result<()> {
    let args = cli::build().get_matches();

    match args.subcommand() {
        Some(("update", args)) => {
            update::run(args)?;
        }
        _ => todo!("implement other subcommands"),
    }

    Ok(())
}
```

We can see that it works by running the update command:

```text
$ cargo run -- update
updating toolchain...default

$ cargo run -- update footoolchain
updating toolchain...footoolchain
```

This is a perfectly valid approach! For a small number of subcommands, or CLIs
with single-layer subcommands this approach is usually fine. However, it can
start to go sideways quickly when using multiple layers of configuration or
nested subcommand layers, especially when context/run-actions need to happen at
each individual layer.

#### Adding `Ctx`

It's so tempting to just use `clap::ArgMatches` as the passed in context like
we did above. And for a simple CLI, it'd probably be fine. But we're pretending
to build a large and complex CLI.

Based on everything we learned when talking about `Ctx` above, we're already
convinced we should be using a Context Struct. And we know initializing and
updating one can be a complex process.

But we're starting small and we won't be adding configuration files or
environment variables to complicate things in this post. So let's just create
our `Ctx` and pass that to `update::run`:

```rust
// src/main.rs
mod cli;
// 👇 new
mod context;

//                             👇 new
use crate::{cli::cmds::update, context::Ctx};

fn main() -> anyhow::Result<()> {
    let args = cli::build().get_matches();

    match args.subcommand() {
        Some(("update", args)) => {
            // 👇 new
            let ctx = Ctx::from_update(args);
            update::run(&ctx)?;
        }
        _ => todo!("implement other subcommands"),
    }

    Ok(())
}

```

```rust
// src/context.rs
use clap::ArgMatches;

pub struct Ctx {
    pub toolchain: String,
}

impl Ctx {
    pub fn from_update(args: &ArgMatches) -> Self {
        Self {
            toolchain: args.get_one::<String>("toolchain").unwrap().to_string(),
        }
    }
}
```

```rust
// src/cli/cmds/update.rs
use anyhow::Result;

// 👇 new
use crate::context::Ctx;

//         👇 new
pub fn run(ctx: &Ctx) -> Result<()> {
    //                                 👇 new
    println!("updating toolchain...{}", ctx.toolchain);
    Ok(())
}

```

This also works!

But for more complex CLIs, this will get tedious and error prone as well for a
few reasons:

- In the above code we're creating the `Ctx` from scratch which wouldn't be an
  option with nested subcommands that each need to *update* a context
- As we nest subcommands the code to update the context and call the next
  subcommand is going to become pure boilerplate noise.

## Next Time

In the [next post][part_iii] we'll see how to use traits to perform some magic and enforce
structure on what could otherwise become unbridled chaos.

[part_i]: https://kbknapp.dev/cli-structure-01/
[part_iii]: https://kbknapp.dev/cli-structure-03/
[part_iv]: https://kbknapp.dev/cli-structure-04/
[part_v]: https://kbknapp.dev/cli-structure-05/

