---
layout: post
title:  "Exclusive interview: Santa switches to Cilium"
date:   2024-01-02
author: Quentin Monnet
employer: isovalent
excerpt:
categories: Cilium
tags: [Cilium, humor]
---

* ToC
{:toc}

_We are very lucky to produce an exclusive interview with Santa, who kindly
answered our questions just a few days after his last tour. Believe it or not,
Santa now runs Cilium!_

#### Dear Santa, welcome to this interview! We just finished 2023's tour. How did it go, this year?

Ho! Hey! Hello, thanks for having me. That's right, the Elves and I finished
our tour in the early hours in the morning of the 25th of December. Once again,
we managed to bring toys and presents to children, young or grown-up, all
around the world. This year, the team was particularly well organised, and we
managed to deliver more presents than ever before, even with the increased
delays due to the decreasing number of chimneys.

#### These are great news. How do you explain the progress you've made?

For some parts, this is due to cultural changes. In 2023, we realised that we
were behind in terms of well-being and mental healthcare. We've been asking
children to be well-behaved for centuries, but now we started to run
initiatives to encourage our team to be nice at work, too. Another thing is
that many of us decreased our consumption of meat and rich food, and we've felt
more on shape.

But we also implemented technical changes on our infrastructure, and observed
huge improvements from it. The truth is: we switched to Cilium.

#### What infrastructure? And what were you using before Cilium?

What do you mean, "what infrastructure"? How do you think we've been tracking
wish lists for billions of children over the years? Or ordering parts and
components for building toys? Or handling stocks and delivery? We've been in
business for centuries, and we're really good at it. We were pioneers in terms
of databases. At the beginning though, we only had pens and paper. But when
computers took off, we soon got equipped. We grew our own infrastructure, first
with mainframes, then with racks of servers. The North Pole is ideal for
cooling hardware, you know. Ten years ago, we started to deploy all our infra
with OpenStack, but this ended badly. The Elves threatened to quit if we
couldn't find an alternative. So we turned to external providers. We do have a
few rebel Elves running clusters of Raspberry Pis internally while muttering
things about Vim or Emacs, but in reality most of our workflows now take place
in hosted clouds. We ported all of our databases there, even if there has been
some overhead due to data privacy restrictions. [GDPR] didn't help, but we
found our way to compliance.

[GDPR]: https://worldbuilding.stackexchange.com/questions/114033/how-can-santa-keep-his-lists-when-the-gdpr-is-around

More recently, we started to get our hands dirty with Kubernetes and to deploy
pods, services, and volumes at scale, in a more convenient way.

#### How did these previous solutions work for you?

Well, we got the presents sorted all these years, haven't we? But some
solutions worked better than others, as we've discovered through the years.

On the early days of the Internet, for example, we tried to provide online
forms for wish lists, but the lack of encryption made it easy for people to
intercept the requests and add useless items to play pranks on their friends.
We also had a good time rebuilding our database after that child named
`Robert'); DROP TABLE Children; --` filled the form. Robert, don't be surprised
you stopped receiving presents after that day.

Also, a few years back, we decided to follow the hype and we deployed a
solution based on blockchain. We had all presents registered into a chain, so
that we could guarantee their provenance and their delivery. As it turns out,
no child has ever cared where the toys come from, or how they all fit nicely
into the chain, or whether you can use cryptographic primitives to ensure that
a toy car has not been tampered with.

Thankfully, at the end of the year we've always managed to bend the tools to
our needs, and to deliver. It's just that some years are easier than others.

#### So why did you try Cilium?

Since we've started managing clusters with Kubernetes, we've had a good
experience. However, the IT Elves have always found it difficult to connect
multiple clusters, or to observe what's going on in the cluster. We first heard
about Cilium when fans of the project asked for some Cilium swag for Christmas
in 2021. In 2023, we learned that Cilium entered (then completed) its
[graduation] process, and felt that the project was mature enough to give it a
try. We deployed it on our clusters, and it turned out this was exactly what we
needed.

[graduation]: https://www.cncf.io/announcements/2023/10/11/cloud-native-computing-foundation-announces-cilium-graduation/

#### How did Cilium change the situation?

Ho! Cilium has been a game-changer. A revolution. A new era. We deliver in so
many countries! We have clusters at AWS, GKE, Azure, and several other
providers. Cilium allows us to connect the clusters seamlessly together, so
that all our services can communicate together. We have XDP acceleration or
bandwidth management to handle the peak of traffic when Christmas comes near.
We have transparent encryption to make sure that no data leaks and that nobody
sees whether a child has been good or bad.

In terms of performance, we observed an increase of 80% of requests processed
by seconds, which translates into 564 additional toys produced each minute and
a 7% improvement in order accuracy, with a 12% reduction of the duration of the
tour.

With Cilium, we've been using this eBPF thing to run the dataplane in the
kernel, and it's fast, and it's resilient! I'm all excited. It's as if I got a
Christmas present to myself, this year. The Elves are excited, too: they
changed their traditional green liveries to light green, yellow, blue, purple,
and red ones, matching the colours of the Cilium logo!

#### Rumour has it you renamed the reindeers, can you tell more about that?

Oh, yeah I did! When I noticed all the benefits we obtain with Cilium, I got so
excited that I started to rename them all. Dasher and Dancer now go as
Clustermesh and Servicemesh. Vixen is VXLAN. We also have XDP, Metrics, DSR,
Hubble, and Tetragon.

<!--
Dasher
Dancer
Prancer
Vixen
Comet
Cupid
Donner
Blitzen
Rudolph
-->

To be accurate, there is one I didn't rename. Rudolph remains Rudolph. It's not
really that I was attached to the name, you see. But the name has been
hardcoded by everyone in so many workflows everywhere that, when we simulated a
change, it completely crashed our CI. I had to abandon the idea.

#### What do you plan for the year ahead?

Usually, I take a couple months to rest after the rush before Christmas, and to
recover from the stress of the delivery night. But this year was easier on the
team, and we're all so excited, that we won't rest for long. We have a few
plans for 2024. Some Elves want to integrate AI into our workflow. They claim
that it can help reading and sorting the letters we receive, or designing the
ideal presents.

Tetragon is also on our list. Our servers are mostly secure, but Bad Folks have
occasionally managed to divert a few toys. Remember that year when you asked me
for this wonderful gift, but didn't get it? That was the Grinch exploiting a
zero-day vulnerability against our infra to steal presents. But I am hopeful
that Tetragon will help detect any attempt in the future, and all requests
should be addressed. Too bad you're a grown-up now!

We also have plans to explore Cilium itself even further. We're following the
project very closely, looking out for new features. In fact, I brought [an
early present][cisco] to some of the Cilium developers, and I'm curious to see
where this will take them. In the meantime, we consider getting involved and
contributing to Cilium ourselves. I already created [a Pull Request][pr] to add
myself to the list of users.

[cisco]: https://isovalent.com/blog/post/cisco-acquires-isovalent/
[pr]: https://github.com/cilium/cilium/pull/30083

#### Many thanks for your replies! We wish you a very Happy New Year 2024

There's no doubt that 2024 will be an interesting year. And when December comes
back, the Elves and I will be back to town, ready like never before. Enjoy the
presents we brought, and Happy New Year to you all. Ho! Ho! Ho!
