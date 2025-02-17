---
author: "Emi Stein"
title: "Mach v0.3 released - Zig game engine & graphics toolkit"
date: "2024-02-02"
draft: false
categories:
- mach
- zig
- gamedev
description: "Mach aims to be a Zig game engine & graphics toolkit for building high-performance, truly cross-platform, robust & modular games, visualizations, and desktop/mobile GUI apps."
images: ["/img/2024/mach_v0.3_release.png"]
---

Mach is a Zig game engine & graphics toolkit for building high-performance & modular games, visualizations, and desktop/mobile apps. [Learn more](https://machengine.org/)

We are working towards Mach 1, and have just released v0.3 which includes [6 months of work](https://github.com/hexops/mach/milestone/5?closed=1) - here are the highlights!

## Coming soon: intro to 2D gamedev workshop

The first-ever **intro to 2D gamedev workshop using Mach** will be hosted at the [Software You Can Love](https://sycl.it/) conference in Milan, Italy, May 14-17. The workshop will use Mach's currently in-development higher level 2D graphics APIs.

<a class="imglink" href="https://sycl.it/"><img src="/img/2024/sycl-workshop.png"></a>

If you're interested in Zig or Mach, then [check out the SYCL conference](https://sycl.it/agenda/workshops/intro-to-2d-gamedev/)! It's an amazing experience, a great opportunity to meet a ton of Zig community members, core team members, as well as enjoy some of the best food that Italy has to offer!

## Community highlight: Pixi and Scoop'ems

[@foxxne](https://github.com/foxnne) is an early adopter of [Mach core](https://machengine.org/core/), largely pushing it to its limits. They make use of Mach's new experimental sysgpu graphics API (which we intend to be a successor/descendant of WebGPU), as well as other libraries like flecs and dear-imgui. They're developing [Pixi](https://github.com/foxnne/pixi) - a pixel art editor:

<a class="imglink" href="/img/2024/pixi1.png"><img src="/img/2024/pixi1.png"></a>
<a class="imglink" href="/img/2024/pixi2.png"><img src="/img/2024/pixi2.png"></a>

And have used it to make games like [Scoop'ems](https://github.com/foxnne/scoop-ems):

<video loop controls height="4000px">
<source src="https://media.machengine.org/showcase/scoopems.mp4" type="video/mp4">
</video>

[@foxxne](https://github.com/sponsors/foxnne) is making awesome tools and games in Zig, pushing things to their limits, I encourage watching [how humble Colton is when speaking about their work](https://www.youtube.com/watch?v=7K9Vzcr7vJg). We are very excited to make Mach support foxxne's projects better in the future, and enable others to build things like this too.

Please consider [sponsoring their work on GitHub](https://github.com/sponsors/foxnne) - you could be their second-ever sponsor!

## Mach core

[Mach core](https://machengine.org/core/) aims to provide just a window, input, and truly cross-platform graphics API.

We think of it as an alternative/competitor to the classic options of SDL+OpenGL, GLFW+Vulkan, etc. Today, it's not quite there yet - it uses GLFW behind the scenes for desktop support, and WebGPU as its graphics API, but we're actively working on making it a genuine competitor written in Zig.

In this release, it saw general bug fixes - as well as some [libmach](https://github.com/hexops/libmach) development - which aims to provide a C API to both Mach core and engine APIs.

## sysgpu

In Mach v0.2, we announced an experiment - that we were working on a WebGPU implementation written in Zig, as an alternative to using Dawn (Google Chrome's WebGPU implementation.) In the past 6 months, this experiment saw an immense amount of development and exceeded our expectations!

<a class="imglink" href="https://machengine.org/pkg/mach-sysgpu">
    <picture><source media="(prefers-color-scheme: dark)" srcset="https://machengine.org/assets/mach/sysgpu-dark.svg"><img alt="mach-sysgpu" src="https://machengine.org/assets/mach/sysgpu-light.svg" style="height:7rem;margin-top:1rem"></picture>
</a>

[sysgpu](https://machengine.org/pkg/mach-sysgpu/) today is a nearly fully-functional WebGPU native implementation (minus browser-level safety checks), thanks to [two amazing contributors](https://github.com/hexops/mach-sysgpu/graphs/contributors). It has functional D3D12, Vulkan, Metal, and OpenGL backends. It has it's own WGSL shader compiler, and nearly all mach-core examples are runnable using it. We've even seen real applications (the Pixi pixel editor from foxxne, for example) begin to adopt it.

As we continued development of it over the past six months, we identified key design tradeoffs where we could differ from WebGPU's API choices and gain a faster, more modern, featureful graphics API. As a result, we've come to view sysgpu as a leaner and meaner _successor and descendant of_ WebGPU for native graphics, rather than just another implementation of it. As a result, it builds on the back of WebGPU's design choices, but ultimately has its own distinct API and will not be ABI-compatible.

We have plans to alleviate some _major_ pain points of WebGPU, specifically around pipeline creation / descriptor boilerplate, supporting push constants when available via a better API design (not as an extension), a more integrated/seamless approach to binding resources to shaders with type-correctness, and more.

We are also evaluating using Zig itself as the shading language, instead of WGSL, and are looking to enable fully offline shader compilation as an optional feature.

sysgpu is still under heavy development, particularly all of the 'successor/descendant' API design choices noted above have not been implemented yet. It is disabled in the v0.3 release by default, and after this release we plan to invest more aggressively in it - so expect more details and specifics to come soon.

## sysaudio

As a bit of background, [mach-sysaudio](https://machengine.org/pkg/mach-sysaudio/) started out as Zig bindings to Andrew Kelley's fantastic C library [libsoundio](https://github.com/andrewrk/libsoundio), but ultimately it grew to stand on its own two feet - becoming a brand new library written in Zig from first-principles and the ground-up to achieve similar goals: providing just low-level audio input/output, nothing else. It saw a good deal of polish in this iteration:

* [SIMD sample conversion support](https://github.com/hexops/mach-sysaudio/pull/33) - and sample conversion is now explicitly optional (so the default is to work with the driver's active audio format.)
* Major API design improvements
* Fixed issues with microphone/input devices, specifically multi-channel devices on macOS with CoreAudio.
* Fixed [various issues](https://github.com/hexops/mach/issues?q=is%3Aissue+sysaudio+is%3Aclosed+milestone%3A%22Mach+0.3%22) with the WASAPI/Windows backend.

### Audio synthesizer hack-project

As a quick hack project over the holidays, I leveraged Mach's audio libraries, reading midi input from a piano keyboard, synthesizing audio using Zig code / Mach, and playing it back through digital piano speakers. Here's a few vertical videos of me being silly & having fun with it (skip to 2:24 to hear how I think a Ziguana might sound!):

<iframe width="720" height="480" src="https://www.youtube.com/embed/b8WDjaZC1C8?si=SgojGMwdD_lfncgn" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

## Nominated Zig versions

Mach has always needed a sweet-spot between stable Zig and nightly Zig, a better balance of latest-and-greatest features and bug fixes, but less of a moving target than nightly Zig. To address this, we formalized how we [nominate Zig versions](/2024/announcing-nominated-zig/) for use, enabling others to synchronize their Zig version with the one Mach supports more easily.

## Mach engine as a standard library of modules

We began [documenting](https://machengine.org/engine/stdlib/) how we view Mach _engine_ as a standard library of modules for game development, and how we'll enable you to use just the parts you wish. This is a small-but-important step in showcasing how the engine's higher level APIs will be more modular than the monolithic big engines of today.

## Entity component system

The Mach entity component system provides a key role in Mach's modularity, in this iteration it saw numerous polish / bug fix improvements - the ability to actually query entities, a more clear/concise API, etc. It is still under heavy development, however.

## mach.math

[mach.math](https://github.com/hexops/mach/tree/main/src/math) was introduced to the Mach standard library: a custom math library tailored towards our graphics API conventions, matrix representations, coordinate systems, etc.

Today it includes many of the basics: vectors, matrices, quaternions - though it is still missing some basic tablestakes. It also has ray-triangle intersection, and we intend to expand it to cover more general collision utilities later.

A new set of [math docs](https://machengine.org/engine/math/) were added to the website with some cute diagrams/visualizations.

## mach.gfx.Sprite

[mach.gfx.Sprite](https://github.com/hexops/mach/blob/main/src/gfx/Sprite.zig) was introduced, which is the start of a 2D sprite-rendering module. It is largely usable, though we anticipate its API to change a fair amount and are looking to add animated sprite support among other key features.

It has been useful in letting us test basic rendering of hundreds of thousands of sprites, each as separate entities with their own transformation matrices calculated CPU-side, and get more of an end-to-end feel for how things are looking with our ECS:

<iframe width="720" height="480" src="https://www.youtube.com/embed/ciuSYf7dcuE?si=SgojGMwdD_lfncgn" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

## mach.gfx.Text

Development of a basic text rendering module is underway, but not ready for use yet.

## Status of 'simple 2D game' support

In short, we're still working on it. More to come soon.

## General project maintenance

* Two new examples [rgb-quad](https://github.com/hexops/mach-core/tree/main/examples/rgb-quad) and [textured-quad](https://github.com/hexops/mach-core/tree/main/examples/textured-quad) showing off super basic 2D rendering were added.
* Began to formulate our [hardware support plans](https://github.com/hexops/mach/issues/989), such as when we will target certain SIMD instruction sets.
* All of our `build.zig` scripts went through a great deal of changes and improvements, as Zig's build system and package manager matured greatly.
* Various [other issues](https://github.com/hexops/mach/milestone/5?closed=1) were addressed.

## A personal note

<div style="display: flex;">
    <a href="https://github.com/emidoots">
        <img style="width: 420px" src="https://machengine.org/img/emidoots-profile.png">
    </a>
    <div>
        <p>I work a normal tech job, and most days after I sign off from work I go online to build Mach, often like working two jobs. I've been doing this for a few years now, and dreaming of being able to build Mach for a decade before that.</p>
        <p>FOSS <a href="https://devlog.hexops.com/2021/increasing-my-contribution-to-zig-to-200-a-month#i-grew-up-playing-linux-games-like-mania-drive">is in my roots</a> and I believe we should own our tools, they should empower <em>us</em>-not be part of <a href="https://kristoff.it/blog/the-open-source-game/">the 'open source' game</a> which is all too prevelant today (even among 'open source' engines.) Mach <em>needs</em> to be for people like you and me-it needs to genuinely be <a href="https://softwareyoucan.love">software you can love</a>.</p>
        <p>My dream is one day to live a simple, modest, future earning a living building Mach for you and creating high-quality games for everyone. Please consider <a href="https://github.com/sponsors/emidoots">sponsoring my work</a> if you believe in this vision.</p>
    </div>
</div>

## Thanks

Immense thank you to all those who helped make this release possible, to those who contribute regularly or in the past, and those who sponsor development. It means the world!

<div style="display: flex; flex-direction: row; align-items: center;">
    <img align="left" style="max-height: 12.5rem;" src="https://user-images.githubusercontent.com/3173176/187348488-0b52e87d-3a48-421c-9402-be78e32b5a20.png"></img>
    <ul>
        <li>Join the <a href="https://discord.gg/XNG3NZgCqp">Mach Discord server</a></li>
        <li>Check out <a href="https://machengine.org">machengine.org</a></li>
        <li>Consider <a href="https://github.com/sponsors/emidoots">sponsoring development</a> so we can do more of it!</li>
    </ul>
</div>
