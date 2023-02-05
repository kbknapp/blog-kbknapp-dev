
+++
title = "Diving into Rust with a CLI"
slug = "rust-cli"
draft = false
template = "page.html"
date = 2020-05-28
in_search_index = true
[taxonomies]
categories = ["programming", "rust", "blog", "clap"]
tags = ["rust", "cli", "clap"]
+++

A blog post titled, ["Diving into Go by Building a CLI
Application"](https://eryb.space/2020/05/27/diving-into-go-by-building-a-cli-application.html)
has been making it's rounds of the internet. It uses a small
[XKCD](https://xkcd.com/) downloader as the subject. I thought was small and
self contained enough, that it'd be interesting to see the same example in Rust!

<!-- more --> 

**Updates: Updated on 2021-10-09 to clap v3.0-beta.4**

**NOTE:** This article assumes the reader is at least mildly familiar with Rust.

This will be an *almost* one to one comparison, however there are slight
differences that I will call out as we go along.

We'll call this `grab-xkcd` since we're dropping the `go`.

... ( •_•)>⌐■-■   (⌐■_■) ...

```
$ cargo new grab-xkcd
```

## Overview 

The basic structure will be this:

* A `main()` function will drive the application, and exit with any error messages
* An `Args` struct will hold and control our CLI
* A `XkcdClient` struct will hold our logic for making requests and running 
  the application
* A `Comic` struct will hold our representation of a single comic
* A `ComicResponse` will represent the JSON returned by the XKCD API

## CLI

With those components in mind, I like to start all CLI applications by building,
or at least stubbing the CLI. This allows me to get a feel for what it's like to
use the tool, and often leads to small changes for a better user experience
(UX). Granted, we're talking about the CLI here, so UX is on a relative scale,
but there is nothing that says a CLI *has* to be terrible!

As per the other article, we'll accept four arguments:

* `--number`: which allows picking a certain comic, or defaults to `0` which the
 XKCD API defines as the most recent comic
* `--save`: to fetch and save the actual comic image to our current directory
* `--output`: to allow selecting between JSON or Text representations of our
 `Comic` (defaulting to `text`)
* `--timeout`: which allows customizing the request timeout, defaulting to 30 seconds

We can do this all in just a few lines of Rust via the
[`clap`](https://github.com/clap-rs/clap) crate. For those familiar with the
[`structopt`](https://github.com/TeXitoi/structopt) crate, which allows one to
define a Rust struct that contains all the CLI logic, as of `clap` 3.0 that code
has been merged together. We will use the `3.0.0-beta.4` release of `clap` to
demonstrate.

We'll use [`cargo-edit`](https://github.com/killercup/cargo-edit) to add our
dependencies:

```
$ cargo add clap --allow-prerelease
    Updating 'https://github.com/rust-lang/crates.io-index' index
      Adding clap v3.0.0-beta.4 to dependencies
```

Here is our CLI:

```rust
use clap::{ArgEnum, Clap};

/// A utility to grab XKCD comics
#[derive(Clap)]
pub struct Args {
    /// Set a connection timeout
    #[clap(long, short, default_value = "30")]
    pub timeout: u64,
    /// Print output in a format
    #[clap(long, short, arg_enum, default_value = "text")]
    pub output: OutFormat,
    /// The comic to load
    #[clap(long, short, default_value = "0")]
    pub num: usize,
    /// Save image file to current directory
    #[clap(long, short)]
    pub save: bool,
}

#[derive(ArgEnum, Copy, Clone)]
pub enum OutFormat {
    Json,
    Text,
}
```

I'll do a quick walk through of highlights to explain a few differences from 
the first article.

* `#[derive(Clap)` is where all the magic happens, it tells clap to use the
  struct `Args` as the CLI.
* `/// ...` is a documentation comment, which gets translated into the "about"
  section of a CLI or argument
* `long, short` tells clap to create a long `--foo` and short `-f` switch
  automatically based off the field name.
* `arg_enum` tells clap to use the field's enum as the value variants.
* `#[derive(ArgEnum)` this allows clap to use the enum variants as values
  for a particular argument. It does things like auto-derive
  `std::str::FromStr` and handles ASCII upper/lowercase differences as
  well as comparing the supplied value at runtime to the one of the
  variants.

Notice we included an enum `OutFormat` with two variants. This allows us to
limit the possible values provided on the CLI to known values. It also allows us
to not have to use `String`s to represent it, reducing errors and typos!

If we provide some value not expected, `clap` informs the user an exits:

```
$ grab-xkcd --output yaml
error: 'yaml' isn't a valid value for '--output <FMT>'
	[possible values: json, text]

USAGE:
    grab-xkcd --num <num> --output <FMT> --timeout <timeout>

For more information try --help
```

Nice!

Ok, so the CLI is pretty much complete. Let's drive our `main()` function with
what we've got so far:

```rust
fn main() {
    let args = Args::parse();
    // ... todo
}
```

Since `clap` handles the errors on invalid CLI use, we don't need to worry about
any sort of validation or exiting on errors.

**NOTE:** `clap` *does* provide the ability to handle errors manually. But for
this quick example it's out of scope.

Let's also add some of those constants from the other article:

```rust
const BASE_URL: &str = "https://xkcd.com";
const LATEST_COMIC: usize = 0;
```

As noted comments from the other article, `0` represents the latest comic
according to the API.

## `XkcdClient`

Alright, now we can start on the client to actually fetch the data from the
server.

We'll use this client to hold the logic for our application, that means it needs
to do a few things:

* Make a GET request against the XKCD API
* Parse the response JSON into a `Comic`
* Print the `Comic` in the format specified by `--output`
* If the user requested to save the image:
	* Fetch the image from the server via another HTTP GET request
	* Write the data to a file
* Return any errors that happen along the way

Since we'll need to know a few items from our CLI, we can make our `XkcdClient`
contain a field holding our `Args` struct. And a way to instantiate a new
instance of this client:

```rust
struct XkcdClient {
    args: Args,
}

impl XkcdClient {
    fn new(args: Args) -> Self {
        XkcdClient { args }
    }
}
```

Now we can add that to our `main()` method:

```rust
main() {
    let args = Args::parse();
    let client = XkcdClient::new(args);
}
```

At this point, I usually stub out the rest of the main function:

```rust
main() {
    let args = Args::parse();
    let client = XkcdClient::new(args);
    client.run();
}
```

Hmm...but we already said errors can happen along the way, so `XkcdClient::run`
will most likely return some form of `Result`.

We have some options, either `match` the return of `run()` and print the
error/exit the process on error, or just let `main()` handle it. For this simple
example, we'll let `main()` handle it. This has the expense that some error
messages may be a little cryptic `error: OS error 1`, but small tool like this
it's fine. We can leave it as an exercise for the reader to implement real error
handling.

To let `main()` handle our errors, it needs to return a `Result` as well. Since
we'll be dealing all kinds of different errors from different crates, it's
helpful to have some convenience methods and representation for handling all
this. The [`anyhow`](https://github.com/dtolnay/anyhow) crate does just this!

```
$ cargo add anyhow
    Updating 'https://github.com/rust-lang/crates.io-index' index
      Adding anyhow v1.0.31 to dependencies
```

We can now use the `anyhow::Result` type who's `E` generic uses the `anyhow::Error`
which is a lot like a `Box<dyn std::error::Error>` under the covers.

So we update our `main()` method:

```rust
use anyhow::Result;

fn main() -> Result<()> {
    let args = Args::parse();
    let client = XkcdClient::new(args);
    client.run()
}
```

Now if our `run()` method returns an `Err(..)`, Rust will insert a pretty:

```
error: foo
```
message when it exits. This also allows us to bubble up *all* the various errors
with a simple `?` operator.

Now it looks like we should implement `XkcdClient::run`, however since that will
drive the entire application, it will utilize a bunch of stuff we haven't
created yet. So let's hold off and come back to it later.

## `ComicResponse`

First, we know we'll make an HTTP GET request against the API, so lets create a
struct to hold the response:

```rust
pub struct ComicResponse {
    month: String,
    num: usize,
    link: String,
    year: String,
    news: String,
    safe_title: String,
    transcript: String,
    alt: String,
    img: String,
    title: String,
    day: String,
}
```

Not too bad. But the API will return JSON, so we'll need to marshall the data
into our struct. Luckily, the `serde` crate exists for serializing and
deserializing data with ease.

I can't stress this enough. `serde` is bar none, the best serialization library
I've used in any language. Much like the `clap` crate for CLIs (but I'm biased
about that, since I wrote `clap` ᕕ(⌐■_■)ᕗ ♪♬).

We can add some crates used for this serialization now:

```
$ cargo add serde serde_derive serde_json
    Updating 'https://github.com/rust-lang/crates.io-index' index
      Adding serde v1.0.110 to dependencies
      Adding serde_derive v1.0.110 to dependencies
      Adding serde_json v1.0.53 to dependencies
```

Why three crates?

* `serde` provides the traits
* `serde_derive` provides the procedural macros so we can just tag our structs
  with a `#[derive(..)]`
* `serde_json` uses these to convert to/from JSON strings.

So all we do is update our `ComicResponse` struct with the attribute:

```rust
#[derive(Deserialize)
struct ComicResponse {
    // .. same as before
}
```

We only care about `de`serialization (from JSON to Rust struct), but if we cared
about going the other direction we could have just as easily added `Serialize`
to the list of `#[derive]`ed traits.

## `Comic`

So we have our response, which is the Rust representation of the raw JSON, but
we only care about a few elements. Likewise, we'd like to display the date in a
more friendly format.

Let's first create our `Comic` struct to hold the data we care about:

```rust
struct Comic {
    title: String,
    num: usize,
    date: String,
    desc: String,
    img_url: String,
}
```

The most idiomatic Rust way to get from one type to another is the `From` trait.
So we can now implement the `From<ComicResponse>` trait for our `Comic` which
will give us some sweet `.into()` abilities.

```rust
impl From<ComicResponse> for Comic {
    fn from(cr: ComicResponse) -> Self {
        Comic {
            title: cr.title,
            num: cr.num,
            date: format!("{}-{}-{}", cr.day, cr.month, cr.year),
            desc: cr.alt,
            img_url: cr.img,
        }
    }
}
```

Pretty self explanitory.

Ok, so we have a `Comic` and we have a `ComicResponse` and the ability to
convert between them, what else do we need to implement `run`?

Although we `derive`d the ability to go from a JSON formatted string to a
`ComicResponse` we haven't actually *utilized* that code anywhere yet. So lets
wire that up with another `From` impl:

```rust
impl From<String> for ComicResponse {
    fn from(json: String) -> Self {
        serde_json::from_str(&json) // ...uh oh, returns a Result
    }
}
```

Hmm. Ok so `serde_json::from_str` returns a `Result` which makes sense. *Any*
string may not be valid JSON, and especially not valid for whatever struct we're
trying to create.

So we can either `expect` that the JSON string we're given is always valid, and
panic otherwise, or if we allow `From` express fallibility.

Turns out there is a version of `From` for exactly this,
`std::convert::TryFrom`!

With quick update, we're back at it:

```rust
impl TryFrom<String> for ComicResponse {
    type Error = anyhow::Error;
    fn try_from(json: String) -> Result<Self, Self::Error> {
        serde_json::from_str(&json).map_err(|e| e.into())
    }
}
```

Notice we used `anyhow::Error` as the error type, but that meant we needed to
`map` the error from the `serde` error, to the `anyhow` error. Not hard, but
something to note.

Alright! We finally have enough of the scaffolding to write `run` with some
minor stubs!

## `XkcdClient::run`

First, we know we'll be making an HTTP request so we'll need some crate to do
that for us. There are a ton of options out there. One I've used before is
[`reqwest`](https://docs.rs/reqwest/0.11.5/reqwest/) which supports both `async`
and blocking I/O. We'll be using the blocking I/O version, so let's add that
crate now:

```
$ cargo add reqwest
    Updating 'https://github.com/rust-lang/crates.io-index' index
      Adding reqwest v0.11.5 to dependencies
```

But since we'll be using the blocking I/O we need to manually edit the
`features` array.

```toml
# Cargo.toml
[dependencies]
# .. snip
reqwest = { version = "0.11.5", features = ["blocking"]}
```

Now we can write the actual method! Here goes!

```rust
impl XkcdClient {
    // ... same as above

    fn run(&self) -> Result<()> {
        let url = if let Some(n) = self.args.num {
            format!("{}/{}/info.0.json", BASE_URL, n)
        } else {
            format!("{}/info.0.json", BASE_URL)
        };
        let http_client = reqwest::blocking::ClientBuilder::new()
            .timeout(Duration::from_secs(self.args.timeout))
            .build()?;
        let resp: ComicResponse = http_client.get(&url).send()?.text()?.try_into()?;
        let comic: Comic = resp.into();
        if self.args.save {
            comic.save()?;
        }
        comic.print(self.args.output)?;
        Ok(())
    }
```

Ok, that's *a lot* to take in.

Let's go over the highlights:

* Lines 5-9: We build a request URL based on if the user requested a particular
  comic, if not we get the latest comic
* Lines 10-12: We build an HTTP client with a custom timeout based on
  `--timeout` or the default
* Line 13: We make the GET request, convert it to text (JSON), then attempt to
  convert to a `ComicResponse`
* Line 14: We convert the `ComicResponse` into a `Comic`
* Lines 15-17: If the user wants to save the image, we stub out save call
* Line 18: Prints out the `Comic` repsresentation in the format requested, or
  the default
* Line 19: Returns no errors

Notice all the `?` points! Turns out errors can happen *everywhere* but thanks
to `anyhow::{Result, Error}` we can easily convert between them. A more proper
solution would be to create our own error representation that contains the
original errors and knows how to print errors and exit appropriately. But that
is out of scope for this article.

If we didn't care about custom timeouts, lines 10-13 could have been reduced to
a single line using
[`reqwest::blocking::get`](https://docs.rs/reqwest/0.10.4/reqwest/blocking/fn.get.html).
Oh well.

We've also gone ahead and added fallibility to the `save` and `print` functions
becuase both of which could realistically fail for many reasons, so it's a safe
assumption that we'll be returning some kind of `Result`.

So let's implement those now.

## `Comic::print`

This should be a simple addition, all we need to do is `match` on the
`OutFormat` and print the `Comic` representation appropriately.

Let's stub that out:

```rust
impl Comic {
    // .. snip

    fn print(&self, of: OutFormat) -> Result<()> {
        match of {
            OutFormat::Text => println!("{}", todo!("print self as Text")),
            OutFormat::Json => println!("{}", todo!("print self as JSON")),
        }
        Ok(())
    }
}
```

**NOTE:** The signature of `print` takes `OutFormat` by value, which only works
because we `#[derive]`d `Copy` and it's a small struct that fits into a register 
so this is fine. If we had larger variants, that caused `OutFormat` to be a
large enum, we could have taken it by reference `&OutFormat` instead.

**NOTE:** `todo!()` is like `unimplemented!()` and will allow code to compile
but you'll receive a nice todo message if this code is reached during execution.

### `OutFormat::Text`

Ok, so first what does it mean to print the `Comic` struct as text? Well we can
implement `std::fmt::Display` for `Comic` which will allow an instance of
`Comic` to be printed using things like `println!` directly.

So if we implement `Display` for the `OutFormat::Text` variant, it would look
something like this:

```rust
impl fmt::Display for Comic {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(
            f,
            "Title: {}\n\
            Comic No: {}\n\
            Date: {}\n\
            Description: {}\n\
            Image: {}\n",
            self.title, self.num, self.date, self.desc, self.img_url
        )
    }
}
```

Now instances of `Comic` and be passed directly to `println!` and will be
formatted correct. So updating our `print` method now looks like:

```rust
impl Comic {
    // .. snip

    fn print(&self, of: OutFormat) -> Result<()> {
        match of {
            OutFormat::Text => println!("{}", self), // <-- New
            OutFormat::Json => println!("{}", todo!("print self as JSON")),
        }
        Ok(())
    }
}
```

### `OutFormat::Json`

Now for JSON!

What if we could just use `serde_json` again, but in the reverse of before (Rust 
struct to JSON)? `#[derive(Serialize)]`!

```rust
#[derive(Serialize)]
struct Comic {
    // .. snip
}
```

And we update our `print` method:

```rust
fn print(&self, of: OutFormat) -> Result<()> {
    match of {
        OutFormat::Text => println!("{}", self),
        OutFormat::Json => println!("{}", serde_json::to_string(self)?), // <-- New
    }
    Ok(())
}
```

Yep, we knew it could fail somehow! `serde_json::to_string` returns a `Result`
in case it can't turn the struct in JSON for some reason.

## `Comic::save`

It's also debatable if the logic for save should be in `Comic` or the
`XkcdClient`. So arbitrarily I picked `Comic`. In a larger application it would
become more apparent which location is more appropriate.

The logic will be similar to that of `run` where we need to:

* Determine our current working directory (which can be done with
  `std::env::current_dir`)
* Determine the image file name
* Add the two together to form a file path for saving
* Perform an HTTP GET on the image URL
* Save the bytes to a file

We already have the image URL from the earlier request, but in order to get the
file name, we'll need to split the URL parts into the final segment. There are
many ways to do this, but the `url` crate makes it pretty easy with iterators.

First we add the url crate:

```
$ cargo add url
    Updating 'https://github.com/rust-lang/crates.io-index' index
      Adding url v2.1.1 to dependencies
```

Next we will need to create an instance of `Url` from the `String`
we've been storing, and iterate it's "segments" to get the last one which will
be the file name.

So let's do that much so far:

```rust
impl Comic {
    fn save(&self) -> Result<()> {
        let url = Url::parse(&*self.img_url)?;
        let img_name = url.path_segments().unwrap().last().unwrap();
        let p = std::env::current_dir()?;
        let p = p.join(img_name);
        let mut file = std::fs::File::create(p)?;

	todo!("do HTTP GET and save the file!");
    }
}
```

There a little bit to unpack here:

* Line 3: parses the string URL into an instance of `Url` with fallibility
* Line 4: iterates the segments, and takes the last one.
* Line 5: Gets the current directory
* Line 6: Joins the current directory and the image file name to create a file
  path
* Line 7: Creates a file from the given file path for writing.

Since we already know how to perform a GET request, this next part should be a
breeze. The only added part is writing the bytes of the response to a file which
can be done with the `std::io::Read` trait, which provides the `write!` method
for writing arbitrary bytes, and the `bytes()` method of the response which
turns the response into raw bytes, which can be dereferenced into `&[u8]` that
`write!` accepts.

```rust
fn save(&self) -> Result<()> {
    use std::io::Read;

    // .. snip, same as before

    let body = reqwest::blocking::get(&self.img_url)?;
    file.write_all(&*body.bytes()?).map_err(|e| e.into())
}
```

We once again have to map the error type so it fits our `anyhow::Error`, but not
to much drama.

And we're done!

## Notes on Differences

Even though this article isn't to point out differences between Rust and Go,
it's still interesting to compare the two.

We can see that the Rust code is a little more verbose than the Go code in some
areas, while the Go code is more verbose others. This, in my opinion stems from
the Rust code being much more explicit about the errors that are possible at all
levels.

We've essentially, ignored most errors and let them bubble up to the terminal
function for a simple `error: foo` message, which we've already acknowledge may
include some errors that seem mysterious to the users. Again, if this were a
real application, we could create our own errors and utilizing (or `map`) the
ones the originating errors into more friendly messages.

The Rust code also relies on more external libraries. Reasonable people can
disagree on if this is good, bad, or a non-issue. I personally like the power
that these libraries bring, and their quality is astounding in most cases.

`clap` handles *a ton* for us, that makes some differences between the Go and
Rust version for almost no boilerplate. We've already seen what happens if we
pass an invalid option to `--output`, but consider what happens if we typo
`--timeout` and instead type `--timeotu`:

```
$ grab-xkcd --timeotu
error: Found argument '--timeotu' which wasn't expected, or isn't valid in this context

	Did you mean '--timeout'?

If you tried to supply `--timeotu` as a PATTERN use `-- --timeotu`

USAGE:
    grab-xkcd --timeout <timeout>

For more information try --help
```

We can also handle any of the following permutations of options (only using
`--output` as example):

* `-o text`
* `-otext`
* `-o=text`
* `--output=text`
* `--output text`

We can even "stack" short flags (in our case only `--save` exists), making any
of these valid as well ("stacking" `--save` with `--output`):

* `-so text`
* `-so=text`
* `-sotext`

Another note about `serde` is we could have forgone the `ComicResponse` struct
altogether and just used `Comic` to even further reduce the code. This works
because `serde` can be told to `#[serde(skip)]` fields that we don't care about.
However, I left it in to be closer to the Go version.

## Conclusion

Hopefully this article shed some light on building a tiny CLI program in Rust.

Rust is an amazing language for building CLI utilities! In fact, it's replaced
Python as my go-to language for CLIs of any size quite some time ago.

In the [next post](../shell-completions/index.html) we will add CLI Shell completions to our program!

The full code from this article can be found at
[github.com/kbknapp/grab-xkcd](https://github.com/kbknapp/grab-xkcd)

Discussed this post on [DEV.to](https://dev.to/kbknapp/diving-into-rust-with-a-cli-4gap)!
