---
layout: post
title:  "Implementing eBPF for Windows"
date:   2023-11-06
author: Quentin Monnet
employer: isovalent
excerpt: An analysis of what its Windows implementation can bring to eBPF.
categories: BPF
tags: [BPF]
---

* TOC
{:toc}

_This article was initially published in June 2021 [on
LWN.net](https://lwn.net/Articles/857215/)._

[Extended BPF (eBPF)](https://ebpf.io/), the general-purpose execution engine
inside of the Linux kernel, has proved helpful for tracing and monitoring the
system, for processing network packets, or generally for extending the behavior
of the kernel. So helpful, in fact, that developers working on other operating
systems have been watching it. Dave Thaler and Poorna Gaddehosur, on behalf of
Microsoft, [recently published an implementation of eBPF for
Windows](https://cloudblogs.microsoft.com/opensource/2021/05/10/making-ebpf-work-on-windows/).
A Linux feature making its way to Windows, in itself, deserves attention. Even
more so when that feature has brought new degrees of programmability to the
Linux kernel over the last few years. This makes it especially interesting to
look at what the new project can do, and to ponder how the current ecosystem
might evolve as eBPF begins its journey toward Windows.

# Gearing up with open-source components

Simply named "eBPF for Windows", the project targets Windows 10 and Windows
Server 2016 (and later). It is not a fork of the Linux code, presumably to avoid
any GPL-related issues. Instead, Windows gets a different implementation, but
not from scratch. Microsoft's project relies on pre-existing open-source
components; it adds the missing pieces along with the necessary shims to
interconnect them and to integrate them with Windows internals. It is available
from GitHub as
[ebpf-for-windows](https://github.com/microsoft/ebpf-for-windows), under the MIT
license.

One of those components is [uBPF](https://github.com/iovisor/ubpf), an
independent user-space eBPF interpreter and an x86\_64 just-in-time (JIT)
compiler for eBPF. In 2015, which was the early days for eBPF, uBPF was
introduced as a non-GPL implementation of the technology. It has seen little
activity since then, but several projects, including
[DPDK](https://doc.dpdk.org/guides/prog_guide/bpf_lib.html), reuse its code. The
Windows project pulls uBPF as a Git submodule, and [literally does a `#include`
of its
code](https://github.com/microsoft/ebpf-for-windows/blob/d765bd616dafb33aea19fd62bf0d9230fcd3720a/libs/ubpf/user/ubpf_user.c)
to embed the interpreter and the JIT compiler. uBPF only supports basic
programs; the code in ebpf-for-windows extends it, for example to implement eBPF
maps and their related helper functions.

The tooling side for handling eBPF programs on Windows should look familiar to
Linux users. The Clang/LLVM back-end remains the main tool to compile programs
written in C into eBPF bytecode instructions. The `netsh` command-line tool,
which offers program management and introspection similar to some extent to
`bpftool`, relies on a shared library that should provide an API compatible with
[libbpf](https://github.com/libbpf/libbpf), thus ensuring the portability of
loader applications.

The last component used by the project is the [PREVAIL
verifier](https://github.com/vbpf/ebpf-verifier), which is another Git submodule
that gets pulled into ebpf-for-windows. PREVAIL was developed as part of a
research project involving VMware Research, and is best described in the
[accompanying publication](https://vbpf.github.io/assets/prevail-paper.pdf)
(note that some of the content is now out-of-date). Unlike the Linux verifier,
it relies on formal methods to determine the safety of programs. It uses an
[abstract interpretation](https://wiki.mozilla.org/Abstract_Interpretation)
layer that aggregates the states after several execution streams rejoin, meaning
that, for example, the states of the registers for the different branches of an
`if`/`else` block are merged together at the end of that block to avoid
producing too many combinations of paths.

By contrast, the Linux verifier checks all paths independently: two consecutive
`if`/`else` blocks produce four paths to explore, rapidly leading to a "path
explosion". But the Linux verifier mitigates this by "pruning" some parts of the
paths that are known to be safe, once they have been reached and validated from
another branch.

PREVAIL is [claimed to be better at recognizing valid
programs](https://github.com/vbpf/ebpf-verifier#counter-and-artificial-examples)
than the Linux verifier, including programs with loops. It also scales better
with complexity, and can validate some programs for which Linux would go over
its limit of one-million validated states, thus would fail verification.
However, the Linux verifier completes validation more quickly when it accepts a
program, and consumes less memory. It is also finely tailored to support all of
the features that the programs can use and their corner cases, while PREVAIL
misses checks for several elements like eBPF-to-eBPF function calls, packet
reallocation (when its size changes), or 32-bit subregister tracking. PREVAIL is
under active development, and Thaler is contributing to it along with the
creator and maintainer, Elazar Gershuni. Together they intend to address some of
these limitations: they recently added support for [checking
termination](https://github.com/vbpf/ebpf-verifier/commit/fdddca019166e3a8e6eaf6eb7ee4f920dfd50585)
of eBPF programs.

# Taking marks

Equipped with these components, eBPF for Windows provides a working eBPF
runtime. But the project is in its infancy. It brings support for [two eBPF
hooks](https://github.com/microsoft/ebpf-for-windows/blob/d724d3b079e771d814ffe2e3ce130a36bde3cd83/libs/api/windows_platform.cpp#L53):
XDP (for fast packet processing) and socket binding (equivalent to
`BPF_CGROUP_INET4_BIND` and other related attach points on Linux). It implements
the most basic [map
types](https://github.com/microsoft/ebpf-for-windows/blob/d724d3b079e771d814ffe2e3ce130a36bde3cd83/include/ebpf_helpers.h#L19):
arrays and hash maps.

The repository contains a few examples. The [Getting Started
guide](https://github.com/microsoft/ebpf-for-windows/blob/2a928a6c717b2775345b74863858d12b453fa85c/docs/GettingStarted.md#using-ebpf-for-windows)
explains how to set up an XDP program to [drop UDP packets with an empty
payload](https://github.com/microsoft/ebpf-for-windows/blob/2a928a6c717b2775345b74863858d12b453fa85c/tests/sample/droppacket.c)
in order to protect a DNS server from UDP flood attacks. Another [sample
program](https://github.com/microsoft/ebpf-for-windows/blob/2a928a6c717b2775345b74863858d12b453fa85c/tests/sample/bindmonitor.c)
attaches to the hook for socket binding, and [can be
configured](https://github.com/microsoft/ebpf-for-windows/blob/2a928a6c717b2775345b74863858d12b453fa85c/tests/end_to_end/end_to_end.cpp#L463)
to enforce a limit on the number of sockets to which processes can bind. The
program increments or decrements per-process counters (stored in a map) on bind
and unbind operations.

# Picking a direction

These samples are simple, but the project is just getting started: Where is it
heading? The authors made it clear in their blog post that they aim at providing
good compatibility with Linux eBPF: <q>The intent is to provide source code
compatibility for code that uses common hooks and helpers that apply across
Operating System ecosystems.</q>

Some existing hooks and helper functions, of course, are tightly linked to Linux
internals. Some features, like [direct calls to kernel
functions](/Articles/856005/), will be downright dependent on the operating
system. But for the rest, the objective is to reach some level of parity with
Linux, [including for
tracing](https://github.com/microsoft/ebpf-for-windows/issues/206) kernel and
user-space functions by reusing, for example, [Event Tracing for
Windows](https://docs.microsoft.com/windows-hardware/drivers/devtest/event-tracing-for-windows--etw-)
or [DTrace](https://en.wikipedia.org/wiki/DTrace). Tracing user applications can
probably work as they do for Linux.

Regarding kernel tracing, the two systems do not have the same internal
functions but, at least, through eBPF they should be able to share a common way
and workflow to attach probes. Interestingly, the authors of the project already
have a new hook in mind to [implement file-system "mini-filters" with
eBPF](https://github.com/microsoft/ebpf-for-windows/blob/2a928a6c717b2775345b74863858d12b453fa85c/docs/eBpfFileSystemHookProposal.md)
on Windows. It seems likely that they will add more in the future. In defining a
common way to probe a kernel, perhaps the program hooks themselves will end up
as an implementation detail if the eBPF workflow, tooling, and runtime has the
same capabilities on all systems and constitute the core of that "interface".

Currently, the disparity in features translates into a number of issues being
opened on the GitHub repository. Contributions to the different components will
also be necessary. The authors have pushed some of their changes to uBPF and
have been contributing to PREVAIL for several months already. It is worth noting
how they [intend to deal with
maps](https://github.com/microsoft/ebpf-for-windows/issues/21): they would like
to reuse the implementation from
[generic-ebpf](https://github.com/generic-ebpf/generic-ebpf), which is yet
another uBPF-based runtime that [targets FreeBSD in
particular](https://papers.freebsd.org/2018/bsdcan/hayakawa-ebpf_implementation_for_freebsd/).

Alan Jowett, from Microsoft,
[summarized](https://github.com/microsoft/ebpf-for-windows/issues/21#issuecomment-823462378)
the project's motivation as: <q>Any code that could be generic should be moved
to a more generic project and pulled in via a sub-module.</q> This reads as an
intent to upstream everything possible to uBPF, PREVAIL, and possibly other
projects in the future, so as to retain only the minimal amount of shims in
ebpf-for-windows and to make the generic components reusable by other projects.

# The road ahead

Ideally, most eBPF programs would behave the same way on Linux and Windows. Some
engineering effort will be necessary to reach that point. By reimplementing an
existing technology, Microsoft saves on the design process, but it will have to
catch up with the specific functionalities. There is more than a simple
instruction set to eBPF: it comes with many hooks, maps, helpers, user space
operations, and features like tail calls, function calls, [BPF type
format](https://www.kernel.org/doc/html/latest/bpf/btf.html) (BTF) information,
and many others. uBPF lacks most of these. These items may gain support in time,
a fair portion of them should hopefully be generic enough to make eBPF programs
portable. Adding JIT compilers for new architectures should not be a major
issue, [DPDK has one for
arm64](https://git.dpdk.org/dpdk/tree/lib/bpf/bpf_jit_arm64.c) already. Any
feature that relies on kernel BTF information will likely be tied to an
operating system, though.

Compatibility for the verifiers is another concern. Of all the design choices
for the Windows implementation, the use of PREVAIL is one of the most
interesting because it already has some advantages over Linux. It accepts a
larger range of valid programs and is less affected by the path explosion, to
the detriment of performance and exhaustiveness of the checks. As a consequence,
some programs may be considered valid only by Linux, others only on Windows.
This should provide an incentive to improve the Linux verifier, to make it
faster on complex sequences, and perhaps help it validate more programs. Full
parity will definitely be a challenge.

One may wonder how important it is to ensure cross-operating-system
compatibility for eBPF. For a start, Microsoft's interest in the technology
would likely be limited if it could not support existing programs. The company
could just as well develop its own solution, without bothering to integrate
external components. Portability will be necessary to limit ecosystem
fragmentation and, more generally, to warrant adoption on Windows. It will be
particularly important at a time when the generic-ebpf project might benefit
from the exposure and regain some activity. That would provide support on yet
more systems and, since static linking [is
being](/ml/bpf/20210318194036.3521577-1-andrii%40kernel.org/)
[introduced](/ml/bpf/20210423181348.1801389-1-andrii%40kernel.org/) in libbpf,
paving the way to the creation of eBPF libraries.

However, the effort cannot come from Microsoft alone. Synchronization with the
existing eBPF community will be essential to make the technology progress in a
way that all systems can support. On Linux, eBPF has been used in production for
some time, at scale, by big players like Facebook, Google, and Netflix. It has
been spreading over the Kubernetes ecosystem for networking, observability, and
security with projects like [Cilium](https://cilium.io/). There is no way that
Linux developers will put eBPF "on hold" to wait for Windows. Thaler and
Gaddehosur are aware that a lot of communication will be required:

> We are announcing this now while the project is still relatively early in
> development because our goal is to work in collaboration with the robust eBPF
> community to make sure that eBPF works great on Windows, and everywhere else.

The project has a dedicated Slack channel named `#ebpf-for-windows` on
<https://ebpf.io/slack>. [Regular
meetings](https://github.com/microsoft/ebpf-for-windows/issues/162) are set up
until July, at least, to triage and address the GitHub issues. A dedicated
mailing list should follow.

The degree of involvement that will come from Linux developers will be decisive,
but is hard to predict yet. The announcement of eBPF for Windows is still fresh:
there were comments on social networks (enthusiastic overall, with a few
expected mentions of the "embrace, extend, and extinguish" strategy ---
Microsoft has a history), but little reaction has emerged from the Linux world
for now.

# The journey is just starting

The "eBPF for Windows" project is ambitious but, in its early days, it misses
many features supported on Linux: some parity on program types, map types,
helpers, and other items will be required to make it viable. This could explain
the absence of a clear reaction from the Linux developers, who may be waiting to
see if the new implementation lives up to its promise. To show that it does, the
authors will need to catch up with development to support more complex use cases
on Windows. They must also draw the interest and participation of more
contributors, external to Microsoft, if they want to ensure the success of the
project.

If they succeed, developers may eventually benefit from eBPF code blocks shared
as libraries common to the two systems. User-space libraries, tooling,
applications, and compatibility test suites could be extended and run across
several environments. The eBPF ecosystem would then profit from fresh ideas
coming from the Windows world, and friendly competition will hopefully lead to
performance improvements on both sides. As an optimistic vision for the long
term, assuming a common basis can be found to express equivalent probes for both
systems, eBPF programs might end up as some form of control-plane API to manage
networking and observability on different types of nodes. Such interfaces [are
starting to emerge](https://github.com/cosi-project/community) in cloud-native
computing, and a cross-environment eBPF would likely play a role in this.

Of course, there is much to accomplish before that; the project is just
starting. Yet, as it stands, this is already a formidable acknowledgment of the
benefits eBPF brought to Linux and of its potential for the future. The
realization of this potential will depend a lot on the dialogue and
synchronization between the different communities. eBPF has just stepped outside
of its Linux home to venture into the world.
