---
layout: post
title:  "An introduction to SDN"
date:   2016-07-08
author: Quentin Monnet
categories: BEBA
tags: [SDN]
---

* ToC
{:toc}

# Introduction to Software Defined Networking

So here we are: this is the first article of this blog. As 6WIND's products are
closely related to what is called _Software Defined Networking_, it made sense
to me to start with some basic notions about this topic. Some following
articles should focus on my work on BEBA, a European research project that aims
at making programmable switches “faster and smarter”. But you may wonder: what
do I mean by “programmable” switches? This is precisely one of the questions to
which you will find an answer in this article: here we will see how networking
has evolved over the last years to become (even) more complex, with computers
or switches no more being delimited by their hardware.

## Physical connections: messing up with the wires

Once upon a time… there were computers. At the very beginning, they could not
talk with one another. At [some point during the
1960s](https://en.wikipedia.org/wiki/Computer_network#History), the few people
that would use them figured out that, for reasons that seem quite obvious
today, connecting several computers could be a good idea. And so computer
networks were created! For some decades, setting up a network would basically
consist in plugging cables between pairs of devices; and to connect more than
two hosts together on local networks, hubs or
[switches](https://en.wikipedia.org/wiki/Network_switch) would be required. To
connect several networks between them would be the task of routers.

<figure>
  <img src="{{ site.baseurl }}/img/net/oldnetwork.svg" alt="Traditional network"/>
  <figcaption>
    A traditional wired LAN, with hardware switches
  </figcaption>
</figure>

Actually, correctly setting up a network is way more complicated than that.
This is why so many protocols have been invented and implemented over the
years: making it run, and addressing the requirements in terms of stability,
scalability, security, quality of service and so on… it could require quite a
lot of skills and knowledge from the administrator. Nevertheless, it would come
down to connecting different physical hosts together. Even since [wireless
networks](https://en.wikipedia.org/wiki/Wireless_network) have been deployed,
this has remained a basic principle.

## Enters virtualization

[Virtualizing hardware](https://en.wikipedia.org/wiki/Virtualization) is not
something new, it was in use back in the 1960s to divide the system resources
provided by mainframe computers between different applications. But it has
evolved a lot with the years. As hardware resources became more affordable and
computers more powerful in the 2000s, it became possible to emulate entire
operating systems along with their desktop applications. It soon became a
considerable opportunity for the industry, and the use of this technology has
been increasing ever since. Deploying virtual applications, either inside
virtual machines or in logical containers, has several advantages indeed:

* It enables faster deployment, with reproducible builds (the emulated
  environment is easier to reproduce on every instance than it would be on
  different physical hosts with distinct hardware).
* It is more scalable, and much more “agile”: if the virtual applications
  running over a physical machine are too demanding in resources, some of them
  may be moved onto another physical host. It makes it possible to ensure an
  optimal consumption of resources for all running physical hosts.
* Regarding security, it is usually interesting to isolate sensitive
  applications inside a system that has no access to other sensitive data, and
  that can be reinitialized easily.
* …

Of course, with the democratisation of Internet and of web services, and the
rise of cloud services, those virtual machines would need to be connected
between themselves as well as to a physical network. That would be the birth of
virtual switches.

<figure>
  <img src="{{ site.baseurl }}/img/net/virtualizednetwork.svg"
    alt="Software defined network"/>
  <figcaption>
    Network now also includes “<acronym title="Virtual Machine">VM</acronym> to
    <acronym title="Network Interface Controller">NIC</acronym>” and “VM to VM”
    traffic
  </figcaption>
</figure>

Virtual switches usually run on top of normal computers and
servers instead of dedicated hardware. This also enables them to take advantage
of the complex network stacks of the system acting as an hypervisor (e.g.
Linux' or \*BDS's stacks); and because it is usually more complex than the
stack available on dedicated hardware, virtual switches were made able to
address more and more complex requirements for packet processing operations, to
the detriment of speed performance.

And virtualization keeps growing. But simple switches are no longer able to
meet the industry's requirements.

## Software Defined Networking (SDN)

Since networks have been increasing a lot in size and in requirements, moving
around hardware switches has become a burden. Even manually setting up
individual software switches has become a complicated and error-prone task for
companies running strongly virtualized environments along with large networks.
This is where [Software Defined
Networking](https://en.wikipedia.org/wiki/Software-defined_networking)—SDN
for short—enters the game, in the early 2010s.

“SDN” is a bit like “cloud computing”: it is a trendy topic in computing, and
it is often sold as a “miracle” solution to all network infrastructure
problems. But beside marketing, it also corresponds to a new type of network
architecture. “Software defined” does not mean that it only uses virtual
switches instead of dedicated hardware (even if it mostly does): it refers to
the fact that switches can be _programmed_. Their behavior is defined by a
_software_ configuration. Indeed the main feature of SDN is that the switch
control plane is decoupled from the data plane. What does this mean?

* The **control plane** is where the administration of the network takes place:
  it corresponds to the setting up of the packet processing rules, and from
  there to the establishment of the whole network switching policy.
* **Data plane** encompasses the application of those rules defined on control
  planes: this is the actual packet processing. When some packets require some
  particular, more complex processing, they can be handled to the control
  plane, where the decision regarding this packet will occur.

As a generic principle, the data plane must be very fast to handle a very high
number of packets with simple rules so as to obtain a good bit rate in the
network. By contrast, the control plane is slower. For legacy switches, the
whole control and data planes would be tightly linked into a same hardware box.
They would not be clearly delimited, and the switch would usually present a
single setup interface. But virtual switches offer new possibilities: they make
it doable to _centralize_ the management of the switches and to remotely
program them. The control plane is organized in such a way that switches take
orders from a centralized controller in the network. This controller
dynamically installs packet processing rules onto the switches, and receives
and (slowly) handles exception packets when needed.

<figure>
  <img src="{{ site.baseurl }}/img/misc/sdn.svg" alt="SDN stack"/>
  <figcaption>
    SDN architecture concept
  </figcaption>
</figure>

If the controller is to be represented at the center of a diagram, it is
attached to network-requiring applications—on top—by so-called _northbound
APIs_, while it is linked to the data plane—below—by what are called
_southbound APIs_, used both to set up the switches configuration and to handle
the exceptions packets.

So this is what SDN looks like! Or… what it used to be in the first years. In
fact, the distinction between control and data plane should be considered as an
“implementation choice” of SDN, and not as a definition of SDN by itself. And
as technology evolves, so does the architecture. Now the switching equipments
often include separated components, thus creating an architecture with three
levels:

* “Control-management” plane is where the administration takes place, this is
  the controller configuring the remote switches.
* “Control-plane APIs” of a programmable switch can be used to form a component
  that both receives setup instructions from the controller, _and_ is able to
  somewhat perform simple updates of the data plane without always referring to
  the controller.
* Data plane, as before, handles fast packet processing.

Some other architectures have also been designed; for example, the Neutron
component of the [OpenStack](https://www.openstack.org/) suite embeds both the
controller and the data plane. They remain logically decoupled, but they run on
the same host.

## The expected benefits of SDN

This new architecture model provides a way to programmatically configure the
switches at runtime, and to manage the network resources in a more efficient
way. It enables the network operator to provide bandwidth ”on demand”. As
virtualization made application development much more flexible and scalable,
the same thing is occurring to networks with SDN.

Another advantage resides in the fact that SDN usually involves vendor-neutral
technology: the software and protocols in use are often open-source, and in any
case, as there is a move from hardware to software, there is no risk to get
locked in on the technical side because of vendor-specific hardware
requirements.

Also, since it is less hardware-dependant, and because all the switches need no
more embed the control part, this paradigm is expected to lower the barrier to
entry of the networking industry for new actors.

Because of those properties, SDN is becoming an essential concept to deploy
cloud services, or to incorporate new paradigms such as
[BYOD](https://en.wikipedia.org/wiki/Bring_your_own_device) (_Bring Your Own
Device_) or [IoT](https://en.wikipedia.org/wiki/Internet_of_Things) into
industrial networks. These concepts represent new network usages, and they come
with increasing needs for bandwidth: it seems today that some of these needs
can only be addressed by Software Defined Networks indeed.

## Challenges

Along with SDN, new challenges have emerged. The basic functionalities of
programmable switches have become somewhat independent from the hardware in
use, so the software part must provide efficient switching capabilities. New
algorithms or protocols (think about the way the controller should configure
the switches, for example) had to be designed, both for the control plane and
the data plane.

But even with new software tools, the functionalities of the data plane remain
at a basic level, so as to gain on processing speed. But this has a cost in
terms of available features, and of ease of use: indeed the addition of a new
feature (new protocol, modified network topology, …) can require an upgrade of
all data planes, thus representing a heavy constraint on production
environments. Therefore one of the challenges consists in developing data
planes with high performances but presenting a powerful programmable,
“updatable” interface. Good software design is of paramount importance!

Hardware is not completely put aside, though. It is mandatory to interface the
software side with the hardware cards in an efficient way to obtain good
performances. And getting good performances is one of the principal objectives
of SDN! Performances for bit rates, but also for resources consumption—the more
CPUs remain available to user applications, the better—or even for other
subsystems such as storage: higher throughputs mean more data, which in turn
must be forwarded to fast and efficient storage backends.

Another huge challenge of SDN is security. The network topology evolves: the
basic architecture gives way to a decoupling of control and data planes. This
new architecture makes it even more feasible and easy to update the network
topology at runtime. This, in turn, makes network components harder to secure
and to monitor. In particular, it is essential that commands on the control
plane remain protected. And the use of virtualization makes things even worse:
when several appliances run on a same physical host, they must share its
resources but must not leak their data. There is a lot of ongoing research on
this subject—because much remains to be done!

# Around SDN

Here are a couple of concepts and technologies that are tightly linked to SDN.

## Network Function Virtualization (NFV)

Network Function Virtualization is an alternative to SDN. It is another network
services architecture based on virtualization. Instead of focusing on the
separation between control and data plane, it places the network
functions—routing, firewalling, load balancing, etc.—inside virtual
machines that act as some sort of “boxes”. Those blocks can be assembled or
chained to compose the whole network architecture. Note that sometimes, we also
hear about Virtual Network Function (VNF), but this seems to be [just another
name for
NFV](https://www.sdxcentral.com/nfv/definitions/virtual-network-function/).

SDN and NFV are two “novel designs” for network services. And yet, they are
[not two concurrent visions, but rather complementary
solutions](https://www.sdxcentral.com/nfv/resources/which-is-better-sdn-or-nfv/):
SDN takes in charge the automation of the network, implementing policy-based
forwarding to establish dynamic and resilient traffic rules, while NFV focuses
on the services that are to be provided over the network, enabling a good
provision and distribution of the virtualized resources.

So NFV and SDN are very close concepts, both relying on virtualization to
provide on-demand and scalable network architectures; and concepts or
mechanisms that can be used with one of them can often be used for the other as
well. Finally, it is worth noting that some articles or presentations (mostly
for marketing) will focus only on NFV even though the principles could also
apply to SDN—or the other way round.

## Specific technologies

The obvious technology involved with SDN are related to virtualization and to
networking. About virtualization, the usual hypervisors are used. they are
either proprietary software, such as VMWare technology, or open-source, as for
Qemu. Technologies in use for the deployment of “cloud” architectures, such as
[OpenStack](https://www.openstack.org/), are often closely related to
virtualization technologies, and of course to SDN.

But the main point here is about networking technologies. Besides encapsulation
protocols such as VXLAN, GRE or MPLS, that are often used in data planes to
deploy flexible architectures, there is an essential protocol which is used to
link control to data plane: this is OpenFlow®. OpenFlow, managed by the [Open
Networking Foundation (ONF)](https://www.opennetworking.org/) and currently in
its [1.5.1 version][spec 1.5.1] (published in March 2015), is used between the
controller and the switch. The controller uses OpenFlow messages to configure
or to monitor the software switches. It is an implementation of the
aforementioned southbound API. This protocol is considered as a historical
basis for SDN, and as of today, it is part of most deployed SDN architectures.

As for the virtual switches themselves, it seems that the main software in use
is [Open vSwitch](http://openvswitch.org/), an OpenFlow-compliant software
switch with additional extensions from Nicira (VMWare).

[spec 1.5.1]: https://www.opennetworking.org/images/stories/downloads/sdn-resources/onf-specifications/openflow/openflow-switch-v1.5.1.pdf

On the controller side, many solutions exist. There are large frameworks such
as [OpenDaylight](https://www.opendaylight.org/) platform, or simpler
controllers such as [Ryu](https://osrg.github.io/ryu/) (Python)… and many
others.

## Standardization

There are a couple of organisms in charge of developing and managing standards
for the SDN universe.

The [Open Networking Foundation (ONF)](https://www.opennetworking.org/), as
stated above, handles the specification of OpenFlow protocol.

The [European Telecommunications Standards Institute
(ETSI)](http://www.etsi.org/) work and develop industrial standards on a
variety of topics related to telecommunications and networks, including SDN.
Also, after intensive work sessions, they sometimes have “beer” events with
excellent vodka, or so I heard; but I am afraid this might bring us off-topic,
so let's focus on the standardization part…

The [Open Platform for NFV Project (OPNFV)](https://www.opnfv.org/) is a
platform supported by the Linux Foundation and develops material for NFV, so as
to provide improved _Virtualized Network Functions_ (VNF) and _Management and
Network Orchestration_ (MANO) components.

# All in all…

Software Defined Networking, along with Network Function Virtualization, are
two new paradigms that heavily rely on virtualization to move networking
services and architecture from hardware to software. For wide architectures, in
datacenters and networks of telecommunication or cloud-based services
operators, _baremetal_ switches are being abandoned for _virtual_ ones. Thanks
to automation of policy-based forwarding and to dynamic resources allocation,
they scale much better and provide a better use of resources, while making
deployment, configuration and monitoring of the services much easier. They are
mostly based, for the network components, on open-source technologies, keeping
away the constraints associated to locked-in proprietary solutions.

There has been a lot of activity on SDN ever since it appeared in early 2010s,
and there is still much to work on… Providing by the way some matter for more
blog articles!

# Additional references

* [SDxCentral](https://www.sdxcentral.com) is a website gathering information
  and news about SDN. It is often related to events or marketing campaigns from
  companies working in this field (including
  [6WIND](https://www.sdxcentral.com/listings/6wind/)), but they also have some
  technical resources about
  [SDN](https://www.sdxcentral.com/flow/sdn-software-defined-networking/) or
  [NFV](https://www.sdxcentral.com/nfv/resources/whats-network-functions-virtualization-nfv/).
* [RFC 7426](https://tools.ietf.org/html/rfc7426) is an informational document
  from the IETF, and defines _Software-Defined Networking (SDN): Layers and
  Architecture Terminology_. It may not be the easiest introduction to SDN, but
  it contains precise definitions of the terms in use in the domain.
* The [Open Networking
  Foundation](https://www.opennetworking.org/sdn-resources/sdn-definition)
  (ONF) has some resources as well, including [the specification of OpenFlow
  protocol](https://www.opennetworking.org/sdn-resources/technical-library#tech-spec).
* Wikipedia, as usual, has a number of articles related to [SDN
  technologies](https://en.wikipedia.org/wiki/Software-defined_networking).
  Some parts of the article about SDN itself are strangely similar to what is
  available on the ONF website.
* … aaaand as I work at 6WIND, of course I encourage you to have a look at the
  resources available on our website! There are both technical and commercial
  information about
  [SDN](http://www.6wind.com/solutions/software-defined-networking/),
  [NFV](http://www.6wind.com/solutions/network-function-virtualization/) or
  [technical
  solutions](http://www.6wind.com/resources/white-papers-presentations/). Also,
  if you love SDN already and would like to push packets much faster into the
  pipe, **check our [job
  offers](http://www.6wind.com/company-profile/careers/)!**

{% comment %} vim: set syntax=markdown spell tw=79 fo+=a: {% endcomment %}
