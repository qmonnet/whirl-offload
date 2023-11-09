---
layout: post
title:  "Bpftool mirror now available"
date:   2022-01-19
author: Quentin Monnet
employer: isovalent
excerpt: Announcement for the bpftool mirror repository on GitHub.
categories: Bpftool
tags: [BPF]
---

"Hi, I have the pleasure to announce the availability of a mirror for
bpftool on GitHub, at the following URL:

<https://github.com/libbpf/bpftool>

This mirror is similar in spirit to [the one for libbpf][0], and its
creation was lead by the following motivations.

1. The first goal is to provide a simpler way to build bpftool. So far,
building a binary would require downloading the entire kernel
repository. By contrast, the code in the GitHub mirror is mostly
self-sufficient (it still requires libelf and zlib, and uses libbpf from
its mirror as a git submodule), and offers an easy way to just clone and
compile the tool.

2. Because it is easier to compile and ship, this mirror should
hopefully simplify bpftool packaging for distributions.

3. Another objective was to help other projects build on top of the
existing sources for bpftool. I'm thinking in particular of
eBPF-for-Windows, which has been working on [a proof-of-concept port of
the tool for Windows][1]. Bpftool's mirror keeps the minimal amount of
necessary headers, and stripped most of them from the definitions that
are not required in our context, which should make it easier to uncouple
bpftool from Linux.

4. At last, GitHub's automation features should help implement CI checks
for bpftool, very much like libbpf is using today. The recent work
conducted on libbpf's CI and turning some of the checks into [reusable
GitHub Actions][2] may help for bpftool.

Just to make it clear, bpftool's mirror does not change the fact that
all bpftool development happens on the kernel mailing-lists (in
particular, the BPF mailing-list), and that the sources hosted in the
kernel repository remain the reference for the tool. At this time the
GitHub repository is just a mirror, and will not accept pull requests on
bpftool's sources.

Regarding synchronisation, the repository contains a script which should
help cherry-pick all commits related to bpftool from the kernel
repository. The idea is to regularly align bpftool on the latest version
from libbpf's mirror (that is to say, to cherry-pick all bpftool-related
commits from bpf-next and bpf trees, up to the commit at which libbpf's
mirror stopped), to avoid any discrepancy between the tool and the
library it relies on.

GitHub was the original home of bpftool, before Jakub moved it to
kernel's tools/ with commit 71bb428fe2c1 ("tools: bpf: add bpftool")
back in 2017. More than four years and five hundred commits later, it is
time to have a stand-alone repository again! But over time, the build
system and the header dependencies got somewhat intertwined, and
extracting the tool again required a few careful steps. Some of them
happened upstream: we addressed the licensing of a few bpftool
dependencies, we switched to libbpf's hash map implementation (instead
of kernel's), we fixed the way bpftool picks up the development headers
from libbpf, and so on. Some other changes happened in the mirror, to
adjust bpftool's Makefile and build system. In the end, I'm rather happy
with the result: the main Makefile in bpftool's mirror only has a few
minor changes with the reference one, and the C sources need no change
at all. The tool you get from the mirror is the same as what you can
compile from tools/bpf/bpftool/.

I hope that this mirror will make it easier to work and develop with
bpftool. Thanks to Andrii for his feedback, and to the folks behind
libbpf's mirroring for their work, some of which I was happy to reuse.

Best regards,<br />
Quentin"

[0]: https://github.com/libbpf/libbpf
[1]: https://github.com/dthaler/bpftool
[2]: https://github.com/libbpf/ci

Find the original message in
[the BPF mailing list archives](https://lore.kernel.org/all/267a35a6-a045-c025-c2d9-78afbf6fc325@isovalent.com/).
