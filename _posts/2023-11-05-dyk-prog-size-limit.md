---
layout: post
title:  "Did you know? eBPF program size limit"
date:   2023-11-05
author: Quentin Monnet
employer: isovalent
excerpt: What is the size limit for an eBPF program? 4k instructions? 1M
  instructions? Something else?
categories: BPF
tags: [BPF]
---

_This explanation on eBPF program size limit was initially published
in February 2021 by the Cilium community as part of the
[eBPF Updates #4](https://ebpf.io/blog/ebpf-updates-2021-02) on
[ebpf.io](https://ebpf.io/)._

Do you know what the maximum size of an eBPF program is? You may have heard of
programs limited to 4k instructions, but this has changed some time ago.

One particularity of eBPF programs, enforced at load time by the kernel
verifier, is that they must run and eventually terminate within a relatively
short delay. Allowing for long runs would slow down the kernel too much.
Permitting users to run arbitrary programs, possibly containing infinite loops,
could even hang the kernel completely.

To avoid that, the verifier builds the direct acyclic graph (DAG) representing
the possible paths of execution in the program, and ensures that each one leads
to termination. Sometimes, some branches “overlap” between several paths, and
under certain conditions the verifier can skip verifying them after the first
occurrence. This is called “state pruning”. Without this mechanism, the number
of instructions to validate would be too high and slow down program loading
beyond what is acceptable.

When eBPF was introduced, there were two parameters that would limit its size:

- The maximum number of instructions for a program: 4096
- The complexity limit: 32768

The second number represents the number of instructions that the verifier is
allowed to check before forfeiting the verification and rejecting the program.
You may think of it as “the total number of instructions cumulated over all the
execution paths, minus those on branches that the verifier is able to prune”.
So if a program had many logical branches and would require too much effort
from the verifier, it would fail to load, even if it had fewer than 4096
instructions.

But both limits were changed[^1] in
[a commit in Linux 5.2](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=c04c0d2b968ac45d6ef020316808ef6c82325a82).
**The complexity limit was raised to one million verified instructions**. As
for the maximum limit of instructions, it simply disappeared, meaning that the
size of program is now limited by the complexity induced by their verification.
There is still a _de facto_ hard limit at one million instructions, for a
program that would have a single logical branch (no “if” and comparisons
anywhere). In practice, such program would be of little interest. Programs have
branches, their verification is more complex, and their allowance for
instructions decreases accordingly.

The 4096-instruction limit did not, in fact, disappear entirely. The kernel
still enforces it for programs loaded by non-root users (more precisely, users
without `CAP_SYS_ADMIN` or, starting with Linux 5.8, `CAP_BPF`).

eBPF programs tend to be small, and the one-million-state complexity limit is
big enough that most use cases will never hit it. Some advanced projects using
eBPF to implement more complex features may be facing it, and
[Cilium](https://cilium.io/) for example is regularly adjusting to satisfy the
verifier's requirements. Some ways to work around complexity may include the
use of tail calls and bounded loops (introduced in Linux 5.3), or reorganizing
the code in such a way that the number of branches decreases or that the
verifier can prune them more efficiently.

Hardware offload is yet another story, and has entirely different constraints
since the program must fit into the hardware's memory. The bound is set as much
by the verifier as by the hardware's capacity, with the efficiency of bytecode
generation and then of the JIT-compiler
[both playing a role](https://www.netronome.com/blog/optimizing-bpf-smaller-programs-greater-performance/).

The new, one-million-state complexity limit should be flexible enough for most
use cases, and in the end, programs have truly one bound: Your imagination!

[^1]: The complexity limit was actually changed several times since
      [the 32k value from its introduction in Linux 3.18](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=17a5267067f3c372fec9ffb798d6eaba6b5e6a4c):
      it was raised to
      [64k in Linux 4.7](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=07016151a446d25397b24588df4ed5cf777a69bb), then to
      [96k in Linux 4.12](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=3c2ce60bdd3d57051bf85615deec04a694473840), again to
      [128k in Linux 4.14](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=8e17c1b16277cba0e9426de6fe78817df378f45c), and at last to
      [1M in Linux 5.2](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=c04c0d2b968ac45d6ef020316808ef6c82325a82).
