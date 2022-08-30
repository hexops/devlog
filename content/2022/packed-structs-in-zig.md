---
author: "Stephen Gutekanst"
title: "Packed structs in Zig make bit/flag sets trivial"
date: "2022-08-29"
draft: false
categories:
- zig
- zigtips
description: "As we've been building Mach engine, we've been using a neat little pattern in Zig that enables writing flag sets more nicely in Zig than in other languages. Here's a brief explainer."
images: ["https://user-images.githubusercontent.com/3173176/184735519-cc78d19d-73e8-4914-8f3d-fc3a15d00bb7.png"]
---

As we've been building [Mach engine](https://machengine.org/), we've been using a neat little pattern in Zig that enables writing flag sets more nicely in Zig than in other languages.

## What is a flag set?

We've been rewriting `mach/gpu` (WebGPU bindings for Zig) from scratch recently, so let's take a flag set from the WebGPU C API:

```c
typedef uint32_t WGPUFlags;
typedef WGPUFlags WGPUColorWriteMaskFlags;
```

Effectively, `WGPUColorWriteMaskFlags` here is a 32-bit unsigned integer where you can set specific bits in it to represent whether or not to write certain colors:

```c
typedef enum WGPUColorWriteMask {
    WGPUColorWriteMask_None = 0x00000000,
    WGPUColorWriteMask_Red = 0x00000001,
    WGPUColorWriteMask_Green = 0x00000002,
    WGPUColorWriteMask_Blue = 0x00000004,
    WGPUColorWriteMask_Alpha = 0x00000008,
    WGPUColorWriteMask_All = 0x0000000F,
    WGPUColorWriteMask_Force32 = 0x7FFFFFFF
} WGPUColorWriteMask;
```

Then to use it you'd use the various bit operations with those masks, e.g.:

```c
WGPUColorWriteMaskFlags mask = WGPUColorWriteMask_Red | WGPUColorWriteMask_Green;
mask |= WGPUColorWriteMask_Blue; // set blue bit
```

This all works, people have been doing it for years in C, C++, Java, Rust, and more. In Zig, we can do better.

## Zig packed structs

<img class="color-auto" style="max-height: 300px;" src="https://user-images.githubusercontent.com/3173176/184735519-cc78d19d-73e8-4914-8f3d-fc3a15d00bb7.png" />

Zig has `packed struct`s: these let us pack memory tightly, where a `bool` is actually a single bit (in most other languages, this is not true.) Zig also has arbitrary bit-width integers, like `u28`, `u1` and so on.

We can write `WGPUColorWriteMaskFlags` from earlier in Zig using:

```zig
pub const ColorWriteMaskFlags = packed struct {
    red: bool = false,
    green: bool = false,
    blue: bool = false,
    alpha: bool = false,

    _padding: u28 = 0,
};
```

This is still just 32 bits of memory, and so can be passed to the same C APIs that expect a `WGPUColorWriteMaskFlags` - but interacting with it is much nicer:

```zig
var mask = ColorWriteMaskFlags{.red = true, .green = true};
mask.blue = true; // set blue bit
```

In C you would need to write code like this:

```c
if (mask & WGPUColorWriteMask_Alpha) {
    // alpha is set..
}
if (mask & (WGPUColorWriteMask_Alpha|WGPUColorWriteMask_Blue)) {
    // alpha and blue are set..
}
if ((mask & WGPUColorWriteMask_Green) == 0) {
    // green not set
}
```

In Zig it's just:

```zig
if (mask.alpha) {
    // alpha is set..
}
if (mask.alpha and mask.blue) {
    // alpha is set..
}
if (!mask.green) {
    // green not set
}
```

## Comptime validation

Making sure that our `ColorWriteMaskFlags` ends up being the same size could be a bit tricky: what if we count the number of `bool` wrong? Or what if we accidently get the padding size wrong? Then it might not be the same size as a `uint32` anymore.

Luckily, we can verify our expectations at comptime:

```zig
pub const ColorWriteMaskFlags = packed struct {
    red: bool = false,
    green: bool = false,
    blue: bool = false,
    alpha: bool = false,

    _padding: u28 = 0,

    comptime {
        std.debug.assert(@sizeOf(@This()) == @sizeOf(u32));
        std.debug.assert(@bitSizeOf(@This()) == @bitSizeOf(u32));
    }
}
```

The Zig compiler will take care of running the `comptime` code block here for us when building, and it will verify that the byte size of `@This()` (the type we're inside of, the `ColorWriteMaskFlags` struct in this case) matches the `@sizeOf(u32)`.

Similarly we could check the `@bitSizeOf` both types if we like.

Note that [`@sizeOf`](https://ziglang.org/documentation/master/#sizeOf) may include the size of padding for more complex types, while [`@bitSizeOf`](https://ziglang.org/documentation/master/#bitSizeOf) returns the number of bits it takes to store `T` in memory _if the type were a field in a packed struct/union_. For flag sets like this, it doesn't matter and either will do. For more complex types, be sure to recall this.

## Explicit backing integers for packed structs

It's worth noting that in Zig 0.11 (shipping in Nov), the new self-hosted compiler has support for [explicit backing integers for packed structs](https://github.com/ziglang/zig/pull/12379) which will simplify this even further.

Instead of manually adding padding to make up 32 bits, one could simply write `packed struct(u32)`:

```zig
pub const ColorWriteMaskFlags = packed struct(u32) {
    red: bool = false,
    green: bool = false,
    blue: bool = false,
    alpha: bool = false,
}
```

## Thanks for reading

<img align="left" style="max-height: 150px;" src="https://user-images.githubusercontent.com/3173176/187348488-0b52e87d-3a48-421c-9402-be78e32b5a20.png"></img>
Be sure to join the new [Mach engine Discord server](https://discord.gg/XNG3NZgCqp) where we're building the future of Zig game development.
<br><br>
You can also [sponsor my work](https://github.com/sponsors/slimsag) if you like what I'm doing! :)
