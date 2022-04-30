---
author: "Stephen Gutekanst"
title: "CLAs are not for open source, use a Developer Certificate of Origin"
date: "2021-07-01"
draft: true
categories:
- licenses
- foss
---

Startups trying to embrace open source today readily adopt Contributor License Agreements (CLAs) as mandatory requirements for contributing. This is not in the spirit of OSS, and you should use a Developer Certificates of Origin (DCO) instead.

## Update: 2021-07-01 5:46pm MST

A professional lawyer I look up to has [rebuked this article and stated it contains worrisome amounts of misinformation](https://lobste.rs/s/lvmb5i/clas_are_not_for_open_source_use_developer#c_n2k3zd). I encourage reading that thread, and also [the article he has written](https://writing.kemitchell.com/2018/01/06/CLAs-Are-Not-a-Sham.html) instead.

## Why do we need either? An imaginary tale of emulators

Let's say you work on a Playstation emulator. Someone who finds your work interesting begins sending you contributions on GitHub.

Unbeknownst to you, that person actually works at Sony. They didn't realize this, but their contributions in their spare time are _not owned by them_ and Sony's lawyers now want to sue you for using their proprietary, patented technology.

If you didn't get a written statement from that person saying they owned those contributions, and actually intended to release them under your open source license terms - then you could indeed be found in the wrong!

Now, you might want to argue that the mere act of _sending a pull request_ meant implicitly that the author intended to do that, but realize that is an _untested legal ground_. We don't know how a court would rule in that case.

This is one of the problems that CLAs and DCOs are used to solve: it makes that agreement explicit.

## Why CLAs are terrible for open source

If you're contributing under a CLA, you are more frequently 'giving the company your code' not 'releasing it as open source'.

CLAs are arbitrary legal agreements drafted by lawyers. They can contain absolutely anything, and often _assign ownership of your change fully to the company to do with as they please, including relicensing it under different terms._

[Eric S. Raymond](https://en.wikipedia.org/wiki/Eric_S._Raymond), Co-founder of the Open Source Initiative, [stated CLAs are harmful in the Linux Journal](https://www.linuxjournal.com/content/contributor-agreements-considered-harmful).

## Why a Developer Certificate of Origin is better

The Developer Certificate of Origin (DCO) was created for the Linux kernel.

It's an incredibly simple, consistent agreement (unlike CLAs!) that simply says:

1. You have the right to submit the contribution under the open source license of the project.
2. You agree it can and will be licensed under the open source license of the project.
3. You agree the contribution will be shared as open source.

You can read the full text here: https://developercertificate.org

It's not an arbitrary legal agreement. It doesn't change between projects. It doesn't give a company full ownership over your change.

## Who uses a Developer Certificate of Origin (DCO) instead of a CLA?

* The Linux kernel
- GitLab [in 2017 switched to DCOs away from CLAs](https://about.gitlab.com/blog/2017/11/01/gitlab-switches-to-dco-license/), with applause from the Debian and GNOME projects.
- An [internal GitLab analysis memo](https://docs.google.com/document/d/1zpjDzL7yhGBZz3_7jCjWLfRQ1Jryg1mlIVmG8y6B1_Q/edit) notes that CLAs did not provide additional protections over a DCO for GitLab themselves.

## Using a DCO for your project is _easy_

There exists [a GitHub check](https://probot.github.io/apps/dco/) which verifies contributors have added a line at the end of their commit message confirming they agree to the DCO:

> My change...
>
> Signed-off-by: Random J Developer <random@developer.example.org>

And `git` even has an `-s` signing option built in to write this signed-off-by line for you:

```
git commit -s -m 'My change...'
```

It is that easy!

#### Git commit hook

You can even add a Git commit hook to your repository in a `hooks/commit-msg` file [like this](https://raw.githubusercontent.com/hexops/ztemplate/main/hooks/commit-msg) and contributors can easily install it after running `git clone` via:

```
git config core.hooksPath hooks
```

So if you forget to sign off on your commit, it'll tell you the exact Git commands to fix it:

```
Your commit message is not signed off with the DCO.

Please read and agree to the DCO: https://developercertificate.org/

Hint: Use 'git commit -s' flag to sign your commits

Hint: Use 'git commit -seF .git/COMMIT_EDITMSG' to recover your commit message and sign.
```

## Be sure to choose a license with patent and IP protection

One thing to note is that DCOs just say a change was made in good faith under the terms of the license.

If the project's license does not explicitly grant you the ability to use patents/IP surrounding what that code _does_, then you do not have explicit permission to _use_ that code - even if it is under an open-source license. However, you could argue that the author meant to implicitly grant this permission by nature of contributing it under an open source license. This has, to date, never been tried in a court and thus there is no legal precedence either way.

Licenses are another topic entirely, and you should carefully consider the options available to you, but this is why Hexops dual-licenses all code under both the MIT and Apache licenses.

MIT is popular as it is very short and simple, while Apache has an explicit patent grant. The consumer of the code gets to choose which license they prefer.

## Conclusion

- Please use DCOs, not CLAs.
- CLAs are harmful for the open-source community.
- Not using a DCO is _potentially_ risky (for users of the code too!)
- Not using a license that has an explicit patent grant is _potentially_ risky (for users of the code too!)

In my view, a DCO and dual-licensing under MIT + Apache-2.0 licenses grants users and contributors to an open source project the most freedom and guarantees.

_Disclaimer: I am not a lawyer, just a spirited advocate. This is not legal advice. Consult a legal professional for advice._
