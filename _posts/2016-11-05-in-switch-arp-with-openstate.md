---
layout: post
title:  "In-switch ARP handling with OpenState"
date:   2016-11-05
author: Quentin Monnet
categories: BEBA
excerpt: ARP is a basic component of most IPv4 networks, where it is used by
  the machines that want to retrieve the MAC address of a recipient possessing
  a given IP address. Of course, it is often implemented in Software-Defined
  Networks (SDN)… where it comes with a number of disadvantages. And _of
  course_ we can improve it with the mechanisms designed as part of the BEBA
  research project!
tags: [BEBA, SDN, ARP]
---

_Please note that even though this article is not endorsed by [the BEBA
Consortium](http://www.beba-project.eu/our-team), its contents may include some
material that is under copyright of the Consortium._

* ToC
{:toc}

**ARP**! That is what we talk about here. ARP is a basic component of most IPv4
networks, where it is used by the machines that want to retrieve the MAC
address of a recipient possessing a given IP address. Of course, it is often
implemented in Software-Defined Networks (SDN)… where it comes with a number of
disadvantages. And _of course_ we can improve it with the mechanisms designed
as part of the [BEBA research project](http://www.beba-project.eu/)!

This article is a follow-up of [the description of BEBA]({{ site.baseurl }}{%
post_url 2016-07-15-beba-research-project %}), then [the description of
OpenState]({{ site.baseurl }}{% post_url
2016-07-17-openstate-stateful-packet-processing %}), and even to the [example
applications]({{ site.baseurl }}{% post_url
2016-08-26-stateful-processing-use-cases %}) of this abstraction layer. ARP
handling is yet another possible use case for OpenState. Let's see how it can
improve your network!

# The basics

ARP has been used for years in IPv4 networks in order to make the association
between an IP and a MAC address: when a host wants to send a packet to a given
IP (L3) address, but does not know what the corresponding MAC (L2) address is,
it broadcasts an ARP request to query all the hosts it can reach. When the host
with the targeted IP address receives the request, it replies with a unicast
message to provide its MAC address. With this knowledge, the initiator of the
query can then send IP packets to their next hop.

<figure>
  <img src="{{ site.baseurl }}/img/net/arp_old.svg" alt="ARP — Normal functioning"/>
  <figcaption>
    Classic ARP example, with broadcast diffusion of the requests
  </figcaption>
</figure>

# ARP handling in datacenter

ARP works very well on the whole[^1], but depending on the context, it may be
preferable not to deploy it. In a software-defined network, the broadcast
queries can make it mandatory for the controller to be able both to handle
broadcast and to prevent loops in the network, for example by implementing
<acronym title="Spanning Tree Protocol">STP</acronym>. Worse, in overlay
networks with L2 over IP, the size of the L2 network is no more limited by
geographical constraints: three virtual machines, for example, can be in three
different locations in the world. The “LAN” is no more “local”, and yet ARP
remains necessary to ensure L2 communication. But broadcasting requests in such
conditions gets more expensive in resources, and the resolution time increases
a lot.

[^1]: ARP has some issues, such as—regarding security—vulnerability to
      [ARP cache poisoning attacks](https://en.wikipedia.org/wiki/ARP_spoofing).
      They are not covered by this article.

So “classic” ARP can add an unnecessary layer of complexity (or even latency),
that can be prevented if ARP requests can be answered by an intermediary agent
instead of systematically being broadcast all over the network. In such a
setup, an ARP responder is responsible for generating the replies to the
unicast queries from the hosts.

<figure>
  <img src="{{ site.baseurl }}/img/net/arp_sdn.svg" alt="ARP — With SDN controller"/>
  <figcaption>
    ARP with SDN controller acting as an ARP resolver
  </figcaption>
</figure>

In SDN networks, this traditionally translates into intermediary switches
acting as responders and having to refer to the controller—which is the one to
know the association between L2 and L3 addresses—whenever they receive an ARP
query.

# A use case for BEBA

But with OpenState, there is yet a simpler solution. Indeed the switch may
already be aware of the MAC address of a given host, if it keeps state memory,
and has already seen a recent packet with both MAC and IP addresses from this
host (i.e. any IP packet). In this case, the switch directly answers to the
requesting host: there is no broadcast, or no query to send to the controller.

<figure>
  <img src="{{ site.baseurl }}/img/net/arp_beba.svg" alt="ARP — With BEBA switch"/>
  <figcaption>
    ARP performed in-switch thanks to stateful processing
  </figcaption>
</figure>

It sounds nice! But, well, I lied: that was an overly simplified version.
Actually the controller remains involved a little in ARP resolution even with
BEBA scheme, and the BEBA switch does not obtain the IP and MAC address from
the IP packet. Instead, what really happens is:

1. The controller is still the one entity that knows about IP ↔ MAC
   association. But it does not respond to queries from the hosts itself.
2. Instead, when it learns about a new association (when one switch sends up a
   notification about a network change for example), it installs a _Packet
   Template Entry_ (PTE) as well as _Flow Table Entries_ (FTEs) into the
   switches, both relying on OpenState.
3. When receiving an ARP query from neighbor hosts:
      * Either the switch does not have the associated PTE and FTEs, and it
        asks for them to the controller.
      * Or they already received the correct information and can use it to
        answer to the querier.

How do those PTEs and FTEs work? A Packet _Template_ Entry is, obviously, a
template. And as such it is used in order to generate flows. These templates
are of the following shape. Considering that the PTE is used to associate IP
address 10.1.1.1 to MAC address aa:bb:cc:dd:ee:ff, the template “says”,
roughly:

> Create an ARP reply to MAC XX:XX:XX:XX:XX:XX and IP XX.XX.XX.XX stating that
> IP 10.1.1.1 is at MAC address aa:bb:cc:dd:ee:dd.

It is triggered by a FTE, which matches flows and instructs the following:

> For any new ARP request received from host with IP address XX.XX.XX.XX and
> concerning IP 10.1.1.1, create an ARP answer with the correct PTE (above) and
> sends it back to the querying host.

Technically, this involves copying the values from the (MAC and IP) address
fields to fill in the template. A more formal representation of a **PTE** would
look like this:

|------|----------+----------------------------+
| ID   | Template | Copy operations to perform
|------|:---------|:---------------------------|
| 1337 | Ether (aa:bb:cc:dd:ee:ff, X1)<br />ARP (reply, aa:bb:cc:dd:ee:ff, 10.1.1.1, X2, Y2) | Copy MAC source address in X1<br />Copy MAC source address in X2<br />Copy ARP IP source address in Y2
|------|----------+----------------------------+
{: .decorated-table }

And this PTE would be triggered by the following **FTE**s:

|-------------------------------+-----------------+
| Matching pattern              | Actions
|:------------------------------|:----------------|
| Ether (\*, broadcast)<br />ARP (request, \*, \*, 10.1.1.1) | GENERATE\_FROM\_PTE(1337),<br />OUTPUT (flow\_table)
| Ether (\*, aa:bb:cc:dd:ee:ff) | OUTPUT(port\_n)
|-------------------------------+-----------------+
{: .decorated-table }

So if we take it in chronological order: the controller installs the PTEs and
FTEs into the switches. When an ARP request reaches a BEBA switch, the latter
searches its flow tables for an associated FTE. When a pattern match (i.e. if
the packet _is_ an ARP request, for a known IP address), the relevant FTE calls
its corresponding PTE to generate the ARP reply, by completing the template
with the IP and MAC addresses of the _querier_ (addresses of the ARP _target_
are part of the template, contained in the PTE). Then this ARP reply is sent
back to the host.

In this way, the controller is not involved in _each_ ARP request that is sent,
as it would if it was to assume the role of a simple ARP resolver as on the
second diagram. This makes ARP deployment much more scalable, and limits the
amount of ARP paquets transiting through the network. A nice use case for
BEBA's OpenState!

# Additional resources

* As for all other use cases for designs developed for BEBA, [the project's
  list of public
  deliverables](http://www.beba-project.eu/dissemination/public-deliverables)
  is the reference. For this use case, see in particular
  [deliverable D5.2][D5.2].
* The mechanisms to generate packet “in-switch” and to handle FTEs and PTEs
  are actually part of the InSPired switch design, also part of BEBA project:
  [_Improving SDN with InSPired
  Switches_](http://conferences.sigcomm.org/sosr/2016/papers/sosr_paper42.pdf),
  Roberto Bifulco, Julien Boite, Mathieu Bouet and Fabian Schneider, 2016.

[D5.2]: http://www.beba-project.eu/public_deliverables/BEBA_D5.2.pdf

---

{% comment %} vim: set syntax=markdown spell tw=79 fo+=a: {% endcomment %}
