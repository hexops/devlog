---
author: "Emi"
title: "Mach Engine: The future of graphics (with Zig)"
date: "2021-10-17"
draft: false
categories:
- mach
- zig
- gamedev
- graphics
description: "In the coming months, we'll begin to have truly cross-platform low-level graphics, with the ability to cross compile GPU-accelerated applications written in Zig from any OS and deploy to desktop, mobile, and (in the future) web."
images: ["https://user-images.githubusercontent.com/3173176/137651926-3734c3b2-4875-47de-b42f-0ece854756f7.png"]
---

In the coming months, we'll begin to have truly cross-platform low-level graphics, with the ability to cross compile GPU-accelerated applications written in Zig from any OS and deploy to desktop, mobile, and (in the future) web.

## Mach engine

<img class="color-auto" alt="Mach: Game engine & graphics toolkit for the future" src="https://user-images.githubusercontent.com/3173176/137651926-3734c3b2-4875-47de-b42f-0ece854756f7.png">

I've been working on [Mach Engine](https://github.com/hexops/mach) for about 4 months now, although it as a project is many years in the making, and I believe in the next 4-6 months we'll have completion of the first key milestone: truly cross platform graphics and seamless cross compilation.

## Vision

Today, I share only the first milestone: Mach engine core. I've been working on this for around 1 year now, and we're close (maybe 4-6 months away) from completion:

<a class="imglink" href="https://user-images.githubusercontent.com/3173176/137649720-072ff7fe-323d-49c6-ae88-51344e04e3e5.png"><img class="color-auto" alt="Zero fuss installation, out of the box cross compilation, and a truly cross-platform graphics API" src="https://user-images.githubusercontent.com/3173176/137649720-072ff7fe-323d-49c6-ae88-51344e04e3e5.png"></a>

## Zero fuss installation & cross compilation

Only `zig` and `git` are needed to build from any OS and produce binaries for every OS. You do **not** need any system dependencies, C libraries, SDKs (Xcode, etc.), C compilers or anything else.

We're able to achieve this thanks to two things:

1. Zig has fantastic cross-compilation support, including its own custom linker `zld` written by [Jakub Konka](http://www.jakubkonka.com/) which is capable of supporting MacOS cross compilation.
2. Mach doing the heavy lifting of packaging the required system SDK libraries and C sources for e.g. GLFW so our Zig build scripts can simply `git clone` them for you as needed for the target OS you're building for, completely automagically.

## Truly cross-platform graphics API

### DirectX 12, Metal, Vulkan & OpenGL

Imagine a low-level, little to no overhead graphics API that unifies DirectX, Metal, Vulkan, and OpenGL (if no others are available):

<a class="imglink" href="https://user-images.githubusercontent.com/3173176/137646296-72ba698e-c710-4daf-aa75-222f8d717d00.png"><img class="color-auto" alt="Simple, low-level unified graphics API mapping to DirectX 12, Metal, Vulkan, and OpenGL" src="https://user-images.githubusercontent.com/3173176/137646296-72ba698e-c710-4daf-aa75-222f8d717d00.png"></a>

_This isn't anything new:_ all modern engines provide this, Godot has been working towards this for _years_ (and still is), and there exist abstraction layers for Vulkan over most of these APIs as well.

### Vendor support

**An API is only as good as the momentum behind it.** What modern API can target the largest array of platforms with the most vendor backing?

<a class="imglink" href="https://user-images.githubusercontent.com/3173176/137647361-3340e33a-9b2f-4c0d-aba5-6bb99ffd1cd8.png"><img class="color-auto" alt="Google to Vulkan, Microsoft to DirectX, Apple to Metal, AMD and NVidia to everything." src="https://user-images.githubusercontent.com/3173176/137647361-3340e33a-9b2f-4c0d-aba5-6bb99ffd1cd8.png"></a>

* **Microsoft sees DirectX as the future, not Vulkan.** (DirectX 13 is coming by the end of 2022.)
* **Apple sees Metal as the future, not Vulkan.** OpenGL and OpenCL are deprecated, and private legal arguments with Khoronos make it unlikely we'll ever see OpenGL or Vulkan on Apple hardware ever again.
* Google, with their Fuschia OS [appears to be primarily into Vulkan](https://fuchsia.dev/fuchsia-src/concepts/graphics/magma) from a system-level POV.
* **NVIDIA, AMD, and Intel generally support as many graphics APIs as possible**, they want to sell hardware.

### One API that Apple, Microsoft, and Google can all agree on

<a class="imglink" href="https://user-images.githubusercontent.com/3173176/137647342-abf2bde6-a8bb-4276-b072-95c279c5d92f.png"><img class="color-auto" alt="Mozilla, Google, Microsoft, Apple, and Intel all to WebGPU" src="https://user-images.githubusercontent.com/3173176/137647342-abf2bde6-a8bb-4276-b072-95c279c5d92f.png"></a>

Outside the bounds of traditional graphics APIs there exists an attempt to provide a unified API across all platforms, [WebGPU](https://en.wikipedia.org/wiki/WebGPU) (not to be confused with the much older _WebGL_).

Mozilla, Google, Apple, and Microsoft all got together to build an abstraction layer over the modern graphics APIs - finding the common ground between Direct3D 12, Metal, and Vulkan - plus a safe way to expose that functionality in browsers.

The name _WebGPU_ might lead you to believe that this is only for browsers, and that it may not be low-level or fast - but this really couldn't be further from the truth.

### Apple & Google's role is what makes WebGPU unique, and why we chose it

<a class="imglink" href="https://user-images.githubusercontent.com/3173176/137648560-e15820d7-6427-4ebd-95bb-c7c9f026477a.png"><img class="color-auto" alt="Khronos group out of the piture in the future" src="https://user-images.githubusercontent.com/3173176/137648560-e15820d7-6427-4ebd-95bb-c7c9f026477a.png"></a>

What is new about WebGPU in my view is the vendors playing key roles in its development, and the fact that it grew outside the Khronos Group.

Although abstraction layers over modern graphics APIs are nothing new - as Apple, Google, and Microsoft continue to get more into manufacturing their own hardware (it's clear this is a strategic move for them) we should ask ourselves how this will change the landscape, and WebGPU is the first cross-vendor API to be produced by this new ecosystem.

### WebGPU extended thoughts

<details>
<summary>Is WebGPU "native enough"? Yes</summary>

<p>For browsers, WebGPU will require sandboxing and validation layers. But in native uses, this can all be turned off, and the WebGPU developers are clearly thinking about this use case:</p>

<ul>
<li>Google's implementation of WebGPU, <a href="https://dawn.googlesource.com/dawn">Dawn</a>, can be configured to effectively turn off all browser sandboxing / validation that could harm performance due to its client/server architecture.</li>

<li>Mozilla / gfx-rs Rust engineers have published articles such as <a href="http://kvark.github.io/web/gpu/native/2020/05/03/point-of-webgpu-native.html">"The point of WebGPU on native"</a>.</li>
</ul>

<p>As for the quality of implementations, we could compare the amount of resources going into e.g. Google's WebGPU implementation vs. the amount of resources going into Unity/Unreal/MoltenVK/other graphics abstraction layers - but I suspect they're <em>about equal</em>.</p>

</details>

<details>
<summary>Will WebGPU be implemented on GPUs natively? Maybe someday</summary>

<p>Not anytime soon. We get some insight into this <a href="https://github.com/gpuweb/gpuweb/issues/847#issuecomment-642883924">via @kvark</a>, a WebGPU developer:</p>

<blockquote>
  <p>[...] We are not in Khronos, and therefore we have limited participation from IHVs (only Intel and Apple are active). WebGPU was never designed to be implemented by the drivers. I mean, it would totally be rad, in the context of how usable WebGPU <a href="http://kvark.github.io/web/gpu/native/2020/05/03/point-of-webgpu-native.html">can be on native</a>, but it couldn't be the requirement from the start.</p>
</blockquote>

<p>But as WebGPU usage grows or even becomes prodominate due to it being the most powerful API in browsers, and as Microsoft, Google, and Apple continue to develop their own hardware - I think it's not unreasonable to think that it's possible some day WebGPU will be an even more direct 1:1 mapping between a cross-platform API and low-level APIs, more direct than Vulkan abstraction layers such as MoltenVK (which is required to get Vulkan working on top of MacOS's Metal API) - with the potential that some vendor starts asking "what would a GPU native WebGPU implementation look like?"</p>

</details>

<details>
<summary>Momentum of WebGPU vs. Vulkan</summary>

<p>To <a href="https://news.ycombinator.com/item?id=23090432">quote</a> <a href="http://kvark.github.io/about/">Dzmitry Malyshau / kvark</a>, a Mozilla engineer working on gfx-rs and WebGPU:</p>

<blockquote>
  <p>At some point, it comes down to the amount of momentum behind the API. In case of WebGPU, we have strong support from Intel and Apple, which are hardware vendors, as well as Google, who can influence mobile hardware vendors. We are making the specification and have resources to appropriately test it and develop the necessary workarounds. It's the quantity to quality transition that sometimes just needs to cross a certain threshold in order to succeed.</p>
</blockquote>

<p>According to some, Nvidia and AMD tend to develop new features with Microsoft as part of DirectX. Only then are they "ported" back to Vulkan and OpenGL. I think that says a lot.</p>

</details>

## What progress has been made so far on Mach Engine?

Today, we have cross-compilation of GLFW on all desktop OSs working out of the box with nothing more than `zig` and `git`:

<a class="imglink" href="https://user-images.githubusercontent.com/3173176/137650099-cd370046-eb43-4fe4-a72a-f54ebe3153c1.png"><img class="color-auto" alt="Cross compilation from Mac, Linux, and Windows to eachother on all major architectures." src="https://user-images.githubusercontent.com/3173176/137650099-cd370046-eb43-4fe4-a72a-f54ebe3153c1.png"></a>

This involved:

* [Packaging MacOS SDKs](https://github.com/hexops/sdk-macos-11.3) and [Linux system X11/Wayland libraries](https://github.com/hexops/sdk-linux-x86_64) into SDKs, and creating Zig build scripts that could merely `git clone` them and utilize them for cross-compilation.
* Purchasing Apple M1 hardware to test on, and for GitHub Actions as it doesn't support it.
* Normalizing symlinks in Mac/Linux SDKs everywhere so that Windows users don't have a hard time with Git symlink management.
* Contributing [a small fix to the Zig linker](https://github.com/ziglang/zig/pull/9734)

All this to say, we're really taking a holistic approach to achieve this.

## What's next? WebGPU

I'm happy to report that a fair amount of progress on this front has been made.

Here is Google's WebGPU implementation, Dawn, compiled using `zig`:
<a class="imglink" href="https://user-images.githubusercontent.com/3173176/137650403-290c6a94-7ee4-44be-8ed0-94f96adcde4e.png"><img alt="A red triangle in a black window titled 'Dawn Window', the" src="https://user-images.githubusercontent.com/3173176/137650403-290c6a94-7ee4-44be-8ed0-94f96adcde4e.png"></a>
<a class="imglink" href="https://user-images.githubusercontent.com/3173176/137650621-f304f20b-5f74-4a3d-956d-7feb3838351d.png"><img alt="A Zig code file, hello_triangle.zig showing Dawn and WebGPU API usage in Zig" src="https://user-images.githubusercontent.com/3173176/137650621-f304f20b-5f74-4a3d-956d-7feb3838351d.png"></a>

This includes:

* A ~500 line port of the `hello_triangle` example from Dawn to Zig
* A ~1200 line `build.zig` file which compiles all the Dawn sources using Zig, without using Google's ninja/etc development tools.
* A hack to workaround a bug in Zig where ObjC++ `.mm` files are not yet recognized.
* C shims for the `dawn_native` C++ API and utility APIs, which are required in order to bind Dawn to an actual GLFW window.

There are a few weeks of work to do before this can be merged and will be usable by others, please stay tuned for that.

After that will be development of idiomatic Zig bindings to the [WebGPU C API](https://github.com/webgpu-native/webgpu-headers) which is shared between implementations such as Dawn and the Rust's [gfx-rs/wgpu-native](https://github.com/gfx-rs/wgpu-native) implementation (we could theoretically switch between them at startup in the future, but we'll probably stick with Dawn as it does not require a separate Rust toolchain and it would prevent out-of-the-box cross compilation.)

## When will there be games, examples, etc.?

It'll be a while because I am focusing purely on the groundwork first. It's unlikely you'll see anything with _real demo value_ before later next year.

I'm sure that will be disheartening to hear - and may make you to think there's nothing of substance here. I totally understand that view, but I hope you'll stay tuned because I'm in this for the long haul and it's not my first rodeo (I previously spent 4 years writing [a game engine in Go](https://azul3d.org), and have worked [at a devtools startup for 7 years](https://sourcegraph.com), with my biggest lesson from of those experiences being the importance of demos and examples.

## Follow along

Major developments will be posted here, as well as on Twitter [@machengine](https://twitter.com/machengine).

You can also follow the project at [github.com/hexops/mach](https://github.com/hexops/mach).

If you like what I'm doing, you can [sponsor me on GitHub](https://github.com/sponsors/emidoots).

Thanks for reading!
