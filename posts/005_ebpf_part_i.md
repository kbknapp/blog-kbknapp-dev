+++
title = "eBPF and Rust (Part 1)"
slug = "ebpf-part-i"
draft = false
template = "page.html"
date = 2020-12-13
in_search_index = true
[taxonomies]
categories = ["programming", "bpf", "linux", "blog"]
tags = ["linux", "bpf", "networking"]
+++

This is a multi-part series on my adventure into eBPF with Rust.

Part 1 details the problem set encountered that led to researching eBPF and what
we actually being trying to solve as our example problem throughout the series.

<!-- more -->

Writing eBPF code is somewhat new and not for the feint of heart. It's typically
done in a low level subset of C with a myriad of gotchas and edge cases. If you
follow me, you'll probably guess Rust is my goto language, but writing eBPF code
in Rust is even trickier. Luckily, a collection of crates exist to make this job
easier, but in using them I found a few areas that could use some improvement.
This series goes over the where the ecosystem stands right now, and ultimately
the details of changes I'm proposing to make it even better.

I believe Rust can be the goto eBPF language with a little work.

## Series Overview

- Part 1: we lay out the problem set
- [Part 2](../ebpf-part-ii/index.html): we dive into eBPF in a general sense to
  familiarize ourselves with the technology before reaching out to Rust
- [Part 3](../ebpf-part-iii/index.html): we look at current state of eBPF
  networking in Rust and what it would look like to solve our Part 1 problem
- [Part 4](../ebpf-part-iv/index.html): We do a deep dive into the current networking implementation of
  `redbpf-probes` and note areas we want to improve
- Part 5 (Coming Soon): We propose ideas for improving the `redbpf-probes`
  networking modules

## The Why

At `$DAYJOB` we've been experimenting with a new(ish) Linux kernel technology
called eBPF. Before I dive into the specifics of eBPF, let me explain the simple
use case we had so that we can keep it in the back of our mind as a concrete
example.

We have a set of servers that accept traffic on a particular port and then
aggregate and display this data to the users. This traffic was coming from a
single source and contained data about a single entity.

![fig. 01](../imgs/ebpf_01.png)

The server application was designed to handle multiple external sources, where
each source comes in over a different port.

![fig. 02](../imgs/ebpf_02.png)

{% fk(after=true, emoji="(´ ͡༎ຶ ͜ʖ ͡༎ຶ )") %}

Before we go any further I should (I want to) explain that we did not design or
build this server application or architecture. We don't have control over how
or why it operates this way. 

{% end %}

One day we get notified that we will be receiving multiple entities from a
single external source over a single port. This is a problem for the server
application which has no knowledge of the "entities" and only differentiates
ports.

![fig. 03](../imgs/ebpf_03.png)

The boss says we need a solution to trick the server application into thinking
all these entities came from different ports. Why can't we just actually use
different ports, you ask? As mentioned, we don't control those external sources.

{% fk(after=true, emoji="ᕕ(⌐■_■)ᕗ ♪♬") %}

...hence the "external"

{% end %}

Luckily, the packets as the come across the wire have byte tags in the payload
that differentiate entities.

I can hear you now, "Easy! Just use `iptables` with `-m string` to match the
bytes and clone the packets with `-j TEE` to another port via the `mangle`
table!"

Slow down buckaroo. It turns out I didn't give you all the details in order to
keep this simple enough for a blog series. We also have some strict validation
requirements along with this new multiple-entities-from-a-single-source
"upgrade." For the sake of this series we'll just use a narrow window for
payload size narrow payload size (in addition to this entity:port mapping).

Just to be clear, we're in a situation where we don't control the server
application, don't control the source of the packets, are getting new
requirements piled on, and are expected to produce some magic to keep things
operating as if nothing changed.

The solution should:

- Validate the packet
- Inspect packet payload to determine entity
- Apply some kind of mangle with an entity:port mapping
- Ideally we'd like something that is easily extendable to meet any surprise
follow on requirements as well.

Oh, and we could be receiving a high volume of these packets so performance is
key (i.e. a userspace implementation will probably not fly).

![fig. 04](../imgs/ebpf_04.png)

So there are probably many ways to solve this particular issue. 

## **eBPF has entered the chat**

Having heard of eBPF and specifically XDP, I wanted to dig in and see how
difficult this would be for us to implement. In theory, it'd be a quick program
that could be implemented in almost no time, not require any modifications to
either the server application or external source system, and not impact
performance. Win, win, win.

This did, and didn't play out.

eBPF can absolutely solve this problem well. It isn't that difficult, and is
extremely performant.

{% fk(after=true, emoji="( ͡° ͜ʖ ͡°)") %}

You feel a "but" coming on, don't you?...me to

{% end %}

BUT, it required a lot C.

{% fk(after=true, emoji="(☞ﾟ∀ﾟ)☞") %}

Here (and always) "C" also stands for confusion

(jokes, just jokes)

{% end %}

Just getting into eBPF was more of a challenge than I'd like. Because it's such
a low level technology, and there are "competing" ecosystems I found the whole
space confusing at first. Of course once I navigated the waters, everything
makes sense and I can see why things are the way they are. Call it survivors
bias, perhaps.

So the teaser is that it works, but that getting into the ecosystem was
difficult. And even though the C wasn't too bad, I'd *much* rather be writing
Rust. As it turns out, there is quite an excellent set of eBPF crates for Rust.

But this is still part 1, so lets first back up and go over the high level wave
tops of what eBPF actually is.

Discussed this post on [DEV.to](https://dev.to/kbknapp/ebpf-networking-in-rust-3nee)!
