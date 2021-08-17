---
layout: post
title:  "eBPF assembly with LLVM"
date:   2020-04-12
author: Quentin Monnet
employer: netronome
excerpt: Clang and LLVM, used to compile from C to eBPF, got support for eBPF
  assembly in version 6.0. Let's have a look at it.
categories: BPF
tags: [BPF, LLVM]
---

* ToC
{:toc}

_This post was left aside as a draft for a long time. Most of it was written in
December 2017. I publish it with the hope it can still be helpful today, even
though the Cilium guide also covers the feature._

_Arthur Chiao has kindly
[translated this post into Chinese (Simplified)](https://arthurchiao.art/blog/ebpf-assembly-with-llvm-zh/),
in case you prefer to read in that language._

One of the most useful evolution of eBPF (_extended Berkeley Packet Filter_)
over the old BPF version (or cBPF, for _classic BPF_) is the availability of a
back end based on clang and LLVM, allowing to produce eBPF bytecode from C
source code.[^gcc]

[^gcc]: As I finish this article, there is now a GCC backend as well, but it
    seems nowhere as complete as the clang/LLVM version, which clearly remains
    the reference tool to produce eBPF bytecode.

# From C to Object File

For example, a simple eBPF program returning zero can be compiled from the
following code:

```
$ cat my_bpf_program.c
int func()
{
        return 0;
}
```

The command line looks like:

```
$ clang -target bpf -Wall -O2 -c my_bpf_program.c -o my_bpf_objfile.o
```

Note: some programs, more evolved than this sample, might need to pass the
`-mcpu` option to `llc` and would use something closer to the following command
instead:

```
$ clang -O2 -emit-llvm -c my_bpf_program.c -o - | \
	llc -march=bpf -mcpu=probe -filetype=obj -o my_bpf_objfile.o
```

This creates an object file in ELF format that contains the compiled bytecode.
By default, the code is under the `.text` ELF section. Let's dump it:

```
$ readelf -x .text my_bpf_objfile.o

Hex dump of section '.text':
  0x00000000 b7000000 00000000 95000000 00000000 ................

```

It worked! We have two eBPF instructions here:

```
b7 0 0 0000 00000000    # r0 = 0
95 0 0 0000 00000000    # exit and return r0
```

If you are not familiar with eBPF assembly syntax, you may be interested in
[this short reference](https://github.com/iovisor/bpf-docs/blob/master/eBPF.md)
(or at
[the complete documentation](https://www.kernel.org/doc/Documentation/networking/filter.txt),
but it is dense).

# Down to the Instructions

Compiling from C to eBPF bytecode as an object file is really useful. The ELF
file produced can be directly used to attach the programs to the various hooks:
TC, XDP, kprobes, etc. Writing advanced programs as bytecode would be very time
consuming, and as I finish to edit this article in 2020, leveraging more
complex features such as
[CO-RE](https://facebookmicrosites.github.io/bpf/blog/2020/02/19/bpf-portability-and-co-re.html)
simply cannot be done by hand. Clang and LLVM are an integral part of the eBPF
workflow.

However, it may not be a convenient solution for someone who needs to test a
very specific eBPF instruction sequence, or to fine-tune a particular aspect of
the program. What if I want to tinker with the instructions? What if I want to
add a third eBPF instruction at the end of my program to set another value to
register r0 after the program exited? Granted, this example is useless, but
still, I should be able to try that!

To do so, we can:

* Write an eBPF program in eBPF bytecode, from scratch. This is perfectly
  doable, but can be long and tedious, and absolutely not user-friendly. And to
  remain compatible with tools like tc, the program has to be turned into an
  object file anyway, which adds an additional step to the process.

* Use an assembly language to write the program from scratch, at least not in
  bytecode, then compile it with a dedicated assembler (for example:
  [`ebpf_asm` in Python by Solarflare](https://github.com/solarflarecom/ebpf_asm)).

As part of recent improvements to LLVM, another solution has emerged, and we
can also:

* Compile from C to an eBPF assembly language. Edit the assembly, then assemble
  it as bytecode in an object file.

Clang and LLVM now allow to do just that! Generating a human-readable version
of the program on one side, then assembling it on the other side. Bonus:
`llvm-objdump` can even be used to dump the program contained in an
object-file.

# eBPF Step by Step with LLVM

To use all clang and LLVM features presented in this section, you need to use
these tools in version 6.0 or higher. It was the development branch when I
started to draft this article, but by the time I finished it version 10 has
been released, so it should not be problematic.

## Compiling from C to eBPF Assembly

Let's use clang to compile the same program as before from C to eBPF assembly.
This is actually done in the same way as you would normally produce assembly
for your processor from C source code, except you tell clang the target is
`bpf`.[^bpf_target]

[^bpf_target]: See the [_BPF and XDP Reference Guide_ from Cilium][cilium] for
    further information on the `bpf` target, and on the differences with the
    default target.

[cilium]: http://docs.cilium.io/en/latest/bpf/#llvm

```
$ cat bpf.c
int func()
{
	return 0;
}

$ clang -target bpf -S -o bpf.s bpf.c
$ cat bpf.s
	.text
	.globl	func                    # -- Begin function func
	.p2align	3
func:                                   # @func
# %bb.0:
	r1 = 0
	*(u32 *)(r10 - 4) = r1
	r0 = r1
	exit
                                        # -- End function

```

Great, now let's modify it and add our instruction at the bottom!

```
$ sed -i '$a \\tr0 = 3' bpf.s
$ cat bpf.s
	.text
	.globl	func                    # -- Begin function func
	.p2align	3
func:                                   # @func
# %bb.0:
	r1 = 0
	*(u32 *)(r10 - 4) = r1
	r0 = r1
	exit
                                        # -- End function

	r0 = 3
```

## Assembling to an ELF Object File

We can assemble this file into an ELF object file containing the bytecode for
this program. It requires the `llvm-mc` tool that deals with machine code and
comes along with LLVM.

```
$ llvm-mc -triple bpf -filetype=obj -o bpf.o bpf.s
```

We have the ELF file! Let's dump the bytecode:

```
$ readelf -x .text bpf.o

Hex dump of section '.text':
  0x00000000 b7010000 00000000 631afcff 00000000 ........c.......
  0x00000010 bf100000 00000000 95000000 00000000 ................
  0x00000020 b7000000 03000000 b7000000 03000000 ................

```

The first two instructions are unchanged in regard to the original version of
the program. The third instructions corresponds to what we added to the
assembly file: it loads 3 into register r0. We successfully edited the
instructions. We could now load the program into the kernel with `bpftool`, for
example. Mission complete!

# A Human-Friendly Output with llvm-objdump

Note that LLVM also provides a way to dump an eBPF object file in a
human-readable fashion (since version 4.0, if I remember correctly). This is
done with `llvm-objdump`:

```
$ llvm-objdump -d bpf.o

bpf.o:	file format ELF64-BPF

Disassembly of section .text:
func:
       0:	b7 01 00 00 00 00 00 00 	r1 = 0
       1:	63 1a fc ff 00 00 00 00 	*(u32 *)(r10 - 4) = r1
       2:	bf 10 00 00 00 00 00 00 	r0 = r1
       3:	95 00 00 00 00 00 00 00 	exit
       4:	b7 00 00 00 03 00 00 00 	r0 = 3
```

We get the assembly instructions, with the syntax used by LLVM (the one we need
to write or edit eBPF assembly, so this is useful). Note the dead instruction
we added at the end of the program.

In addition to the bytecode and assembly instructions, LLVM is able to embed
debug symbols so they can be dumped for inspection. Concretely, we can have the
C instructions at the same time as the bytecode. It comes handy to remember
what your original program was, but most of all it is super helpful to
understand how the C instructions map to the eBPF code. Embedding the
instructions is done by compiling from C with the `-g` flag passed to clang.
Let's give it a try:

```
$ clang -target bpf -g -S -o bpf.s bpf.c
$ llvm-mc -triple bpf -filetype=obj -o bpf.o bpf.s
$ llvm-objdump -S bpf.o

bpf.o:	file format ELF64-BPF

Disassembly of section .text:
func:
; int func() {
       0:	b7 01 00 00 00 00 00 00 	r1 = 0
       1:	63 1a fc ff 00 00 00 00 	*(u32 *)(r10 - 4) = r1
; return 0;
       2:	bf 10 00 00 00 00 00 00 	r0 = r1
       3:	95 00 00 00 00 00 00 00 	exit

 ```

Note that we passed `-g` to `clang`, and that we also changed the command (now
`-S` instead of `-d`) passed to `llvm-objdump`. The `return 0;` instruction
appeared, and is mapped (placed just above) the relevant instructions in the
eBPF program. Nice.

# Inline Assembly

Since clang and LLVM know how to produce and compile eBPF assembly, there is
another way to handle instructions. eBPF assembly can now be used directly
inline in a C program, again to produce specific sequences in the bytecode. See
the example below, somewhat inspired by the example from the [_BPF and XDP
Reference Guide_ from Cilium][cilium].

```
$ cat inline_asm.c
int func()
{
    unsigned long long foobar = 2, r3 = 3, *foobar_addr = &foobar;
    asm volatile("lock *(u64 *)(%0+0) += %1" :
         "=r"(foobar_addr) :
         "r"(r3), "0"(foobar_addr));
    return foobar;
}

$ clang -target bpf -Wall -O2 -c inline_asm.c -o inline_asm.o
$ llvm-objdump -d inline_asm.o
inline_asm.o:	file format ELF64-BPF

Disassembly of section .text:
func:
       0:	b7 01 00 00 02 00 00 00 	r1 = 2
       1:	7b 1a f8 ff 00 00 00 00 	*(u64 *)(r10 - 8) = r1
       2:	b7 01 00 00 03 00 00 00 	r1 = 3
       3:	bf a2 00 00 00 00 00 00 	r2 = r10
       4:	07 02 00 00 f8 ff ff ff 	r2 += -8
       5:	db 12 00 00 00 00 00 00 	lock *(u64 *)(r2 + 0) += r1
       6:	79 a0 f8 ff 00 00 00 00 	r0 = *(u64 *)(r10 - 8)
       7:	95 00 00 00 00 00 00 00 	exit
```

It produces an atomic increment of the value at the address pointed by r2.
Because the instruction was written in the C source, we do not need the
intermediate step of compilation to assembly.

Should we use inline assembly or intermediate compilation? Both methods are
useful in my opinion: insert a couple instructions as assembly in your C source
files, or create small programs for testing specific sequences of instructions.
At Netronome, we have often used the latter for unit tests to check the eBPF
hardware offload feature of the nfp driver.

# Conclusion

In short, instead of compiling an eBPF program from C to an ELF object file,
you can alternatively compile it to an assembly language, edit it according to
your needs, and then assemble this version as the final object file. For this
you need clang and LLVM in version 6.0 and higher, and the commands are:

```
$ clang -target bpf -S -o bpf.s bpf.c
$ llvm-mc -triple bpf -filetype=obj -o bpf.o bpf.s
```

And to dump that file in human-readable format:

```
$ llvm-objdump -d bpf.o
$ llvm-objdump -S bpf.o         # add C code, if -g was passed to clang
```

Additionally, the `asm` keyword can be used to include eBPF assembler inline in
a C program.

eBPF assembly support in LLVM allows one to write any eBPF instruction sequence
they want. Including bugged programs. Do not forget: Even if it compiles, it
still has to pass the verifier. Good luck, and have fun!

---

{% comment %} vim: set syntax=markdown spell tw=79 fo+=a: {% endcomment %}
