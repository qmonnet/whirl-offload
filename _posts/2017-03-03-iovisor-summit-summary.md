---
layout: post
title:  "Summary of the 2017 IO Visor Summit"
date:   2017-03-03
author: Quentin Monnet
employer: 6wind
categories: BPF
tags: [BPF]
---

* ToC
{:toc}

The 2017 IO Visor summit took place at the Computer History Museum in Mountain
View, California, on February 27th, 2017. It is at least the second summit of
its kind, being preceded by the XDP Summit on June 23rd, 2016. I could not
attend physically to the meeting, but VMware, organizer of the event, kindly
provided remote access through WebEx, and I was able to attend from remote.

# Author's foreword

This article is a summary of what I heard and saw during the Summit. As a
disclaimer, the audio quality I experienced was not so good (and to be honest,
not being a native English speaker does not help grasping all what is said),
and could not hear everything the speakers said. In particular, I would
understand nearly nothing of the questions asked by people in the assistance.
We had access to projected slides, but no image of the room (just [the ones
from Twitter](https://twitter.com/IOVisor/status/836265618500288512)).

So of course, do not take my writings as truth, I may have misheard some stuff.
In particular, if you also attended, or even talked during the Summit, and have
corrections to provide, please do not hesitate to reach out! Other readers
might be interested to know that [the talks were recorded and should eventually
be available on the
net](https://twitter.com/IOVisor/status/836310835844689920).

The agenda of the talk [was
here](https://docs.google.com/document/d/1tw3BlY0fq9yZYVakcuoXMLW1mL-KeB-zHUJGCUw8qfk).

Many thanks to Deepa Kalani from VMware for providing remote access, and for
relaying my questions from the WebEx chat during the Summit.

# Intro, greetings

* _Pere Monclus, VP, R&D at VMware_
* _Alexei Starovoitov, Software Engineer, Facebook_

I missed most of it, because of connection issues. Happily, seeing the title I
missed nothing too technical.

From what I got from the talk, it was about the current state of IO Visor
project as a whole. From the first “PLUM“ engine launched in January 2011, the
project has evolved and now has a 6-year old community. It joined the Linux
foundation, developed a number of tools, produced many kernel patches. The
community keeps growing and works on use cases such as networking, tracing, but
also storage and security (regarding security, I think about Cilium or XDP for
DDoS protection, but I do not know what are the use cases for storage).

Most tools developed for the project are of course available at
<https://github.com/iovisor/>.

# bcc tools: Strategy, current tools, future challenges

* _Brendan Gregg, Senior Performance Architect at Netflix_

You can measure the influence of Netflix when the speaker uses a title slide _à
la_ _Stranger Things_… This talk is about the [bcc framework and set of
tools](https://github.com/iovisor/bcc/). If you read this article, you most
probably already heard of it or even played with them.

Brendan presented the current state of the tools. There is already an
impressive number of utilities, targeting all layers of the Linux system. He
explained that bcc contains both “single-purpose tools”, such as `opensnoop`
(only showing calls to `open()` syscall), as well as `multitools` such as
`trace`, that have a more complex command syntax but are able to perform a
variety of tasks.

Beyond adding new tools, the project aims at maintaining the current ones into
a coherent set:

* Output of tools is not made on a whim, but follows some well-defined
  templates, to look more familiar to Linux users (per-vent output, time
  intervals, histograms…).
* They try to use POSIX-style arguments.
* The tools have to be documented.
* They also have to be tested. Quoting the slide, “If you can't write the
  workload, you can't write the tool”.

Regarding future work, there is not much to do at the moment regarding BPF
support, since bcc already supports nearly all BPF features from the kernel.
There is still a list of issues on the GitHub repository, though. Other work
should include:

* Some more tools, including some that Brendan has not ported from DTrace yet.
* Cleaning up, maintaining existing tools (since kernel evolves).
* Working on USDT.
* Reducing overhead, in particular with uprobes.
* Maybe make it easier to develop tools (in this regard, see also
  [ply](https://wkz.github.io/ply/)).
* Work on visualizations (flame graphs etc.)

There was some question about having bcc on OS X I believe? But I could not
hear the answer. Other questions I did not understand. Personally, I asked
about plans to add new tutorials or tools related to networking aspects in bcc.
It seems that Brendan leaves it to people working on networking stuff (so, not
from him I get).

# Tracing TCP connections for Weave Scope

* _Alban Crequy, Director & Software Engineer at Kinvolk_

[Slides available on the net](https://docs.google.com/presentation/d/1HaU0zBsesm-S2umsOvVnQPVGV7sos_T6dygBj8gRIHU).

Full title was:

> Tracing TCP connections for Weave Scope — Visualization and monitoring tool
> for Docker and Kubernetes as used by Weave Works

Under this descriptive headline hides a brief introduction to Weave Scope, and
to the use case it constitutes for eBPF.

[Weave Scope](https://www.weave.works/products/weave-scope/) is an Open Source
solution for container managing and monitoring, available [on
GitHub](https://github.com/weaveworks/scope). The website says:

> Zero configuration or integration required — just launch Weave Scope and go!
> Automatically detects processes, containers, hosts. No kernel modules, no
> agents, no special libraries, no coding. Seamless integration with Kubernetes
> and AWS ECS.

As such, it is deployed on a set of nodes (for example,
[Kubernetes](https://www.kubernetes.io/) nodes) and needs to trace some
parameters from those nodes. For example, it needs to monitor IP and L4 port,
source and destination, between a client and a server, or NAT details, etc.

To do so, Weave Scope has used or explored a number of technologies, each
having their limitations:

* procfs and conntrack,
* Netlink sockets, and
* tracepoints and eBPF.

It turns out the best solution so far is yet something else: eBPF attached to
kprobes. In a succession of steps, they moved to using bcc (and Python) to Go
language and an ELF loader, in particular to lighten the dependencies needed in
the Alpine OS image (thus getting rid of llvm, elfutils libraries and kernel
headers).

As a consequence they use a single eBPF program for all the kernel version. But
this implies having to guess the offset of the attributes in some structs! From
what I understood this is done by checking the offset of some expected values
in the memory, and by keeping states (uninitialized, checking, checked) in
maps.

A number of challenges have not be solved to this date. This include things
like perf event ordering or endianness with struct `sock` (see slides for
details). Also, they had some troubles with continuous integration tests since
most free services such as Travis CI use old kernel with no eBPF and some
restrictions. Their solution: using SemaphoreCI with rkt and a [specialised
stage1](https://github.com/kinvolk/stage1-builder).

The “Tracing TCP connections“ part of the title refers to [this
example](https://github.com/weaveworks/tcptracer-bpf).

# ebpf-cgroups in Kubernetes and Prometheus

* _Alban Crequy, Director & Software Engineer at Kinvolk_

[Slides available on the net](https://docs.google.com/presentation/d/1i7MNy8ozTLVmICCicVjZL8HYEasXkmR05uE5WRXv9ss).

Alban went on and presented Kinvolk's work on
[ebpf-cgroups](https://github.com/kinvolk/cgroup-ebpf), a frontend leveraging
the eBPF capacity to get attached to cgroups. If I understood correctly, they
use it with [Prometheus](https://prometheus.io/), an Open Source tool using
“metrics-based time series database for monitoring”, to indeed monitor
Kubernetes nodes.

They seem to be using
[LRU](https://en.wikipedia.org/wiki/Cache_replacement_policies#LRU) eBPF hash
maps to collect metrics from the nodes.

The talk was short, around 10 minutes, and did not bring many questions as far
as I remember.

# Tracing the CDN with LuaJIT-BPF / bcc

* _Marek Vavrusa, Programmer (R&D) at Cloudflare_

Marek developed a compiler from LuaJIT to eBPF. Though started as a side
project, it eventually ended up in bcc repository as one of the existing front-
and back-ends. The talk comes back on the motivation of this project, the way
it works, and presents the next steps.

Motivation comes from the will to trace events in Cloudflare's infrastructure,
in particular: sockets for DNS services. But they run several services on up to
27k addresses! Hence the idea to create socket groups for each process, and to
attach `SO_REUSEPORT` BPF programs to those groups. Programs can update
per-destination maps to provide statistics. The need for a LuaJIT compiler
itself comes from the reluctance of people doing the monitoring to develop
complex tools.

Hence the creation of the tool. LuaJIT has a number of practical features in
this context (and it is easier than C), even though several elements differ
from eBPF (LuaJIT has “slots” where eBPF has registers and stack, LuaJIT has no
pointer arithmetic, etc.). But it works well for writing both user and kernel
space code.

Future steps include further eBPF code optimisation, and some work outside of
the compiler (visualization, access control for examples).

# A control / management plane for IO Modules

* _Fulvio Risso, Associate Professor at Politecnico di Torino_

This is one of the few academic projects using eBPF I heard of. Fulvio Risso
and his team use [IO Modules](https://github.com/iovisor/iomodules) for NFV,
thus providing [a new
backend](https://github.com/iovisor/iovisor-ovn/tree/master/iomodules) to OVN
(the second repository seems to be the one containing the up-to-date
IO Modules). To explain this choice, they state that:

* IO Modules are fast (1.3 64-byte long Mpps, when OvS is a bit higher at 1.4+).
  In particular, chaining services provide good performances, even without XDP
  (36 Gbps in TCP throughput for an IO Visor-based VM, against 24 Gbps for
  OpenStack).
* IO Modules are flexible, they easily produce the desired chain of services.
* IO Modules are powerful and can express a lot of services. Or are they?

The speaker argued that eBPF has its own limitation and cannot implement
everything, hence the need for a “slow path”. Example: broadcasting a packet to
a variety of IO Modules (cannot be done through tail calls). This slow path,
implemented in userspace, is opposed to the “fast path” they build with eBPF in
the kernel. They are not necessarily on the same device, and they communicate
through a management and control layer.

This slow path seems to be implemented in the “Hover” module coming with [the
IO Modules](https://github.com/iovisor/iomodules). There are two parts. All
modules use a local Hover controller that registers a callback in the global
Hover controller, which can later call it to redirect exception packets it
receives. On the dataplane level, each local controller acts as a proxy between
the IO Module and the global centralized Hover module. The full slow path detail
is:

    IO Module → encapsulator → perf ring buffer → local ctrler → global ctrler
        → back to local ctrler → tap → decapsulator → back to IO Module.

The local controller also acts as a “user space helper”, in the sense it
provides some features that can be used from the IO Modules without having to
reach each time for the global controller. People are encouraged to tell what
“helpers” of this kind they need.

An example use case for this work is a L2 switch, whose code has been
considerably reduced after the implementation of the slow path. The team also
worked on a router prototype, and have a DHCP implemented in the slow path (but
intend to make a smarter version in the datapath).

All the code and examples, for IO Modules and for the control plane, can be run
with and without OVN, from the code [in this
repository](https://github.com/iovisor/iovisor-ovn). Work to come include
improving the controller, maybe move to a generic SDN controller (ONOS?), or
stabilizing the specifications and API of the control path. Also they would
like to gather a community around the project, even if they are not so many
companies working with eBPF for networking, on the dataplane, right now.

Answering a question I could not understand, Fulvio explained that he expect
few packets to be raised as exceptions and sent through the control path.
Fewer, in fact, than for current SDN solutions, since eBPF, even if it can not
be used in all cases, remains pretty powerful.

Personally, I had not heard much about a “slow path” for eBPF before that. Of
course, when deploying eBPF in a vast network, there would be some need for a
kind of controller that would, for example, generate the eBPF programs and send
them to the switching nodes. This is, I believe, what happens with Cilium, or
what I would like to have for [BEBA project]({{ site.baseurl }}{% post_url
2016-07-15-beba-research-project %}). But here it is about dealing with
exception packets, not only configuration.

Also, it is interesting to see that the slow path is in userspace, while the
concept of XDP would rather be to remain on the kernel for all tasks: use eBPF
for fast dataplane, and default to the network stack otherwise. I do not know
how complementary this would be with the material described in this talk, sadly
I did not have time to formulate and ask a question at the end of the
presentation.

# OvS eBPF datapath

* _Joe Stringer, Software Engineer at VMware_

The Open vSwitch (OvS) team has been looking at eBPF for a while, and several
patches have already been written to clean the path in that direction.

The objective is to get better performances, more flexibility and more
stability. The principle leading the transition is: implement datapath in eBPF,
see what it's like, fallback on userspace when necessary. Then extend it
incrementally.

The current OvS design would be adapted to eBPF components, with hash map
instead of flow table, perf ring instead of “upcall” etc. The parser could be
generated with P4. One eBPF program could be used per action, then they can be
chained with tail calls (somewhat similar to what Cilium and IO Modules do, in
fact).

There are open questions on megaflows, connections tracking (involving IP
defragmentation), and packet cloning:

* Current version of OvS uses both exact match and megaflow, adapting this to
  eBPF would require a way to use masks for lookup.
* Connexion tracking could use either a specific eBPF helper, or some specific
  eBPF bytecode (consumes instructions, but easier to port and to use with
  XDP).
* I don't remember the issues about packet cloning. Maybe the same thing as
  with IO Modules: eBPF tail calls (or eBPF in general) does not allow to clone
  packets?

There were a lot of discussions here, in particular about the “tuple space
search table” (to replace flow table; not yet in the kernel as far as I know),
but I could not grasp much of the discussion.

Someone asked about the upcoming work on the topic. From what I heard (and from
what Joe later told me by email, thanks!), there is a number of patches that
already exist to implement some basic features of this eBPF datapath, not
pushed yet. People work on their private trees for now and, quoting Joe,
“Probably when [they] get something that implements a pretty reasonable amount
of the OvS functionality [they] will start pushing out to the lists for further
comments and review”. And of course, there are those open issues to fix one way
or another.

This is really interesting to see some production-oriented software such as
OvS, which is the reference for virtual switching in the industry, slowly turn
to eBPF to implement its dataplane—the very core of packet processing. Of
course the process cannot be instantaneous, but it will be nice to see what an
eBPF-based OvS is able to do. I am also curious to see how they generate
programs from OpenFlow commands.

# Cilium — Status of recent work and current limitations

* _Thomas Graf, CTO & Co-Founder at Covalent IO_

Thomas is doing a lot of dissemination work about
[Cilium](https://github.com/cilium/cilium), this is maybe the fifth talk he
gives on the subject. So for this one, he announced he would focus on the new
work achieved over the last year. In the end I was a bit disappointed, since
most of the talk remained pretty similar to [the previous presentations about
Cilium]({{ site.baseurl }}{% post_url 2016-09-01-dive-into-bpf
%}#about-other-components-related-or-based-on-ebpf).

So if you do not know about Cilium, this is some Open Source solution to manage
networks of IPv6 (mostly) containers, regarding network fabric as well as
security. To this end, Cilium generates one eBPF program for each container /
task / service / endpoint, that is attached to tc (or XDP?) interface. The
security is enforced _via_ rules based on _identities_, themselves based on
metadata on endpoints and carried through the network thanks to _labels_
contained in the packets. The Cilium architecture naturally adapts to any
container framework (Docker, Kubernetes, etc.). And it scales pretty well with
the number of cores (two linear factors: one from 1 to 8 cores, and a slower
one from progression after 8 cores).

Happily, there _was_ some new elements since the presentation of last submit.
The load balancer use case, for example, did not exist at that time. Now,
Cilium also has IPv4 support, or L4 policy support. Further improvements are
expected: more integration with Kubernetes, in particular an “ingress
implementation” (though I do not know what it refers to), transparent
redirection into proxies, or collection of metrics and statistics to send them
to some monitoring solution such as Prometheus.

There were not many questions. Well, it was lunch break time…

# XDP for virtual machines

* _John Fastabend, Software Engineer, Intel_

[Slides available on the net](https://github.com/jrfastab/iovisor_summit_feb/raw/master/Fastabend-XDP-Summit-Feb.pdf).

The two successive talks on XDP by John Fastabend are my greatest source of
regrets on the Summit: it looked very promising, and most probably it was: John
is involved in all discussion about XDP design and optimizations on the kernel
networking mailing list, and is currently working on porting XDP to Intel's
ixgbe and i40e drivers. But the talk turned out to be more like a workshop or
Q&A session, and I just could not understand most of the talks and discussions.
Plus the “here” and “there” in the speech suggest that John was pointing at the
diagrams on the slides, and I could not see such gestures either. I'll
definitely have to watch it if the videos are released.

From the slides, we see that the objective of the work is to use XDP to
implement a virtual switch and to attach it to a virtual host; then to add also
a virtual switch with XDP on the hypervisor itself, just above the physical
NIC. By using the `XDP_REDIRECT` action, it is able to steer packets directly
to the XDP switch attached to the virtual host. This action is still to add
into the kernel at this date.

In the end, the idea is to have traffic between virtual machines, or between
VMs and hypervisor, driven by XDP components for high performances. TSO (for
TX) and LSO (for RX) support is also expected.

John has no benchmark results ready to show yet, but he should have some in a
near future.

# XDP hardware metadata integration

* _John Fastabend, Software Engineer, Intel_

[Slides available on the net](https://github.com/jrfastab/iovisor_summit_feb/raw/master/Fastabend-XDP-Summit-Feb.pdf).
The slides are at the end of the previous set.

Similarly to the previous talk, this one was mostly a discussion session that I
did not manage to follow.

The couple of slides show that some network features, namely checksum, TSO or
other inline programming metadata that hardware is able to compute today need
some specific space along with the packet data buffer to be forwarded to
software. WIth XDP, this “headroom space” is limited (192 bytes for now), hence
the open question: what should be done to be able to preserve those features?

Again, I'll have to watch the video to catch up with the discussion… Sorry for
the poor coverage!

# Design and implementation of P4/XDP

* _William Tu, Senior Member of Technical Staff at VMware_
* _Mihai Budiu, Senior Staff Researcher at VMware_

The first slides are a brief presentation to [P4 language](http://p4.org). It
is used in order to describe the architecture of a switch or, since the
P4<sub>16</sub> version, of any network function, and then to be
compiled towards a specific target. This P4<sub>16</sub> version
now supports a large number of devices.

In particular, there is [a compiler from P4 to eBPF for
XDP](https://github.com/williamtu/p4c-xdp) that William has developed. The XDP
switching model is described: parser, then {match + action}, with maps
potentially used to communicate with control plane, then deparser. This can be
described with P4, then the p4c-xdp tool produces both a .c file (holding the
eBPF program, to be compiled from C) and a .h header file (describing the maps
for user space programs). The program is compiled into an object file and
injected into the kernel, where it passes the verifier and eventually gets
attached to the XDP hook. The slides provide code samples for the parser, table
match/action, and deparser blocks in P4, that I will not reproduce here. The
deparser is used to update the packet, for example by pushing new headers.

Dependencies to run the program include the P4<sub>16</sub>
prototype compiler, and recent versions of stuff that is easy to get on Linux
(kernel, iproute2, clang/LLVM).

The project had some issues with the eBPF in-kernel verifier, but fixed it on
the kernel side. It still have some open issues, such as the state of registers
not being tracked when the data is spilled to the stack, or another one that is
not related to the verifier, but instead to the limited size of the stack (512
bytes as of this writing).

Then comes the presentation of three demos (not shown during the talk, only
resumed on slides). The videos of the demos are available on [William's YouTube
page](https://www.youtube.com/channel/UCXvv9ht8tr1qMDnB10JX3Yg) (the last three
at this date; search for P4 or XDP). They were about: swap Ethernet, ping4/6
and stats, and encapsulation. The code, as you can guess, [is on
GitHub](https://github.com/williamtu/p4c-xdp/tree/master/tests).

The presentation ends with a slide on “Future Wok”: next steps include work on
forwarding to any port with XDP (not yet supported in kernel), broadcasting or
cloning packets (same problem as everyone else), recirculation, and providing
more use cases. This seems good (sorry, couldn't help!).

There was a question I wanted to ask: is it possible to have it the other way
round, to compile eBPF to P4 in order to program ASICs? I asked on the chat,
but the question was noticed too late to be answered: the next talk had already
begun. Too bad, maybe next time!

# XDP use cases: DDoS, load balancer, IPVS and ILA router

* _Alexei Starovoitov, Software Engineer, Facebook_
* _Satendra Gera, Software Engineer at Versa Networks Inc._
* _Nikita Shirokov, Production Engineer at Facebook_
* _Huapeng Zhou, Software Engineer at Facebook_

This was a four-in-one presentation about several use cases for XDP at Facebook
and Versa Networks:

* Protection against Distributed Denial of Service (DDoS) attacks, by Huapeng
  Zhou.
* L4 load balancing, by Nikita Shirokov.
* XDP vs. IPVS, and
* ILA router, by Satendra Gera and Alexei Starovoitov, though I am unsure who
  presented what.

## Droplet: DDoS protection using BPF

The first use cases presented is protection against DDoS. Something I was
impatient to hear about: I have some previous experience with DDoS, but most of
all, this is sold as one of the main use case for XDP, and discussions about
the design are regularly referring to this use case.

So the talk is about Droplet, which is a DDoS protection framework from
Facebook. It uses the classical bcc workflow, and attaches programs to tc or
XDP interface. The Droplet “handler” deals with runtime compilation and loads
the program into the kernel. Well, I thought that was just what bcc was about
already? Is Droplet another wrapper layer around bcc Python script?

Probably it brings some helpers more related to DDoS protection. The talk
mentioned “generic handler”, “IP handler”, “prefix handler”. The user still has
to write some eBPF program (in C language), it is not automatically generated.

It is possible to “chain” the rules, the countermeasures, well in fact to chain
eBPF programs. With tc, this is easy, since the interface is able to drive the
packet through several successive programs. No such thing with XDP, but
chaining remains possible with tail calls—as all other people chaining programs
on talks about XDP seem to do anyway. They “control filter runtime behavior via
CFLAGS”, though I am not sure what this means, and it does not seem an ideal
solution. So the work is not finished yet.

There was no benchmark, and the bad news: I asked if it is Open Source, but at
this time it is not, there is an confidentiality issue on some part. There
seems to be plans to open it at some point, though, so let's cross fingers. I
would like to experiment with this!

## XDP for L4 load balancing

The second use case was about L4 load balancing. It was short, and I do not
remember it well. The idea is to use XDP to implement a L4 load balancer. Then
the L4 load balancer seems to encapsulate the packet and to push it to a L7
load balancing device, where the packet is decapsulated, and probably forwarded
to the relevant service. I am not sure XDP intervenes on the L7 level. You'd
probably better wait for the videos if you're really interested in this one…

## XDP vs. IPVS

We stay on load balancing with this one.
[IPVS](https://en.wikipedia.org/wiki/IP_Virtual_Server) is a L4 load balancing
feature available from the Linux Kernel, and based on Netfilter. Here it is
described as “too generic”, hard to extend. The speaker also explains that XDP
brings much better performances: the tests performed show that it can do
between 3 and 6 times better with high (99%) cache hit, and 10 times, or even
up to 25 times (without session tracking) when the cache hit is at 0%.
Impressive figures!

## ILA router

ILA is for Identifier Locator Addressing. It consists in splitting an IPv6
address into two parts, locator (“where”) and identifier (“who”). Routing to
end points use locators, then identifier are used as endpoint addresses of
virtual nodes. You can find some documentation about it on the web, often by
Tom Herbert, who also works at Facebook (at the time of this Summit) and
contributes to XDP (he organized the previous summit, by the way).

Here the talk is about implementing an ILA router with XDP. Since the routing
operations seem relatively simple, with no encapsulation, the use case looks
well-suited for XDP. There is no particular technical detail that drew specific
attention, as far as I remember: the router uses XDP with hash maps for the
tables. On the diagram, with ILA hosts on the sides, the router seems to be
placed “above” a switch placed in the middle? This could allow to send the
packet from switch to router, then back to the switch on the same port, since
XDP is currently limited and cannot forward a packet on another port that the
one it comes from.

ILA routing itself faces some issues, just search the web if you want more
details. There was a question about checksums I believe, but I failed to
understand the discussion. The question of open sourcing the project was raised
again: while the control plane of the ILA router seems to be generic, the
address resolution stuff would not be ready for a release at this time, so
again this may happen in the future, not immediately. There was also some talks
about resistance of the router to DDoS, and in that XDP seems to do a pretty
good job, so ultimately the limit is the speed at which the driver can drive
packets.

## Additional use cases

People from Facebook also stated that they had some other use cases involving
XDP, and named two of them. One had to do with storage, and the other one I
could not hear about.

# Transparent XDP / BPF offload to hardware

* _Nick Viljoen, Research and Development at Netronome_

The presentation of Netronome does not present a specific use case for XDP or
eBPF. Instead, it explains how the company managed to offload eBPF execution…
to their hardware platform. That's super interesting, and really promising!

Their NFP board is made of “islands” containing 12 “Flow Processing Cores”
(FPC) each, for a total ranging from 72 to 120 FPCs. Some of those cores are
used to process packets as a standard NIC would do, others (half of them it
seems) are used to process JIT-compiled eBPF instructions. A special island is
responsible for communicating with the kernel module.

This kernel driver, in particular, can be used to perform the offload. It
registers a NDO (“Network Device Operation”, if I remember correctly?), as
would any other driver, except this one is used to extract the eBPF code from
tc of XDP hook (they are working on XDP offload).

Regarding the offload, Nick insisted on the fact they did not want to focus on
_what_ to offload, but instead on _how_ to offload eBPF stuff, to avoid
whack-a-mole problems. As I take it, this means you are pretty free to offload
any eBPF stuff you want, since Netronome has designed a nice workflow to do
that.

So back to the offload itself: there is this NDO. The module can get an eBPF
program from there. When the program is provided to the driver, it has already
been injected into the kernel, and it has passed the verifier (a first time).
The driver will call a specific function to perform a second pass, slightly
different, of the verifier on the program, possibly registering some custom
verifications (in order to reject programs using features unsupported by the
board, for example).

Then the program is JIT-compiled, and sent to the board _via_ PCIe, where it
can be executed as it would be in the kernel. But on dedicated hardware. XDP is
fast, XDP on hardware sounds… pretty awesome. They get about 3 Mpps per FPC for
write \+ redirect operations, and just a bit less for read \+ write operations.

All of this was already in at least one earlier presentation by Netronome. What
was new however was their detailed plans regarding maps. They intend to share
read-only maps with the kernel, but to have a single copy on hardware for
read/write maps. In both cases, some new mechanisms must be added to the kernel
to intercept update and lookup operations, or to lock the maps in the kernel.
The access type could be determined by the verifier.

They also explained some optimizations that the JIT-compiler is able to bring
to the eBPF code, such as sometimes merging several instructions (e.g. shift
right, logical “and” and move) into a single one (in this case “`alu_shf`”, I
don't know what it stands for exactly).

There is still some work to do, it seems, before map offload reaches the
kernel. But I'm really curious to see what offloaded eBPF with maps could be
like. Dedicated hardware for hashmaps, anyone?

# Gobpf — Utilizing BPF from Go

* _Alban Crequy, Director & Software Engineer at Kinvolk_

Alban comes back in a third talk, this time to present the Go implementation of
tools for handling eBPF. Using Go was motivated by personal preferences and by
their working environment (Weave Scope is in Go).

With [Gobpf](https://github.com/iovisor/gobpf), it is possible to reuse the
helpers from the bcc tools, or to load eBPF programs directly from ELF object
files. Some examples were provided for each case, and they can be found on the
GitHub repository. For eBPF-based programs, the continuous integration remains
an issue for now (old drivers, non-root access etc.). Kinvolk's solution
remains the same as explained earlier: SemaphoreCI with a custom stage1 for
rkt.

I don't remember of any question on this talk, but maybe I forgot them.

# Libiov — A Framework for building IO Modules

* _Brenden Blanco, Staff Engineer at VMware_

[Slides available on the net](https://docs.google.com/presentation/d/1yv2MQ30B47YLIQYXzR1EiSIw7j-h2hWebNcMIp-Calg).

And finally the last presentation was about
[libiov](https://github.com/iovisor/libiov/blob/master/docs/event-api.md), a
library, or even an interface, to “connect IO Modules”. It is based on the
following observation: the current eBPF tools are quite disparate in their way
to be called and used. The attach points are specific to each type of program
(kprobes uses tracefs, tc uses a classifier, XDP uses the driver, etc.), the
debugging facilities are not the same, maps handling is somewhat opaque, and
map themselves can be tied to a process as well as pinned on the file system…

All of this makes it difficult to somehow find a way to unify eBPF program and
maps handling. While this does not seem to be a major limitation when using a
single tracing tool or networking feature, this may become a handicap when
trying to couple more complex IO Modules, or to use for example programs on
cgroup and tc and kprobes at the same time… This is what libiov tries to solve.

This is based on a model with “events”, “actions” and “buses”. It seems to be
described with more details on the GitHub repository. The buses seem to be used
as an intermediary between events and actions: it is possible to subscribe to
events (that are sent on the buses), and to describe actions to perform on
those buses (such as `filter()` or `notify()`).

Ultimately, this library is expected to help modularize the eBPF objects so
that they can support different backends: C language, P4,
[ply](https://wkz.github.io/ply/), LuaJIT etc. An objective for future work
would be to also provide support for several eBPF implementations: kernel of
course, but also hardware offload, and userspace virtual machines for eBPF like
[uBPF](https://github.com/iovisor/ubpf/) or…
[rbpf](https://github.com/qmonnet/rbpf)! My eBPF implementation in Rust was
mentioned, I was proud of it. Made it totally worth to follow the Summit until
the end, even if that was at 2 a.m. in Paris…

Libiov could also be used to implement a number of utilities, not in the sense
of bcc tracing tools, but rather utilities to work with eBPF and improve the
workflow: map dumping tools (similar to
[bpf-map](https://github.com/cilium/bpf-map) from Cilium), lsof for eBPF
programs, etc. This looks really interesting, those tools would be of great
assistance to develop more eBPF-based projects.

The motivation behind the library is excellent, and the technical concept is
intriguing. I will have to look deeper into the examples and code, it would be
really nice to have rbpf compatible with bcc for instance.

# Conclusion

The high-quality talks in this summit offered an interesting state-of-the-art
overview of the work on XDP and eBPF in general. The community is going well,
growing and multiplying projects based on this technology.

In regard to last year's summit, some companies did not intervene (or I did not
spot them), I'm thinking in particular at Huawei or at Mellanox (I don't know
what the current state of CETH is).

Even though the PLUMgrid company, who initiated the IO Visor project, was
recently acquired, the fact that VMware (the buyer) organized the summit shows
that the people remain engaged in the project.

On the technical side, eBPF is spreading to a number of layers: OvS datapath,
kernel and process tracing, cgroup filtering… This comes with a number of use
cases: programmable switching and routing, DDoS protection, managing
containers. We can expect that eBPF-based project will keep spreading, and that
new use cases are yet to appear. I am really eager to see what comes out from
XDP, and what impact it will have on high-performance networking.

{% comment %} vim: set syntax=markdown spell tw=79 fo+=a: {% endcomment %}
