---
layout: post
title:  "Run the BPF CI on your GitHub Linux fork"
date:   2022-08-05
author: Quentin Monnet
employer: isovalent
excerpt: The Linux kernel has a CI system to check all submissions to the eBPF
    subsystem. It runs the BPF selftests on all patches reaching the
    development mailing list. But you can also run it before posting your
    patches, by using your own Linux fork in GitHub. Let's see how to do this
    in a few simple steps - or with a script.
categories: BPF
tags: [BPF]
---

* ToC
{:toc}

All patches submitted to the Linux BPF subsystem go through the Continuous
Integration (CI) system. This is to make sure that all the BPF selftests are
still passing with the proposed changes applied, to catch as many bugs as
possible _before_ merging the patches. This post gives an overview of the CI
architecture, and explains how to run it on your own Linux fork on GitHub.

# Validating BPF patches

I've been contributing to bpftool and occasionally to the kernel for several
years, sending upstream [a number of
patches](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?qt=author&q=quentin+monnet).
Each time I post something, the reviewers, maintainers, and contributors at
large [expect of me that I have correctly tested my
patches](https://www.kernel.org/doc/html/latest/bpf/bpf_devel_QA.html#q-what-is-the-minimum-requirement-before-i-submit-my-bpf-patches)
before submitting---before the patches reach the CI. This usually includes
running locally the whole suite of BPF selftests with my changes.

Occasionally, I've run into issues when trying to run the selftests. Some of
the tools were missing on my setup, or didn't come in the correct version. The
selftests are shipped with [a handy
script](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/tools/testing/selftests/bpf/vmtest.sh?h=v5.19)
to run them locally on a QEMU virtual machine, but even so, I've faced issues
such as [libc
compatibility](https://lore.kernel.org/bpf/CAGG-pUTpppu-voYuT81LiTMAUA5oAWwnAwYQVAhyPwj3CwnZPA@mail.gmail.com/)
in the past. At other times, my setup would work well on a given machine, but I
simply didn't have access to that machine when working on a more trivial patch
and had to delay submissions until I could properly run the tests.

Running the CI on my own Linux fork on GitHub gives me a chance to test my
patches, without sending them upstream, and without requesting a review from
the maintainers if the work is still in progress. It also means that I do not
have to care about setting up the environment (or the libc) needed to run these
tests: I simply defer this to GitHub Actions, the CI platform from GitHub,
which compiles and installs everything required.

Plus, this means running the official BPF CI. One hundred percent guaranteed to
match what your patches will go through when you submit them at last. Makes it
a safe bet!

# Architecture of the BPF CI

Let's begin with an overview of the BPF CI architecture. All patches targetting
the BPF subsystem are run against the CI, to detect and prevent regressions.
The workflow looks like this:

<figure>
  <img src="{{ site.baseurl }}/img/misc/ci_flow.svg" alt="BPF CI arcchitecture: Overview"/>
  <figcaption>
    BPF CI architecture (overview)
  </figcaption>
</figure>

1. Patches are submitted by contributors, to the `bpf@vger.kernel.org` mailing
   list.

2. They are automatically added to
   [Patchwork](https://patchwork.kernel.org/project/netdevbpf/list/).

3. There is a daemon running somewhere (in fact it runs currently on Meta's
   infrastructure), to catch the patches arriving on Patchwork and to copy them
   to GitHub.

4. Magic happens, and all BPF tests run in a VM with all patches applied.

Okay, let's come back to that last step with a few more details.

1. The daemon picking up the patchsets from Patchwork prepares a Pull Request
   (PR) for the
   [`kernel-patches/bpf` repository](https://github.com/kernel-patches/bpf/pulls).

2. The first commit for that PR is always the same: it consists in picking up
   the files that describe the CI actions (the GitHub Actions workflow) to run,
   in a YAML file that GitHub can interpret. These files are under the
   `.github` directory in the
   [`kernel-patches/vmtest` repository](https://github.com/kernel-patches/vmtest/tree/master/.github).

3. On top of this first commit, the patches from the patchset retrieved from
   Patchwork are added one after the other, each as a separate commit in the
   PR.

4. Once that the PR is finalised and created on GitHub, the GitHub workflow is
   automatically triggered and runs the CI.

5. Magic happens, and all BPF tests run in a VM with all patches applied.

<figure>
  <img src="{{ site.baseurl }}/img/misc/ci_pr.svg" alt="Pull Request creation"/>
  <figcaption>
    Creating the Pull Request from the patchset
  </figcaption>
</figure>

Right, let's zoom in again on this last sub-step. What's happening at this
stage is simply the workflow from GitHub Actions unfolding. There are a few
steps involved, but roughly, the idea is to download a VM image, to checkout
the kernel sources, to apply the patches from the PR under test, and to run all
the BPF selftests on this patched kernel in the VM. Any test failure will make
the workflow fail. Ideally, a failure highlights a bug in the PR (although at
times, bugs creep in and some selftests are broken).

If you want more details about the BPF CI, you should have a look at
[the slides from Mykola Lysenko's presentation at bpfconf 2022](http://vger.kernel.org/bpfconf2022_material/lsfmmbpf2022-bpf-ci.pdf).
It comes with all the links and snapshots to help you retrace all the steps. As
for GitHub Actions, more details are available on
[GitHub Docs](https://docs.github.com/en/actions).

Over time, the CI has been catching a number of issues in patches submitted by
developers. It has proved immensely useful. The BPF CI is great. Send your
patches---but keep the CI happy!

# Running the BPF CI on your own GitHub Linux fork

So you want to run the BPF CI on your patches. How does it work? There are a
few prerequisites for running the BPF CI from your own GitHub account:

- You have Linux patches (commits) on your laptop, in a dedicated branch of
  your local Linux repository (for example, `new-feature`), on top of a local
  base branch (for example, `bpf-next`), and you want to run them through the
  BPF CI.

- You must have a GitHub account.

- You must have a fork of [Linux](https://github.com/torvalds/linux/) on your
  account, with GitHub Actions enabled for the repository.

- This fork must have a branch which is up-to-date with your local base branch
  (in our example, `bpf-next`).

Running the BPF CI is simple. First, we need to download the files defining the
GitHub workflow, and to add them to the feature branch as a new commit. Then,
we can create a new Pull Request, which will automatically trigger a run of the
workflow.

Let's download the files now. We can obtain an archive with the contents of the
`kernel-patches/vmtest` repository, directly from GitHub:

```
$ cd <local-linux-repository>
$ curl -sLO https://github.com/kernel-patches/vmtest/archive/refs/heads/master.zip
```

We can extract this archive and copy its contents to the local repository.

```
$ unzip master.zip
$ rsync -a --exclude README vmtest-master/. .
```

Now we add the files to Git's index, and create a new commit.

```
$ git add -f .github travis-ci
$ git commit -m "add CI files"
```

Our branch is ready. We need to create a new Pull Request on GitHub. To do
this, we need to push the `new-feature` branch:

```
$ git push <your-fork> -u new-feature
```

On the GitHub interface, we can proceed with the creation of the PR. Mind that
it should _not_ be opened against the original Linux repository. Instead,
select your base branch (`bpf-next` in our example) from your own fork as a
base for the PR. Obviously, you should use your `new-feature` branch, also from
your own fork, as the branch to compare with the base.

Once the PR is created, you're all set! GitHub detects the workflow described
in the files we added to the branch, and after a few seconds, it should start
building the environment then running the BPF CI on your commits.

# bpf-ci-check

We have just seen that running the BPF CI on your own Linux fork on GitHub is
easy. And it can be automated! To save time when I want to run the CI on a new
patchset, I created `bpf-ci-check`, a simple script that downloads the workflow
files, creates the new commit, pushes the development branch and creates the
Pull Request. All in one command!

You can download the script from [the bpf-ci-check repository on GitHub](https://github.com/qmonnet/bpf-ci-check).

To use it, go to your local Linux repository. Make sure that your remote base
branch (on your GitHub Linux fork) is up-to-date with your local base. Then
just download and run the script:

```
$ ./bpf-ci-check
```

There are a few available options to configure the development and base
branches, as well as the repository to use.

```
$ ./bpf-ci-check -h
Usage: ./bpf-ci-check [options]
Test in CI - Open a GitHub PR for 'ci-test/<branch>' against <base> on <repo>
Options:
        -h              display this help
        -n              dry run (create branch, but do not push or create a PR)
        -d branch       development branch to test in CI (default: current)
        -b base         base branch to use (default: 'bpf-next')
        -r repo         GitHub repository to use, for both branches and for PR
                        (default: gh-resolved base or first remote found)
```

I hope you'll find it useful!

# Parting notes

## The last line of defense

The BPF CI is becoming more robust and increasingly important in the BPF
development workflow, and some other kernel subsystems may also rely on it in
the future. It is simple and convenient to run it on your patches using your
own Linux fork on GitHub.

However, keep in mind that the existing checks have their limitations. They do
not replace thorough testing of new features and careful proofreading of the
code. Nor do they implement the new tests that might be required for your
feature. So, use common sense. Run your patches through the CI, but keep
testing your code. As Andrii says, CI and maintainers should be the last line
of defense against bugs, not the main one!

## Improving the CI

If you can help improve the BPF CI, do open Issues or Pull Requests on the
[`vmtest` repository](https://github.com/kernel-patches/vmtest).

Note that the [libbpf GitHub mirror](https://github.com/libbpf/libbpf) also
relies on the workflows defined in this repository, to run tests on new libbpf
commits.

And if you're interested in contributing, I would love having something similar
for [bpftool](https://github.com/libbpf/bpftool). Get in touch!
