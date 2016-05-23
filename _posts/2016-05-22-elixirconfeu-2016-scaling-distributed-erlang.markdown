---
layout: post
title:  "ElixirConfEU 2016 talks: scaling distributed Erlang"
date:   2016-05-22 18:13:34 +0200
---

I was lucky enough to attend ElixirConfEU 2016. There were loads of great
talks! I'm going to try my hand at writing up my impressions of a few. To
start, I'll focus on one that could have a significant impact on all the Erlang
VM languages, Elixir included.

# Aside: the event

The organizers did an excellent job. Both sides of my experience (attendee,
speaker) were wonderful. So, ElixirConfEU organizers: thank you
for a fantastic time!

My only disappointment is that I didn't manage my goal of connecting with
a wider range of people: attending with a cohort of $work colleagues makes it
all too easy to hang out in familiar social territory, and I should have worked
a bit harder to connect with more Elixir folks. Oh well -- catch you on IRC :)

As an added bonus, evening strolls on the tarmac at Tempelhof were _great_.

# Talks I loved, part one: [Scaling Distributed Erlang](http://www.elixirconf.eu/elixirconf2016/zandra-norman)

#### Preface: Distributed Erlang

For folks who don't know, Erlang VM languages primarily orchestrate programs
using message passing -- similar to remote procedure calls (RPC) on other
platforms, but between different pieces of code on the same host. The amazing
bit is that by writing our code using RPC _locally_, we can transform our
program to run in a _distributed_ way with minimal adjustment.

_Woah!_

Not woah-ing? Just think about rewriting your Perl/Python/Ruby/etc. app to split
functionality (or have redundancy) across two servers, and you'll have an idea
of how neat this is. No futzing with sockets, no central coordinator like
ZooKeeper to potentially fail: just send a message.

What follows is my personal narrative arc on learning about the Erlang VM's
distribution capabilities. As a quick language note, when talking
about sending messages between different Erlang VMs (usually on different
computers), we usually call each VM a 'node' -- this is because 'server' is
overloaded in the Erlang world.

1. Initial discovery -- __wild enthusiasm__: this tool-set has incredible support for distributed
systems! The sky is the limit!

2. Learning about the limitations -- __rejection__: this was designed for
telephone hardware connected by physical backplanes! It doesn't scale beyond 50
nodes! Can't be used outside of a local network! I work on systems
with lots of computers in multiple data centers, and I can't use this!

3. Watching [some](https://www.youtube.com/watch?v=tW49z8HqsNw)
[talks](https://www.youtube.com/watch?v=c12cYAUTXXs) from WhatsApp engineers
-- __acceptance__: ok, it isn't magic, but it _does_ provide a powerful building
	block for real systems: small-ish Erlang clusters as independent components
	in the architecture, where communication between clusters is handled like
	any other platform (RPC, protobuf, JSON API, etc.).  WhatsApp knows a thing
	or two about scaling, after all...

4. After "Scaling Distributed Erlang" -- __hope__

#### Limitations?

The basic arrangement in a distributed Erlang cluster is a full-mesh network;
every node connected to every other. As a result, cluster sizes are typically
limited to somewhere around 50: in this area (depending on hardware,
network, user code, and so on) the number of messages being sent through the
cluster just to keep it functioning starts to overwhelm a node's ability to do
real work. In other words, heartbeats start contesting with RPCs for
VM time and bandwidth, which results in a flaky cluster.

While there's a huge world of useful services you can run on a cluster of
20-40 modern server blades, and advanced tricks[^1] exist to scale beyond that, this
puts a bit of a damper on the most naive reading of the "massive
scalability!" excitement that sometimes surrounds the Erlang VM.

The Erlang VM _is_ highly scalable, and the built-in distribution _is_ cool,
but -- like most technology -- the design involves some trade-offs, and the
design of distributed Erlang focuses more on reliability (as in "make sure my
system still runs after a server fails") than on large[^2] scale distributed
systems.

#### The Talk

Fortunately, this talk was all about how the OTP team is exploring ways to
move beyond those limits! The key idea had two parts:

1. garbage collect inactive connections
2. change the connection topology from a fully-connected mesh network to
a sparse network using a [Kademlia](https://en.wikipedia.org/wiki/Kademlia)
distributed hash table to discover other cluster members

#### Kademlia?

I couldn't quite keep up with Zandra's description of the algorithm, but my
basic understanding is:

* Nodes are arranged into a trie-ish binary prefix tree, where the prefixes are pieces of the binary representation of the node's
numeric ID, and nodes are the leaves.

* Each node maintains a small list of specially selected peers throughout the
	tree structure: each node has at least one peer in each sub-tree.

* When looking up another cluster member to send a message, you can do some XOR
	magic to figure out who to contact to get you closer to your goal
	recipient.

* The process of finding another cluster member will happen in `O(log(n))`
	steps.

`O(log(n))` is (of course) slower than the `O(1)` lookup in a fully connected
mesh, _but_ it means that we don't pay the price of a quadratically-growing
number of connections maintaining the cluster as it grows.

That's pretty neat, but there's one more important piece: if Kademlia tells us
how to find and connect to other nodes, we need to make sure that those
connections eventually go away, or else we'll end up back with
a fully-connected mesh of mish-mash. This is where connection garbage
collection kicks in -- once the VM determines that a connection between two
non-peer (that is, you had to query the distributed hash table to discover the
other end) nodes is inactive, it will close it: this preserves the sparse
connection topology, and avoids the massive flood of heartbeats you get with
a full mesh network.

#### Why is this neat?

Remember the ~50 node limit on distributed Erlang? Well, this work
addresses one of the main culprits: the fully connected mesh.

If nodes are no longer arranged in a full mesh, we might be able to have Erlang
clusters that look very different: consider a cluster of $hundreds of tiny VMs
rather than $dozens of 128gb/24 core physical servers.

How many nodes are we talking about? Well, Zandra mentioned that she's been
testing with clusters of ~350 nodes, but that the approach should be able to
grow to much larger clusters than that. Cool!

#### Is there a catch?

I have a vague notion that using a distributed hash table in place of the full
mesh might trickle down to the developer in the form of different behaviors
under partition, different guarantees, or just a different mental model to work
with, but we'll have to see how it turns out. Full mesh is a really simple
model to reason about, while Kademlia has a great deal more nuance.

That said, I'm confident that the OTP team + the Erlang VM community will
arrive at a good set of design decisions. Here's to scalable distributed
Erlang!

#### Footnotes
[^1]: see chapter 13 of _Designing for Scalability with Erlang and OTP_. In short: you can federate your Erlang clusters of 20-50 in various ways (message bus, Dynamo-style ring, service discovery), or connect VMs to the cluster that do not participate in the full mesh: these are called "hidden nodes".
[^2]: Where "large" means "hundreds to thousands of computers in a logical cluster, working together". Think Hadoop at a $bigcorp, like [Yahoo and the ~4500 server cluster](https://wiki.apache.org/hadoop/PoweredBy#Y).
