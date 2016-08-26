---
layout: post
title:  "BEBA research project: stateful processing for fast programmable switches"
date:   2016-07-15
author: Quentin Monnet
categories: BEBA
tags: [BEBA, SDN]
---

* ToC
{:toc}

# BEBA project overview

At 6WIND we work on accelerating packets for network equipments. As for me, I
work in the research and development team, and hence I realize little
development on our core products. Instead, I am currently working on a European
research project called [BEBA][]. The name derives from _BEhavioural BAsed
forwarding_, and indeed it aims at providing stateful packet processing
capabilities to programmable switch.

<figure>
  <img src="{{ site.baseurl }}/img/misc/logo-beba.svg" alt="BEBA logo"/>
  <figcaption>
    BEBA logo—Copyright 2015, BEBA research project
  </figcaption>
</figure>

To put it in another way, BEBA tries to bring back some of the “smartness” of
the network architecture from the controllers to the switches. We have seen in
a previous article how the controllers would program the switches, and how they
would handle the “exception” packets: well, this model still applies with BEBA,
but we try to make less of those packets considered as exceptions.

To that end, an abstraction layer has been devised so as to implement [finite
state machines](https://en.wikipedia.org/wiki/Finite-state_machine) on the
switches. This text is a brief overview of the project, but I intend to write
future articles about the technical contents, starting with details about this
stateful packet processing.

[BEBA]: http://beba-project.eu/

# Related articles on this blog

1. [Introduction to SDN]({{ site.baseurl }}{% post_url
   2016-07-08-introduction-to-sdn %}) — not directly related to BEBA.
2. The present article, a general overview of the project.
3. [OpenState]({{ site.baseurl }}{% post_url
   2016-07-17-openstate-stateful-packet-processing %}), the first version of
   the abstraction layer.
4. [BEBA's use cases]({{ site.baseurl }}{% post_url
   2016-08-26-stateful-processing-use-cases %}), a description of some of the
   principle use cases defined in the project.
5. Hopefully, more to come!

{% comment %}
These are work in progress, or simply post ideas
5. Software acceleration, because _speed matters_.
6. Open Packet Processor, the final version of BEBA's abstraction layer.
{% endcomment %}

**Please note** that all the articles of this series are some articles I wrote
on my own, on my spare time, and that they are not endorsed by the BEBA
consortium. They might contain inaccuracies or even mistakes, and in no case
should their contents be considered as official statements from the BEBA
consortium, nor from any of the project's partners.

# Main lines

## The work packages

BEBA project is split in a number of work packages, themselves divided in a
number of tasks. Here is a summary list of the work packages.

* **WP1** is related to management and administrative stuff. Nothing technical
  here, so now let's put it aside.

* **WP2** is a really interesting work package: it includes the conception, the
  definition and the validation, through proof-of-concept prototypes—be it
  software or hardware implementations—of the BEBA abstraction layer that is
  meant to provide stateful processing capabilities to the switches. So it is
  in this work package that the core mechanisms of the project are made
  explicit, and that the state machines are described in details. Some of those
  mechanisms should provide material for the next articles of this blog.

* **WP3** is based on WP2 and its prototypes, and brings them further: it
  consists in developing efficient and near-production implementations for the
  abstraction layer. Also, it encompasses another aspect of the project:
  besides bringing “**smartness**” to the switches, we want to bring them
  **speed**. So in this work package, we try to accelerate as much as
  possible—and a great part of what I do in the project takes place in here. So
  for this work package again, there should be a following article.

* **WP4** is about verification of the switches. It includes methods to assess
  and validate the correctness of the configuration, the security, the
  reliability and the resilience of the switches in the BEBA context.

* **WP5** includes an identification and a definition of several use cases for
  BEBA switches, that should be of interest for a great number of industrial
  entities. These use cases are considered either as “middlebox applications”
  (and they focusses on a particular processing that happens at the level of a
  switch, seen as a box) or as network-wide applications (and they rely on the
  interactions between several BEBA switches). I do not work specifically on
  this part. But the examples used for BEBA switches are fun to describe, so
  you may expect yet another article on this topic.

* **WP6** is for tests. We devise the switches, implement and accelerate them,
  find ways to verify their settings, and find uses for them, in the previous
  work packages. At last it will be time to check if all of this works! So
  there are two kinds of tests here, in short: in “laboratory”-like testbeds,
  and under real traffic conditions (thanks to CESNET, that may provide real
  traffic conditions for the tests).

* **WP7**, at last, focuses on the dissemination of the concept that will have
  emerged from the project, in particular through scientific publications or
  presentations, standardization activities, or **open-source** contributions.
  The last important characteristic for the project is that the technologies
  introduced to improve the switches—in terms in functionalities as well as
  in speed—must be “**vendor-agnostic**”, meaning that they must not rely on
  the hardware or on a SDK provided by a single vendor. On the contrary, they
  should be easy to deploy on all platforms, hence the importance of
  standardization and open-source code.

## Time schedule

The project started in January 2015, and runs until March 2017. At the time of
this writing, the first half of the project has ended a couple of months ago.

## The partners

At his point a question remains: _Who's in there?_ Here is the list of the
partners involved in the project.

Academics:

* [CNIT](http://www.cnit.it) (Consorzio Nazionale Interuniversitario per le
  Telecomunicazioni): an Italian research organism. They have many people
  working on SDN technologies, and are leaders of the BEBA project.
* [KTH Royal Institute of Technology](https://www.kth.se) is a University in
  Stockholm, Sweden.

Association:

* [CESNET](https://www.cesnet.cz) is an operator for research and education in
  Czech Republic. They maintain a network for universities in the whole
  country; and they do research as well.

Industrials:

* [NEC](http://www.nec.com) (German branch) is a big company selling IT
  products and services, including technologies related to SDN and packet
  processing.
* TCS (Thales Communication and Security) is a branch of the French
  [Thales](https://www.thalesgroup.com) group, specialized in electronical, IT,
  defense, aerospace businesses (and probably a few others).
* And [6WIND](http://6wind.com), of course, is a medium French company, and
  your favorite partner for network acceleration solutions!

# “_I want to know more!_”

All right, here are some additional leads. Of course, you can find all details
on [the project's website][BEBA]. BEBA being a European project funded on the
Horizon 2020 research plan, the results are public. In particular, you can
consult the [scientific
articles](http://www.beba-project.eu/dissemination/papers) published during the
project, or even better if you want the very technical details of BEBA's
internals, the [project
deliveries](http://www.beba-project.eu/dissemination/papers) that are all
publicly accessible, save for the management part. It represents quite dense,
but very exhaustive reading.

And also… why not read [my other articles](#related-articles-on-this-blog)
about BEBA?

{% comment %} vim: set syntax=markdown spell tw=79 fo+=a: {% endcomment %}
