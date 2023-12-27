+++
title = "CLI Structure in Rust - Part 3"
slug = "cli-structure-03"
draft = false
template = "page.html"
date = 2023-12-22
in_search_index = true
[taxonomies]
categories = ["programming", "rust", "blog"]
tags = ["rust", "cli", "clap"]
+++

In Part 3 of this series we define a trait to help control the chaos and
enforce some structure on our CLI programs.

<!-- more -->

## Series Contents

* [Part 1 - Contexts][part_i]
* [Part 2 - Builder Based CLIs][part_ii]
* Part 3 - Using Traits in Builder Based CLIs (You are here)
* [Part 4 - Derive Based CLIs][part_iv]
* [Part 5 - Traits with Derive Based CLIs][part_v]

## Previously On...

In [Part 2][part_ii] we saw how using context structs was a better idea, but
the method in which we niavely applied it has the potential to lead to pain the
larger our CLI program grows.

## Traits have entered the chat

We can solve this by using a trait to define the context updating, running, and
traversing down the subcommand call stack. I call this `Cmd` and it looks like
this:

```rust
// src/cli.rs

use clap::ArgMatches;
use anyhow::Result;

pub trait Cmd {
    fn update_ctx(&self, _args: &ArgMatches, _ctx: &mut Ctx) -> Result<()> {
        Ok(())
    }
    fn run(&self, _ctx: &mut Ctx) -> Result<()> {
        Ok(())
    }
    fn next_cmd<'a>(
        &self,
        _args: &'a ArgMatches,
    ) -> Option<(Box<dyn Cmd>, &'a ArgMatches)> {
        None
    }
}

// .. all else unchanged
```

There is a lot going on here, so let's break it down.

### `Cmd::update_ctx`

First we have a method that is responsible for taking a CLI values struct (in
our case `clap::ArgMattches`) and normalizing any values into `Ctx`.

By default this does nothing because there may be no context to update.

### `Cmd::run`

This can be thought of as the entry-point to whatever functionality the command
itself does.

Notice that it only takes a `Ctx` as context.

This is important because it means you can fully test your commands by building
a mock `Ctx` and don't have to worry about mocking CLIs, etc. It *also* means
you can use the same backing code for a CLI, a TUI, or a GUI because the code
only need a `Ctx` to run. It doesn't know anything about CLI values or
environments, or config files.

By default this method also does nothing allowing for intermediate subcommands
that have no distinct action.

### `Cmd::next_cmd`

Finally, we have a method for retrieving the next subcommand to run, if any.

Like all other methods, it also does nothing by default returning that there
are no further subcomands to run.

The interesting bit about this method is the return when there *is* another
subcommad to run. It returns a boxed trait object of something else that also
implements `Cmd`.

## Traversing the call-stack with `Cmd`

So we've added the definition of `Cmd`, but there is another bit of magic
required to really reap the ergonomic benefits.

We implement a function on the trait object itself for doing the subcommand
traversal.

```rust
// src/cli.rs

//  ... all else unchanged

impl dyn Cmd + 'static {
    pub fn walk_exec(&self, args: &ArgMatches, ctx: &mut Ctx) -> Result<()> {
        self.update_ctx(args, ctx)?;
        self.run(ctx)?;
        if let Some((c, m)) = self.next_cmd(args) {
            return c.walk_exec(m, ctx);
        }
        Ok(())
    }
}
```

This updates the context, runs the action, and if there is another subcommand
to call it calls `walk_exec` on it.

> **WARNING**
> Some may smell a whiff of recursion and be wary of any such tactics, however
> by design subcommands cannot, so any such loop would be an inherent error
> where a stack overflow would be desirable. Additionally, I have never seen
> subcommands nested so heavily that an overflown stack would be possible. Even
> on very poorly designed subcommand-based CLIs the nesting does not go above
> 5-6 levels which is a far cry away from the *hundreds* that would be required
> to overflow the stack even on platforms like Windows which use small 1MB
> stacks by default.

> **PRO TIP**
> You can add another method on the trait object that allows for simply
> updating the context all the way down the call-stack without actually
> executing any of the code. This is super useful in testing where you don't
> actually want to run the code for real, but you'd like the context to be
> accurately updated. I usually call this `walk_update_ctx` and simply omit the
> `selef.run(..)` call while recursively calling `walk_update_ctx` instead of
> `walk_exec`

## Our new `main`

Ok, so we have all the glue, now we just need to fill in the implementations.
Let's start with `main` because it contains some interesting notes, here's the
entire file:

```rust
// src/main.rs
mod cli;
mod context;

use crate::{
    cli::{Bustup, Cmd},
    context::Ctx,
};

fn main() -> anyhow::Result<()> {
    let args = cli::build().get_matches();

    let mut ctx = Ctx::default();
    let cmd: Box<dyn Cmd> = Box::new(Bustup);
    cmd.walk_exec(&args, &mut ctx)
}
```

Notice we create a boxed trait object and then call `walk_exec` on it to kick
off the entire show.

> **NOTE**
> `Box` has a specialization for Zero-Sized Types (ZSTs) that doesn't actually
> allocate. `cmd` however is a pointer to a vtable.

Notice we reference `cli::Bustup` so let's take a look at that next.

## CLI Structs

When using this pattern with `clap`'s builder based CLIs I like to create empty
Zero Sized structs which I can actually implement the `Cmd` trait on.

This forces me to keep no state except in `Ctx`, and allows the neat
no-allocation for `Box`.

I also prefer to strictly name these structs after their path in the CLI
hierarchy. For example `Bustup` is the top level command, and `bustup target
add` would end up being named `BustupTargetAdd`. This is handy to always know
exactly what command you're operating on in code.

For the top-level command I typically place it in the `cli` module, but that
isn't strictly necessary.

```rust
// src/cli.rs
pub struct Bustup;

impl Cmd for Bustup {
    fn next_cmd<'a>(&self, args: &'a ArgMatches) -> Option<(Box<dyn Cmd>, &'a ArgMatches)> {
        match args.subcommand() {
            Some(("update", m)) => Some((Box::new(cmds::update::BustupUpdate {}), m)),
            Some(("target", m)) => Some((Box::new(cmds::target::BustupTarget {}), m)),
            _ => None,
        }
    }
}
```

Notice I don't need to do anything for `run` or `update_ctx` in our toy
example, but adding code to execute at those hooks would be trivial.

One annoyance of the builder-pattern based CLIs is that we have manually handle
dispatching to another subcommand at each level, but this is a minor complaint
that could easily be handled by a macro if we so desired.

Looking over at our `update.rs` file next we see:

```rust
// src/cli/cmds/update.rs
pub struct BustupUpdate;

impl Cmd for BustupUpdate {
    fn update_ctx(&self, args: &ArgMatches, ctx: &mut Ctx) -> Result<()> {
        ctx.toolchain = args.get_one::<String>("toolchain").cloned();
        ctx.force = args.get_flag("force");
        Ok(())
    }

    fn run(&self, ctx: &mut Ctx) -> Result<()> {
        println!(
            "Updating {} toolchain...{}",
            ctx.toolchain.as_ref().unwrap(),
            if ctx.force { " (forced)" } else { "" }
        );
        Ok(())
    }
}
```

## Intermediate Subcommands

In our toy example `bustup target ...` is an interesting example because it's
an intermediate layer subcommand that has no functionality of it's own
(although it could), and it defines a "global argument" (an argument that is
present in all of it's child subcommands).

So even though we don't have a distinct action to take, we do need to update
the context with our global CLI option. We also dispatch to our child
subcommands exactly how the top level `Bustup` command does.

```rust
// src/cli/cmds/target.rs

pub struct BustupTarget;

impl Cmd for BustupTarget {
    fn update_ctx(&self, args: &ArgMatches, ctx: &mut Ctx) -> Result<()> {
        ctx.toolchain = args.get_one::<String>("toolchain").cloned();
        Ok(())
    }

    fn next_cmd<'a>(&self, matches: &'a ArgMatches) -> Option<(Box<dyn Cmd>, &'a ArgMatches)> {
        match matches.subcommand() {
            Some(("add", m)) => Some((Box::new(add::BustupTargetAdd), m)),
            Some(("list", m)) => Some((Box::new(list::BustupTargetList), m)),
            Some(("remove", m)) => Some((Box::new(remove::BustupTargetRemove), m)),
            _ => None,
        }
    }
}
```

## The Rest of the Owl

Everything should fairly self-evident at this point, so we'll breeze through
the next few files only calling out interesting aspects.

Just like `BustupUpdate` the following are all leaf subcommands and thus do not
need to implement `Cmd::next_cmd`.

```rust
// src/cli/cmds/target/add.rs
pub struct BustupTargetAdd;

impl Cmd for BustupTargetAdd {
    fn update_ctx(&self, args: &ArgMatches, ctx: &mut Ctx) -> Result<()> {
        ctx.target = args.get_one::<String>("target").cloned();
        Ok(())
    }

    fn run(&self, ctx: &mut Ctx) -> Result<()> {
        println!(
            "Adding the {} target to the {} toolchain",
            ctx.target.as_ref().unwrap(),
            ctx.toolchain.as_ref().unwrap(),
        );
        Ok(())
    }
}
```

```rust
// src/cli/cmds/target/list.rs
pub struct BustupTargetList;

impl Cmd for BustupTargetList {
    fn update_ctx(&self, args: &ArgMatches, ctx: &mut Ctx) -> Result<()> {
        ctx.installed = args.get_flag("installed");
        Ok(())
    }

    fn run(&self, ctx: &mut Ctx) -> Result<()> {
        println!(
            "Listing {} targets for the {} toolchain",
            if ctx.installed { "installed" } else { "all" },
            ctx.toolchain.as_ref().unwrap(),
        );
        Ok(())
    }
}
```

```rust
// src/cli/cmds/target/remove.rs
pub struct BustupTargetRemove;

impl Cmd for BustupTargetRemove {
    fn update_ctx(&self, args: &ArgMatches, ctx: &mut Ctx) -> Result<()> {
        ctx.target = args.get_one::<String>("target").cloned();
        Ok(())
    }

    fn run(&self, ctx: &mut Ctx) -> Result<()> {
        println!(
            "Removing the {} target from the {} toolchain",
            ctx.target.as_ref().unwrap(),
            ctx.toolchain.as_ref().unwrap(),
        );
        Ok(())
    }
}
```

## One Final Change

If you look at the repository's final code, you'll notice that one other thing
I typically change, although it's not at all mandatory is to move the CLI
construction into the respective modules instead of being fully contained at
`src/cli.rs`. I find this lets me better encapsulate shared arguments and keep
smaller subsets of the code in my head at any given time. To do this I usually
place a `fn build() -> clap::Command` function in each module that contains a
subcommand. For example, the `cli.rs` would then look this:

```rust
// src/cli.rs
pub fn build() -> Command {
    Command::new("bustup")
        .about("Not rustup")
        .subcommand(cmds::update::build())
        .subcommand(cmds::target::build())
}
```

And for example `src/cli/cmds/target.rs`:

```rust
// src/cli/cmds/target.rs

pub fn build() -> Command {
    Command::new("target")
        .about("manage targets")
        .arg(
            Arg::new("toolchain")
                .help("toolchain to use")
                .long("toolchain")
                .action(ArgAction::Set)
                .default_value("default")
                .short('t')
                .global(true),
        )
        .subcommand(add::build())
        .subcommand(list::build())
        .subcommand(remove::build())
}
```

And so on and so forth.

The [full code][full_code] can be found in the repository under the `builder`
branch.

## Summary for Builder based CLIs

The Builder Pattern is an older and more verbose way to define CLIs in `clap`.
When used with this method for structuring your CLIs it has the benefit of
being easy to enforce single-source-of-truth `run` functions and `clap` does a
lot of the heavy lifting when it comes to providing our `ArgMatches` structs
that we can use to update the `Ctx`.

## Next Time!

That's it for Builder Pattern based CLIs!

In the next post we'll start back over using `clap`'s Derive based approach and
see how that changes our trait definitions and concerns.

[part_i]: https://kbknapp.dev/cli-structure-01/
[part_ii]: https://kbknapp.dev/cli-structure-02/
[part_iv]: https://kbknapp.dev/cli-structure-04/
[part_v]: https://kbknapp.dev/cli-structure-05/
[full_code]: https://github.com/kbknapp/bustup
