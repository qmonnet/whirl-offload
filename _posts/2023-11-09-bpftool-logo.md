---
layout: post
title:  "A logo for bpftool"
date:   2023-11-09
author: Quentin Monnet
employer: isovalent
excerpt: Introducing bpftool's new logo
categories: BPF
tags: [BPF]
---

* ToC
{:toc}

At last, bpftool has a logo! But finding the right one was a long process.

# The journey

I have thought about a logo for bpftool for a while. Something that would
accurately represent the tool, and its relationship with BPF objects. I'm
decently creative, and I don't lack ideas; I've just been struggling to find
_the_ idea, the one that passes the bar that I mentally set. And I know how to
fiddle with Inkscape, but I've got poor design skills, and I'm usually terrible
when it comes to drawing.

So I've explored various ideas, and lucky you! You get to see what bpftool got
away with.

## Bees, and more bees

Given that bpftool is related to eBPF, the logo has to represent some kind of
bee, _of course_. So I started with stylised bees, first as some sort of
ribbon:

<figure>
  <img src="{{ site.baseurl }}/img/bpftool_logos/ribbon.png" height="200px" alt="A stylised bee made of a rolled ribbon, creating the stripes"/>
  <figcaption>
    Not the logo for bpftool. A ribbon-bee. All rights reserved.
  </figcaption>
</figure>

But it turned out to be very limited by my skills at drawing nice-looking
ribbons. I tried another variant more in the shape of a helix, but I didn't go
beyond the stage of a rough sketch for this one:

<figure>
  <img src="{{ site.baseurl }}/img/bpftool_logos/helix.png" height="300px" alt="Sketch of a stylised bee made of a helix-like ribbon"/>
  <figcaption>
    A helix-bee sketch. All rights reserved.
  </figcaption>
</figure>

Then I got an Idea, a good one. As bpftool is, well, a _tool_, the logo should
probably reflect that. What if the bee was also a Swiss-army knife, just like
bpftool is sort of a Swiss-army knife for managing eBPF objects? I came up with
something decent:

<figure>
  <img src="{{ site.baseurl }}/img/bpftool_logos/army-knife-bee1.png" height="300px" alt="A Swiss-army knife shaped as a bee, with the wings as blades"/>
  <figcaption>
    A Swiss-army knife bee. All rights reserved.
  </figcaption>
</figure>

And I think the next iteration was actually rather nice (don't mind the typeset
"bpftool" underneath, the colour and font were just a test). I have the honour
to introduce you to the Swiss-army knife bee, Pikachu variant!

<figure>
  <img src="{{ site.baseurl }}/img/bpftool_logos/army-knife-bee2.png" height="300px" alt="A Swiss-army knife shaped as a bee, with the wings as blades, and a cute head"/>
  <figcaption>
    Another Swiss-army knife bee. All rights reserved. Anyone has a lightning
    rod?
  </figcaption>
</figure>

I really considered using this one for a time. I kept it in the drawer for a
while because I hoped it would receive some improvements from a designer,
except it turned out the project ended up with a priority too low in their list
of tasks. I'm also unsure how similar to a Pokémon you can make your logo
before you get into troubles for copyright infringement.

Trivia: You can observe the improvement on the colour gradient on the wings
between the two versions. This was just before I discovered that mesh
gradients, that I used in Inkscape for the second version, are not part of the
SVG standard and are not supported yet in browsers, so the original SVG would
display with completely transparent wings.

## Bee smokers

After considering the Swiss-army knives for a while, I thought of something
else. What if we picked a logo that was _not_ a bee? There are many variants of
bees in eBPF-related projects already, so why not use something different, for
a change? After all, bpftool is not a project _based on_ eBPF in the sense that
it would use eBPF for the binary to work[^use-bpf], so maybe it shouldn't be an
eBPF bee itself. Instead, it could be something that we use to _work with
bees_.

[^use-bpf]: To be accurate, bpftool does use eBPF at runtime, for displaying
    PIDs of process holding references to eBPF programs, or for profiling eBPF
    programs. But this is marginal, and is not representative of how the main
    features of bpftool work.

... Wait, what do we use to work with bees?

One answer came to mind: when harvesting honey, beekeepers use smokers in order
to mask the pheromones from the bees and to prevent them from attacking all at
once. Can a bee smoker make a nice logo? Yes! And no. I mean, it probably can.
You just need more skills than I have, I suppose.

Let's be honest: the first draft was, huh, very bad.

<figure>
  <img src="{{ site.baseurl }}/img/bpftool_logos/smoker0.png" height="217px" alt="A badly-drawn bee smoker and smoke cloud"/>
  <figcaption>
    A bee smoker, with a bit of imagination. You can do whatever you want with
    this one, really.
  </figcaption>
</figure>

As a colleague kindly commented, it looked like a fart. Okay, so let's move on
to some other versions of the smoke, just the smoke, without the tool this
time:

<figure>
  <img src="{{ site.baseurl }}/img/bpftool_logos/smoke_cloud1.png" height="300px" alt="A stylised smoke cloud"/>
  <figcaption>
    A smoke cloud. All rights reserved.
  </figcaption>
</figure>

Or another one:

<figure>
  <img src="{{ site.baseurl }}/img/bpftool_logos/smoke_cloud2.png" height="300px" alt="Another stylised smoke cloud"/>
  <figcaption>
    Another smoke cloud. All rights reserved.
  </figcaption>
</figure>

The two logos above are colourful, but they don't really depict smoke well,
they look more like clouds. Could someone understand, just by seeing them, that
they relate to the smoke used to tame the bees? Very unlikely. We need the
smoker itself to be apparent. Hence a new version:

<figure>
  <img src="{{ site.baseurl }}/img/bpftool_logos/smoker1.png" height="300px" alt="A stylised bee smoker end piece and smoke cloud"/>
  <figcaption>
    A bee smoker and smoke cloud. All rights reserved.
  </figcaption>
</figure>

The smoke looks better, but the smoker is not recognisable. Let's make it
smaller so we can fit more of it in the frame:

<figure>
  <img src="{{ site.baseurl }}/img/bpftool_logos/smoker2.png" height="300px" alt="A stylised bee smoker and smoke cloud"/>
  <figcaption>
    A more recognisable smoker and smoke cloud. All rights reserved.
  </figcaption>
</figure>

Probably not the worst thing we could think of, but I was still not convinced
by the result. The thickness of the lines is not balanced between the smoker
and its smoke. Or are there too many details? But removing them would make the
smoker hard to recognise... Anyway, do we really want smoke on the logo? It's a
mean to disturb the normal behaviour of the bees, it's not healthy, and it's
not environment-friendly. Perhaps not the right image we need, after all.

See how far we've come! Ribbon bees, Swiss-army knives, Pokémons, and bee
smokers. None of them satisfying. However, I got some new inspiration and found
something that I really like. By some miracle, I even managed to draw something
that (I think) looks great. Here comes the official logo for bpftool, at last!

# Hannah the Honeyguide

<figure>
  <img src="{{ site.baseurl }}/img/bpftool_logos/bpftool_stacked_color.svg" alt="The logo for bpftool: a juvenile greater honeyguide"/>
  <figcaption>
    The final logo: Hannah the Honeyguide
  </figcaption>
</figure>

Meet **Hannah the Honeyguide**, bpftool's mascot. She is a [greater
honeyguide](https://en.wikipedia.org/wiki/Greater_honeyguide), but a juvenile
one, as can be seen from her yellow throat. We accentuated her shades, because
Hannah really wanted to share colors with eBee and Tux (the mascots for eBPF
and Linux, respectively).

Living in sub-Saharan Africa, greater honeyguides are known for guiding humans
to the nests of wild bees. They use a specific call to attract human attention,
then they fly towards the hive. Once the honey hunters have found and harvested
the nest, greater honeyguides feed on the remnants of the hive, eating bee eggs
and larvae, and even beeswax.

Like a honeyguide, bpftool guides humans towards bees, or to be more accurate,
towards BPF objects loaded on a system: after all, one of the primary use cases
for bpftool is to load and inspect BPF programs and maps. Don't worry, bpftool
will not eat your programs. Although, it could well detach programs and have
them removed from the kernel, if you asked it to. Of course, bpftool is a piece
of software and cannot "expect" to receive something in return for its
services. But think of it this way: for guiding humans to BPF, it feeds on
software maintenance and new features. Isn't that some form of mutualism, after
all?

Greater honeyguides are also brood parasites: the females lay their eggs in the
nests of birds of different species, and the chicks attempt to get rid of any
competitors as soon as they hatch. Thankfully, Hannah chose not to fight at
birth. As for bpftool? Shhhh, we may well have placed it in a particular
penguin's nest, so it could thrive. But we're happy to report that bpftool
never pushed any other project out of the Linux repository!

Other variants of the logo (horizontal, icon only, monochrome) are available on
[the GitHub mirror repository][assets].

All variants for the logo are licensed under the [Creative Commons Attribution
4.0 International (CC-BY-4.0)][cc-by-4.0]. Reuse them as you want, but please
credit the bpftool authors. The font used to typeset "bpftool" is
[Raleway](https://www.theleagueofmoveabletype.com/raleway).

---

[assets]: https://github.com/libbpf/bpftool/blob/main/.github/assets/
[cc-by-4.0]: https://creativecommons.org/licenses/by/4.0/
