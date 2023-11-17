---
layout: post
title:  "Buzzing Across Space: The Illustrated Children's Guide to eBPF"
date:   2023-11-17
author: Quentin Monnet
employer: isovalent
excerpt: Based on The Illustrated Children's Guide to Kubernetes, Bill and
    Quentin have created a children's book for eBPF. This post is a quick
    introduction to the book, and links to the freely-available PDF.
categories: BPF
tags: [BPF]
---

One of my first contributions to the eBPF ecosystem was to [list the available
resources about eBPF]({{ site.baseurl }}{% post_url 2016-09-01-dive-into-bpf
%}), to help people get started with the topic (and, let's be honest, so I
could remember where to find these resources myself when I needed them). Since
then, eBPF has gained in notoriety, and we now have many more available
resources, from books to tutorials, a variety of blog posts, better kernel
documentation, and a lot more. The website <https://ebpf.io> is an excellent
starting point to learn the basics about the technology and to find the
pointers for the next steps of your journey. Whether you're a Linux user who
heard about eBPF and want to learn what you can do and how to use it, or an
experienced eBPF hacker trying to understand how the verifier is implemented,
there are resources out there to help you.

However, people who are unfamiliar with Linux or technology in general will
struggle to find documents that could help them. Have you ever tried explaining
eBPF to your parents? To your neighbours? To this decision taker who has no
idea what a kernel is? To your... children? This is hard!

To make eBPF accessible to all people, Bill, a colleague from Isovalent, had
the brilliant idea to do a children's book about eBPF. We took inspiration from
[The Illustrated Children's Guide to Kubernetes][k8s], which introduces the
basic concepts of Kubernetes in a fun way, with the story of Phippy, a young
application lost in a cloud-native world, unfolding on the left-hand side pages
of the book, accompanied with diagrams and explanations on the right side.

And so we created a brand new story: _Buzzing Across Space_!

<figure>
  <img src="{{ site.baseurl }}/img/misc/buzzing-across-space.png"
    alt="Captain Tux (a penguin), eBee (a bee) and a gopher waiving from inside
      the cockpit of a starship flying in space. In the background, a bee is
      working on a console in an adjacent room of the ship. On the top, the
      title of he book is displayed."
  />
  <figcaption>
    <i>Buzzing Across Space: The Illustrated Children's Guide to eBPF</i> -
    Front cover.
  </figcaption>
</figure>

Meet Captain Tux, at the commands of its starship, the Silver Lining (every
cloud has one), and struggling to find an efficient way to fix and improve the
vessel. Meet eBee and her fellow bees, aboard the ship, helping Captain Tux
optimise various components, directly from the engine room. Will they succeed
in making the Silver Lining faster and more resilient? Will they be able to
adapt their setup to new pieces to avoid the ship to become obsolete? Will they
survive to the evil powers attracted by their prowesses? If you want the
answers to all these questions, just read the book! The PDF is freely available
from <https://ebpf.io>:

[**_Buzzing Across Space: The Illustrated Children's Guide to eBPF_**][pdf]

The book was authored by Bill Mulligan and myself. I focused on the story,
while Bill wrote most of the explanations, and we both contributed diagrams.
All the illustrations are the work of [Dacil C][dacil]. The book was produced
by [Isovalent]. For my part, I thoroughly enjoyed working on this project, and
I took great pleasure in writing the story of eBee and Tux.

In case you want to check it out, I also did a short introduction to the book
during the Linux Plumbers Conference 2023 ([session][lpc], [slides], [video]).
Bill had some physical copies of the book [printed for the latest
KubeCon][kubecon], but we managed to keep a few for LPC, and I handed over
twenty copies or so. Some people asked me to sign it (which I consider a great
honour). If you find yourself a copy and manage to find me at an event, I'll
gladly sign it, too (provided I don't forget my pen, like for LPC; maybe bring
your pen just in case). We'd like to find a way for people to buy physical
copies online, but nothing's been decided yet. We also plan to publish the
source files for the book, this should happen Soon™.

Now that this children's book is done, we also have a few ideas for future
related works. Again, nothing's been decided yet, we'll see where this takes
us. All aboard, the Silver Lining is taking off soon!

[k8s]: https://www.cncf.io/phippy/the-childrens-illustrated-guide-to-kubernetes/
[pdf]: https://ebpf.io/books/buzzing-across-space-illustrated-childrens-guide-to-ebpf.pdf
[dacil]: https://pro.fiverr.com/freelancers/artkrieg
[Isovalent]: https://isovalent.com/
[lpc]: https://lpc.events/event/17/contributions/1651/
[slides]: https://lpc.events/event/17/contributions/1651/attachments/1175/2421/LPC2023%20-%20Buzzing%20Across%20Space.pdf
[video]: https://youtu.be/voQMOQ1a7QM?t=31357
[kubecon]: https://isovalent.com/blog/post/kubecon-north-america-2023-wrap-up/#cilium-celebrations
