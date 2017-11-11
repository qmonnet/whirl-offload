---
layout: page
title: About
permalink: /about/
---

# About {{ site.title }}

Engineering and doing research in fast networking environments can bring out
many interesting technical or conceptual elements. At one point I wanted to
share some of it, so here we are: this is my technical blog about my work in
the field of fast networking.

<figure>
  <img alt="{{ site.title }}'s logo" src="{{ site.baseurl }}/img/site/frigate.svg" style="width:250px;" />
</figure>

If you wish to react to something I wrote here, please feel free to send me an
email or to ping me on Twitter.

# Disclaimers

I always wanted to associate this blog to the companies that pay me to do the
work I write about, so initially I cited my first employer on all pages. Since
at some point I took a new position in a different company—these things
happen!—I modified my template, and now you can find on each post the logo and
name of my employer at the moment I did the work. This does not imply any
specific partnership between the companies mentioned on this website.

Please note that this remains a _personal blog_: while my employer is fully
aware of its existence, the articles are not reviewed in details by the company
before posting. This is not even a roadmap blog. So opinions, points of view,
technical mistakes and so on are my own; experiments do not necessarily result
in new functionalities for the company's products; and **at no point should the
responsibility of my current or past employers be engaged about something that
is written here**. For your information, the companies I have been working for
do have official blogs, even though you may find that they are more commercial
than technical. [Here you will find 6WIND's](http://www.6wind.com/blog/), and
[there is Netronome's](https://www.netronome.com/blog/).

# About me

I am Quentin <span style="font-variant: small-caps;">Monnet</span>, a French
R&D engineer. I joined 6WIND in fall 2015, after completing my PhD in computer
science, and changed for Netronome two years later, in fall 2017.

<figure>
  <img alt="Little Helper" src="{{ site.baseurl }}/img/site/littlehelper.svg" style="width:150px;" />
</figure>

I enjoy research, programming, open-source software, improving and sharing my
technical skills. There is a bit of all those things in my current position,
and in this blog I try to share the most interesting parts of it. You can find
more details about me on [my personal webpage](https://qmo.fr/).

# About 6WIND

[6WIND][] is a French medium company based close to Paris (with regional
offices on other continents). It specialises in producing and selling quality
software for high-speed packet processing in Linux environments. The company
works on accelerating virtual networking infrastructure, and at the same time
it tries to optimise hardware management on multicore architectures in order to
provide extremely high bit rates with a variety of network protocols, while
saving as much processing cores as possible for the software appliances. People
there go by one motto: _Speed Matters_!

<figure>
  <img alt="6WIND's logo" src="{{ site.baseurl }}/img/site/6WIND.svg" style="width:250px"/>
</figure>

6WIND also contributes to several open-source projects, including DPDK, the
Linux networking stack, OpenStack, Open vSwitch, Quagga / FRRouting and a few
others.

It is also involved in research activities, and takes part in projects such as
[BEBA](http://www.beba-project.eu/our-team), with industrial but also academic
partners.

More details are available on [the company's webpage][6WIND].

[6WIND]: http://6wind.com/

# About Netronome

_Bringing Software Innovation Velocity to Networking Hardware_. This motto is
not as catchy, but it exposes clearly [Netronome][]'s technical objectives.
Again, speed and performances are at stake, but instead of working on pure
software solutions, The company designs network cards. These devices, based on
a number of “Network Flow Processors” (NFPs) are highly programmable. They can
run as standard NICs, or they can serve as a support for hardware offload of
technologies such as Open vSwitch or Linux eBPF. Optimised for x86
“Off-The-Shelf” servers, the cards can efficiently relieve the nodes from
packet processing tasks and attain excellent performances in data center
networking. Netronome is based in California, with offices other parts of the
US and in various part of the world. I work in the team in Cambridge, United
Kingdom.

<figure>
  <img alt="Netronome's logo" src="{{ site.baseurl }}/img/site/Netronome.svg" style="width:300px"/>
</figure>

Netronome takes part in several open-source projects and is an active
contributor to the Linux kernel, and is involved in many other projects such
DPDK or IO Visor.

Again, comprehensive information is available on [the company's
website][netronome].

[netronome]: https://www.netronome.com/

# Credits

This blog is hosted on [GitHub](https://github.com/) and is powered by
[Jekyll](https://jekyllrb.com/). The theme is Jekyll's default, slightly
modified, and with custom colors. The style in use for code blocks was inspired
by [Luna.vim](https://github.com/notpratheek/vim-luna) scheme. Some schemas
include icons derived from [an SVG
version](https://github.com/Xaviju/inkscape-open-symbols/tree/master/font-awesome)
of the [Font Awesome icons](http://fontawesome.io/).

If everything goes well, it should be displayed with [Fira
Sans](https://mozilla.github.io/Fira/) fonts, designed by Mozilla (and Fira
Mono for code snippets).

The logo is based on a shot of a [male magnificent
frigate](https://commons.wikimedia.org/wiki/File:Magnificent-Frigate-male.jpg)
(“magnificent” is part of the name, actually) photographed by Wikimedia
Commons' user Benjamint444 and released under the terms of the [GNU Free
Documentation License (GFDL)
1.2](https://www.gnu.org/licenses/old-licenses/fdl-1.2.html) (and as a
consequence, so is the logo of this blog).

# License

In the absence of other specific mention, contents and materials presented on
this website are licensed under a [Creative Commons Attribution 4.0
International License](https://creativecommons.org/licenses/by/4.0/).

<figure>
  <img alt="CC-BY 4.0" src="{{ site.baseurl }}/img/site/by.svg" style="width:150px;" />
</figure>

The logos of the companies, of course, are under copyright or their respective
owners, and not subject to this license.

I launched this blog in order to share my work with the community, so if you
want to reuse some of the contents, please do not hesitate: help yourselves.
And if you do so, it would be kind not to omit to attribute the work to its
author and to the company he works for—thanks!

{% comment %} vim: set syntax=markdown spell tw=79 fo+=a: {% endcomment %}
