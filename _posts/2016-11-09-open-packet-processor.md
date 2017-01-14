---
layout: post
title:  "Open Packet Processor"
date:   2016-11-09
author: Quentin Monnet
categories: BEBA
excerpt: BEBA research project has a strong focus on in-switch stateful packet
  processing for SDN networks. A first abstraction layer for state management
  and efficient packet switching was made available through the OpenState
  layer. This article presents an advanced concept that extends OpenState to
  provide additional functionalities, as well as better performances (in
  particular for hardware implementations)—please meet Open Packet Processor.
tags: [BEBA, SDN]
---

_Please note that even though this article is not endorsed by [the BEBA
Consortium](http://www.beba-project.eu/our-team), its contents may include some
material that is under copyright of the Consortium._

* ToC
{:toc}

[BEBA research project]({{ site.baseurl }}{% post_url 2016-07-15-beba-research-project %})
has a strong focus on in-switch stateful packet processing for SDN networks.
A first abstraction layer for state management and efficient packet switching
was made available through the
[OpenState layer]({{ site.baseurl }}{% post_url 2016-07-17-openstate-stateful-packet-processing %}).
This article presents an advanced concept that extends OpenState to provide
additional functionalities, as well as better performances (in particular for
hardware implementations): please meet Open Packet Processor.

# A successor to OpenState

## OpenState lacks simple registers

Open Packet Processor (_a.k.a._ OPP) is an advanced abstraction layer, still
providing stateful packet processing facilities. In fact its design heavily
relies on OpenState, and extends with two main additions: registers, and
Boolean conditions on those registers.

Those registers are used to store values. For instance, they can be used as
counters: it is especially useful to have counters for packet switching. They
can be used to measure a number of packets, of connections, to collect
statistics and so on. Advanced features such as service monitoring or attack
detection, for example, make an extensive use of it. With OpenState, we have
Mealy state machines that make it possible to use counter-based processing, but
each possible value of the counter must be associated to a new state.

<figure>
  <img src="{{ site.baseurl }}/img/fsm/counter.svg" alt="No counters in OpenState"/>
  <figcaption>
    To emulate counters with OpenState, a succession of state is needed.
  </figcaption>
</figure>

As each of those states must in turn be stored as a new entry into an XFSM
table (please see [the description of OpenState]({{ site.baseurl }}{% post_url
2016-07-17-openstate-stateful-packet-processing %}) for details), the memory
footprint increases very fast. And also, this may quickly become a pain to
manage. Also, performing arithmetic and logical operations on those values tend
to represent a non-trivial challenge.

## Registers and Boolean condition

So instead of having an enormous amount of states to deal with counters, Open
Packet Processor implements registers that can be used to store values. In a
way, they are used like “variables” for traditional programs. There are two
types of registers, and both can be updated with algorithmic and logical
operations:

* Flow-based registers, that are associated to a given flow and retain a value
  related to the packets of this particular flow.
* Global registers, that are used as variables in which values for metrics
  related to the global traffic can be stored.

Of course, we do not store values for fun. So to actually draw benefits from
those registers, we need at some point to evaluate their content and to check
whether or not they verify some conditions, so that we know how to process the
next packets. With Open Packet Processor, this is implemented as Boolean
conditions, based on the current values of a subset of registers, that can be
evaluated at any state transition.

Now that we have registers and conditions, let's see how it translates into a
simple use case. Imagine we have an application in which we want to forward the
first five packets on UDP port 1234. With the previous model, OpenState, we
need to chain 6 states to do so.

<figure>
  <img src="{{ site.baseurl }}/img/fsm/drop5_OS.svg" alt="Example with OpenState"/>
  <figcaption>
    Open State example: Drop after first five packets on UDP port 1234<br />
    (Not represented: packets other than UDP on port 1234 do not trigger any
    change of state)
  </figcaption>
</figure>

By contrast, with Open Packet Processor, we can use a register r<sub>0</sub>,
and we only need two states. The register r<sub>0</sub> is initialized at 0,
and incremented each time a packet is received on UDP port 1234. For each of
those packet, the Boolean condition “_Is the value of r<sub>0</sub> superior to
5?_” is evaluated; if the answer is _true_, we change state and stop forwarding
these packets. If, on the contrary, it is _false_, we remain in the same state.

<figure>
  <img src="{{ site.baseurl }}/img/fsm/drop5_OPP.svg" alt="Example with OpenState"/>
  <figcaption>
    Same example with Open Packet Processor: only two states needed
  </figcaption>
</figure>

# Extended finite state machines

As a reminder, OpenState uses a Mealy machine, an abstract structure comprising
a 5-tuple (S<sub>0</sub>, S, I, O, T), where:

* S is a finite set of states.
* S<sub>0</sub> is an initial starting state S<sub>0</sub>, belonging to S.
* I is a finite set of input symbols (events).
* O is a finite set of output symbols (actions).
* T : S × I → S × O is a transition function mapping (state, event) pairs into
  (state, action) pairs. For “standard” Mealy machines, these transitions would
  be decoupled into two functions T : S × I → S and G : S × I → O.

With Open Packet Processor, we use an *eXtended Finite State Machine* (XFSM)
extending the former model. This time, it is a 7-tuple (S, I, O, D, F, U, T),
where:

* S remains a finite set of states.
* I remains a finite set of input symbols (events).
* O remains a finite set of output symbols (actions).
* D = D<sub>1</sub> × D<sub>2</sub> × … × D<sub>n</sub> is a _n_-dimensional
  linear space (set of registers).
* F is a set of functions f<sub>i</sub> : D → {0, 1} (Boolean conditions on
  registers).
* U is a set of functions u<sub>i</sub> : D → D (register update functions).
* T : S × I × F → S × O × U is a transition function mapping (state, event,
  Boolean conditions) tuples into (state, action, register update) tuples.

This is exactly what we saw in the example above: states, and actions triggered
on events. Plus registers, and on each event, the possibility to evaluate
Boolean conditions on those, such that the result of this evaluation can affect
the resulting state, action to perform, and may lead to a register update.

Concretely, this abstract model is implemented in the switch in the form of
state tables and XSFM tables in a way similar to OpenState. We will not recall
the details here, please refer to the additional resources at the bottom of
this article for details.

# Runs with TCAMs

## Architecture

So now we have a model to perform stateful processing. Fine. But why did we
choose such a low-level model in the first place, instead of defining some
higher-level API? The answer is simple: one essential strength of OPP is that
it is particularly well adapted to run on hardware.

Indeed, the simplicity of the design makes it very close to hardware
capabilities, while the expressiveness enabled by the use of registers and on
conditions makes it almost as flexible as any ordinary programming language.
This sounds promising: OPP provides a way to easily implement any stateful
application, while maintaining a low number of CPU cycles for processing the
packets! Concretely, the layer is ideal for being implemented on a
[FPGA](https://en.wikipedia.org/wiki/Field-programmable_gate_array) that will
act as the network processing unit for the system.

<figure>
  <img src="{{ site.baseurl }}/img/misc/OPP_archi.svg" alt="OPP architecture for FPGA"/>
  <figcaption>
    OPP architecture — Copyright BEBA consortium, 2016. All rights reserved.
  </figcaption>
</figure>

Indeed:

* Per-flow registers are stored in a RAM block, in a flow context table, along
  with pattern used for flow matching.
* Global registers are also stored in a RAM block.
* The XFSM table can be implemented as a
  [TCAM](https://en.wikipedia.org/wiki/Content-addressable_memory#Ternary_CAMs).
  Conditions on the registers and event matching can use wildcards. The table
  returns the next state on which the transition is to end, the action to
  perform (drop, forward…), and a set of custom algorithmic and logics
  instructions used to update the registers.
* The update logic block is a parallel array of ALUs: it executes the
  microinstructions returned by the TCAM in order to update the registers.

In fact, the model uses the TCAM as the processing unit: it kind of replaces a
traditional processor. This design presents the advantage of being
platform-agnostic: the packet processing is guided by the model architecture,
with the TCAM performing state transitions, and does not rely on any specific
CPU architecture.

At this point, only the essential question remains: what about performances? I
have not run tests myself, but [the OPP
article](https://arxiv.org/abs/1605.01977v1) published by people from CNIT—who
did implement OPP on a FPGA—says:

> The system latency, i.e. the time interval from the first table lookup to the
> last context update is 6 clock cycles. The FPGA prototype is able to sustain
> the full throughput of 40 Gbits/sec provided by the 4 switch ports.

Not bad.

## Processing network flows with FPGAs

We have seen that the goal of OPP is to provide very fast and flexible stateful
packet processing and programmable actions. The reference use case for this
design is, of course, network switching. In the context of BEBA project, OPP
has been successfully implemented on a
[COMBO-100G](https://www.liberouter.org/technologies/cards/) card, with good
performances (see citation above).

But beside switching, it is interesting to note that the OPP layer can also
apply to more generic network processing units (NPUs), at a time when there is
a rising demand for so-called “SmartNICs”. Indeed, a variety of FPGA-based
network products have been hitting the market over the last few years, and
their number keeps growing. [It is
expected](https://www.nextplatform.com/2016/01/27/offloading-the-network-like-a-hyperscaler/)
that more and more SmartNICs and CPUs will combine FPGAs or other hardware
packet processors—the estimate in the article is that more than 500,000 servers
a year are to be equipped with such devices in the coming years—thus opening
solutions for “smart” packet processing on the <acronym title="Peripheral
Component Interconnect">PCI</acronym> boards as an alternative to the (less
scalable) virtual switches running on dedicated CPU cores.

Would you like a few example names, maybe? Here are some available or future
concepts of CPU-based PCI <acronym title="Network Interface
Controller">NIC</acronym>s, in no particular order:

* [Cavium LiquidIO](https://www.cavium.com/LiquidIO_Adapters.html)
  (“Application acceleration adapters”),
* [Intel combined FPGA (from Altera, bought in 2015) + Xeon processors](http://itpeernetwork.intel.com/accelerating-open-innovation-for-emerging-cloud-workloads/)
  (it seems Intel
  [started to ship a development module](http://www.eweek.com/servers/intel-begins-shipping-xeon-chips-with-fpga-accelerators.html),
  but I could not find references about a product yet),
* [Konic80 for Kalray](http://www.kalrayinc.com/kalray/products/#platforms)
  (CPU-based accelerators),
* [Mellanox
  BlueField](https://www.mellanox.com/page/products_dyn?product_family=256)
  (Network-oriented multicore <acronym title="System on a
  Chip">SoC</acronym>s),
* [FPGA + ConnectX from
  Mellanox](https://www.mellanox.com/related-docs/prod_adapter_cards/PB_Programmable_ConnectX-3_Pro_Card_EN.pdf)
  (The FPGA is attached to a programmable network adapter device),
* [Napatech FPGAs](http://www.napatech.com/products/accelerators) (A series of
  FPGA-based network accelerators),
* [Netronome Agilio](https://www.netronome.com/products/agilio-cx/) (“Server
  adapters” based on cards with multiple hardware accelerators—“Flow Processing
  Cores”, “Packet Processing Cores” etc.),
* …

Hardware programmable switching seems to be in fashion, and Open Packet
Processor is definitely meant to be part of it!

# Additional resources

* All details, including background information, design, implementation,
  experiments, limitations and future work for OPP, are exposed in the related
  scientific publication by researchers from CNIT: [_Open Packet Processor: a
  programmable architecture for wire speed platform-independent stateful
  in-network processing_](https://arxiv.org/abs/1605.01977) Giuseppe Bianchi,
  Marco Bonola, Salvatore Pontarelli, Davide Sanvito, Antonio Capone, Carmelo
  Cascone — 2016.
* The model is also described in one of the project's deliverables,
  [D2.3](http://www.beba-project.eu/public_deliverables/BEBA_D4.3.pdf),
  accessible from [BEBA's list of public
  deliverables](http://www.beba-project.eu/dissemination/public-deliverables).
* [Extended finite state
  machine](https://en.wikipedia.org/wiki/Extended_finite-state_machine) on
  Wikipedia.

{% comment %} vim: set syntax=markdown spell tw=79 fo+=a: {% endcomment %}
