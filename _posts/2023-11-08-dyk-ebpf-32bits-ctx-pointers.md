---
layout: post
title:  "Did you know? eBPF 32-bits context pointers"
date:   2023-11-08
author: Quentin Monnet
employer: isovalent
excerpt: Why are eBPF context pointers 32-bit long, when the memory is
  addressed with 64-bit long pointers?
categories: BPF
tags: [BPF]
---

_This post was left aside as a draft for a long time. Most of it was written in August 2022, but it should still be accurate as of its publication._

Networking eBPF programs take a pointer `ctx` to a `struct __sk_buff` (or a
`struct xdp_md`) as their only argument. This struct is a lighter version of
the _socket buffer_, SKB (or an XDP equivalent), that contains some metadata
about the packet to process. In particular, it contains 32-bit long unsigned
integers (`__u32`), `ctx->data` and `ctx->data_end`, pointing respectively to
the beginning and the end of packet data[^linear]. This is convenient for
reading or writing directly from/to packet data when writing eBPF programs (and
this is known as _direct packet access_).

[^linear]: This is a simplification. `ctx->data_end` does point to the end of
    packet data for XDP programs, but not always for programs taking a `struct
    __sk_buff` pointer, because the data does not always fit in a single linear
    buffer (the socket buffer [may contain additional
    data pages](http://vger.kernel.org/~davem/skb.html)).

This post answers to a related question, that I saw some time ago:

> Why are `ctx->data` and `ctx->data_end` 32-bit long pointers? Doesn't that
> ever cause issues at runtime? When the XDP hook invokes the program, and
> constructs the `struct xdp_md`, does it cast addresses to 32-bit unsigned
> integers and if so, isn't there a risk of data loss due to truncation?

(I slightly rephrased the question---Sadly I forgot to take note of its author,
apologies if you read these lines!)

In other words, why use a 32-bit long pointer on 64-bit architectures, and how
can we reliably use it to find the correct address? Spoiler: of course, the
eBPF developers have made it so there is no issue at runtime.

To better comprehend how `ctx->data` works, it is important to dissociate the
32-bit pointer appearing in the C code (and then in the eBPF instructions) from
the actual pointer used inside of the kernel. They are not the same; the 32-bit
pointer is _not_ used at runtime.

What happens instead is that, when the eBPF program is loaded into the kernel,
it goes through the verifier to ensure that it is safe. Among the various
verifications performed at that stage, the verifier checks all accesses to the
program context, as well as to packet data.

Then it converts all eBPF instructions doing such read or write accesses
from/to the context (`ctx->data`, `ctx->data_end`, but also `ctx->data_meta`
and the other existing fields), at verification time. So that at runtime, there
is no cast or additional translation required: the program stored in the kernel
is ready to read at the correct locations.

For a more concrete example, let's consider a networking program receiving a
pointer to a socket buffer. The C code for this program uses the `struct
__sk_buff`:

```c
SEC("classifier")
int prog(struct __sk_buff *ctx)
{
	void *data_end = (void *)(long)ctx->data_end;
	void *data = (void *)(long)ctx->data;
	struct eth_hdr *eth = data;

	if (eth + 1 > data_end)
		return TC_ACT_SHOT;

	/* Block IPv4 traffic */
	if (eth->eth_proto == bpf_htons(ETH_P_IP))
		return TC_ACT_SHOT;

	return TC_ACT_OK;
}
```

But at runtime, the kernel will pass to the program the real `struct sk_buff`.
The conversion from `ctx->data` (from C code) to the real `skb->data` (kernel
object) happened at verification time, when we loaded the program. It was just
a matter of adjusting the offset for the `data` field, from the one in the
`struct __sk_buff` to its offset in `struct sk_buff`. Then once the program is
attached and runs on a packet, `R1`, the first eBPF register, points like
always to the program context, in our case the kernel SKB. When the program
attempts to read at `skb->data`, it looks for the pointer at `R1 +
offsetof(struct sk_buff, data)` (this is simplified) and naturally finds the
64-bit pointer (depending on the architecture, of course) to packet data.

Here is the code from the kernel, in `bpf_convert_access()` in
[`net/core/filter.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/net/core/filter.c?h=v5.19#n9187),
where the conversion happens. It replaces the instruction with a load of the
kernel pointer at `skb->data` into the register:

```c
	case offsetof(struct __sk_buff, data):
		*insn++ = BPF_LDX_MEM(BPF_FIELD_SIZEOF(struct sk_buff, data),
				      si->dst_reg, si->src_reg,
				      offsetof(struct sk_buff, data));
		break;
```

That function is called by the verifier to translate each access to the
context.

As for `data_end`, it is not typically present in the kernel's SKB, but the
pointer address is computed prior to running the eBPF program and stored in
some field of the SKB (somewhere in the _control buffer_). From the [same
function](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/net/core/filter.c?h=v5.19#n9202):

```c
	case offsetof(struct __sk_buff, data_end):
		off  = si->off;
		off -= offsetof(struct __sk_buff, data_end);
		off += offsetof(struct sk_buff, cb);
		off += offsetof(struct bpf_skb_data_end, data_end);
		*insn++ = BPF_LDX_MEM(BPF_SIZEOF(void *), si->dst_reg,
				      si->src_reg, off);
		break;
```

After this translation is done, the real pointers are used at runtime. A
similar translation step happens for XDP programs.

In conclusion, given that this translation happens at verification time, it
doesn't matter whether the pointer is 32 or 64-bit long in the eBPF bytecode.
If you really want the details on why we picked `__u32`, [Alexei's
email](https://lists.linuxfoundation.org/pipermail/iovisor-dev/2016-August/000355.html)
is likely the best explanation available. But in the end, all we need is a
convention, a way to tell the compiler and then the kernel: “Here I want you to
access the `data` pointer from my `ctx`. Please make it work.”

_Direct packet access_ makes it easier to write TC and XDP programs, but
remember to add the boundary checks before reading from your packets, or the
verifier will reject your code. I hope this post helped you understand how the
program context is handled. Have fun!

---
