---
layout: post
title:  "Use cases for BEBA’s stateful packet processing"
date:   2016-08-26
author: Quentin Monnet
employer: 6wind
excerpt: In previous articles I gave an overview of the BEBA research project,
  and then of the OpenState abstraction layer that enables stateful packet
  processing and has been developed as part of this project. I provided a
  simple example, port knocking, to illustrate the concepts involved in this
  design, but port knocking is not an essential network feature. So what would
  one need stateful processing for? You are looking for example use cases?
  Well, this is exactly what we present in this article!
categories: BEBA
tags: [BEBA, SDN]
---

_Please note that even though this article is not endorsed by [the BEBA
Consortium](http://www.beba-project.eu/our-team), its contents may include some
material that is under copyright of the Consortium._

* ToC
{:toc}

# Stateful packet processing

In previous articles [I gave an overview]({{ site.baseurl}}{% post_url
2016-07-15-beba-research-project %}) of the [BEBA research
project](http://www.beba-project.eu/), and then of [the OpenState abstraction
layer]({{ site.baseurl }}{% post_url
2016-07-17-openstate-stateful-packet-processing %}) that enables stateful
packet processing and has been developed as part of this project. I provided a
simple example, port knocking, to illustrate the concepts involved in this
design, but port knocking is not an essential network feature. So what would
one need stateful processing for? You are looking for example use cases? Well,
this is exactly what we present in this article!

Some applications take place at the level of a single switch, to enhance it
with new forwarding or processing capabilities, while some others make it able
to deploy network-wide applications. Of course, describing all the examples
used for the project in full length would be much too verbose for the present
article, so I will provide details just for a couple of them—a subset of the
applications considered for BEBA, which are themselves but some examples among
all applications permitted by the OpenState layer. For more details, just go
for the official deliveries (see [“Additional
resources”](#additional-resources) section), and you will find more complete
descriptions!

# Example use cases

## Forwarding Consistency

Some user applications sometimes require that all their packets be forwarded
along the same path in the network. At the switch level, this means that even
though there may exist several paths leading to the destination host, the
packets of a given flow must always be forwarded on the same output port.

<figure>
  <img src="{{ site.baseurl }}/img/misc/consistency.svg" alt="Switch as a multiplexer"/>
  <figcaption>
    The switch acts as a multiplexer. It differentiates packets on criteria
    other than MAC address and classifies them: all packets of a given class
    must be forwarded to a same port.
  </figcaption>
</figure>

In-switch stateful processing is an efficient way to realize this setup without
involving the controller: if the switch keeps a state for each new flow
encountered, and associates to this state a certain output port, it becomes
able to consistently forward the packets of those flows along the same ports.

## Adaptive QoS management and admission control

More complex setups could associate states so as to monitor the evolution the
traffic and to apply traffic shaping, scheduling, and policing according to the
quality of service (QoS) policy that has been chosen for the network. This
makes it possible to perform actions such as:

* Adjusting the size of the queues, when some queues are overloaded and some
  others are nearly empty.
* Adjusting scheduling or filtering so as to adapt the filling or the
  consumption of the packets in the queues according to the current needs.
* Dropping packets if the queue is full and no other adjustment is possible.
* …

Again, the fact that these adjustments occur in-switch (and do not involve the
controller) allows for a very fast reaction, which is particularly useful in
networks where traffic shape is constantly evolving.

## DDoS mitigation

This one is a very interesting example—and my favorite in the list. The purpose
of this application is to mitigate a distributed denial of service (DDoS)
attack at the switch level. The attack model consists in sending a lot of TCP
SYN packets so as to exhaust the resources of the targeted server. Remember?
The TCP three-way handshake to initiate a connection works like this:

<figure>
  <img src="{{ site.baseurl }}/img/misc/tcp3wh.svg" alt="TCP 3-way handshake"/>
  <figcaption>
    Sequence diagram of the TCP three-way handshake
  </figcaption>
</figure>

So to launch their attack, rogue machines make the server entering state `SYN
received` for as many potential connections as possible, making it allocate
great amounts of resources.

<figure>
  <img src="{{ site.baseurl }}/img/net/ddos.svg" alt="DDoS diagram"/>
  <figcaption>
    Server under distributed denial of service (DDoS) attack
  </figcaption>
</figure>

The first step for defense consists of course in detecting the attack, either
at the controller level or at the switch level, usually by measuring traffic
and by detecting an abnormal rate or number of SYN packets not followed by
ACKs.

When the alert threshold is reached, the controller loads the mitigation
application onto the switches, without unloading the previously established TCP
connections. The concept of the application relies on two restrictions:

  1. The programmable switch accepts only SYN packets as first packet of a
     connection (to prevent distributed RST flood).
  2. The programmable switch drops the first SYN packet of each connection
     (based on source and destination IP and port).

The idea behind the second one is that attackers are usually not waiting for
answers for the SYN packets they send. Therefore they would not notice the
dropping of their packets, and will not bother to send it again (with same
source IP and port). On the other hand, a legitimate client would care for the
absence of answer, and send again a new initiating SYN packet. Since the switch
keeps a state history for TCP flows, it can recognize and allow forwarding of
the second SYN packet, thereby establishing the connection.

<figure>
  <img src="{{ site.baseurl }}/img/misc/tcp3wh_beba.svg" alt="TCP connection sequence diagram with BEBA switch"/>
  <figcaption>
    Sequence diagram of the TCP connection establishement under attack with
    DDoS mitigation scheme
  </figcaption>
</figure>

<br />

<figure>
  <img src="{{ site.baseurl }}/img/net/ddos_mitigation.svg" alt="DDoS mitigation with BEBA switch"/>
  <figcaption>
    DDoS mitigation scheme: legitimate clients resend SYN packets and get an
    answer, attackers do not.
  </figcaption>
</figure>

This mitigation method has a low impact for the end user, especially if it
spares enough resources for the server to answer correctly to legitimate
queries. While the application is run, the number of connections is still
monitored so as to detect the end of the attack and to be able to unload the
mitigation scheme.

Here is what the state machines for the anti-DDoS use case look like:

<figure>
  <img src="{{ site.baseurl }}/img/fsm/ddos_mitigation.svg" alt="DDoS mitigation FSM"/>
  <figcaption>
    State machine of the BEBA switch for the DDoS mitigation use case
  </figcaption>
</figure>

It is quite simple to implement with the BEBA interface, even though the switch
must be able to handle enough states to mitigate a large number of TCP
connection requests (otherwise, the denial of service happens at the switch
level!).

Many more details about the DDoS mitigation use case, including additional
state machines and flow charts, are available in [one of the
deliverables](http://www.beba-project.eu/public_deliverables/BEBA_D5.2.pdf) of
the BEBA project.

# More examples…

Enough! We have seen a small bunch of possible applications already, we will
not go further in this article. Just in case you were interested, here is the
list of the remaining use cases we considered for BEBA… But here provided with
no description whatsoever!

* Adaptive treatment of unexpected traffic
* Advanced packet processing
* ARP handling in datacenter (I should come back on this one in a future
  article)
* DDoS detection and mitigation at network level
* Deep monitoring for (proactive) failure detection
* Distributed, usage-based data rate limiters
* Failure recovery
* Flexible evolved packet core
* In-switch support of legacy control protocols for SDN networks
* Network fault-tolerance
* Programmable network flow measurement

# Additional resources

* [**BEBA**'s list of **public
  deliverables**](http://www.beba-project.eu/dissemination/public-deliverables).
  The most interesting use cases are covered by [deliverable D5.2][D5.2]. Some
  additional use cases are detailed in [deliverable D5.1][D5.1], but some of
  them also use features that have not been presented on this blog (such as the
  need to maintain global or per-flow counters). D5.1 was written at the very
  beginning of the project, while D5.2 is somewhat more focused on the
  possibilities brought by OpenState.

[D5.1]: http://www.beba-project.eu/public_deliverables/BEBA_D5.1_Use%20Case%20and%20Application%20Scenarios_vFinal.pdf
[D5.2]: http://www.beba-project.eu/public_deliverables/BEBA_D5.2.pdf

{% comment %} vim: set syntax=markdown spell tw=79 fo+=a: {% endcomment %}
