---
layout: post
title:  "OpenState: an abstraction design for stateful packet processing"
date:   2016-07-17
author: Quentin Monnet
categories: BEBA
tags: [BEBA, SDN]
---

_Please note that even though this article is not endorsed by [the BEBA
Consortium](http://www.beba-project.eu/our-team), its contents may include some
material that is under copyright of the Consortium._

* ToC
{:toc}

# What is it all about?

This article is a brief presentation of the OpenState abstraction layer for
stateful packet processing for programmable switching. Did you ever wonder how
to model, for your network switches, complex applications that behave
differently based on the history of received traffic? No? (_What? How comes?
You strange folks…_) Well, here is a possible solution all the same! OpenState
is part of the [BEBA](http://www.beba-project.eu/) European research project.
An introduction to BEBA is available in [a previous article on this blog]( {{
site.baseurl }}{% post_url 2016-07-15-beba-research-project %}).

# Stateful packet processing

## The principle

Concretely, we try to provide the switches with stateful packet processing
capabilities: this means that when a packet arrives, it will not be processed
solely according to the values contained it its headers (such as the
protocol(s) in use, TCP/UDP ports, sender/receiver addresses and so on), but
instead the switch will also take into account the packets that were
_previously_ received. The datapath keeps _state memory_: it actually
implements a set of _finite state machines_, that will help determine how to
handle a packet based on its proper characteristics as well as on the traffic
history.

## A simple example: port knocking

This is easier to understand with a simple example. Let's consider port
knocking: this is a basic security method that consists in closing a (TCP|UDP)
port on host, until a specific series of packets has been received from the
sender.

So let's imagine that the client _foo_, with IPv4 address 10.3.3.3, wants to
connect to the server _bar_ (10.1.1.1) through SSH (TCP port 22).

<figure>
  <img src="{{ site.baseurl }}/img/net/portknocking.svg" alt="Example topology"/>
  <figcaption>
    Simple network topology: the clients want to connect to the server through
    SSH.
  </figcaption>
</figure>

The server _bar_ can be directly reached on the Internet, and its administrator
does not wish the TCP port 22 to look open to people performing port scans. So
they have enabled port knocking: to open the port to a client, _foo_ for
instance, the server must receive from this client a specific sequence of
“knocks”. Here we will say that the server _bar_ is waiting for UDP packets on
ports 1111, 2222, 3333, then TCP port 4444, in this order. Once this sequence
of packets has been received from host _foo_, the server opens its TCP port 22
for this host, and the SSH connection becomes possible.

The graphical representation of this operation, in the shape of a state
machine, is represented on the figure below. The connection (for TCP port 22)
in initially closed, and each correct new packet makes the sequence progress.
If a wrong packet is sent, or if a timeout delay is reached, the sequence is
considered as false and is reinitialized. When the full correct sequence has
been sent, the connection is (and hence remains) open.

<figure>
  <img src="{{ site.baseurl }}/img/fsm/fsm-portknocking.svg" alt="FSM: port knocking"/>
  <figcaption>
    Port knocking state machine
  </figcaption>
</figure>

Now let's imagine that we want to replicate the port knocking mechanism inside
a switch: all packets from _foo_ and addressed to _bar_ are discarded, until
_foo_ provides the correct secret sequence. This enables to protect the server
_bar_ from undesired traffic until the client has identified itself as
trustful. To recognize the sequence of packets, the switch has to keep in
memory the previously received packets belonging to this sequence. In other
words, it must implement the state machine associated to port knocking.

And this is exactly what we do with BEBA!

# Technical solution: OpenState

## The old way: one-step flow tables

Before we look at BEBA's internals, let us remember how the legacy switches
proceed to process the packets.

Simple programmable switches are usually not able to maintain a history, a
set of states for the traffic they receive. They only match the incoming
packets again a set of patterns to determine to which “flow” these packets
belong, and search their flow tables for the action associated to this flow.

Here is an example flow table:

|---------------------------------------------------------+--------------------+
| Flow matching pattern                                   | Action             |
|:--------------------------------------------------------|:-------------------|
| …                                                       | …                  |
| IP src = 10.1.1.1                                       | Forward            |
| IP src = 10.2.2.2, IP dst = 10.1.1.1, TCP dst port = 22 | Forward            |
| IP src = 10.3.3.3, IP dst = 10.1.1.1, TCP dst port = 22 | Drop               |
| ARP                                                     | Send to controller |
| …                                                       | …                  |
|---------------------------------------------------------+--------------------+
{: .decorated-table }

SSH traffic coming from the host with IP address 10.2.2.2 and targeting the
server at 10.1.1.1 is allowed by the switch, while SSH packets to the same
server but coming from host at 10.3.3.3 is dropped. There is no simple way to
make it change when the port knocking secret sequence is received. The only way
we could hope to do so would consist in sending up to the controller all
packets that could be part of the secret sequence, as we do already for ARP
packets. So we would have a flow table looking like this:

|-----------------------------------------------------------+--------------------+
| Flow matching pattern                                     | Action             |
|:----------------------------------------------------------|:-------------------|
| …                                                         | …                  |
| IP src = 10.1.1.1                                         | Forward            |
| IP src = 10.2.2.2, IP dst = 10.1.1.1, TCP dst port = 22   | Forward            |
| IP src = 10.3.3.3, IP dst = 10.1.1.1, TCP dst port = 22   | Drop               |
| IP src = 10.3.3.3, IP dst = 10.1.1.1, UDP dst port = 1111 | Send to controller |
| IP src = 10.3.3.3, IP dst = 10.1.1.1, UDP dst port = 2222 | Send to controller |
| IP src = 10.3.3.3, IP dst = 10.1.1.1, UDP dst port = 3333 | Send to controller |
| IP src = 10.3.3.3, IP dst = 10.1.1.1, TCP dst port = 4444 | Send to controller |
| ARP                                                       | Send to controller |
| …                                                         | …                  |
|-----------------------------------------------------------+--------------------+
{: .decorated-table }

Then, it would be up to the controller to check that the packets of the secret
sequence have been sent in the correct order, and without reaching the reset
timeout delay. But unless all traffic from 10.3.3.3 to 10.1.1.1 is forwarded to
the controller, it could not know whether other packets not belonging to this
sequence would have arrived in between (such packets would be expected to reset
the sequence progression, as described per the above state machine).

And anyway, sending all those exceptions to the controller would be consuming,
in resources (processing, bandwidth) as well as in time. This is not a good
solution for this problem.

## OpenState, on the formal side: Mealy machines

So instead, the BEBA project proposes OpenState. OpenState brings back some
_smartness_ to the switches! In fact, it brings stateful processing of packets,
by proposing an implementation of simplified _extended finite state machines
(<acronym title="eXtended Finite State Machine">XFSM</acronym>)_, also known as
[Mealy machines](https://en.wikipedia.org/wiki/Mealy_machine). A simplified
Mealy machine in use with our model is an abstract structure comprising a
4-tuple (S0, S, I, O, T), where: 

* S is a finite set of states.
* S0 is an initial starting state S0, belonging to S.
* I is a finite set of input symbols (events).
* O is a finite set of output symbols (actions).
* T : S × I → S × O is a transition function mapping (state, event) pairs into
  (state, action) pairs. For “standard” Mealy machines, these transitions would
  be decoupled into two functions T : S × I → S and G : S × I → O.

Concretely, this comes down to having states (such as the states on the machine
of our previous port knocking example); and for each state, for any given input
symbol, that is to say for a given incoming packet, a pair made of a new state
and of an action is associated. So, if I am a switch proposing port knocking
features: if I am in state STEP\_3 and I receive a TCP packet on port 22, I go
to a new state, OPEN, and I perform an action: I forward the packet to its
recipient. If, on the contrary, I receive any other packet, or if the timeout
fires, then the new state will be the initial state (S0), and the action
consists in dropping the packet.

That was for the formal definition. Now, how can we integrate these machines
into the data plane?

## The new way: state tables and XFSM tables

The OpenState abstraction layer is implemented by means of new tables. Two new
tables, precisely, that replace the former flow table.

The first of those tables is the _state table_. To each flow pattern, it
associates a state key instead of a direct action. Let us come back once again
to our port knocking example: we keep the previous configuration, that is to
say:

* SSH from 10.1.1.1 is allowed.
* SSH from 10.2.2.2 to 10.1.1.1 is allowed.
* SSH from 10.3.3.3 to 10.1.1.1 is forbidden at start, but can be allowed after
  the host at 10.3.3.3 sends the correct sequence.

In this setup, at a given time, the traffic from each host is in one of the
following states:

* DEFAULT: no SSH allowed.
* OPEN: SSH traffic (TCP towards port 22) is allowed.
* STEP\_1; or STEP\_2, or STEP\_3: the port knocking sequence is ongoing.

For 10.1.1.1 and 10.2.2.2, the state is always OPEN. But for 10.3.3.3, it depends
on the progress of the secret sequence, so the state may evolve. SO **the state
table itself can evolve, without the intervention of the controller**: this is
precisely the way we can track the succession of states.

For instance, assuming that 10.3.3.3 has sent the first packet of the sequence
(UDP on destination port 1111), the state table of the OpenState switch is as
follows. Note that when searching for a match, the tables that are provided
here are scanned from top to bottom, so the packets not matching any other
pattern will be assigned the DEFAULT state by the last flow pattern.

|--------------------------------------+---------+
| Flow matching pattern                | State   |
|:-------------------------------------|:--------|
| …                                    | …       |
| IP src = 10.1.1.1                    | OPEN    |
| IP src = 10.2.2.2, IP dst = 10.1.1.1 | OPEN    |
| IP src = 10.3.3.3, IP dst = 10.1.1.1 | STEP\_1 |
| …                                    | …       |
| IP src = other                       | DEFAULT |
|--------------------------------------+---------+
{: .decorated-table }

But how exactly are these states expected to evolve? Furthermore, how does the
switch associate state OPEN to the actual “forwarding” action? The solution to
these questions resides in the use of another table, called _<acronym
title="eXtended Finite State Machine">XFSM</acronym> table_.

This table has two main blocs: _Flow matching pattern_ and _Actions_. The
former is subdivided into two columns: _State_ and _Event_. The _State_ is a
state name, like OPEN or DEFAULT. The _Event_ is a traditional pattern that a
packet can match. The principle is the following: when a packet is received,
the _State table_ is scanned first. Depending on the pattern matched by the
packet, the table returns the current state associated to the flow. Then once
we know this state, the _XFSM table_ is scanned. The matching row is the one
for which the first (_State_) column is matched by the current flow state (e.g.
STEP\_1) _and_ the second (_Event_) column is matched by the packet (e.g. TCP
dst port = 2222.

The second bloc of columns gathers two columns, _Action_ and _Next state_. So
once the matching row has been determined with the state and events associated
to the packet, the corresponding action, found in the third column, is applied
(e.g. Drop); and at last the state for this flow is modified, as indicated by
the state name found in the _Next state_ column. In our example, if the second
packet of the secret port knocking sequence has just been received, then we
switch to STEP\_2.

Here is the example _<acronym title="eXtended Finite State
Machine">XFSM</acronym> table_ coming with the port knocking use case:

|-----------------------+-------+---------+------------+
| Flow matching pattern |       | Actions |            |
| State   | Event               | Action  | Next state |
|:--------|:--------------------|:--------|:-----------|
| …       | …                   | …       | …          |
| DEFAULT | UDP dst port = 1111 | Drop    | STEP\_1    |
| STEP\_1 | UDP dst port = 2222 | Drop    | STEP\_2    |
| STEP\_2 | UDP dst port = 3333 | Drop    | STEP\_3    |
| STEP\_3 | TCP dst port = 4444 | Drop    | OPEN       |
| OPEN    | TCP dst port = 22   | Forward | OPEN       |
| OPEN    | Port = *            | Drop    | OPEN       |
| …       | …                   | …       | …          |
| *       | Port = *            | Drop    | DEFAULT    |
|---------+---------------------+---------+------------+
{: .decorated-table }
{: #xfsm-table }

<script>
let t = document.getElementById('xfsm-table')
for (let i = 0 ; i < t.childNodes.length ; i++ ) {
  for (let j = 1 ; j < t.childNodes[i].childNodes.length ; j+=2 ) {
    let cell= t.childNodes[i].childNodes[j].childNodes[5];
    i < 2 ?
      cell.style = 'border-left: solid 2px #D6E7E0;' :
      cell.style = 'border-left: solid 2px #277553;'
  }
}
</script>

So if the first packet of the sequence has been previously received, meaning
that we are in state STEP\_1, and that a new packet from host 10.3.3.3 is
received, the _State table_ is scanned first. The third row from the top (not
counting the filling dots) is hit. It tells that this flow is currently in
state STEP\_1. OK, so now we know the state and can search the second table.
Two cases may occur:

1. The event on row 2 is matched: the packet is a TCP packet, with dst port =
   22.
2. The packet is anything else, and the very last row is matched instead.

In the first case, the packet is dropped, but watch out: the state for this
flow, for the flow of packets sent from 10.3.3.3 to 10.1.1.1, is changed to
STEP\_2. This means that the _State table_ is modified, and now looks like
this:

|--------------------------------------+-------------+
| Flow matching pattern                | State       |
|:-------------------------------------|:------------|
| …                                    | …           |
| IP src = 10.1.1.1                    | OPEN        |
| IP src = 10.2.2.2, IP dst = 10.1.1.1 | OPEN        |
| IP src = 10.3.3.3, IP dst = 10.1.1.1 | **STEP\_2** |
| …                                    | …           |
| IP src = other                       | DEFAULT     |
|--------------------------------------+-------------+
{: .decorated-table }

The next packet from 10.3.3.3 will match the third row again, but this time it
will return STEP\_2, so we will look for a different row in the _<acronym
title="eXtended Finite State Machine">XFSM</acronym> table_. If again a new
packet from the port knocking sequence is received, the state will move to
STEP\_3, and ultimately to OPEN.

If, on the contrary, the packet received is not a TCP packet towards port 22,
then the last row is matched: the packet is dropped all the same, but the state
is moved to DEFAULT instead. As a consequence, the port knocking sequence is
“reset”.

## Mapping with the formal definitions

In regard to the previous formal definition provided for the simplified Mealy
machines, you may have guessed how the elements map to the Mealy machine
components. Just to make it clear, let's sum up the association:

* S is the set of states.
* S0 is the initial state, DEFAULT in our example.
* I is the finite set of input symbols. It represents the events to match with
  the packet, such as “TCP dst port = 22”.
* O is the finite set of output symbols: it is the set of the possible
  resulting actions, such as “drop” or “forward”.
* T : S × I → S × O maps a pair (state, event) with another pair (state,
  action). In fact, this is exactly what the _<acronym title="eXtended Finite
  State Machine">XFSM</acronym> table_ does: the first pair is an element of
  the _Flow matching pattern_ column bloc, and the two last cells of this row
  that belong to the _Actions_ bloc are associated to it.

## About the actions

It is worth noting that the actions performed once the _<acronym
title="eXtended Finite State Machine">XFSM</acronym> table_ has been read can
vary and that they are not restricted to “drop packet” or “forward it through
output port _n_”. Instead, there is a number of possibilities such as:

* push or pop some header on or from the packet,
* create a new flow rule,
* read the value from a given field of a header of the packet and update a rule
  with this value,
* _et cætera_.

Along with state management, these actions allow for the implementing of a wide
range of applications.

# Leveraging OpenState

These were the main lines of OpenState. With a similar setup, it is possible to
implement many applications relying on stateful processing. BEBA research
project heavily relies on this concept, but of course it goes a little further.
Here are some other works based on OpenState. Some may be turned into future
articles on this blog.

* OpenState extension with per-flow registers and conditions. This would be the
  evolution of OpenState, and is called Open Packet Processor. It is some work
  in progress from BEBA's partners.
* Hardware proof of concept of OpenState abstraction layer (see references).
* Software acceleration of programmable switches running the OpenState layer.

Before we tackle any of these topics, I believe it would be interesting to
present some real expected use cases for OpenState switches. In-switch port
knocking is a nice stateful example to study, but to my knowledge it is not and
will most probably not get widely implemented. So, next article: OpenState
switch use cases. Get ready!

# Resources

* [**BEBA**'s list of **public
  deliverables**](http://www.beba-project.eu/dissemination/public-deliverables).
  In particular, deliverables D2.1 and D2.2 relate to the design and basic
  implementation of the OpenState layer.
* **OpenState**: G. Bianchi, M. Bonola, A. Capone, and C. Cascone, “OpenState:
  Programming Platform-independent Stateful OpenFlow Applications Inside the
  Switch” ACM SIGCOMM Computer Communication Review, vol. 44, no. 2, pp. 44–51,
  2014.
  ([link](http://openstate-sdn.org/pub/openstate-ccr.pdf))
* **Hardware implementation**: S. Pontarelli, M. Bonola, G. Bianchi, A. Capone,
  C. Cascone, “Stateful Openflow: Hardware Proof of Concept”, IEEE HPSR 2015,
  Budapest, July 1−4, 2015.
  ([link](http://www.beba-project.eu/presentations/edas.paper-1570113167.pdf))

{% comment %} vim: set syntax=markdown spell tw=79 fo+=a: {% endcomment %}
