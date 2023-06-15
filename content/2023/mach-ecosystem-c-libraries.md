---
author: "Stephen Gutekanst"
title: "Mach: providing an ecosystem of C libraries using the Zig package manager"
date: "2023-06-14"
draft: false
categories:
- mach
- zig
- gamedev
description: "For Mach engine, we're maintaining an ecosystem of C libraries packaged up with the new experimental Zig package manager. Whether you use Zig the language, or just its build system to build C/C++ applications, you may find this overview of our ecosystem informative."
images: ["https://github.com/hexops/devlog/assets/3173176/cc6c6035-a2e4-4384-9a15-134679cbb1b0"]
---

[Andrew Kelley](https://github.com/andrewrk) gave a keynote speech at [Software You Can Love 2023](https://softwareyoucanlove.ca) in Vancouver last week (a recording will be available later), the outline was:

> **How to Build Software From Source**
> 
> [...] then I'll take things in a completely different direction, by **showing you how to rip apart a project's build system and replace it with the zig build system.** This will make building things from source work effortlessly for more people and more platforms, as well as annoy a lot of boomers. It's going to be super fun and spicy!

## As we all know, Zig is a C/C++ compiler and its own build system

Zig is really three things:

* Programming language
* Build system (build.zig) replacing makefiles/cmake/ninja/etc
* C/C++ compiler with an emphasis on cross-compilation

We've been leveraging all three in [Mach engine](https://github.com/hexops/mach) for a while now. For example, we maintain a version of Google Chrome's WebGPU implementation (Dawn) with its rather complex build system (code generation, python scripts, depot_tools, ninja, cmake, depot_tools, etc.) replaced with `build.zig`.

That let's us say that if you have [a recent Zig version](https://github.com/hexops/mach#supported-zig-version) you can get started with Mach in ~60s on Windows, Mac, and Linux:

```
git clone --recursive https://github.com/hexops/mach-examples
cd mach-examples/
zig build run-textured-cube
```

And instead of getting a bunch of dependency errors that you might need to `apt-get` install or whatever, you'll just get something that works out of the box:

<video autoplay loop muted playsinline style="width:24rem">
<source src="https://user-images.githubusercontent.com/3173176/210317154-90e7a41c-2b44-4ee6-956f-5a93285e19ef.webm" type="video/webm"></video>

## Zig has a new package manager (for C/C++ too!)

Those in the Zig community know that Zig has a new package manager, it's built into the compiler. Effectively you describe your dependencies in a `build.zig.zon` file, and then the `zig` compiler is able to fetch them for you as part of `zig build`. You're then able to link against/use dependencies in your `build.zig` file, which declaratively says how to build your project (except, using a real language instead of a DSL like cmake/etc use.)

It's still very experimental, [has little to no documentation yet](https://github.com/ziglang/zig/pull/14265) - it's not ready for widespread use. But one strong point is that it also aims to address the issue of building C/C++ projects, not just Zig ones. You can write a `build.zig` file in Zig, describing how to build your C/C++ project using Zig as the toolchain. Then for free you get quite solid cross-compilation (since Zig bundles clang, every glibc version, and more), plus now a dependency manager, as well as a declarative way to describe your build using the Zig language.

One example of this is in Andrew Kelley's [fork of ffmpeg](https://github.com/andrewrk/ffmpeg/), where he merely forked the ffmpeg repository, removed their build system & unnecessary files, and added [a `build.zig` file](https://github.com/andrewrk/ffmpeg/blob/main/build.zig). This allows you to clone the repository and `zig build` will fetch all the required dependencies and build ffmpeg for you. Fancy!

## Mach engine

[Mach engine](https://github.com/hexops/mach) is an upcoming game engine built in Zig, that we're building with the aim of becoming competitive with Unity/Unreal/Godot - but with an emphasis on _modularity_:

* **Mach core**: If you choose to use _core_, then it's like an alternative to GLFW+OpenGL or SDL, you just get a window+input+WebGPU with minimal dependencies. Your application runs natively on Windows/Linux/Mac using their respective graphics APIs (DirectX12, Vulkan, Metal), you get cross-compilation and zero-fuss installation, and also web/mobile support in the future with the same codebase. Write once, run everywhere.
* **Mach engine**: If you choose this option, you _additionally_ get an entity component system - with a library of standard modules that you can 'plug and play' with for rendering/audio/etc.

### Keeping our runtime C dependencies small

One way that we're keeping _runtime C dependencies_ (the ones your game/app would ship with!) a smaller, focused, set - is by building tooling: a `mach` CLI and fully-fledged GUI editor like other engines have. But how does that help reduce runtime dependencies? Well, at runtime you may need:

* Harfbuzz: for Unicode text layout
* GLFW (and some headers): for window management)
* Basisu and PNG: for GPU supercompressed textures / lossless textures
* Opus and FLAC: for lossy and lossless audio

Mach will 'bless' certain formats, being opinionated in what you _ship_ with your game. You're free to pull in other formats, if you like, but the default/easy path will be these ones. As a result, there's a lot we _won't_ need at runtime:

* Freetype
* JPEG, TGA, or other image formats
* MP3, ffmpeg, or other audio formats

We don't need these because our _tooling_ (the CLI and GUI editor) is going to make it easy to convert whatever format you want into the 'blessed' runtime formats. One major benefit of this is that we can nudge you to the right defaults, without you being an expert. For example, you probably want to be using texture compression formats that GPU hardware itself understands, instead of say shipping a JPEG that just gets expanded to an uncompressed texture, eating a bunch of GPU memory and harming your texture bandwidth.

## Providing an ecosystem of C libraries

Similar to Andrew Kelley's ffmpeg fork (although, with a few niceties to verify the supply chain) - Mach is now maintaining forks of various C libraries that we make use of. These aren't Zig bindings to these libraries (which we have separately), but rather are just forks of the actual project with their build system replaced by `build.zig`.

A massive special thanks to [@LordMZTE](https://mzte.de/git/) who has been tirelessly pushing us along here over the past month-ish, helping to inch us ever-closer to fully adopting the new package manager.

### Forks we maintain

We have _forks_ of these projects which switch their build systems to `build.zig`:

* [hexops/harfbuzz](https://github.com/hexops/harfbuzz)
* [hexops/freetype](https://github.com/hexops/freetype)
* [hexops/brotli](https://github.com/hexops/brotli)
* [hexops/glfw](https://github.com/hexops/glfw)
* [hexops/basisu](https://github.com/hexops/basisu) (basis_universal, supercompressed textures)

### A note about supply chain verification

I personally care a lot about supply chain security - and more importantly, bugs. In general, I don't ever want anyone to have to 'wonder' if our fork of a library has some strange patches applied to it or something.

As a result, in each of these forks we've taken the time to ensure _you_ know the exact `git diff` command you can run to verify that our fork _exactly_ matches the upstream version - with the only difference being _removing the project's old build system, and unnecessary files_.

### Header packages we're maintaining

In addition to the above, we're maintaining the following which aren't strict forks (a repository for each would simply be too much for us to maintain), but rather are collections of common headers that you very often need together. These can help you build GLFW, SDL, and other such applications.

Some headers are generated with platform-specific tools (e.g. in the case of Wayland this is needed.) We always provide the exact steps we used to produce the headers from upstream repositories in an `update-upstream.sh` script, with the intent that you can fully reproduce what's in these packages and have confidence it came from the upstream repository.

* [hexops/vulkan-headers](https://github.com/hexops/vulkan-headers)
* [hexops/linux-audio-headers](https://github.com/hexops/linux-audio-headers) includes:
  * ALSA
  * Jack
  * PipeWire
  * PulseAudio
  * SPA
* [hexops/x11-headers](https://github.com/hexops/x11-headers) includes:
  * x11
  * xcb
  * xkbcommon
  * xcursor
  * xrandr
  * xfixes
  * xrender
  * xinerama
  * xi
  * xext
  * xorgproto
  * GLX
* [hexops/wayland-headers](https://github.com/hexops/wayland-headers) includes:
  * xdg-shell
  * xdg-decoration
  * viewporter
  * pointer-constraints-unstable-v1
  * relative-pointer-unstable-v1
  * idle-inhibit-unstable-v1

### How do I use these?

Later we'll provide more details specifically on how to use these. Effectively you just add them to a `build.zig.zon` file next to your `build.zig` and then call `b.dependency("name")` to retrieve each one. If you need more help than that, you might need to [join our Discord](https://discord.gg/XNG3NZgCqp) because as mentioned previously the Zig package manager is pretty immature and has sharp edges today. There are [lots of known issues & bugs](https://github.com/hexops/mach/issues/721) that prevent even us from using it fully today.

But, it is coming along rather quickly! We wanted to let the broader Zig community know we're maintaining these packages to help with collaboration.

## Help us become sustainable

We're working towards Mach v0.2, this article was one of the first steps in beginning to share the progress we've been making towards that behind the scenes over the past several months. We have some exciting things to share next, this was the 'boring' article that had to go first. :)

<img align="left" style="max-height: 150px;" src="https://user-images.githubusercontent.com/3173176/187348488-0b52e87d-3a48-421c-9402-be78e32b5a20.png"></img>
<br><br>
Consider [sponsoring my work](https://github.com/sponsors/slimsag) to help us become a sustainable OSS project and enable us to do more in the future.
<br><br>
Join the [Mach Discord](https://discord.gg/XNG3NZgCqp) where we're building the future of Zig game development in realtime!
