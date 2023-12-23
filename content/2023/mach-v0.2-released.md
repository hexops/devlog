---
author: "Stephen Gutekanst"
title: "Mach v0.2 released - Zig game engine & graphics toolkit"
date: "2023-08-12"
draft: false
categories:
- mach
- zig
- gamedev
description: "After about a year and a half of work, Mach v0.2 has just been released. Mach aims to be a Zig game engine & graphics toolkit for building high-performance, truly cross-platform, robust & modular games, visualizations, and desktop/mobile GUI apps."
images: ["https://github.com/hexops/mach/assets/3173176/54743b27-79da-498c-a7de-57da30f77f5a"]
---

Mach is a Zig game engine & graphics toolkit for building high-performance, truly cross-platform, robust & modular games, visualizations, and desktop/mobile GUI apps. [Learn more](https://machengine.org/)

We've been developing Mach for ~2 years; this release includes over a year of work, thousands of commits, and fixes [300 issues](https://github.com/hexops/mach/milestone/2?closed=1).

<video height="800px" autoplay loop muted>
<source src="https://media.machengine.org/core/example/deferred-rendering.mp4" type="video/mp4">
</video>

## On your machine in just ~60 seconds

With [this Zig nightly](https://machengine.org/about/zig-version/) version you can run the above demo on your machine in ~60 seconds:

```sh
git clone https://github.com/hexops/mach-core
cd mach-core/
zig build run-deferred-rendering
```

[Zero system dependencies](https://machengine.org/about/goals/#zero-fuss-installation) to slow you down; only [zig](https://machengine.org/about/zig-version/) is needed, we build and package the few relevant dependencies on our own. <small>[known issues](https://machengine.org/about/known-issues/)</small>

## Engine and Core split

We completely split _Mach engine_ and _Mach core_ apart, so that you get to choose your journey and decide if you just want low-level window+input+GPU and nothing else, or prefer to use our higher level engine (which although not ready for use yet, will be very modular itself):

<p align="center">
    <img class="color-auto" src="https://user-images.githubusercontent.com/3173176/184719710-ebae4fbd-af14-4b2f-80ad-11527c5f3e8f.png" />
</p>

## Mach core

<picture>
    <source media="(prefers-color-scheme: dark)" srcset="https://machengine.org/assets/mach/core-full-dark.svg">
    <img alt="mach-core" src="https://machengine.org/assets/mach/core-full-light.svg" style="height: 7rem; margin-top: 1rem;">
</picture>

Mach core aims to be a _truly cross-platform_ way to get _window+input+GPU, and nothing else._ It supports Linux, Windows, and Mac today, with WebAssembly / browser support in active development, and mobile coming in the future.

It gives you the power of Vulkan, DirectX, Metal, and modern OpenGL in a _single concise graphics API and shader language_ - by compiling Google Chrome's WebGPU implementation natively using Zig's build system.

Seamless multi-threading capabilities are provided, which means that your rendering and input handling are trivially decoupled from one another, you get butter-smooth window resizing, and your render loop and input handling can run at different frequencies. For example, a 60FPS render loop while your application handles keyboard & mouse events at a much faster dynamic rate (as fast as the OS can deliver them.)

<div style="align-self: center;">
    <video autoplay loop muted height="190px">
    <source src="https://media.machengine.org/core/example/gen-texture-light.mp4" type="video/mp4">
    </video>
    <video autoplay loop muted height="190px">
    <source src="https://media.machengine.org/core/example/boids.mp4" type="video/mp4">
    </video>
    <video autoplay loop muted height="190px">
    <source src="https://media.machengine.org/core/example/textured-cube.mp4" type="video/mp4">
    </video>
</div>

You can think of Mach core as an alternative to the classic options of SDL, GLFW+OpenGL, etc.

There are [15+ examples in the showcase](https://machengine.org/core/examples/), and we're [planning a C API](https://github.com/hexops/mach/issues/858) so it can be used from other languages as well.

## Engine development has begun

<strong>Mach engine is not ready for use yet, but we've started breaking ground on higher-level engine APIs.</strong>

The v0.2 release focuses on deep changes and improvements to our infrastructure, primarily building out the Zig gamedev ecosystem and building foundational packages that we needed for Mach core, the engine, and a game we're starting to build.

As a result, we've _finally_ just broken ground on the engine side of things.

<img src="/img/2023/mach-where-we-are.png">

## Breaking up our monorepo

Previously, all of Mach's [standalone packages](https://machengine.org/pkg/) were developed in a single giant monorepo. This was both intimidating for new contributors, and we wanted to better communicate how many standalone Zig gamedev packages we actually provide.

Today, we're happy to report all standalone packages are now developed in separate repositories and available via the package manager!

## Zig package manager

We migrated 100% to the self-hosted Zig compiler and the new experimental Zig package manager, every Git submodule has been banished!

We created [pkg.machengine.org](https://pkg.machengine.org/) - a mirror for downloading Mach packages and Zig downloads as well.

## Introducing Wrench the Machanist

<img src="https://raw.githubusercontent.com/hexops/media/b71e82ae9ea20c22a2eb3ab95d8ba48684635620/mach/wrench_rocket.svg" style="width: 300px">

Wrench is the Mach engine mascot (artwork contributed by [Keyla Jones](https://keylajones.me)); and also [our infrastructure automation](https://wrench.machengine.org/) tool, written in Go, to help us with various tasks:

Giving us an overview of our many repositories [CI statuses](https://wrench.machengine.org/projects/) and [pull requests](https://wrench.machengine.org/pull-requests/):

<a href="https://github.com/hexops/mach/assets/3173176/62c3e118-faa0-40c7-a663-e9e68e000bbf"><img src="https://github.com/hexops/mach/assets/3173176/62c3e118-faa0-40c7-a663-e9e68e000bbf" style="width: 300px"></a><a href="https://user-images.githubusercontent.com/3173176/210106853-3e72f102-e8a8-4154-bec3-e169922fd9af.png"><img src="https://user-images.githubusercontent.com/3173176/210106853-3e72f102-e8a8-4154-bec3-e169922fd9af.png" style="width: 300px"></a>

Sending us [pull requests](https://github.com/hexops/mach/pull/953) to automatically update our CI pipelines to the latest Zig version, and update our [`build.zig.zon`](https://github.com/hexops/mach/pull/926) dependencies - in a fully atomic way across all our repositories at once:

<a href="https://user-images.githubusercontent.com/3173176/210106831-afe34f1e-38fc-40ab-bee3-72374067838c.png"><img src="https://user-images.githubusercontent.com/3173176/210106831-afe34f1e-38fc-40ab-bee3-72374067838c.png" style="width: 300px"></a>

...and much more:

* Checking our website and docs for [broken links](https://github.com/hexops/mach/issues/931) and sending us GitHub issues.
* Performing automatic updates of involved dependencies, such as updating our fork of Google Chrome's WebGPU implementation, which involves [pull requests](https://github.com/hexops/mach-gpu-dawn/pull/22) across a few different repositories, pushing branches, running out-of-band commands, and ultimately presenting us with helpful/pretty diffs so we can just do the human work.
* A custom CI job runner system, running on a custom mini server with actual GPUs - so we can do screenshot-based testing of graphical applications in the future.

<video height="800px" controls>
    <source src="https://user-images.githubusercontent.com/3173176/234467605-ed255a5f-325e-449b-934b-971c2d4b4739.mp4" type="video/mp4">
</video>

<video height="800px" controls>
    <source src="https://user-images.githubusercontent.com/3173176/210107110-3ac1df91-1192-4113-96cd-748ae581c4f9.mp4" type="video/mp4">
</video>

## mach-gpu: rewritten for perfection

<picture>
  <source srcset="https://raw.githubusercontent.com/hexops/media/839b04fa5a72428052733d2095726894ff93466a/gpu/logo_dark.svg" media="(prefers-color-scheme: dark)">
  <img style="height: 100px;" src="https://raw.githubusercontent.com/hexops/media/839b04fa5a72428052733d2095726894ff93466a/gpu/logo_light.svg">
</picture>

[mach-gpu](https://machengine.org/pkg/mach-gpu/) is the WebGPU interface for Zig, and last year we [completely rewrote](../2022/perfecting-webgpu-native/) it, achieving:

* Zero overhead, using comptime interfaces
* 100% API coverage
* Default values for 100% of the API (which makes writing descriptors, and makes examples, look much simpler.)

## Audio development

Contributor [@alichraghi](https://github.com/alichraghi) has been relentless in pushing our audio capabilities (and more) forward

<div style="align-self: center">
    <picture>
        <source media="(prefers-color-scheme: dark)" srcset="https://machengine.org/assets/mach/flac-full-dark.svg">
        <img alt="mach-flac" src="https://machengine.org/assets/mach/flac-full-light.svg" style="width: 250px">
    </picture>
    <picture>
        <source media="(prefers-color-scheme: dark)" srcset="https://machengine.org/assets/mach/sysaudio-full-dark.svg">
        <img alt="mach-sysaudio" src="https://machengine.org/assets/mach/sysaudio-full-light.svg" style="width: 250px">
    </picture>
    <picture>
        <source media="(prefers-color-scheme: dark)" srcset="https://machengine.org/assets/mach/opus-full-dark.svg">
        <img alt="mach-opus" src="https://machengine.org/assets/mach/opus-full-light.svg" style="width: 250px">
    </picture>
</div>

[mach-sysaudio](https://machengine.org/pkg/mach-sysaudio/) started as Zig bindings to Andrew Kelley's awesome [libsoundio](https://github.com/andrewrk/libsoundio) library, and ended up being a fully-fledged new library written in Zig to achieve similar goals:

* Truly cross-platform, low-level, audio IO in Zig - playback and recording with backends for:
  * Linux
    * PulseAudio
    * PipeWire
    * Jack
    * ALSA
  * Windows: WASAPI
  * macOS/iOS: CoreAudio
  * WebAssembly: WebAudio

Then just recently we got [mach-flac](https://machengine.org/pkg/mach-flac/) and [mach-opus](https://machengine.org/pkg/mach-opus/), which combined give you FLAC (lossless audio) and Opus (lossy audio) via the respective battle-hardeneed xiph.org libraries.

## New website

<img src="https://github.com/hexops/mach/assets/3173176/6a1167bd-330c-47b4-8d8b-69e0c5cd0de2">

We built a [brand new website](https://machengine.org) that will serve us well into the future, featuring:

* [Project roadmap](https://machengine.org/engine/roadmap/)
* Documentation for Engine, Core, Packages, and more.
* WebGPU documentation and [learning material](https://machengine.org/engine/gpu/)
* Offline-viewing support (see link in the footer)
* All around better design, landing page, etc.

**Note:** Mobile support isn't great right now, we'll fix it later.

## Browser support: in development

Chrome has already shipped WebGPU, and others will follow soon. Mach support for WebAssembly is not yet ready, but is coming along nicely:

* Input, audio, etc. is working already ([piano demo](https://slimsag.com/mach/piano/), click in the frame and type with your A-Z keys.)
* `mach build` is a new CLI command written in Zig which:
  * Starts an HTTP development server
  * Invokes `zig build` for you when you reload the page
  * Generally provides a nice browser development experience

We currently do not have graphics support in the browser: we are currently doing a full rewrite of [mach-sysjs](https://machengine.org/pkg/mach-sysjs/) to use a code generation approach to enable Zig and JavaScript to communicate across the WebAssembly boundary, while being able to pass complex types (strings, slices, etc.) instead of just pointers and integers as normal. Once this rewrite is finished, we will generate a `gpu.Interface` implementation which simply invokes the browser's JavaScript WebGPU API with minimal overhead.

## Dusk: Experimental pure-Zig WebGPU implementation

<picture>
    <source media="(prefers-color-scheme: dark)" srcset="https://machengine.org/assets/mach/dusk-full-dark.svg">
    <img alt="mach-dusk" src="https://machengine.org/assets/mach/dusk-full-light.svg" style="height: 7rem; margin-top: 1rem;">
</picture>

Dusk is a highly experimental WebGPU implementation in Zig, aiming to be blazingly fast, lean & mean.

[@alichraghi](https://github.com/alichraghi) has been quietly working on Dusk tirelessly and consistently over the past year, and we think it may begin to be usable in Mach v0.3. Although it is not usable today, it already features:

* A [full WGSL shader parser and compiler](https://github.com/hexops/mach-dusk/tree/main/src/shader), based loosely on the Zig compiler, capable of emitting SPIRV.
* A partial Vulkan implementation.

Dusk is a long-term bet / investment for us, we intend to always have the option of using Dawn (the Google Chrome WebGPU implementation), and we don't expect Dusk will be the default very soon. Since both will implement the same `gpu.Interface`, it'll just be another backend you can select from at build time.

Learn more about our goals with Dusk [here](https://machengine.org/pkg/mach-dusk/#goals), and feel free to join the `#dusk` channel in Discord or check out the repository if you're interested in contributing some Vulkan, Metal, or Direct3D knowledge.

## Model loading

[mach-model3d](https://machengine.org/pkg/mach-model3d/) provides Zig bindings to [Model3D](https://gitlab.com/bztsrc/model3d/), a compact, featureful model format & alternative to glTF. We may replace this with our own model format in the future, but for now this enables us to load models from Blender in a decent, performant way.

<video height="800px" autoplay loop muted>
<source src="https://media.machengine.org/core/example/pbr-basic.mp4" type="video/mp4">
</video>

## Sprite / 2D examples

<img src="https://media.machengine.org/core/example/sprite2d.jpg">

mach-core now has a [sprite2d example](https://github.com/hexops/mach-core/tree/main/examples/sprite2d) which is ~400 lines and demonstrates loading sprites from a JSON file and sprite atlas, basic keyboard movement, etc.

We've already begun making a higher-level API for 2D graphics, as well - though not ready for use yet.

## Community

* Our [Discord community](https://machengine.org/discord) grew to over 700+ members, though we aim to keep all valuable information in GitHub issues and on the new website.
* We attended [Software You Can Love](https://softwareyoucan.love/) in Milan, Italy - gave a talk, had a table full of Zig gamedevs, and gave out some cool stickers
* We saw a number of new contributors, both one-off and ongoing.
* Many coffee was drank, and much coding was done over the holidays.

<div style="align-self: center;">
    <img src="https://github.com/hexops/mach/assets/3173176/43573ecf-35ce-4e34-831f-425151d5c281" style="height: 190px">
    <img src="https://github.com/hexops/mach/assets/3173176/ad30a37d-a37c-4950-b7b6-36411c9a51f1" style="height: 190px">
    <img src="https://github.com/hexops/mach/assets/3173176/9cfe64fe-ba2e-49a1-bfbe-f40b66abde2b" style="height: 190px">
</div>

## A personal note

<div style="display: flex;">
    <a href="https://github.com/slimsag">
        <img style="width: 420px" src="https://machengine.org/img/slimsag-profile.png">
    </a>
    <div>
        <p>I work a normal tech job, and every day after I sign off from work I go online to build Mach, almost like working two jobs. I've been working on Mach double-time like this for over two years now, and dreaming of it for a decade before that.</p>
        <p>FOSS <a href="https://devlog.hexops.com/2021/increasing-my-contribution-to-zig-to-200-a-month#i-grew-up-playing-linux-games-like-mania-drive">is in my roots</a> and I believe we should own our tools, they should empower <em>us</em>-not be part of <a href="https://kristoff.it/blog/the-open-source-game/">the 'open source' game</a> which is all too prevelant today (even among 'open source' engines.) Mach <em>needs</em> to be for people like you and me-it needs to genuinely be <a href="https://softwareyoucan.love">software you can love</a>.</p>
        <p>My dream is one day to live a simple, modest, future earning a living building Mach for you and creating high-quality games for everyone. Please consider <a href="https://github.com/sponsors/slimsag">sponsoring my work</a> if you believe in my vision.</p>
    </div>
</div>

## Thanks

Both to everyone who has contributed and sponsored the project, as well as you for reading this far!

<div style="display: flex; flex-direction: row; align-items: center;">
    <img align="left" style="max-height: 12.5rem;" src="https://user-images.githubusercontent.com/3173176/187348488-0b52e87d-3a48-421c-9402-be78e32b5a20.png"></img>
    <ul>
        <li>Join the <a href="https://discord.gg/XNG3NZgCqp">Mach Discord server</a></li>
        <li>Checkout <a href="https://machengine.org">machengine.org</a></li>
        <li>Consider <a href="https://github.com/sponsors/slimsag">sponsoring development</a> so we can do more of it!</li>
    </ul>
</div>
