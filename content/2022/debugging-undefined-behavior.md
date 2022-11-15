---
author: "Stephen Gutekanst"
title: "Debugging undefined behavior caught by Zig"
date: "2022-11-14"
draft: false
categories:
- zig
- zigtips
description: "Unlike other toolchains, Zig enables many more safety checks by default. We've caught undefined behavior in GLFW, the DirectX Shader Comppiler, and Google Chrome's WebGPU implementation as a result. But debugging these situations can be a bit tricky sometimes, so here's a walkthrough of how to debug such an error using Zig and LLDB."
images: ["https://user-images.githubusercontent.com/3173176/201806539-3dfa8d41-60d9-4b4f-ba14-9aa0012a0007.png"]
---

[Mach engine](https://machengine.org/) uses [Zig](https://ziglang.org) as the C/C++ compiler for almost everything. Unlike other toolchains, Zig enables many more safety checks by default - such as clang's undefined behavior sanitizer.

Using Zig, we've caught undefined behavior in established projects like GLFW[[1](/2021/perfecting-glfw-for-zig-and-finding-undefined-behavior/)], the DirectX Shader Compiler[[2](https://github.com/microsoft/DirectXShaderCompiler/pull/4178#discussion_r780733405)], and more. Undefined behavior is everywhere, often relatively innocuous, and hard to catch.

Professional C/C++ developers know to run UBSan fuzzers as part of their test suite, but even with that we've found e.g. Google Chrome's fuzzers weren't running normally at one point, and we caught undefined behavior in Chrome's implementation of WebGPU as a result[[3](https://dawn-review.googlesource.com/c/dawn/+/87380)].

Zig having UBSan enabled by default is valuable, but it can also lead to tricky to debug errors that are a bit annoying to interpret today. And so this article is a walkthrough of how to debug such an error when it arises using Zig and LLDB.

## The situation

We're looking at using [model3d](https://bztsrc.gitlab.io/model3d/) in Mach. Model3D is an up-and-coming compact, featureful, universal model format that tries to address the shortcomings of existing formats (yes, including glTF - see [their rationale](https://gitlab.com/bztsrc/model3d/#rationale).) It is a small, zero-dependency single-header C implementation which in Zig we can simply import.

As we've been testing it with various models, though, we found our Zig program just crashes when we use it:

```
% zig build test 2>&1|cat
1/1 test_0... The following command terminated unexpectedly:
cd /mach/libs/model3d && /mach/libs/model3d/zig-cache/o/26c4104a1643fed2068dfa9244dfe90e/model3d-tests /Users/slimsag/zig-macos-aarch64-0.11.0-dev.38+b40fc7018/zig 
error: the following build command failed with exit code 1:
/mach/libs/model3d/zig-cache/o/679e494577315c1bcc3749ee7068ea2f/build /Users/slimsag/zig-macos-aarch64-0.11.0-dev.38+b40fc7018/zig /mach/libs/model3d /mach/libs/model3d/zig-cache /Users/slimsag/.cache/zig test
```

As you can see, we're not getting much info here on why our tests crashed. This is a telltale sign of undefined behavior in Zig (and [there's an open issue to make this error messaging way more clear](https://github.com/ziglang/zig/issues/5163)). When we compile our program with `-Drelease-fast`, which disables safety checks, we find it runs as expected - which confirms our suspicion about it being a safety check.

## Debugging with LLDB

In the output above, we can grab the path to the executable `model3d-tests`. We can debug it using lldb (I think we should find a nicer way to invoke `lldb` via `zig build test` though!):

```
lldb -- /mach/libs/model3d/zig-cache/o/26c4104a1643fed2068dfa9244dfe90e/model3d-tests
```

Next we just enter the `run` command at the `(lldb)` prompt:

```
(lldb) run
Process 6830 launched: '/mach/libs/model3d/zig-cache/o/26c4104a1643fed2068dfa9244dfe90e/model3d-tests' (arm64)
Process 6830 stopped
* thread #1, queue = 'com.apple.main-thread', stop reason = EXC_BREAKPOINT (code=1, subcode=0x100033634)
    frame #0: 0x0000000100033634 model3d-tests`m3d_load(data="\xc3.:>", readfilecb=0x0000000000000000, freecb=0x0000000000000000, mtllib=0x0000000000000000) at m3d.h:3356:56
   3353	                        model->tmap[i].v = (M3D_FLOAT)(*((uint16_t*)(data+2))) / (M3D_FLOAT)65535.0;
   3354	                    break;
   3355	                    case 4:
-> 3356	                        model->tmap[i].u = (M3D_FLOAT)(*((float*)(data+0)));
   3357	                        model->tmap[i].v = (M3D_FLOAT)(*((float*)(data+4)));
   3358	                    break;
   3359	                    case 8:
```

If you squint, you can see `stop reason = EXC_BREAKPOINT` in the output. This is the sign I was looking for: it tells me that UBSan likely inserted a breakpoint for undefined behavior, and we hit it. There's _some_ form of undefined behavior going on here!

Thanks to lldb, we also got an exact line number, and see the source code where the crash occurred. Sometimes you may not get this much, in which case you may need to run the lldb `up` command until you get to a line you can make sense of.

In this instance, we're running code inside of a for loop - and so one question I have is: what iteration of that loop are we at? We can get this info easily using `p i` to inspect the `i` variable:

```
(lldb) p i
(unsigned int) $1 = 0
```

This shows us that the `i` variable is an unsigned 32-bit int with the value `0`. It crashed on the first iteration.

(Note: we could also use `frame variables` to get a list of variables in the local frame (i.e. our function), but in this case it's quite a few and so I didn't find that useful.)

Next, I wanted to find out: what would the float at that address actually look like? Luckily, the LLDB `p <expr>` command actually interprets C-like expressions for us. It's rather easy to write the C expression for that:

```
(lldb) p *(float*)(data+0)
(float) $2 = 0.181819007
```

Here we can see the float at the address of `data` is `0.181819007` - which is within the normalized range I'd expect for a UV coordinate. It seems correct. My next question was.. is the pointer address aligned? I know UBSan has a check for pointer alignment, so I looked at it:

```
(lldb) p data
(unsigned char *) $3 = 0x0000000101008251 "\xc3.:>"
```

Since we're accessing a float at this address, we'd expect it to be aligned to 4 bytes. We can check this easily by plopping the address into Python and dividing by 4. If there's a remainder, it's not aligned:

```
>>> 0x0000000101008251 % 4
1
```

## What are the consequences?

Many times, UBSan will catch undefined behavior that in practice isn't really harmful on modern machines you might care about. I wasn't sure about the consequences of unaligned pointer accesses like this, so I asked someone smarter than myself and [filed an issue on model3d](https://gitlab.com/bztsrc/model3d/-/issues/19) which led to some very interesting insights [from @bztsrc](https://gitlab.com/bztsrc/model3d/-/issues/19#note_1171783061).

> Short answer: the bug is in UBSan
>
> Long answer: it is true that in ancient times there were CPUs that couldn't handle unaligned access. However that's not the case with today mainstream processors: x86, ARM, RISC-V, etc. all handle unaligned access out-of-the-box. (Ok, for ARM it's not out-of-the-box, you have to enable MMU which is surely done by the OS kernel otherwise virtual memory mapping would be impossible.)
>
> [...] The reason for the unaligned access is pretty simple. In the binary bit-chunk that a compressed model file is, there's obviously no guarantee that a value is aligned. Such compactness is absolutely needed for small file sizes, padding with zeros would insanely increase the required storage requirements.
>
> [...] So the decision I had to make here was: keep UBSan happy but create crappy and slow code, or don't care about UBSan and take advantage of modern CPU features. I've decided on the latter.

Which is a quite compelling argument for this just being noise for our purposes. :)

### RISC-V: a notable exception

It's worth noting (as was pointed out to me by someone much more knowledgable) that RISC-V cores lack hardware support for unaligned accesses[0][1] ('if sifive doesn't do this in hardware (unalignment) there's no way any other risc-v cores do [...due to sifive's sheer popularity in the space]'), unalignment is done by trap handlers instead:

![image](https://user-images.githubusercontent.com/3173176/201804903-5584f318-5832-4c76-9f1a-45a32ce10348.png)

> Officially, programs running in any mode but M-Mode on risc-v are allowed to do unaligned accesses but that doesn't mean it can't be pretty painful

[0] https://forums.sifive.com/t/ld-sd-alignment/5530/6

[1] https://patchwork.kernel.org/project/linux-riscv/patch/60c1f087-1e8b-8f22-7d25-86f5f3dcee3f@gmail.com/#24313195

I don't have any RISC-V hardware to test on, so this doesn't affect us at present, but I figured it worth noting.

## Disabling the sanitizer

In our case, we can just disable the alignment sanitizer for this one function:

```
+__attribute__((no_sanitize("alignment")))
m3d_t *m3d_load(unsigned char *data, m3dread_t readfilecb, m3dfree_t freecb, m3d_t *mtllib)
```

And with this, our tests pass. And we continue to get all the other benefits and safety checks of UBSan elsewhere.

In this case, though, I opted to just disable alignment sanitization entirely when building model3d by [adding](https://github.com/hexops/mach/commit/c96ff64958c241249041856a8ea0e8a4349050a6) the `-fno-sanitize=alignment` compiler flag to our `build.zig` to avoid this surprising us in other model3d functions.

## Thanks for reading

If you're writing Zig, C, or C++ code - then I hope this `zig: Tip` helps you! You can find [other `zig: Tips` here](/categories/zigtips/).

<img align="left" style="max-height: 150px;" src="https://user-images.githubusercontent.com/3173176/187348488-0b52e87d-3a48-421c-9402-be78e32b5a20.png"></img>
Be sure to join the [Mach engine Discord](https://discord.gg/XNG3NZgCqp) where we're building the future of Zig game development.
<br><br>
You can also [sponsor my work](https://github.com/sponsors/slimsag) if you like what I'm doing! :)
