+++
title = "CLI Structure in Rust"
slug = "cli-structure-01"
draft = false
template = "page.html"
date = 2023-12-19
in_search_index = true
[taxonomies]
categories = ["programming", "rust", "blog"]
tags = ["rust", "cli", "clap"]
+++

How should one structure their CLI programs? I've seen this question a few
times, and over the years I've come up with some patterns that I really like.
This post explores the variations of these patterns and trade-offs between
them.

<!-- more -->

## Series Contents

* Part 1 - Contexts (You are here)
* [Part 2 - Builder Based CLIs][part_ii]
* [Part 3 - Using Traits in Builder Based CLIs][part_iii]
* [Part 4 - Derive Based CLIs][part_iv]
* [Part 5 - Traits with Derive Based CLIs][part_v]

## Intro

I recently [came across this question][question] (and [associated
answer][answer]) on the `clap` repository. The answer given is a good one. But
I wanted to expand with my own findings and practices, which spurred the
motivation for this post.

## Type of CLI

First things first. We need to know which type of CLI you're building. For
these purposes there are two broad categories:

- Subcommand based
- Single Command Argument based

An example of "subcommand based" would be `cargo`; each function is represented
by a distinct "command" that is a child (or grand-child, or great-grand-child,
etc.) of the top level command `cargo`. i.e:

```text
$ cargo build
$ cargo clean
$ cargo new
```

Whereas an example of a single command based CLI would be `ripgrep`, where the
tool has a single command, in this case `rg` that is controlled solely by
arguments.

Most often, although not always, subcommand based CLIs each command is a
distinct function, and at some level they could have perhaps been distinct
tools. For example `cargo build` builds the Rust program, whereas `cargo clean`
cleans all build artifacts. These could probably have been implemented as
`cargo-build` and `cargo-clean` respectively. But because those two commands
are closely related in purpose (handling Rust projects) it makes far more sense
to combine them into a single `cargo` command which has the added benefit of
being able to easily share code, cohesion in things like error messages and
arguments, and not cluttering the file system with `cargo-*` commands, etc.

Similarly, single command based CLIs often, but not always, have a single
function. In Ripgrep's case, that is searching through files. This function can
be altered in ways via arguments, but ultimately it's still just searching
through files.

> **NOTE**
> Ripgrep is actually a good example of the "but not always" saying.
> Technically, Ripgrep can also manage and assist the user with "type
> definitions" that allow searching exclusively through specific file types.
> This function is controlled through the `--type-*` arguments, which when used
> add, remove, or list various "type definitions" and thus don't actually do
> the "sole functionality" of searching through files. These could have been
> made subcommands, such as `rg type [OPTIONS]` however, this set of
> functionality is minor enough (and the *only* other functionality beyond
> searching files) that making it a subcommand could have detracted away from
> the primary purpose of searching needlessly. Subcommands can, and do,
> complicate matters.

Ok, so now we know what kind of CLI we're building, we can start to decide on
what pattern to use.

This post will focus almost exclusively on subcommand based, because it's the
most interesting and complicated.

> **NOTE**
> I plan to write a "BONUS POST" that discusses a single-command based pattern
> for completeness sake if that's the route you're taking.

## The CLI - bad rustup...bustup

So we're making a subcommand based CLI. The example we'll use is a toy example
to keep this already long post manageable since the actual tool itself isn't
important here. This tool will mimic a tiny tiny subset of `rustup` _extremely
poorly_...i.e. we're just going to print some messages.

We'll call this `bustup`.

> **NOTE**
> Why  `rustup`? Because it implements a pattern that will be useful to
> explore, and that's using several nested layers deep of subcommands. For
> example `cargo` (and other tools like `git`) typically only go one layer deep
> (i.e. `git clone`, `cargo update`, etc.) whereas `rustup` can go several more
> layers deep, such as `rustup toolchain add ...`. This will be important as
> the pattern we'll use has additional considerations when nesting more than a
> layer or two deep.

### Parsing Library

We also need to decide on a parsing library. I'm biased towards [`clap`][clap]
so that's what we'll be using. However, this pattern should more or less be
applicable to any command line argument parsing library.

`clap` like several other parsing libraries supports two different modes of
building a CLI. The so-called "Builder Pattern" and the "derive based" method.

We'll cover both methods because the pattern used is different in interesting
ways for each.

But this also means we'll have to re-implement `bustup` twice using the
different methods. Another reason to keep `bustup` minimally simple.

## `bustup` Command Structure

Here's what we'll implement:

- `bustup update [toolchain] [--force]`
- `bustup target add [--toolchain=<TOOLCHAIN>]`
- `bustup target remove [--toolchain=<TOOLCHAIN>]`
- `bustup target list [--toolchain=<TOOLCHAIN>] [--installed]`

## Context Structs

Something all (or at the very least *most*) CLI applications deal with the idea
of a "runtime context."

When doing an actual action in code, there is normally some defined "context"
informing the runtime code on what actions to take and how to take them. For
example our `bustup update` will need some contextual knowledge about which
`toolchain` the user wishes to update.

There are many different ways to store and pass around runtime context. The
most common ways are via:

- The CLI value structs themselves
- Context Structs
- Globals

I will omit globals, as it's typically an anti-pattern and frowned upon
especially in Rust as these context objects are typically mutable and having
global mutable state precludes many multi-threading possibilities and negates
many of Rust's safety features.

That leaves us with the CLI value structs, and context structs.

As the heading may suggest, Context Structs are what this post will focus on.
But before doing so I will mention below in the appropriate sections about CLI
value structs; as the types of value structs and how they're used is different
depending on the mode of `clap` one is using.

For now, suffice it to say that Context Structs are nearly always a better
approach, even though they may seem redundant in some circumstances.

### Meeting `Ctx`

A Context Struct (usually abbreviated `Ctx`) is just a local struct containing
*normalized* values from all configuration sources (whereas CLI value structs
typically only get their values from the CLI or perhaps from the environment
depending on argument parsing library features).

In our `bustup update` example, using the `toolchain` as the context we may
have a struct similar to:

```rust
struct Ctx {
    toolchain: String
}
```

> **NOTE**
> It's common to prefer borrowing of values in some context structs, especially
> if the value is only temporary. For instance:
> ```rust
> struct Ctx<'a> {
>     toolchain: &'a str,
> }
> ```
> But for brevity and simplicity I'm leaving out any such concerns in order to
> focus on the topic of this post.

An important distinction about context structs is that they contain the
*normalized configuration values*.

### Single Source of Truth

The primary benefit of using Context Structs is to provide *A Single Source of
Truth* to runtime code.

Although our toy `bustup` doesn't use overriding flags, if we pretend it does
for the sake of argument real quick let's see what could happen.

Imagine something like our `bustup update` taking a `--confirm` flag that says
to ask the user for confirmation before downloading and installing any updates,
and also a `--no-confirm` flag that says *not* to ask the use first.

> **NOTE**
> Naively it may seem silly to provide both `--confirm` and `--no-confirm`,
> especially if one of those values is the default behavior of the program (for
> instance if either flag is provided it acts *as if* `--confirm` was used).
> However, it's actually a super useful pattern to provide because it allows
> users to set up custom aliases. For example a user normally does not want to
> be asked for confirmation, so they set up `alias bustup-up='bustup update
> --no-confirm'`. Now one day, they are on a metered network connection and
> decide they'd like to confirm before downloading a massive update, so run:
>
> ```text
> $ bustup-up --confirm
>
> ... this expands to ...
>
> $ bustup update --no-confirm --confirm
> ```

As it stands, we'd probably start by only passing in the CLI values directly to
the `run_update` function. This would mean it's the function's responsibility
for knowing about and deconflicting both `--confirm` and `--no-confirm` which
is something it should not have to care about. Additionally, if we had other
subcommands with similar flags, they *too* would have to know about and handle
these cases.

This typically leads to people passing in *duplicate* context. e.g. they'd pass
in the CLI values raw and a boolean flag of some kind:

```rust
fn run_update(cli_values: &Args, confirm: bool) { /* .. */ }
```

To properly write `run_update` now one needs to know the difference between the
two locations which contain values for whether or not we should prompt the user
for confirmation.

And that's not all!

*Technically*, the CLI values are only a single layer in a delicious cake that
is program configuration and on their own make for bad context....

### Initializing `Ctx`

Although it's out of scope for this post to fully explore this topic; for
completeness sake let's take an ultra-brief look at what it's like to
initialize a `Ctx` struct.

Let's pretend we had a program that *could* use pretty colors for terminal
output. So we have a Context Struct that looks like:

```rust
struct Ctx {
    // If `true` pain the terminal
    color: bool
}
```

Let's further assume our program supports system-wide, user-level, and
project-level configuration files, as well as CLI flags and environment
variables for controlling settings.

For something as simple as turning color on or off, the ways to control such a
setting could be:

- A default value of `Auto`  meaning to check if STDOUT is a TTY or not
- A system-wide config file setting
- A user-level config file setting
- A project level config file setting
- A set of environment variables
	- E.g. could be any of `TERM`, `NO_COLOR`, or perhaps `PROGRAM_COLOR`
- A set of CLI flags
	- E.g. `--color=...` or `--no-color`

This seems like madness! Yet its exactly how a well behaved CLI program should
function!

So a full initialization sequence probably looks something like this:

- At program startup do something like `Ctx::default()`
- Load system-wide configuration files, if any (e.g. `/etc/...`) perhaps in a
  `Config<System>` struct
    - Normalize values if needed
	- Update the `Ctx` with something like a `Config::update_ctx(&mut ctx)`
 - Load user-level configuration files, if any (e.g. `~/.config/...`) perhaps
   in a `Config<User>` struct
	- Normalize values if needed
	- Update the `Ctx` with something like a `Config::update_ctx(&mut ctx)`
 * Load environment variable values, if any
     * This step may be done, or partially done by your CLI parsing library,
       however things like `TERM` or `NO_COLOR` which are only loosely related
       to your `--color` and `--no-color` flags will probably be handled
       manually via something like a `Ctx::update_from_env` method
 * Parse CLI values
	- Normalize values if needed
	- Update the `Ctx` with something like a `Config::update_ctx(&mut ctx)`
 - You now have a fully initialized `Ctx`

The benefit being any actual functionality in your program no longer needs to
worry about _all of that_ and can just get passed an instance of `&Ctx` as an
argument check a quick `ctx.color` field and trust whatever is written there!

## Next Time!

In the [next post][part_ii] we'll actually dive in using the `clap` Builder based method to
see what using this pattern looks like in practice.

[question]: https://github.com/clap-rs/clap/discussions/5258
[answer]: https://github.com/clap-rs/clap/discussions/5258#discussioncomment-7866256
[clap]: https://crates.io/crates/clap
[part_ii]: https://kbknapp.dev/cli-structure-02/
[part_iii]: https://kbknapp.dev/cli-structure-03/
[part_iv]: https://kbknapp.dev/cli-structure-04/
[part_v]: https://kbknapp.dev/cli-structure-05/
