---
author: "Stephen Gutekanst"
title: "Building the DirectX shader compiler better than Microsoft?"
date: "2024-02-09"
draft: false
categories:
- mach
- zig
- gamedev
description: "This is a story about the messy state of Microsoft’s DirectX shader compiler, and trying to wrangle it into a nicer experience for game developers. In some respects, building the DXC compiler better than how Microsoft does."
images: ["/img/2024/building_dxc_better_than_microsoft.png"]
---

This is a ~~story~~ nightmare about the messy state of Microsoft's DirectX shader compiler, and trying to wrangle it into a nicer experience for game developers. In some respects, we now build the DXC compiler better than how Microsoft does.

## Setting the stage

For [Mach engine](https://machengine.org) we've been building an [experimental graphics API called sysgpu](../mach-v0.3-released/#sysgpu) using Zig, aiming to be a _successor_ and _descendant_ of WebGPU for native graphics. It will support Metal, Vulkan, Direct3D, and OpenGL backends. As part of this, we need to compile shader programs into something that Direct3D 12 can consume. But what does it consume?

## A brief history lesson

The DirectX graphics API uses HLSL as its shading language of choice. In the past, with Direct3D 11 and earlier, this compiler was called 'FXC' (the 'effects compiler')

### FXC is deprecated, DXC enters the scene with Direct3D 12

Unfortunately, FXC as a compiler is rather notoriously slow among game developers, with suboptimal code generation - meaning shaders often both compile and execute fairly suboptimally.

With the release of Direct3D 12 and Shader Model 6.0 (SM6), Microsoft officially deprecated the FXC compiler distributed as part of the Windows OS in favor of a new compiler called 'DXC' ('directx compiler'), which exists as a public Microsoft-official fork of LLVM/Clang v3.7 [Microsoft/DirectXShaderCompiler](https://github.com/microsoft/DirectXShaderCompiler) and prebuilt binaries you can download.

In this Microsoft fork of LLVM, changes are meticulously annotated via `// HLSL Change Start` and `// HLSL Change End` comments making it clear who owns what code:

<a class="imglink" href="/img/2024/hlsl-change-start"><img src="/img/2024/hlsl-change-start.png"></a>

### What a DirectX driver eats for breakfast: DXBC or DXIL

Although HLSL is the language of choice for Direct3D programming, at the end of the day GPUs under the hood all have different compute architectures and requirements: the compiled binary form of a shader program that an Intel GPU needs is going to be different from what an NVIDIA GPU needs, same goes for AMD.

Microsoft's role is to provide the nice game developer frontend APIs (like Direct3D, and the HLSL shanding language), while working with independent hardware vendors (IHV's) like Intel/AMD/NVIDIA who write the drivers - bridging those nice frontend APIs to whatever is hopefully closest to hardware manufacturer's instruction set architecture (ISA) under the hood. You can think of it like web browsers making sure JavaScript can run on both a Windows PC and a macOS Apple Silicon device, though graphics developers would spit at the suggested comparison.

DirectX versions 9-11 had driver manufacturers consuming what is called DXBC (DirectX Byte Code) - game developers would produce DXBC either using a CLI tool to compile their HLSL programs like `fxc.exe`, or at runtime using the `d3dcompiler` APIs, and then the driver's job was to take that decently-optimized shader bytecode and turn it into the actual binary that the GPU would run. This bytecode was an undocumented, proprietary format really only shared between Microsoft and GPU driver manufacturers - excluding a few odd-ball Linux developers who cared to reverse engineer it for Proton.

With the advent of DirectX 12 and Shader Model 6.0, Microsoft aspirationally had intended to create their own standard IR called DXIR, but in 2021 they [removed all language suggesting they might do this](https://github.com/microsoft/DirectXShaderCompiler/commit/61c6573842be58a14e1dfc6b1b3def03d39d9988). The intent _was_ for DXIR to be the 'high level', 'unoptimized' IR form which compilers (think: Rust) could target, and then the DXC compiler could lower DXIR into the optimized DXIL bytecode form, a new 'low level' post-optimization IR format, before handing it off to graphics drivers to muck with as they please before it gets translated to run on the actual hardware.

Asked about DXIR documentation, a [Microsoft employee](https://github.com/microsoft/DirectXShaderCompiler/issues/2389#issuecomment-517076643) had noted this in 2019:

> Unfortunately, documentation on the lowering process [from DXIR to DXIL] is mostly non-existent. [...]
> 
> Oh, and DXIR isn't anything official, but just the first LLVM IR after CodeGen.

As you'll soon see, this theme of 'there are no docs, just whatever our compiler actually does' will become a common pattern.

### DXIL

DXIL (pronunciation?) is the official format that DirectX 12 driver manufacturers consume _today_.

A game developer produces DXIL bytecode using the DXC compiler, which is a fork of LLVM/clang heavily modified to support HLSL compilation, and the DirectX APIs hand that DXIL over to the graphics driver which then converts the IR into their own intermediate languages, performing any secret sauce optimization passes on it, and ultimately boiling down to the actual machine code that will run on the GPU hardware.

Much like the old bytecode format DXBC which DXIL replaced, it is _also_ an undocumented bytecode format, specifically it is LLVM's version 3.7 post-codegen post-optimization-passes bitecode format. It is undocumented not because nobody wants to document it, but rather because the documentation is literally 'whatever the Microsoft fork of LLVM v3.7 with all the HLSL changes we made, after CodeGen and optimization passes have occurred, actually emits as LLVM bitcode - plus a small custom container/wrapper file format on top.'

### Correcting the Microsoft fork of LLVM

Microsoft themselves are well aware that a bunch of independent driver manufacturers relying on and expecting to consume a hyper-specific undocumented LLVM bitcode format specifically produced by their fork is, well, less than ideal - and also aware that their fork of LLVM is not super fun to maintain, either. Quoting [another Microsoft employee](https://github.com/microsoft/DirectXShaderCompiler/issues/5773#issuecomment-1735794551) (Sep 2023) who was asked about the potential of adding DirectX 9/10/11 support to the new/better DXC compiler, they stated:

> DXC's fork of LLVM removed and/or damaged much of the code generation layer and infrastructure [of LLVM]. Given that, supporting DXBC generation in DXC would be a massive task to fix and restore broken LLVM functionality. Due to the large scale of this issue and resource constraints on our team we're not going to address this issue in [the new] DXC [compiler] ever.
>
> We may support DXBC generation in Clang in the future (we mentioned that in the original proposal to LLVM). That work is unlikely to begin for a few years as our focus will be on supporting DXIL and SPIR-V generation first.

As noted above, in March of 2022, Microsoft had proposed and begun work on [upstreaming HLSL compilation support directly into LLVM/clang proper](https://discourse.llvm.org/t/rfc-adding-hlsl-and-directx-support-to-clang-llvm/60783) - work that is still ongoing today - and involved _adding back_ legacy LLVM v3.7 bitcode writing support to modern LLVM/clang versions:

> By isolating as much of the DXIL-specific code as possible into a target we hope to minimize the cost on the community to maintain our legacy bitcode writing support.

i.e. the plan to get away fromn their fork is to upstream HLSL and DXIL support to LLVM/clang proper.

## The challenge for gamedevs, WebGPU, etc.

Graphics abstraction layers which aim to provide a unified interface to modern graphics APIs like Metal, Direct3D 12, and Vulkan.. ultimately need to provide a unified shading language as well. If you look today, you'll find most WebGPU implementations which do this have had a goal of 'in the future we might be able to emit DXIL directly..' but in practice, none actually do.

Instead, basically every WebGPU implementation today behaves as follows:

* The WGSL textual language first gets translated to HLSL at runtime
* HLSL is compiled into DXBC or DXIL using an HLSL compiler
* The optimized DXBC/DXIL is handed to the graphics driver, which then gets converted to the various vendor-specific ILs before finally becoming machine code that runs on the GPU.

### A quick detour: SPIR-V

Vulkan/SPIR-V does much the same as the above, in fact most drivers cannot assume SPIR-V is optimized at all - though some do, and this varies by mobile/desktop GPUs - and have more work to perform to get SPIR-V turned into a _driver-compiled_ native binary.

Valve has [Fossilize](https://github.com/ValveSoftware/Fossilize) and maintains caches of each specific (GPU, driver version, etc.) pairing along with the _actual_ driver-compiled binary for a SPIR-V blob, to enable downloading 'pre-cached shaders' from Valve servers ahead of playing games for this reason: so that you don't spend all day waiting for your computer to go brrr compiling and optimizing SPIR-V shaders into actual native code your GPU understands.

In other words, DXIL is always post-optimization-passes LLVM _bitcode_, while SPIR-V can or cannot be an an optimized form, and GPU manufacturers write their drivers based on what SPIR-V looks like in the wild - which may or may not be a pre-optimized form. SPIR-V is closer to hardware than a textual shading language, but still very far from native machine code a GPU understands.

Only Apple's Metal graphics API supports compiling directly to the actual target hardware's native binary format (thanks to that iron fist they hold over their hardware, I guess.)

## To use dxcompiler.dll or not?

Since WGSL->HLSL->DXIL is happening at runtime, WebGPU runtimes are faced with a challenge: do we use the new DXC HLSL compiler, or the old, officially deprecated FXC compiler which has worse performance and codegen quality? On the surface, this hardly sounds like a difficult choice!

However, despite this, many indie devs and game engines choose to use FXC by default. [Bevy game engine's documentation](https://docs.rs/bevy/latest/i686-pc-windows-msvc/bevy/render/settings/enum.Dx12Compiler.html) puts it really well:

> The Fxc compiler (default) is old, slow and unmaintained. However, it doesn’t require any additional .dlls to be shipped with the application.
>
> The Dxc compiler is new, fast and maintained. However, it requires both `dxcompiler.dll` and `dxil.dll` to be shipped with the application. These files can be downloaded from https://github.com/microsoft/DirectXShaderCompiler/releases.

As a result, much software defaults to the old, slow and unmaintained compiler. And it's not just Bevy: `wgpu` Rust users, Dawn WebGPU users, etc. are all faced with this same question. It's likely one of the reasons WebGPU does not support Shader Model 6.0+ functionality today - using the DXC compiler is not so pleasant: it is after all a large, clunky Microsoft fork of a C++ codebase from nearly a decade ago!

## Well, why not just statically link against it?

You can't.

Firstly, there is the issue that [Microsoft's fork of LLVM doesn't support statically linking](https://github.com/microsoft/DirectXShaderCompiler/issues/4766). On the surface, this appears just to be due to some cmake files assuming `SHARED` instead of `STATIC` when creating libraries, but if you dig into it - as I did - you'll soon find it is _much_ more involved than that.

Switching `SHARED` to `STATIC` everywhere in CMake files will appear to get you a build with ~15 different static libraries to link against (not pleasant compared to just one.) You might think using cmake `OBJECT` libraries could solve this, but with this you will quickly encounter an issue where although the cmake files are structured logically as dependants, they actually have implicit dependencies on eachother due to the HLSL changes Microsoft made. I am 80% sure you would need to rewrite every cmake file in the repository to support OBJECT libraries. I can say this, because I tried!

You might be thinking, linking against ~15 static libraries isn't SO bad as long as the final executable is static, right?

Not so fast - many parts of DXC's COM interface implementation is also explicitly designed to load itself as a DLL, i.e. to load `dxcompiler.dll` and `dxil.dll` as dynamic libraries and self-invoke methods.

OK, we just need to patch the implementation to not call `LoadLibraryW` then, basically, right?

## Introducing dxil.dll - the proprietary code signing blob for DirectX shaders

If you've ever built DirectXShaderCompiler from source, you might notice something: dxil.dll doesn't get built. Why? It's distributed in every release on GitHub, both for Windows (x86/arm) and Linux (x86 only).

Strange, I thought the compiler was supposed to be open source? Well, it wouldn't be the first time[[0]](https://github.com/microsoft/win32metadata/issues/766#issuecomment-1150271300)[[1]](https://github.com/microsoft/Azure-Kinect-Sensor-SDK/issues/1521) I've encountered a Microsoft 'open source' repository that actually completely depends on some proprietary platform-specific code blobs behind the scenes.

Incidentally, I stumbled across the [D3D12 Shader Cache API specification](https://microsoft.github.io/DirectX-Specs/d3d/ShaderCache.html) which mentions the existence of this proprietary code signing blob as a 'good reason for invoking the shader compiler at runtime':

> D3D12 will only accept signed shaders. That means that if any patching or runtime optimizations are performed, such as constant folding, the shader must be re-validated and re-signed, which is non-trivial.

And in the recent ['preview release' for Shader Model 6.8 functionality](https://github.com/microsoft/DirectXShaderCompiler/releases/tag/v1.8.2306-preview), Microsoft notes how they appear to leverage this DLL to restrict new experimental shader functionality:

> The DXIL signing library (dxil.dll/libdxil.so) is not provided with this preview release. DXIL generated with this compiler targeting Shader Model 6.8 is not final, cannot be validated, and is not supported for distribution or execution on machines not running in developer mode.

In other words: if you do not have dxil.dll, then your shaders will not be signed/validated. If your shaders are not signed/validated, then they cannot run on a Windows machine unless it is running in Developer Mode.

## Platform support challenges

For a second, I'd like to go back to something I wrote at the start of this article:

> For [Mach engine](https://machengine.org) [...] we need to compile shader programs into something that Direct3D 12 can consume.

I'd like for us to be able to perform offline shader compilation, and skip out on distributing the heavy DXC dependency, when desired.

But Microsoft only distributes a copy of dxil.dll for Windows (x86/arm) and Linux (x86). There's no Linux aarch64 binary. There's no macOS binary. In other words, you can't produce builds of your cross-platform game for Windows using offline shader compilation on a mac, or in your Arm Linux CI pipeline. You need a Windows or x86_64 Linux machine to run the proprietary blob.

## Recap

To recap:

* We cannot build DXC as a static library, because the decades-old Microsoft fork of LLVM v3.7 has a very messy build-system.
* Even if we could, we cannot build DXC as a static library **because of the proprietary code-signing blob**.
* We cannot compile DirectX HLSL shaders offline on a Mac, or build our cross-platform game in an arm Linux CI pipeline, because Microsoft doesn't distribute copies of **the proprietary code signing blob** for those platforms.

## Going deeper

### Un#$@&%*! the build system

The first problem I wanted to address was how to actually build this codebase into a single static library.

After several days of attempting to fix the implicit dependencies that changing the cmake virtual libraries from `DYNAMIC` -> `OBJECT` surfaces, I gave up. Originally, my intent was to use their existing cmake build system (so as to not diverge from their codebase too much) and just swap the compiler with `zig cc` as the build toolchain for cross-compilation.

After it slowly and painfully became apparent that direction was not going to be _any_ better than maintaining the entire buildsystem myself, I decided to just bite the bullet and rewrite the entire CMake build system they had, some ~10.5k lines of code, using `build.zig` instead. To make things simpler, I chose to build only the two parts we (and others) really care about as consumers of the code: the `dxcompiler.dll` library, and `dxc.exe` binary for offline compilation / testing. (we'll deal with `dxil.dll` later.)

This resulted in somewhere around [~1k lines of build.zig logic](https://github.com/hexops/mach-dxcompiler/blob/bd0cfbe4230133d8d3b50eedf1a0d0c4a00f47d7/build.zig#L1-L956), and in practice it's less than that because much of it is just related to running `git clone` on the source repository, having the ability for Zig package consumers to use a prebuilt binary instead of building the large C++ library from source, and header/source generation (though we're still not done with that, thanks to llvm-tablegen)

### Un#$@&%*! the dynamic library dependency

As mentioned earlier, DXC is written with the expectation that `dxcompiler.dll` and `dxil.dll` exist. Reading the code, it almost appears as if the COM API implementation invokes the DLL, which then invokes itself dynamically depending on which is available.

Taking some advice from Microsoft, I got my hands dirty, _forked their codebase_ and got to work on the actual C++ code. I began annotating my changes with cute `// Mach change start` and `// Mach change end` comments, to know who owns what code. All of this existing as a choice that I hope will come back to haunt my dreams in the future as much as Microsoft's own choice to underemploy the HLSL team and fork LLVM 3.7 originally.

I was off to the races: [simulating dllmain](https://github.com/hexops/DirectXShaderCompiler/blob/4190bb0c90d374c6b4d0b0f2c7b45b604eda24b6/tools/clang/tools/dxcompiler/DXCompiler.cpp#L88) entrypoints, [disabling](https://github.com/hexops/DirectXShaderCompiler/blob/4190bb0c90d374c6b4d0b0f2c7b45b604eda24b6/tools/clang/tools/dxclib/dxc.cpp#L1258) the ability to print the compiler version info derived from the dlls, and [emulating dynamic library function pointer loads](https://github.com/hexops/DirectXShaderCompiler/blob/4190bb0c90d374c6b4d0b0f2c7b45b604eda24b6/include/dxc/Support/dxcapi.use.h#L17).

### Un#$@&%*! the proprietary code signing

All that was left was that pesky `dxil.dll` - what sort of magic might Microsoft be employing in that library to "sign shaders"? How can they prevent unsigned shaders from running on Windows machines that aren't in developer mode? How are they able to distribute that binary on Linux, too?

I won't comment on any of those questions, but will say that [you'll find dxil.dll is NOT a dependency of mach-dxcompiler in any form](https://github.com/hexops/DirectXShaderCompiler/blob/4190bb0c90d374c6b4d0b0f2c7b45b604eda24b6/tools/clang/tools/dxcompiler/MachSiegbertVogtDXCSA.cpp#L178). You can compile an HLSL shader on a macOS machine using mach-dxcompiler, without the proprietary `dxil.dll` blob - and end up with a DXIL bytecode file that is byte-for-byte equal to one which runs it on a standard Windows box. Enjoy!

## Results

We now have prebuilt, static binaries of the `dxcompiler` library, as well as the `dxc` CLI [here](https://github.com/hexops/mach-dxcompiler/releases/tag/2024.02.10%2B2c3635c.1), with zero dependency on the proprietary `dxil.dll`. At the time of writing, we have binaries building in our CI pipeline for:

* macOS (the first ever in history), both Apple Silicon (aarch64) and Intel (x86_64).
* Linux, including musl and glibc, as well as aarch64 (first ever in history) and x86_64.
* Windows, x86_64 and aarch64, including for MinGW/GNU ABI (first ever in history?)

Additionally included is a [small C API](https://github.com/hexops/mach-dxcompiler/blob/main/src/mach_dxc.h) the library now exposes, as an alternative to the COM API traditionally required.

Zig game developers will find the repository also includes a Zig API, see [`src/main.zig`](https://github.com/hexops/mach-dxcompiler/blob/main/src/main.zig) tests for usage. By default prebuilt binaries are downloaded/used.

You can [build from source yourself](https://github.com/hexops/mach-dxcompiler) for any OS/arch with only `zig` and `git`, just make sure you have [the right Zig version](https://machengine.org/about/zig-version/):

```
git clone https://github.com/hexops/mach-dxcompiler
cd mach-dxcompiler/
zig build -Dfrom_source -Dtarget=aarch64-macos
zig build -Dfrom_source -Dtarget=x86_64-windows-gnu
zig build -Dfrom_source -Dtarget=x86_64-linux-gnu
```

## Caveats

It's not all roses - there are some drawbacks:

* Windows MSVC ABI binaries are currently not building due to a small bug in the C bindings - will fix it quickly if important for you, otherwise at our own pace.
* Linux musl binaries are untested, they build fine and I'd be curious to know if they run fine!
* With Mach engine, we plan to use Zig itself as our shading language, not HLSL, so I do not build SPIRV-output support, sorry! I have no plans to add it.
* No plans to update this to support SM6.7 currently (released very recently), though perhaps in the future.
* LLVM's cmake build system is not trivial, there are some aspects yet-to-be-translated. See `generated-include/` for specifics which come from the cmake build system still.
* If you use this, you'll be relying on myself to fix/address any issues. I am the only person working on this, and it exists solely to solve Mach's own problems. If it works for you, great - but there may be a time we find a better path forward for us and it could get deprecated, so keep that in mind.

## On a personal note

<div style="display: flex;">
    <a href="https://github.com/slimsag">
        <img style="width: 420px" src="https://machengine.org/img/slimsag-profile.png">
    </a>
    <div>
        <p>My name is Stephen, I work a normal tech job, and after signing off from work at the end of the day I go online to build <a href="https://machengine.org/">Mach engine</a>. I've been dreaming of being able to build a game engine like this for a long time, and I'm finally doing it!</p>
        <p>FOSS <a href="https://devlog.hexops.com/2021/increasing-my-contribution-to-zig-to-200-a-month#i-grew-up-playing-linux-games-like-mania-drive">is in my roots</a>, I believe we should own our tools, they should empower <em>us</em>-not be part of <a href="https://kristoff.it/blog/the-open-source-game/">the 'open source' game</a> which is all too prevelant today (even among 'open source' engines.) I <em>need</em> Mach to genuinely be <a href="https://softwareyoucan.love">software you can love</a>.</p>
        <p>My dream is one day to live a simple, modest, life earning a living building Mach for everyone and selling high-quality games. Please consider <a href="https://github.com/sponsors/slimsag">sponsoring my work</a> if you believe in my vision. It means the world to me!</p>
    </div>
</div>

## Thanks for reading

<div style="display: flex; flex-direction: row; align-items: center;">
    <img align="left" style="max-height: 12.5rem;" src="https://user-images.githubusercontent.com/3173176/187348488-0b52e87d-3a48-421c-9402-be78e32b5a20.png"></img>
    <ul>
        <li>Check out <a href="https://machengine.org">machengine.org</a></li>
        <li>Consider <a href="https://github.com/sponsors/slimsag">sponsoring development</a> so we can do more of it!</li>
        <li>Join the <a href="https://discord.gg/XNG3NZgCqp">Mach Discord server</a></li>
    </ul>
</div>
