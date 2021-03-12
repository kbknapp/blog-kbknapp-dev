+++
title = "clap v3 Update: Structopt"
slug = "clap-v3-update-structopt"
draft = false
template = "page.html"
date = 2019-03-08
in_search_index = true
[taxonomies]
categories = ["blog", "rust", "clap", "programming"]
tags = ["rust", "clap", "structopt"]
+++

I want to put out some notes on why clap v3 is taking so long. I'd also like to spell out what exactly I've been doing over the past few months and why there hasn't appeared to be much progress publicly.

<!-- more --> 

I want to put out some notes on why clap v3 is taking so long. I'd also like to spell out what exactly I've been doing over the past few months and why there hasn't appeared to be much progress publicly.

With `clap` v3 there have been two primary issues preventing a speedy release. 

The first has been my limited time available to dedicate to `clap`. Late last year I switched jobs, moved houses, and had a baby. Oh and holiday season. The new job has been much more demanding of my time than my previous gig, which means less time for open source. I've considered trying to get some corporate "sponsorship" (i.e. just work letting me work on open source on the clock), but at the current job that isn't really an option. And let's be honest, there aren't many corporations in need to paying someone to work on an argument parser and related crates :) 

Since there's not a ton I can do about my time, we'll leave the discussion to more interesting technical challenges.

`clap` has been pretty widely used, which means I need to be certain when proposing major changes that existing users are not unnecessarily burdened due to my lack of foresight. I've also wanted to increase the maintainability of `clap` so that taking on new contributors can help with the time issues noted above.

In the next series of posts I'm going to detail some of the technical challenges that I've been facing, finding solutions to, and testing. Almost all of which has been behind the scenes. If anything, I apologize for not being more public about the work. As you may see in these posts, some of the challenges are multi-hour or multi-day software challenges. And when you've got about 30 minutes at a time, it's difficult to get anything meaningful done, with context switches and all.

But let's dive in!

# Issue: Importing Structopt
 
Early on, I wanted to incorporate `structopt` into `clap` proper. This has been one of the biggest design goals in v3. Many people have expressed interest in be able to easily go from a command line list of arguments (an ARGV) to a custom user defined “context struct” for their application. This is something `structopt` did beautifully via Custom Derive macros.

```rust
#[derive(Structopt)]
struct Context {
    verbose: bool,
    file: String
}

let ctx = Context::from_args();
println!(“Was verbose used?: {:?}”, ctx.verbose);
```

In order to incorporate `structopt` into `clap` there were a few things I'd need to change or address. I came up with the following design requirements:

* Use of custom derive macros and features provided by `structopt` should feel consistent with `clap` naming. It shouldn't feel "bolted on" but integral. Providing an overall cohesive feel with the rest of `clap` is extremely important.
* Current users of `structopt` should experience as little friction as possible when migrating to `clap` v3 Custom Derive
* Using the custom derive should be optional

Last summer I participated in Increasing Rust Reach, and was paired with Alan K. (@savish) who worked on porting `structopt` into `clap`. This was a tremendous help, and we were able to help each other through the experience.

I'll now go through those requirements and what had to be done for them.

## Challenge: Consistent and Cohesive

`structopt` provided essentially two methods and one function on one’s context struct. The `structopt` method `clap` would generate a `clap::App` struct which could then perform the parsing normally, which returns a `clap::ArgMatches` struct. `structopt` then provides the `from_clap` method to take the `clap::ArgMatches` struct and turn that into your custom context struct. Finally it provides the `from_args` function on ones context struct which does the round trip (generates `clap::App`, parses, takes the `clap::ArgMatches` and turns the context struct).

However, I wanted to be able to do any of the following and keep naming consistent/intuitive:

* Generate a `clap::App` from a Context struct definition (`fn into_app() -> clap::App`)
* Turn a `clap::App` struct into a context struct (`fn from_app(&clap::App) -> Self`)
* A convenience method analogous to `structopt::from_args` which does the full round trip (`fn parse() -> Self`)
* A convenience method which also allows specifying the ARGV, which `clap` natively supports (`fn parse_from(impl Iterator) -> Self`)
* The ability to go from `clap::ArgMatches` into the context struct `fn from_argmatches(&ArgMatches) -> Self`)
* `try_*` versions for all of the above which return `Result`s instead of `panic!`ing

With these in place as building blocks, one could then form any arbitrary chain to get to/from a `clap::App` and into your own context struct. This makes upgrading significantly easier. It also makes the entire API more flexible.

@savish and I then broke that setup up into three distinct Traits instead of a single `Structopt` trait. We created `IntoApp`, `FromArgMatches` and `Clap` which just combines the previous two. This allows one to only bring in the functionality they need/want to support their application.

## Challenge: Low Friction Migration

Because `structopt` is already used in many places, I didn’t want the migration to `clap` v3 to pose unnecessary challenges. This meant not drastically changing the format of the custom derive attributes people were already using.

This was straightforward overall. The hardest part about migration for existing users of `structopt` *should* be simply renaming the methods (`from_args` -> `parse`) and macro attribute name (`structopt`->`clap`).

## Challenge: Using Custom Derive should be Optional

I didn’t want all users of `clap` to have to use custom derive. While it’s super handy, sometimes you just don’t need it. 

Solving this challenge really just mean spinning off this functionality into a `clap_derive` crate which was easy enough as the code was already coming from an external crate anyways.

## Status

This works in `v3-alpha1` already. For the most part, you can simply Find->Replace `structopt` with `clap` as well as change `from_args`->`parse` and things should magically work. Of course, there are probably some bugs left as in any alpha.

There is actually additional work going into `clap_derive` which allows you define things like argument possible value lists via an enum using custom derive as well. However, that’s outside of what `structopt` provides. I’ll speak to this in a later challenge.


## Next

I began writing thing on a plane ride, and it turned out far longer than I meant for it to. So I'll be realeasing each "issue" as an individual post in a series. In the next edition I'll be talking about Removing Strings from `clap` 

Stay tuned!
