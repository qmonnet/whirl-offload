---
layout: post
title:  "Understanding tc “direct action” mode for BPF"
date:   2020-04-11
author: Quentin Monnet
employer: 6wind
categories: BPF
tags: [BPF, tc]
excerpt: The Linux Traffic Control subsystem, “TC”, got support for running
  eBPF programs as classifiers. Then a “direct-action” flag appeared. Let's see
  how it works.
---

* ToC
{:toc}

_This post was left aside as a draft for a long time. Most of it was written in
August 2016, at a time when no documentation for the `direct-action` mode for
TC was available. It is probably less relevant today, but I publish it in case
it might help readers understand a bit more how this flag works._

# A “direct-action” Mode for TC

The Linux Traffic Control subsystem, or “TC” for short, has been in the kernel
for years, and yet it is still under active development. A major addition
occurred with kernel version 4.1, when new hooks were added to run eBPF
programs as TC “classifiers” (also known as “filters”) or “actions”. About six
months later, alongside kernel 4.4, iproute2 got a curious `direct-action`
mode, that has been little documented…[^manpage]

[^manpage]: There was no documentation for the `direct-action` mode other than
    the commit logs when I drafted this article. But by the time I published
    it, the [Cilium Guide][cilium] referenced it, and the `tc-bpf(8)` manual
    page also provides a brief description, stating that the mode _“instructs
    eBPF classifier to not invoke external TC actions, instead use the TC
    actions return codes (`TC_ACT_OK`, `TC_ACT_SHOT` etc.) for classifiers.”_

# Back to the Basics

Before we see what `direct-action` can be used for, we need a short reminder
about the classic usage of traffic control on Linux. Effective traffic control
happens in the kernel: different algorithms can be used to throttle or
prioritise some flows on an interface. When a user wants to set up traffic
control, they usually rely on `tc` utility, from iproute2 package, which is the
user-part counterpart of the kernel TC subsystem, and communicate with the
latter through Netlink messages (mostly).

TC is a powerful, yet complex framework (and it is [somewhat documented]({{
site.baseurl }}{% post_url 2016-09-01-dive-into-bpf %}#about-tc)). It relies on
the notions of “queueing disciplines” (_qdiscs_), “classes”, “classifiers”
(_filters_) and actions. A very simplified description might be the following:

1. The user defines a _qdisc_, a shaper that applies a specific policy to
   different _classes_ of traffic. The _qdisc_ is attached to a network
   interface (ingress or egress).
2. The user defines _classes_ of traffic, and attach them to the _qdisc_.
3. _Filters_ are attached to the _qdisc_. They are used to classify the traffic
   intercepted on this interface, and to dispatch the packets into the
   different _classes_. A _filter_ is run on every packet, and it can return
   one of the following values:
     * 0, which denotes a mismatch (for the default _class_ configured for this
       _filter_). Next _filters_, if any, are run on the packet.
     * -1, which denotes the default classid configured for this _filter_,
     * any other value will be considered as the _class_ identifier refering to
       the _class_ where the packet should be sent, thus allowing for
       non-linear classification.

4. Additionally, an _action_ to be applied to all matching packets can be added
   to a filter. For example, selected packets could be dropped, or mirrored on
   another network interface, etc.
5. New nested _qdiscs_ can be attached to the _classes_, and receive _classes_
   in their turn. The complete policy diagram is in fact a tree spanning under
   the root _qdisc_. But we do not need this information for the rest of the
   article.

An example of this workflow could be the following example (inspired from [the
documentation of the HTB
shaper](http://luxik.cdi.cz/~devik/qos/htb/manual/userg.htm))

    tc qdisc add dev eth0 root handle 1: htb default 11

    tc class add dev eth0 parent 1: classid 1:1 htb rate 100kbps ceil 100kbps
    tc class add dev eth0 parent 1:1 classid 1:10 htb rate 30kbps ceil 100kbps
    tc class add dev eth0 parent 1:1 classid 1:11 htb rate 10kbps ceil 100kbps

    tc filter add dev eth0 protocol ip parent 1:0 prio 1 u32 \
        match ip src 1.2.3.4 match ip dport 80 0xffff flowid 1:10
    tc filter add dev eth0 protocol ip parent 1:0 prio 1 u32 \
        match ip src 1.2.3.4 action drop

In this setup, packets with source IP address `1.2.3.4` **and** with L4
destination port 80 (HTTP) will be sent to the first queue, i.e. the _class_
with a rate of 30 kbps. Packets from same host, to different ports, are
dropped. All other packets go to the second queue, with 10 kbps rate.

# Hooking eBPF Programs

Enters eBPF (extended Berkeley Packet Software). Basically, it consists in a
	restricted assembly-like language that can be used to produce programs run in
the kernel in a safe fashion. They can be hooked at several points in the
kernel, mostly for packet processing or for monitoring tasks. Two of these
hooks are related to TC: eBPF programs, since kernel 4.1, can be attached as
classifiers or actions.

As classifiers, eBPF brings more flexibility for parsing programs, and even
allows stateful processing or interaction with user-space _via_ specific data
structures called _maps_. But in the end, the classifiers remain the same: they
return a value that can be 0 for a mismatch, -1 for a match, or any other
_class_ identifier.

Used as actions, eBPF programs behave differently, their possible return values
indicating what action should actually be performed on the packet (description
from `tc-bpf(2)` manual page):

* `TC_ACT_UNSPEC (-1)`: Use the default action configured from `tc` (similarly
  as returning -1 from a classifier).
* `TC_ACT_OK (0)`: Terminate the packet processing pipeline and allows the
  packet to proceed.
* `TC_ACT_RECLASSIFY (1)`: Terminate the packet processing pipeline and start
  classification from the beginning.
* `TC_ACT_SHOT (2)`: Terminate the packet processing pipeline and drops the
  packet.
* `TC_ACT_PIPE (3)`: Iterate to the next action, if available.
* And a few others. They are defined in file
  [include/uapi/linux/pkt_cls.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/uapi/linux/pkt_cls.h)
  of the kernel tree. The [_BPF and XDP Reference Guide_ from Cilium][cilium]
  gives more details on their usage.
* Values not defined in that file are unspecified return codes.

[cilium]: http://docs.cilium.io/en/latest/bpf/#tc-traffic-control

# Concept of the “direct-action”

Although eBPF has restrictions—it has a limited number of instructions, and
only bounded loops are allowed, for example—it provides a powerful language for
packet processing. A consequence is that for a number of use cases, eBPF
classifiers alone are enough to filter and process the packets, and do not need
additional _qdiscs_ or _classes_ to be attached to them. This is particularly
true when packets should be filtered (passed, or dropped) at the TC interface
level. Classifiers do need, however, an additional action to actually drop the
packets: the value returned by a classifier cannot be used to tell the system
to drop a packet.

To avoid to add such simple TC actions and to simplify those use cases where
the classifier does all the work, a new flag was added to TC for eBPF
classifiers: `direct-action`, also available as `da` for short. This flag, used
at _filter_ attach time, tells the system that the return value from the
**_filter_** should be considered as the one of an **_action_** instead. This
means that an eBPF program attached as a TC classifier can now return
`TC_ACT_SHOT`, `TC_ACT_OK`, or another one of the reserved values. And it is
interpreted as such: no need to add another TC _action_ object to drop or
mirror the packet. In terms of performance, this is also more efficient,
because the TC subsystem no longer needs to call into an additional action
module external to the kernel.

For using TC with eBPF, using the `direct-action` flag is the simplest, the
fastest, and is now recommended way to go.

What about the TC eBPF _actions_ then? Could not they be used to achieve the
same result in the first place, to process the packet and return the correct
“pass” or “drop” value? The answer is negative: _actions_ are not attached
directly to a _qdisc_, they are only used after a packet has been through a
classifier, which means you need a classifier anyway. But then, does this mean
that TC eBPF _actions_ are now useless? Well, eBPF actions could still be used
after other filters. We can imagine attaching a _u32_ classifier on a _qdisc_
to proceed to a filtering based on some bit fields in the packets, then drop
the packets if they meet a given additional condition. eBPF action would work
in that case. But honestly, the use cases I have seen usually just use eBPF
both for filtering and returning the action, without the need for additional
filters.

Yet another question about the change of signification for the return values:
does this mean that when used with the `direct-action` flag, the eBPF
classifier loses its ability to indicate to what class the packet should be
dispatched? Once more, the answer is “no”: the special field `tc_classid` of
the `struct __skb_buff` which is passed as the only argument to the filter
program can be used, instead of the program return value, to tell the system
where to send the packet when the program allows it to pass.

# The “clsact” Qdisc

A few months after the `direct-action` mode was introduced into the kernel and
iproute2, a new _qdisc_ appeared in Linux 4.5: `clsact`. It is similar to the
`ingress` _qdisc_, to which we can attach eBPF programs with the
`direct-action` mode, and which does not perform any queuing. But `clsact` acts
as a superset of `ingress`, in the sense that it also allows to attach programs
with `direct-action` on the egress path, which was not possible before. It is
the recommended _qdisc_ for attaching eBPF programs in `direct-action` mode.

More details on the `clsact` _qdisc_ are available in [the relevant commit
log](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=1f211a1b929c804100e138c5d3d656992cfd5622)
or from [the Cilium Guide][cilium].

# Example Usage

Let's write a sample program that filter packets, possibly modify their data,
and pass or drop them, depending on the policy.

``` c
#include <linux/bpf.h>
#include <linux/if_ether.h>
#include <linux/pkt_cls.h>
#include <linux/swab.h>

int classifier(struct __sk_buff *skb)
{
	void *data_end = (void *)(unsigned long long)skb->data_end;
	void *data = (void *)(unsigned long long)skb->data;
	struct ethhdr *eth = data;

	if (data + sizeof(struct ethhdr) > data_end)
		return TC_ACT_SHOT;

	if (eth->h_proto == ___constant_swab16(ETH_P_IP))
		/*
		 * Packet processing is not implemented in this sample. Parse
		 * IPv4 header, possibly push/pop encapsulation headers, update
		 * header fields, drop or transmit based on network policy,
		 * collect statistics and store them in a eBPF map...
		 */
		return process_packet(skb);
	else
		return TC_ACT_OK;
}
```

Let's compile our program with clang/LLVM:

```
$ clang -O2 -emit-llvm -c foo.c -o - | \
	llc -march=bpf -mcpu=probe -filetype=obj -o foo.o
```

Now let's load it. We first need a _qdisc_ to attach the filter to.

```
# tc qdisc add dev eth0 clsact
# tc filter add dev eth0 ingress bpf direct-action obj foo.o sec .text
# tc filter show dev eth0
$ tc filter show dev eth0 ingress
filter protocol all pref 49152 bpf chain 0
filter protocol all pref 49152 bpf chain 0 handle 0x1 foo.o:[.text] direct-action not_in_hw id 11 tag ebe28a8e9a2e747f
```

The eBPF program loaded from `foo.o` appears on the second line of output. Note
the mention of the `direct-action` flag. This program is enough to run both
classification and action selection on the traffic.

Remove with:

```
# tc qdisc del dev eth0 clsact
```

# The Code

The kernel support for the flag was added in commit 045efa82ff56, with the
following log:

```
cls_bpf: introduce integrated actions

Often cls_bpf classifier is used with single action drop attached.
Optimize this use case and let cls_bpf return both classid and action.
For backwards compatibility reasons enable this feature under
TCA_BPF_FLAG_ACT_DIRECT flag.

Then more interesting programs like the following are easier to write:
int cls_bpf_prog(struct __sk_buff *skb)
{
  /* classify arp, ip, ipv6 into different traffic classes
   * and drop all other packets
   */
  switch (skb->protocol) {
  case htons(ETH_P_ARP):
    skb->tc_classid = 1;
    break;
  case htons(ETH_P_IP):
    skb->tc_classid = 2;
    break;
  case htons(ETH_P_IPV6):
    skb->tc_classid = 3;
    break;
  default:
    return TC_ACT_SHOT;
  }

  return TC_ACT_OK;
}
```

In particular, it adds the following chunk to function `cls_bpf_classify()` in
`net/sched/cls_bpf.c` (the boolean `prog->exts_integrated` is set if the
`direct-action` flag was passed):

``` c
if (prog->exts_integrated) {
	res->class = prog->res.class;
	res->classid = qdisc_skb_cb(skb)->tc_classid;

	ret = cls_bpf_exec_opcode(filter_res);
	if (ret == TC_ACT_UNSPEC)
		continue;
	break;
}
```

So with the `direct-action` mode, the `classid` is retrieved from the
`tc_classid` field accessible to the eBPF program through the `struct __sk_buff
*skb` context, instead of being set to the return value from the classifier.
This return value is assigned to the `ret` value instead, and we break from the
loop, whereas if the `direct-action` mode was _not_ used, we would have
executed `ret = tcf_exts_exec(skb, &prog->exts, res);` to call into the
relevant action module and find the action to perform on the packet.

The iproute2 counterpart was added in commit faa8a463002f, and adds the use of
the `da|direct-action` flag on the `tc` command line.

# Conclusion

Hopefully this post helped you understand what this `da` flag appearing on `tc`
commands mean, and why it is relevant for eBPF programs. As of this publishing,
the `direct-action` mode is not only the recommended way to use eBPF programs
with TC, but also the only one that people use in practice, as far as I know.
eBPF is extremely flexible and powerful, and it is only sensible to use it both
for filtering packets and returning the code of the action to perform. It is
also easier (no action to add) and more performant. Use it, and have fun with
TC and eBPF!

# References

* [*On getting tc classifier fully programmable with
  cls_bpf*](http://www.netdevconf.org/1.1/proceedings/slides/borkmann-tc-classifier-cls-bpf.pdf)
  (Daniel Borkmann, netdev 1.1, Sevilla, February 2016)
* Linux kernel commit
  [045efa82ff56](https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=045efa82ff563cd4e656ca1c2e354fa5bf6bbda4):
  _cls_bpf: introduce integrated_ actions
  (Daniel Borkmann and Alexei Starovoitov, September 2015)
* Linux kernel commit
  [1f211a1b929c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=1f211a1b929c804100e138c5d3d656992cfd5622)
  _net, sched: add clsact qdisc_
  (Daniel Borkmann, January 2016)
* iproute2 commit
  [faa8a463002f](https://git.kernel.org/pub/scm/network/iproute2/iproute2.git/commit/?id=faa8a463002fb9a365054dd333556e0aaa022759):
  _f_bpf: allow for optional classid and add flags_
  (Daniel Borkmann, September 2015)
* iproute2 commit
  [8f9afdd53156](https://git.kernel.org/pub/scm/network/iproute2/iproute2.git/commit/?id=8f9afdd531560c1534be44424669add2e19deeec):
  _tc, clsact: add clsact frontend_
  (Daniel Borkmann, January 2016)

---

{% comment %} vim: set syntax=markdown spell tw=79 fo+=a: {% endcomment %}
