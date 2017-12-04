---
layout: post
title:  "Offload is coming"
date:   2017-12-04
author: Quentin Monnet
employer: netronome
excerpt: A short update about my position, and about this blog in general.
categories: Other
---

Nothing technical in this post. This is a short update about my position, and
about this blog in general.

> _I'm asking the questions! What's up on the blog?_

You might have noticed that I slightly modified the template of this website.
In particular, I removed the company's logo from the generic pages (index,
“About”, etc.) and no longer defines this platform as a blog “about my work at
6WIND”. The reason, as you maybe know or guessed, is that I no longer work for
6WIND! Instead I joined [Netronome](https://www.netronome.com) not long ago.
The company roughly addresses the same issues: getting the best performances
for server networking. But instead of proposing pure software solutions, they
design programmable NICs that are “smart” enough to offload many components
from the host.

> _So you don't like 6WIND anymore?_

I do! I was extremely happy with my job at 6WIND. Doing research on this BEBA
European project, messing around with eBPF, and basically learning all I know
today about fast networking from brilliant colleagues… Their products are
quality software to boost up throughput performances, and they are just the
best in that domain. I had a wonderful time there.

I left my post vacant, so if one of you, dear readers, is looking for awesome
technical challenges about drastic improvements of userspace datapath
performances, I strongly encourage you to apply! I believe [they are
hiring](http://www.6wind.com/company-profile/careers/).

> _Then, why did you change?_

I left because, of course, I wanted a bit more than the perfect job. Describing
what drives such choices is somewhat complex, but I think the essence of my
expectations was threefold.

First, I wanted to go down one layer, and to get closer to the hardware. This
is, maybe, a consequence of working on BEBA with all those nice fellows
obtaining impressive results with their FPGAs. I want to learn how to process
packets at the deepest level. Netronome designs physical smartNICs and will be
able, I believe, to help me in this.

Then of course, I want to keep going with eBPF! I could have done it at 6WIND I
suppose, but not to the same extent. I am now part of a team of people who are
doing their best to offload eBPF to the hardware—how cool is that?—, so it's
really eBPF everywhere.

The last point is a personal choice, not less important in my eyes: I wanted to
move abroad, to become a world citizen, to perfect my English, to live outside
of my native country and to meet another culture—although I did not feel ready
to cross the Atlantic. Netronome invited me to join their team in Cambridge,
United Kingdom, and I seized this opportunity to get the life of a true British
gentleman.

> _I am a loyal reader of your blog. What does this mean for me?_

Well first of all, you have my gratitude for reading this blog! Let's have a
drink and discuss its contents next time we meet, if you will.

Then nothing much should change here. I will use the logo of the company who
paid me when I did the work presented in the article—there may still be a few
posts I started at 6WIND that I would like to finalise and publish. As for
contents, you should expect more eBPF-related stuff, since I remain involved in
this technology. And since I am done with the job-switching process and
relocation part, I will hopefully have more time to write than over the last
months!

> _Sounds cool! Anything else?_

Dear fellows from 6WIND, if you ever read this, thank you so much for those two
excellent years. I hope we meet again and find new ways to work together in the
future. Best of luck, and keep going fast!

Now I'm part of the Netronome team, and we're currently doing our best to run
eBPF directly on the NICs. By the way, [we are hiring as
well](https://www.netronome.com/careers/), we need all kinds of talented people
to help us on this topic. Brace yourselves: Offload is coming!

{% comment %} vim: set syntax=markdown spell tw=79 fo+=a: {% endcomment %}
