+++
title = "Configuration Tetris"
slug = "config-tetris"
draft = false
template = "page.html"
date = 2024-02-03
in_search_index = true
[taxonomies]
categories = ["blog", "programming"]
tags = ["configuration"]
+++

Thoughts on best practices when when building a configurable application.

<!-- more -->

## Background

Recently [this post titled, "Don't prefill Your config files"][post] was making
its round of the various news aggregators I follow. This topic closely tracks
the last few articles I wrote about building command line applications and
passing around an ["Application Context."][ctx]

I still plan on writing an article about loading configuration specifically for
applications written in Rust, but at least for now I wanted to jot down some
thoughts and best practices in a language agnostic manner.

## Play Field

At it's most basic form the runtime context will contain only application
defaults, i.e. the application isn't configurable at all.

There are a multitude of places an application could pull it's configuration
from, not all applications support all methods. But if we look at the most
common options, we'd see something like this for an application with six (6)
configurable options.

![Fig 01][fig_01]

<details>
<summary>Fig 01 Alternate Text</summary>
<hr>
An X by Y matrix where the X axis is the configuration options denoted by empty
cells and the Y axis lists the configuration options of (from bottom to top):

* Application Defaults
* System-wide configuration
* User configuration
* Project configuration
* Environment Variables
* Command line arguments

Where options 3-5 are noted as "configuration files."

Cells for "Application defaults" are filled in with a light blue.

Below the matrix is the "Runtime Context" with cells filled with light blue
indicating values taken from the Application Defaults row.
<hr>
</details>

> **NOTE**
> In the image above the "filled in" (hash checked) boxes are "values"
> provided.

A runtime context for a configurable option will *always* have a value, even if
it's just an application default and that default is "no value" as expressed in
whatever language happens to be used.

## Rules

Notice the order of the Y axis is important. Higher rows will override values
in lower rows.

The lower the row the more "implicit" the value is considered. Likewise, the
higher the row the more "explicit" the value is considered because it is
increasingly likely to have been set by the user directly and for this
particular run of the program.

## Falling Blocks

If we were to "fill in" various cells indidcating that the user or system
administrator changed some default to a new value we may end up with something
that looks like this:

![Fig 02][fig_02]

<details>

<summary>Fig 02 Alternate Text</summary>
<hr>
The same X by Y matrix as in Figure 1 except various cells are "filled in" with
various colors (one color per row) indicating the user supplied a value.

Below the matrix the "Runtime Context" cells are filled with the color of the
"highest" supplied value.
<hr>
</details>

Notice how the final "Runtime Context" contains the "value" of the highest
(most explicit) value.

That's really all there is to it. More explicit values should override implicit
values.

[post]: https://www.makeworld.space/2024/02/no_prefill_config.html
[ctx]: https://kbknapp.dev/cli-structure-01/#context-structs
[fig_01]: ../imgs/configs_01_nobglm.png
[fig_02]: ../imgs/configs_02_nobglm.png
