+++
title = "Network Coordinates and Vivaldi"
slug = "network-coordinates"
draft = false
template = "page.html"
date = 2022-11-26
in_search_index = true
[taxonomies]
categories = ["network", "rust", "vivaldi", "performance", "blog"]
tags = ["network", "performance", "rust"]
+++

Imagine you have a distributed system and need to answer something like, "Which
node is closest to Node X?" or perhaps "Which N nodes are closest to Node Y?"

It turns out answering this question naively is _wildly_ wasteful.

We'll dig in to a popular Network Coordinates system and how to use it
correctly below.

<!-- more -->

# Intro

As an example of the kinds of differences we're talking about, a naive solution
to this question for a system of 1,000 nodes (where a "node" could be a host, a
VM, or even a container) could require between 1GB to 240GB+ network bandwidth
per day depending on how accurate (or the frequency of) one would like the
results to be. Whereas the solution we'll be discussing here would require 272KB
to 65MB for the same "accuracy" levels.

This more efficient system is known as Network Coordinates. But these
coordinates also need to be used properly or you effectively have a fancy
version of the naive solution.

This problem has come up rather frequently for me, and it's been surprising to
me how often the naive solution gets picked.

When initially presented with this problem many people I've talked to decide to
simply conduct a few latency checks (such as a `ping` or ICMP message) and
store a map of the results at each node. Done. Move on the next problem.

We've already shown how wasteful this could be, but let me expand on it just to
prove my case.

# Lets Math

To demonstrate how this naive solution adds up, let's do some quick
back-of-napkin-math for those 1,000 nodes. First I'll mention 1,000 nodes is a
decent sized network yet still entirely possible. If we were to say a "node"
could be a container then the 1,000 number becomes not only feasible, but
entirely likely even for small to medium distributed systems.

## Network Requirements

A `ping` (ICMP message) is an 8 byte header plus some payload size. On Linux
the packet size is 64 bytes by default, although we also need to include the IP
and Ethernet header size as well which adds another 40 bytes (assuming IPv4).
Keep in mind many real world scenarios use additional encapsulation such as
tunnels or application layer messages that add more overhead. But let's assume
the simple scenario of ~100 bytes per message, or 200 bytes network traffic
round trip.

![Fig. 01](../imgs/net-coords-001.svg)

At this rate, a single node conducting a latency check against all other 999
nodes would require about 200KB network traffic round trip. This doesn't sound
too bad yet, is probably why this solution gets picked so often. 

However, all other nodes still need to do the same check.

![Fig. 02](../imgs/net-coords-002.svg)

Lets further assume symmetrical results, meaning if `A` does a check with `B`
then `B` doesn't need to do a check with `A` and just reuse `A`'s result.

> **Note**
> Symmetrical results is close, but not actually that realistic for real world
> network latencies as packets can take different routes.

With this change our total isn't `2KB * 1,000^2` but instead `2KB * (1,000 +
999 + 998 ... 1)` which is slightly better. Although, even this equals just
under 1GB of traffic round trip for a single check.

Because we're talking network latencies which are very unstable and subject to
change over time as well as jitter a single latency check isn't very
representative of actual network latencies. 

It's common to do several latency checks and take an average, say five checks
(on the command line something like `ping -c5`). And then one would repeat this
test over time, such as every 30 minutes.

At this rate (five ICMP messages every 30 minutes), we're in the range of 5GB
of network traffic every 30 minutes, or 240GB a day just to be able to answer
the question, "Which nodes are the closest."

## Storage Requirements

There's also storage considerations as well, which are admittedly less
concerning, however let's briefly address it so we can compare to the Network
Coordinate solution later. 

If we assume the "results" of this latency check are an integer, say four
bytes, representing the average number of milliseconds round trip latency and a
public IPv4 address as the node ID (another 4 bytes) it's around 8KB to store
the entire map. This map is duplicated per node so around 8MB in total storage. 

Granted, a 4 byte result and 4 byte ID is about as small as it could be and
real world requirements are probably much, much higher doing things such as
keeping a history of results, or at least a sliding window, and using longer
node IDs.

# Network Coordinates

So how would these Network Coordinates work, and why are they so much better?

## Vivaldi

The system we're discussing here is based on the [Vivaldi Network
Coordinates](https://pdos.csail.mit.edu/papers/vivaldi:sigcomm/paper.pdf) (PDF). Network coordinates function much like physical
coordinates on a map. Two locations have some coordinate that represents their
location.

In physical space it's relatively simple to take two coordinates and determine
how "close" the points are. Or back to our original problem, trivial to take a
list of coordinates and determine which of those are the closest to any other
given coordinate.

Network coordinates are the same, except that they're trying to model a
"location" not in physical space but within the network. Unlike physical space,
the distance between two "locations" can fluctuate in network space because
packets can take different routes, links could go down or become degraded, and
we must account for jitter and spikes.

Vivaldi takes the approach of giving each node a random euclidean coordinate
using an N-dimensional coordinate. In physical space we may use a 2D or 3D
coordinate, but in Vivaldi you're not limited physical dimensions. In fact,
some research has gone in to picking an optimal number of dimensions for these
coordinates. The more dimensions the more network traffic and storage
requirements, but theoretically the more accurate as well. 

That "optimal" number appears to be around 8D, although in my anecdotal testing
even 6D or 4D coordinates seem to be accurate enough for many scenarios.

Let me briefly describe how Vivaldi works at a high level and what the code is
doing under the hood.

### How Does it Work?

I said that each node starts with a randomly assigned coordinate. Each node
will then conduct round trip latency checks and adjust their coordinate based
off observed latency. The goal is for each node to adjust (or move) to a
coordinate that accurately reflects it's position.

> **Note**
> "Each node conducts latency checks..." This sounds just as bad our naive
> solution! It's not I promise, this will become clear shortly.

The way I think about this is for example in a 2D map, with two nodes, the goal
is for each node to enter any coordinate where the calculated latency between
the two nodes matches (or is within a margin of error) of the observed latency.

Let's assume nodes `A` and `B` are physically 5ms round trip latency from one
another. We start by giving them random coordinates (remember we're using 2D
just to make this easy to visualize). I've added a dashed circle to represent
any coordinate that give a correct distance estimate when comparing the two
coordinates.

![Fig. 03](../imgs/net-coords-003.svg)

> **NOTE:**
> I've deliberately left out the actual coordinate numbers (i.e. `(2,4)`, etc.)
> because the coordinates themselves are irrelevant and meaningless. The only
> meaning comes from comparing two sets within the same system.

Now `A` and `B` do a latency check, observe real world latency and adjust their
coordinates.

![Fig. 04](../imgs/net-coords-004.svg)

Node `B` then does the same thing

![Fig. 05](../imgs/net-coords-005.svg)

At this point the coordinates have been updated enough that a distance estimate
between them is accurate with observed real world latency.

Like I said in the note above, this doesn't look much better than the naive
solution. And with only two nodes it's not.

If we were to add a third node, let's see what happens. We'll repeat the above,
except let's assume this new Node `C` is also doing checks with `A`, but has a
real world latency of 10ms to `A`.

![Fig. 06](../imgs/net-coords-006.svg)

Notice all nodes are now accurate with observed latency. 

But wait! What is the distance between `B` and `C`? For the sake of this post,
and my diagramming skill, lets assume it's also 10ms.

But then that means the coordinates aren't yet accurate! We need to do some
more checking!

![Fig. 07](../imgs/net-coords-007.svg)

In the diagram a single check made everything accurate again. However, a real
world update to `C` could have well made it accurate for `B` but now `A` is not
accurate any longer so it needs to adjust, etc. etc.

> **NOTE:** 
> Additionally, the direction of movement is calculated from the position of
> the remote, which the diagram doesn't accurately reflect because it would
> have made the sequence more confusing as stabilization can take many rounds.

### Error Estimates

Additionally, Vivaldi gives a notion of an "error estimate" or how sure a node
is that it's coordinates are accurate. At first, because the coordinates are
random the error rate will be very high. 

When doing coordinate updates, if the remote node's error rate is high (it's
not sure of it's accuracy) and the local rate is low (you're very sure of your
own accuracy) you won't move much. But the inverse (you're not sure at all of
your accuracy, but the remote is very sure) will result in your local
coordinate updating a lot.

This allows the coordinates to converge quickly on known good points, but then
slowly once the results become more accurate.

### Performance

At this point it's looking like all nodes just need to do actual latency checks
with all other nodes to have any shot at being accurate. This seems just as
bad, if not worse than our naive solution! As we're doing latency checks
anyways, why not just use those real latencies!

This is what I alluded to at the beginning of the post about having a fancy
version of the naive solution. Because all nodes start with random coordinates
they need to conduct updates with more "stable" nodes (low error estimates) to
converge on accurate results. Additionally, in large networks if the coordinate
updates aren't conducted against shared stable nodes, you end up with
"islands."

For example, assume a network of six nodes (`A..F`) where `A` and `B` are
stable accurate coordinates. If `C` and `D` only update against `A` and `E` and
`F` only update against `B` you'll end up with two islands of coordinates.

![Fig. 08](../imgs/net-coords-008.svg)

This can happen accidentally by relying on "random updates" where all nodes
start random, and conduct updates against other nodes randomly or naturally as
a side channel of regular communication. Some nodes will stabilize, while other
nodes will not.

To make matters worse, even if one of the nodes happens to conduct an observed
check against another stable island (say `C` against `B` in our example, or
even `C` against `F` once it's stabilized), that does not actually help `D`, or
`E` at all unless they also start updating against other stable islands/nodes.

As it turns out, the solution to this problem and the solution to orders of
magnitude better performance is the same. Use a single __Origin Node__, which
all nodes conduct their observed latencies checks with. For example, if `A`
became the only origin node.

![Fig. 09](../imgs/net-coords-009.svg)

This does not preclude a node from checking against another node, but it's not
required.

# Lets Math Again

We'll assume 8D coordinates, which is effectively a `[f64; 8]` in Rust plus a
few `f64`'s of metadata like error estimates and jitter smoothing, in total 88
bytes per Vivaldi message as a payload, and we're free to use whichever
protocol best serves our application, say UDP which like ICMP is 8 bytes. We
also have our original overhead of 40 bytes for the IPv4 and Ethernet header.

We add an origin node, so 1,001 nodes, and a single coordinate update for _all
nodes_ is about 272KB of traffic. Let's be fair and say we did five updates per
node, so total network traffic for all nodes and those five updates is 1.36MB.
Even if we did this every 30 minutes, that's only 65.28MB per day (versus our
240GB+ in the naive solution).

# Real Power

However, it's not just more efficient by network traffic standards. The _real
power_ comes from any two nodes being able to estimate accurately their latency
_without ever having observed any latency check between one-another!_ 

In the naive solution the only way to know what the latency between `E` and `C`
was, was to check it (over and over and over). However, in this new system `E`
and `C` need not ever have sent a single packet to each-other! We can just look
at their two coordinates and perform a quick calculation.

If the Origin stores all current coordinates (or even gossips them around to
various local caches) we can just ask, "what are the N closest nodes to
coordinate X?" at which point the origin can just zip through it's list of
coordinates doing these calculations and give back a sorted list if desired!

But how fast are these calculations?

## Speed

In my Rust Vivaldi implementation (called [`violin`](https://github.com/kbknapp/violin)), a single
calculation of two 8D coordinates using stack memory on my 8 core AMD Ryzen 7
5850U laptop with 16GB RAM takes ~16.5ns. A benchmark of 1,000,000 calculations
takes 16.5ms.

# Summary

I hope this post has given you some insight into how to efficiently get answers
to questions like "network closeness." Using a Vivaldi based solution
(_correctly_!) can vastly improve distributed systems performance.

If you're into Rust, check out [`violin`](https://github.com/kbknapp/violin)!
As a quick sample, this is what it looks like to use:

```rust
use std::time::Duration;
use violin::{heapless::VecD, Coord, Node};

// Create two nodes and an "origin" coordinate, all using a 4-Dimensional
// coordinate. `VecD` is a dimensional vector.
let origin = Coord::<VecD<4>>::rand();
let mut a = Node::<VecD<4>>::rand();
let mut b = Node::<VecD<4>>::rand();

// **conduct some latency measurement from a to origin**
// let's assume we observed a value of `0.2` seconds...
//
// **conduct some latency measurement from b to origin**
// let's assume we observed a value of `0.03` seconds...

a.update(Duration::from_secs_f64(0.2), &origin);
b.update(Duration::from_secs_f64(0.03), &origin);

// Estimate from a to b even though we never measured them directly
println!("a's estimate to b: {:.2}ms", a.distance_to(&b.coordinate()).as_millis());
```

As a bonus `violin` also works in `no_std` and no `alloc` environments (but
with some performance penalties due to lack of platform intrinsics).
