+++
title = "CLI Structure in Rust - Part 5"
slug = "cli-structure-05"
draft = false
template = "page.html"
date = 2023-12-27
in_search_index = true
[taxonomies]
categories = ["programming", "rust", "blog"]
tags = ["rust", "cli", "clap"]
+++

In Part 5 we update our trait to work with Derive based CLIs and discuss the
tradeoffs in doing so.

<!-- more -->

## Series Contents

* [Part 1 - Contexts][part_i]
* [Part 2 - Builder Based CLIs][part_ii]
* [Part 3 - Using Traits in Builder Based CLIs][part_iii]
* [Part 4 - Derive Based CLIs][part_iv]
* Part 5 - Traits with Derive Based CLIs (you are here)

## Previously On...

First a quick re-cap on how we used a trait in the Build method so we can
compare.

### Builder Trait Revisited

In the Builder method we defined a trait that enforced updating a context
struct, performing some action with only that context struct, and then calling
the next subcommand, if any.

A reminder about what our trait looked like:

```rust
pub trait Cmd {
    fn update_ctx(&self, _args: &clap::ArgMatches, _ctx: &mut Ctx) -> anyhow::Result<()> {
        Ok(())
    }
    fn run(&self, _ctx: &mut Ctx) -> anyhow::Result<()> {
        Ok(())
    }
    fn next_cmd<'a>(
        &self,
        _args: &'a clap::ArgMatches,
    ) -> Option<(Box<dyn Cmd>, &'a clap::ArgMatches)> {
        None
    }
}
```

We then implemented this trait on Zero Sized Type dummy structs:

```rust
struct Bustup;

impl Cmd on Bustup {
    // impl here...
}
```

We had a helper method implemented on the trait object itself that would do the
walking and enforcing of `update_ctx`->`run`->`next_cmd`.

```rust
impl dyn Cmd + 'static {
    pub fn walk_exec(&self, args: &clap::ArgMatches, ctx: &mut Ctx) -> anyhow::Result<()> {
        self.update_ctx(args, ctx)?;
        self.run(ctx)?;
        if let Some((c, m)) = self.next_cmd(args) {
            return c.walk_exec(m, ctx);
        }
        Ok(())
    }
}
```

Finally, we kicked off this process in `main` by creating a Trait Object out of
top level dummy struct. Because we used ZSTs, creating a `Box<dyn Cmd>` did not
actually need to heap allocate anything.

```rust
let mut ctx = Ctx::default();
let cmd: Box<dyn Cmd> = Box::new(Bustup);
cmd.walk_exec(&args, &mut ctx)
```

## Back to present day

Now that we're using a Derive based method we have a few issues if we were to
try and port that trait directly as is to our current code.

### CLI Value Structs

We are no longer using dummy structs and no longer passing around a
`clap::ArgMatches` as the CLI values.

If we tried to use ZST structs and instead of `clap::ArgMatches` pass in our
CLI structs we run into not being able to define the type of our CLI struct
since they're all distinct types.

```rust
pub trait Cmd {
    fn update_ctx(
        &self,
        _args: /* ... */, // <--- What goes here?
        _ctx: &mut Ctx
    ) -> anyhow::Result<()> { }
}
```

Since we *actually want* to go from our CLI struct to the context struct in
that method, it seems perfectly reasonable to *just use our CLI struct* instead
of a ZST dummy struct.

### Single Source of Truth in `run`

Deciding against a ZST dummy struct also has the consequence of now we have
access to our CLI struct *and* the context struct during `run` which can break
our "single source of truth rule."

There is no good answer this issue, and just like in Builder method we "just
live with" the possibility of "stringly-typed" bugs or other oddities of the
`ArgMatches` API; this is something we can live with.

For starters, we take self by shared reference (`&self`) which keeps us from
being able to easily mutate anything and encourages us to use the `Ctx` struct
if we needd to mutate a field such as for normalization or validation.

Additionally, I've just lived with allowing the CLI struct to be the source of
truth for any values that meet *all* of the following rules:

* No normalization is required for this CLI value (e.g. no need to deconflict
  or check multiple CLI value fields to determine what the user "really
  wanted")
* This CLI value has no complex environment variable equivalents (`clap` can
  handle basic environment variables which is the reason for for "complex"
  qualifier)
* The CLI value has no configuration file equivalents
* THe CLI argument is not global or shared between other commands

If all of these rules hold, I allow myself to use the CLI struct's values,
otherwise use the Context struct.

In practice this rarely an issue.

### Using `Box<dyn Cmd>` will now allocate...a lot

Because we got rid of ZST dummy structs and instead decided to use the CLI
value structs, that means creating a `Box<dyn Cmd>` out of them will allocate.
These structs can also end up being quite large depending on the number of CLI
arguments your program contains.

Luckily, all is not lost. It turns out we can just as easily use a `&dyn Cmd`
as well since our CLI struct is created at program startup and kept around
until after the end of the final `run` command.

This has the slight downside that you must keep the CLI struct around in
memory, but I have yet to find a program where that is a problem as even though
I said these structs can be quite large, it's still typically in well below a
page size (4k) even for enormous CLIs.

> **NOTE**
> See [Appendix A][appendix_a] for a variation on this trait that I've used in
> the past where I wanted to be able to drop the CLI structs but allow the
> program to continue to run.

Switching from `Box<dyn Cmd>` to a `&dyn Cmd` is trivial in our case:

```rust
// src/cli.rs
pub trait Cmd {
    fn update_ctx(&self, _ctx: &mut Ctx) -> anyhow::Result<()> {
        Ok(())
    }
    fn run(&self, _ctx: &mut Ctx) -> anyhow::Result<()> {
        Ok(())
    }
    //                          👇 new
    fn next_cmd(&self) -> Option<&dyn Cmd> {
        None
    }
}
```

Our impl signature for `Cmd` changes slightly:

```rust
// src/cli.rs
impl<'a> dyn Cmd + 'a { /* .. */ }
```

And finally our `main` function changes ever so slightly:

```rust
// src/main.rs
fn main() -> Result<()> {
    let args = Bustup::parse();

    let mut ctx = Ctx::default();
    // 👇 new
    let cmd: &dyn Cmd = &args;
    cmd.walk_exec(&mut ctx)
}
```

## Context

Alright, so we've adjusted our trait objects and our trait, but what about our
`Ctx`? Since we are allowing ourselves to use the CLI structs as a partial
context only for fields that are trivial, that leaves us with only the more
complex, or global values that we need to move into our `Ctx`. For `bustup`
that happens to only be the `toolchain` field used globally by all `bustup
target ...`.

```rust
// src/context.rs
#[derive(Default)]
pub struct Ctx {
    pub target_toolchain: Option<String>,
}
```

### Our First Hard Decision

What about `bustup update`? It takes a `[toolchain]` argument, but it meets all
the rules for us to just use the CLI struct instead of the context, right?
Should we have just named that context field a more general `toolchain` and
shared it?

My answer is, no.

Although those two arguments share a name, they may in the future not share
anything else. I prefer to qualify fields with their path, *or* use child
contexts.

#### Child Contexts

Just a quick aside, our toy example `bustup` isn't complex enough to really
demo this, but in a real CLI it's not a bad idea to instead of qualifying all
`Ctx` fields with their path, instead to use actual sub-contexts. For example
something like:

```rust
struct Ctx {
    ctx_target: TargetCtx,
    ctx_update: UpdateCtx,
}
```

> **PRO TIP**
> You an also use things like [`Lazy`][lazy] initialized fields because due to
> how CLI command work, you typically only walk down a single command path,
> e.g. `bustup update` won't ever require anything out of `bustup target ...`.
> Of course, there are some minor exceptions depending on what you're doing and
> how things are structured, but in general this is the case.

## Enums and Subcommands

Because of how `clap`'s Derive methods parse the subcommands we get back an
enum representing one possible subcommand.

And then when we want to implement `Cmd::next_cmd` we have to do this tedious
dance to match:

```rust
// src/cli.rs

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

impl Cmd for Bustup {
    fn next_cmd(&self) -> Option<&dyn Cmd> {
        match self.cmd {
            BustupCmd::Update(cmd) => Some(cmd),
            BustupCmd::Target(cmd) => Some(cmd),
        }
    }
}
```

While this works, it's tedious to update all the various levels we could have
subcommands and also must be updated any time we add or remove a subcommand!

If you don't mind taking on another dependency it turns out we can just
implement `Cmd` directly on our enum in a low drama way!

We'll use the [`enum_delegate`][enum_delegate] crate to reduce some boiler
plate for us:

```text
$ cargo add enum_delegate
```

Now we change our code slightly to register our trait, and utilize it on our
`BustupCmd` enum:

```rust
// src/cli.rs

// 👇 new
#[enum_delegate::register]
pub trait Cmd { /* .. unchanged */ }

// 👇 new
#[enum_delegate::implement(Cmd)]
#[derive(Clone, Subcommand)]
pub enum BustupCmd {
    Update(update::BustupUpdate),
    Target(target::BustupTarget),
}
```

What does that buy us? Well *all* of the variants of `BustupCmd` implement
`Cmd` so we actually just want to delegate directly to them.

Why that matters is when we call `<Bustup as Cmd>::next_cmd` what we really
want is to call the that method on our _inner_ value of whatever variant
`BustupCmd` happens to currently have.

Well now that we have some boiler plate free method do that delegation, we just
have to return the enum value itself!

```rust
impl Cmd for Bustup {
    // 👇 new
    fn next_cmd(&self) -> Option<&dyn Cmd> {
        Some(&self.cmd)
    }
}
```

Yes, this will call `Cmd::update_ctx` and `Cmd::run` on the enum itself as
well, but since we have default implementation that do nothing it doesn't
actually matter.

The *only* thing we have to do is write those three lines for each CLI struct
that contains subcommands. No more changes even when we add and remove
subcommands!

> **NOTE**
> This IMO is a big improvement over the builder method

## The Rest of the Owl

Back to code-dump time! We have all the bits we need to implement our remaining
commands and see how it all fits together.

Starting with `bustup update`:

```rust
// src/cli/cmds/update.rs
use crate::{cli::Cmd, context::Ctx};

// All else unchanged...

impl Cmd for BustupUpdate {
    fn run(&self, _ctx: &mut Ctx) -> anyhow::Result<()> {
        println!(
            "updating toolchain {}{}",
            self.toolchain,
            if self.force { " (forced)" } else { "" }
        );
        Ok(())
    }
}
```

```rust
// src/cli/cmds/target.rs
use crate::{cli::Cmd, context::Ctx};

// All else unchanged...

impl Cmd for BustupTarget {
    fn update_ctx(&self, ctx: &mut Ctx) -> anyhow::Result<()> {
        ctx.target_toolchain = Some(self.toolchain.clone());
        Ok(())
    }
    fn next_cmd(&self) -> Option<&dyn Cmd> {
        Some(&self.cmd)
    }
}
```

```rust
// src/cli/cmds/target/add.rs
use crate::{cli::Cmd, context::Ctx};

// All else unchanged...

impl Cmd for BustupTargetAdd {
    fn run(&self, ctx: &mut Ctx) -> anyhow::Result<()> {
        println!(
            "Adding the {} target to the {} toolchain",
            self.target,
            ctx.target_toolchain.as_ref().unwrap(),
        );
        Ok(())
    }
}

```

```rust
// src/cli/cmds/target/list.rs
use crate::{cli::Cmd, context::Ctx};

// All else unchanged...

impl Cmd for BustupTargetList {
    fn run(&self, ctx: &mut Ctx) -> anyhow::Result<()> {
        println!(
            "Listing {} targets for the {} toolchain",
            if self.installed { "installed" } else { "all" },
            ctx.target_toolchain.as_ref().unwrap(),
        );
        Ok(())
    }
}

```

```rust
// src/cli/cmds/target/remove.rs
use crate::{cli::Cmd, context::Ctx};

// All else unchanged...

impl Cmd for BustupTargetRemove {
    fn run(&self, ctx: &mut Ctx) -> anyhow::Result<()> {
        println!(
            "Removing the {} target from the {} toolchain",
            self.target,
            ctx.target_toolchain.as_ref().unwrap(),
        );
        Ok(())
    }
}
```

And that's it!

The [full code][full_code] can be found in the repository under the `derive`
branch.

## Summary

By structuring our code in such a way we can have clean delineations between
"setup" and "action" code allowing us to split up code to run at the
appropriate point of program execution logically.

It also helps us to enforce a little bit of rigor in appropriately using our
context structs and keeping the "run" code single purpose.

There will be times when you want to manually "run" one of the commands from
within code, and with this type of setup it should be mostly trivial as you
only need to ensure the context struct is appropriately populated and then call
`Cmd::run`.

> **NOTE**
> It's possible to use an associated type for an "output" of the `Cmd::run`
> method if you prefer, for these manual run scenarios. However I've found that
> it's easier to maintain by just setting a context field if `run` methods need
> to pass data between them which is actually quite rare.

## Appendix A: Short Lived CLI Structs

If for some reason we do not want our CLI structs to remain in memory for the
duration of the program such as either they take up significant memory or
perhaps there is sensitive information that must be cleaned up, we can alter
our `Cmd` trait slightly to allow dropping.

In this variation I think of the `Cmd` trait as an initializer, and thus I
either rename `run` -> `runner` or remove it altogether.

Addressing these options in reverse order...

### Removing `run`

In these scenarios your "real" `main`/`run` will just take the final `Ctx` as
it's only argument. This option can get tricky though if you have multiple
terminal methods that are distinct, e.g. `git clone` versus `git push`. Passing
a `Ctx` to some final `run` method would probably need to match on some field
to figure out what functionality it really needs to execute. Hence this method
I typically only use in single purpose daemons where there is esseentially
always a single well defined "running" state.

The code would look something like:

```rust
pub trait Cmd {
    fn update_ctx(&self, _ctx: &mut Ctx) -> anyhow::Result<()> {
        Ok(())
    }
    fn next_cmd(&self) -> Option<&dyn Cmd> {
        None
    }
}

fn main() -> anyhow::Result<()> {
    let args = Bustup::parse();

    let mut ctx = Ctx::default();
    let cmd: &dyn Cmd = &args;
    cmd.walk_exec(&mut ctx)?;

    // 👇 new
    drop(args);

    // 👇 new
    run(ctx)
}

// 👇 new
fn run(ctx: &mut Ctx) -> Result<()> { /* .. */ }
```

### Renaming `run` to `runner`

In this variation `run` is more of an object initializer that returns something
that can be `run`.

This means I typically make another trait to represent the terminal `Run`
action that should be performed.

Depending on the use case you can also just *not implement* `runner` in any
of the commaneds except the terminal one.

```rust
// 👇 new
pub trait Runnable {
    fn run(&self, _ctx: &mut Ctx) -> Result<()>;
}

pub trait Cmd {
    fn update_ctx(&self, _ctx: &mut Ctx) -> anyhow::Result<()> {
        Ok(())
    }
    // 👇 renamed                                👇 new
    fn runner(&self, _ctx: &mut Ctx) -> Option<Box<dyn Runable>> {
        None
    }
    fn next_cmd( &self,) -> Option<&dyn Cmd> {
        None
    }
}

impl<'a> dyn Cmd + 'a {
    pub fn walk_exec(&self, ctx: &mut Ctx) -> anyhow::Result<Option<Box<dyn Runnable>> {
        self.update_ctx(args, ctx)?;
        // 👇 new (return if we have something to run)
        if let Some(run) = self.runnere(ctx)? {
            return Ok(Some(run));
        }
        if let Some((c, m)) = self.next_cmd(args) {
            return c.walk_exec(m, ctx);
        }
        Ok(())
    }
}

fn main() -> anyhow::Result<()> {
    let args = Bustup::parse();

    let mut ctx = Ctx::default();
    let cmd: &dyn Cmd = &args;
    let runner = cmd.walk_exec(&mut ctx)?;

    // 👇 new
    drop(args);

    // 👇 new
    runner.run(ctx)
}
```

[part_i]: https://kbknapp.dev/cli-structure-01/
[part_ii]: https://kbknapp.dev/cli-structure-02/
[part_iii]: https://kbknapp.dev/cli-structure-03/
[part_iv]: https://kbknapp.dev/cli-structure-04/
[lazy]: https://docs.rs/once_cell/latest/once_cell/unsync/struct.Lazy.html
[enum_delegate]: https://crates.io/crates/enum_delegate
[full_code]: https://github.com/kbknapp/bustup
