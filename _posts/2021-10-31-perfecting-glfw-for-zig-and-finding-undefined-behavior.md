---
layout: post
title: "Perfecting GLFW for Zig, and finding lurking undefined behavior that went unnoticed for 6+ years"
categories: mach, zig, gamedev, glfw
author: "Stephen Gutekanst"
twitter:
  image: https://user-images.githubusercontent.com/3173176/139573985-d862f35a-e78e-40c2-bc0c-9c4fb68d6ecd.png
facebook:
  image: https://user-images.githubusercontent.com/3173176/139573985-d862f35a-e78e-40c2-bc0c-9c4fb68d6ecd.png
---

**Today, I am announcing [mach-glfw](https://github.com/hexops/mach-glfw): Ziggified GLFW bindings with 100% API coverage, zero-fuss installation, cross compilation, and more.**

## Building Mach for everyone

If [Mach engine](https://github.com/hexops/mach) only benefits people interested in using that engine, and not the broader Zig (and even gamedev) community I would consider that _a total failure_.

Whether you're interested in using all of Mach, just some of it with your own engine / project, or just the tools/ideas we develop in the future (with Unity, Unreal, etc.), _I truly aim to produce something that benefits you_.

Mach is in super early stages, I've spent the last four months perfecting a Zig interface to GLFW, and making no-fuss installation and cross-compilation a reality. Today, you can benefit from that work too.

## Building GLFW for every platform

Just `zig` and `git`, that's the idea. The GLFW C code is compiled with `zig`, and the `build.zig` file automatically uses `git` to clone (a very minimal set of) system dependencies for you (X11 libraries, etc.)

No installing apt packages. No dealing with missing header errors. It should just work out-of-the-box, and for every platform:

<a href="https://user-images.githubusercontent.com/3173176/137650099-cd370046-eb43-4fe4-a72a-f54ebe3153c1.png"><img alt="Mach engine platform support, including Windows, Linux, Mac and cross-compilation between them with Android/iOS coming soon." class="color" src="https://user-images.githubusercontent.com/3173176/137650099-cd370046-eb43-4fe4-a72a-f54ebe3153c1.png"></a>

Today, this works for GLFW itself. Cross-compilation of _OpenGL and Vulkan apps_ is not yet fully functional. [We're working on it, though.](https://github.com/hexops/mach/issues/59)

## Perfecting GLFW for Zig

Aside from platform support, mach-glfw now has:

* **100% API coverage** of GLFW v3.3.4. Every function, type, constant, etc. has been wrapped in a ziggified API.
* **130+ tests**, with CI testing Linux, Windows, Mac (x86 and M1/ARM) and cross-compilation between them.

You might be asking: _why Zig bindings, when Zig can interface directly with C?_ Ziggified bindings to GLFW get us:

* Errors as [zig errors](https://ziglang.org/documentation/master/#Errors) instead of via a callback function.
* **Enums**: always know what value a GLFW function can accept as everything is strictly typed. And use the nice Zig syntax to access enums, like `window.getKey(.escape)` instead of `c.glfwGetKey(window, c.GLFW_KEY_ESCAPE)`
* Slices instead of C pointers and lengths.
* [packed structs](https://ziglang.org/documentation/master/#packed-struct) represent bit masks, so you can use `if (joystick.down and joystick.right)` instead of `&` `|` etc. bitwise operators.
* `true` and `false` instead of `c.GLFW_TRUE` and `c.GLFW_FALSE`.
* Generics: use `window.hint` instead of `glfwWindowHint`, `glfwWindowHintString`, etc.
* Methods, e.g. `my_window.hint(...)` instead of `glfwWindowHint(my_window, ...)`

## Explicit error handling solves a real problem

GLFW traditionally passes errors to the user via a callback. This can make errors easy to ignore, as well as difficult to correlate and handle effectively at the time of the function invocation.

We translated a [a Vulkan example to mach-glfw](https://github.com/hexops/mach-glfw-vulkan-example), which you can try for yourself today:

<a href="https://user-images.githubusercontent.com/3173176/139573985-d862f35a-e78e-40c2-bc0c-9c4fb68d6ecd.png"><img alt="mach-glfw and vulkan-zig libraries working together to produce a triangle." class="color" src="https://user-images.githubusercontent.com/3173176/139573985-d862f35a-e78e-40c2-bc0c-9c4fb68d6ecd.png"></a>

After porting it, we found that the example was crashing with a `NoWindowContext` error. Strange?

As it turns out, we had found [a small bug in the vulkan-zig example code](https://github.com/Snektron/vulkan-zig/pull/21), it was calling `glfwSwapBuffers` which is not needed for Vulkan. The error went unnoticed because it's easy to miss errors with GLFW's error callback handling style. But with mach-glfw, it was an explicit error you have to handle e.g. via `try glfw.swapBuffers()` - we literally couldn't miss it.

## Finding lurking undefined behavior in 6+ year old GLFW code

One _particularly frustraing_ issue was tracking down why the last part of the GLFW API we needed to wrap for 100% coverage, the `glfwSetWindowIcon` function, was crashing:

```
Test [76/135] Window.test "setIcon"... Illegal instruction at address 0x2cee09
upstream/glfw/src/x11_window.c:0:0: 0x2cee09 in _glfwPlatformSetWindowIcon (/mach/glfw/upstream/glfw/src/x11_window.c)
upstream/glfw/src/window.c:511:5: 0x2de484 in glfwSetWindowIcon (/mach/glfw/upstream/glfw/src/window.c)
    _glfwPlatformSetWindowIcon(window, count, images);
    ^
/mach/glfw/src/Window.zig:508:28: 0x23a083 in Window.test "setIcon" (test)
        c.glfwSetWindowIcon(self.handle, @intCast(c_int, im.len), &tmp[0]);
                           ^
/usr/local/bin/lib/std/special/test_runner.zig:77:28: 0x25a0d1 in std.special.main (test)
        } else test_fn.func();
                           ^
/usr/local/bin/lib/std/start.zig:517:22: 0x2896bc in std.start.callMain (test)
            root.main();
                     ^
/usr/local/bin/lib/std/start.zig:469:12: 0x25c117 in std.start.callMainWithArgs (test)
    return @call(.{ .modifier = .always_inline }, callMain, .{});
           ^
/usr/local/bin/lib/std/start.zig:434:12: 0x25bec2 in std.start.main (test)
    return @call(.{ .modifier = .always_inline }, callMainWithArgs, .{ @intCast(usize, c_argc), c_argv, envp });
           ^
???:?:?: 0x7f4b7c3280b2 in ??? (???)
```

That's odd? `Illegal instruction at address 0x2cee09` - are we corrupting the stack somehow? Is this a Zig compiler bug?

Running in `lldb` didn't help with shining any light on the problem, either:

<a href="https://user-images.githubusercontent.com/3173176/139576146-775371fd-8003-46ba-aa30-8b81a2f22ce0.png"><img alt="lldb showing nothing particularly useful" class="color" src="https://user-images.githubusercontent.com/3173176/139576146-775371fd-8003-46ba-aa30-8b81a2f22ce0.png"></a>

After poking around at the stack, checking all pointers and lengths were valid, etc. I was at a loss. The mach-glfw code _sure seemed valid_, and yet, this crash. I managed to track the crash down to the first iteration of a loop in GLFW's `x11_window.c`:

```c
void _glfwSetWindowIconX11(_GLFWwindow* window, int count, const GLFWimage* images)
{
    if (count)
    {
        int longCount = 0;

        for (int i = 0;  i < count;  i++)
            longCount += 2 + images[i].width * images[i].height;

        long* icon = _glfw_calloc(longCount, sizeof(long));
        long* target = icon;

        for (int i = 0;  i < count;  i++)
        {
            *target++ = images[i].width;
            *target++ = images[i].height;

            for (int j = 0;  j < images[i].width * images[i].height;  j++)
            {
                // illegal instruction on first iteration?
                *target++ = (images[i].pixels[j * 4 + 0] << 16) |
                            (images[i].pixels[j * 4 + 1] <<  8) |
                            (images[i].pixels[j * 4 + 2] <<  0) |
                            (images[i].pixels[j * 4 + 3] << 24);
            }
        }
...
```

## Reaching my limits

At this point, I feel confident in saying:

* The Zig code is correct, the pointers are valid, the lengths are correct, everything's right.
* The GLFW code is pretty popular, and it's been around for 6 years. Seems unlikely it's a bug in GLFW?

Luckily, my brother (and reverse engineer) [@Andoryuuta](https://github.com/Andoryuuta) was available to help debug, so I pulled him in. Stepping through instructions, we could see clearly that after a bit shift we were stepping into the abyss:

```
* thread #1, name = 'test', stop reason = instruction step over
    frame #0: 0x00000000002c6f84 test`_glfwPlatformSetWindowIcon(window=0x00000000004e53d0, count=1, images=0x00007fffec0b3000) at x11_window.c:2156:58
   2153	                *target++ = (images[i].pixels[j * 4 + 0] << 16) |
   2154	                            (images[i].pixels[j * 4 + 1] <<  8) |
   2155	                            (images[i].pixels[j * 4 + 2] <<  0) |
-> 2156	                            (images[i].pixels[j * 4 + 3] << 24);
   2157	                printf("DID WE GET HERE???x\n");
   2158	            }
   2159	        }
(lldb) 
Process 6516 stopped
* thread #1, name = 'test', stop reason = instruction step over
    frame #0: 0x00000000002c6c21 test`_glfwPlatformSetWindowIcon(window=0x00000000004e53d0, count=1, images=0x00007fffec0b3000) at x11_window.c:0
   1   	//========================================================================
   2   	// GLFW 3.3 X11 - www.glfw.org
   3   	//------------------------------------------------------------------------
   4   	// Copyright (c) 2002-2006 Marcus Geelnard
   5   	// Copyright (c) 2006-2019 Camilla Löwy <elmindreda@glfw.org>
   6   	//
   7   	// This software is provided 'as-is', without any express or implied
(lldb) 
Process 6516 stopped
```

Inspecting the binary in IDA Pro we were able to see that we were jumping into an `__asm { ud1 }` section (ud1 standing for "undefined instruction 1"):

<a href="https://user-images.githubusercontent.com/3173176/139594073-b2159e4c-6764-44b1-882d-802724f424e8.png"><img alt="IDA Pro showing a jump to an undefined instruction 1" class="color" src="https://user-images.githubusercontent.com/3173176/139594073-b2159e4c-6764-44b1-882d-802724f424e8.png"></a>

It turns out that clang's UBSan inserts these instructions as traps for when the compiler thinks there is undefined behavior occurring, such as if a pointer addition leads to an overflow. This is super interesting, but unfortunately doesn't always give a compiler error. We got lucky and found someone else who ran into this through Google:

> I *believe* LLVM explicitly generates a ud2 x86 instruction because &quot;it determined&quot; there&#39;s undefined behavior in the C code. So first I wonder which flags you&#39;re passing it through zig (i.e. how strict are you being with the settings?)</p>&mdash; Abner (@AbnerCoimbre) [December 17, 2020](https://twitter.com/AbnerCoimbre/status/1339396987100168192)

And indeed, compiling via `zig build test -Drelease-fast` (which turns off UBsan) made the crash go away. So where's the undefined behavior?

If we squint at the code and assume all pointers, counts, and indices are correct, you might be able to spot it:

```c
void _glfwSetWindowIconX11(_GLFWwindow* window, int count, const GLFWimage* images)
{
    ...
        long* target = icon;

        for (int i = 0;  i < count;  i++)
        {
            ...
            for (int j = 0;  j < images[i].width * images[i].height;  j++)
            {
                // illegal instruction on first iteration?
                *target++ = (images[i].pixels[j * 4 + 0] << 16) |
                            (images[i].pixels[j * 4 + 1] <<  8) |
                            (images[i].pixels[j * 4 + 2] <<  0) |
                            (images[i].pixels[j * 4 + 3] << 24);
            }
        }
...
```

What is happening here is that:

* `images[i].pixels[j * 4 + 0]` is returning an `unsigned char` (8 bits)
* It is then being shifted left by `<< 16` bits. !!! That's further than an 8-bit number can be shifted left by! So that's UB!

Suddenly, it all makes sense. And [if we load an equal snippet of code into Godbolt](https://godbolt.org/z/ddq75WsYK) we can see what is happening when we compile without UBSan / the `-fsanitize=undefined` flag:

<a href="https://user-images.githubusercontent.com/3173176/139594650-eff35347-3f32-42e5-bc60-da2a1dceb1e1.png"><img alt="Compilation with godbolt with UBSan turned off shows movement into 32-bit EAX register" class="color" src="https://user-images.githubusercontent.com/3173176/139594650-eff35347-3f32-42e5-bc60-da2a1dceb1e1.png"></a>

Without UBsan, clang merely uses the 32-bit EAX register as an optimization. It loads the 8-bit number into the 32-bit register, and then performs the left shift. Although the shift exceeds 8 bits, it _does not get truncated to zero_ - instead it is effectively as if the number was converted to a `long` (32 bits) prior to the left-shift operation.

This explains why nobody has caught this UB in GLFW yet, too: it works by accident! Just because the compiler likes to use 32-bit registers in this context.

And this change benefits all the languages out there using GLFW: [glfw/glfw#1986](https://github.com/glfw/glfw/pull/1986)

## Defaults are _critical_

This code, and undefined behavior, has been in GLFW for over 6 years according to `git blame`.

Anybody using GLFW _could have_ enabled UBSan in their C compiler. Anybody _could have_ run into this same crash and debugged it in the last 6 years. But they didn't.

In mach-glfw, we compile all of GLFW's C code with Zig (which is also a fully functional C and C++ compiler), with UBSan enabled by default.

Only because Zig has good defaults, because it places so much emphasis on things being right _out of the box_, and because there is such an emphasis on having safety checks for undefined behavior - were we able to catch this undefined behavior that went unnoticed in GLFW for the last 6 years.

## Thanks for reading

All key Mach engine developments will be posted here, with incremental updates on Twitter [@machengine](https://twitter.com/machengine).

Follow [Mach engine on GitHub](https://github.com/hexops/mach), and if you like what I'm doing please consider [sponsoring my work](https://github.com/sponsors/slimsag).
