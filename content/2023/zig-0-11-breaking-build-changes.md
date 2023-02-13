---
author: "Stephen Gutekanst"
title: "Zig tips: v0.11 std.build API / package manager changes"
date: "2023-02-13"
draft: false
categories:
- zig
- zigtips
description: "With Zig v0.11 will come several changes to the std.build API. We've just updated Mach to the latest nightly Zig version, and wanted to provide some tips on how to update your own Zig code."
images: ["https://user-images.githubusercontent.com/3173176/218508235-43e18733-d18d-428a-8475-737f804f590c.png"]
---

We've just updated [Mach engine](https://machengine.org/) to use the latest Zig nightly version, which includes a fair amount of improvements and breaking changes to the `std.build` API used in `build.zig` files, and figured now would be a good time to share the general changes you may need to make if you want to update your own code.

## Package manager: incoming!

Zig is finally starting to see its package manager and build system shape up, some notable mentions:

* `std.http.Client` and `std.crypto.tls` were added ([#13980](https://github.com/ziglang/zig/pull/13980))
* The package manager MVP landed almost a month ago and has seen steady improvements since ([#14265](https://github.com/ziglang/zig/pull/14265))
* Zig packages can now expose C headers are part of their public API ([#14449](https://github.com/ziglang/zig/pull/14449))
* Transitive dependencies are now handled better ([#14392](https://github.com/ziglang/zig/pull/14392))
* "zig build: The breakings will continue until morale improves." ([#14498](https://github.com/ziglang/zig/pull/14498))
* Zig Object Notation (ZON, an alternative to JSON) was introduced ([#14523](https://github.com/ziglang/zig/pull/14523))
* The caching system is being moved from the compiler to the std lib to start using it in the bulid system ([#14571](https://github.com/ziglang/zig/pull/14571))
* Zig plans to run the build system in a sandboxed WASM environment ([#14286](https://github.com/ziglang/zig/issues/14286))

You can get an overview of progress on the package manager on this [GitHub project board](https://github.com/ziglang/zig/projects/4)

Mach isn't yet using the new package manager: it's improving rapidly, and we plan to make use of it soon, but things are still changing so we've held off for now. What we have done, though, is updated to the latest API and want to share those changes with you.

## Release options have been renamed to optimization

Previously you would've used `b.standardReleaseOptions()` which would provide your `zig build` command with multiple options like `zig build -Drelease-fast=true`, `zig build -Drelease-safe=true`, etc.

It's been renamed to `b.standardOptimizeOption(.{})` and now exposes a single build option `zig build -Doptimize=ReleaseFast`, `zig build -Doptimize=ReleaseSafe`, etc. instead.

```diff
-pub fn build(b: *std.build.Builder) void {
-    const mode = b.standardReleaseOptions();
-    const target = b.standardTargetOptions(.{});
+pub fn build(b: *std.Build) void {
+    const optimize = b.standardOptimizeOption(.{});
+    const target = b.standardTargetOptions(.{});
```

```diff
-mode: std.builtin.Mode
+optimize: std.builtin.OptimizeMode
```

```diff
-step.build_mode
+step.optimize
```

## Creating tests, libraries, and executables

Creating tests, libraries, and executables now takes a struct with options as the parameter instead of using a setter API:

```diff
-const exe = b.addExecutable("example", "src/main.zig");
-exe.setBuildMode(mode);
-exe.setTarget(target);
+const exe = b.addExecutable(.{
+    .name = "example",
+    .root_source_file = "src/main.zig",
+    .target = target,
+    .optimize = optimize,
+});
```

<details>
<summary>See more examples</summary>

Tests:

```diff
-const main_tests = b.addTestExe("glfw-tests", sdkPath("/src/main.zig"));
-main_tests.setBuildMode(mode);
-main_tests.setTarget(target);
+const main_tests = b.addTest(.{
+    .name = "glfw-tests",
+    .kind = .test_exe,
+    .root_source_file = .{ .path = sdkPath("/src/main.zig") },
+    .target = target,
+    .optimize = optimize,
+});
```

Shared libraries:

```diff
-const lib = b.addSharedLibrary("glfw", null, .unversioned)
-lib.setTarget(target);
-lib.setBuildMode(mode);
+b.addSharedLibrary(.{ .name = "glfw", .target = target, .optimize = optimize })
```

```diff
-const lib = b.addSharedLibrary("machcore", "src/platform/libmachcore.zig", .unversioned);
-lib.setTarget(target);
-lib.setBuildMode(mode);
+const lib = b.addSharedLibrary(.{
+    .name = "machcore",
+    .root_source_file = "src/platform/libmachcore.zig",
+    .target = target,
+    .optimize = optimize
+});
```

Static libraries:

```diff
-const lib = b.addStaticLibrary("basisu-transcoder", null);
-lib.setTarget(target);
-lib.setMode(mode);
+const lib = b.addStaticLibrary(.{
+   .name = "basisu-transcoder",
+   .target = target,
+   .optimize = optimize,
+});
```

</details>

## Renamings

* `std.build.LibExeObjStep` has been renamed to just `std.build.CompileStep` (beautiful!)
* `*std.build.Builder` has been renamed to just `*std.Build` (nice, this is used extensively everywhere!)

## Modules

Units of code you `@import("foo")` (previously known as _packages_) are now known as _modules_, and _packages_ now refers to a piece of code you download/depend on using the Zig package manager. _Libraries_ is reserved for referring to C-style libraries, `.dll`s, etc.

These units of code used to be declared as a `std.build.Pkg` struct:

```zig
pub const my_pkg = std.build.Pkg{
    .name = "earcut",
    .source = .{ .path = "src/main.zig" },
};
```

And added as a dependency using e.g. `exe.addPackage(my_pkg);`

Now, these are called _modules_ and can be created in a few ways. One is using `b.createModule`:

```zig
const my_module = b.createModule(.{
    .source_file = .{ .path = "src/main.zig" },
    .dependencies = &.{
        .{ .name = "core", .module = core.module(b) },
        .{ .name = "ecs", .module = ecs.module(b) },
        .{ .name = "sysaudio", .module = sysaudio.module(b) },
    },
});
```

And then depend on that module using e.g. `exe.addModule("earcut", my_module);`

Notably, modules are created at _runtime_ via the `*std.Build` now - so you may have some reworking to do if you previously depended on `std.build.Pkg` being a global constant you could rely on at comptime.

Another option, which may be preferred, is via [`addModule`](https://github.com/ziglang/zig/blob/fc48467a97021cb872ff2a947f96e882274c39c1/lib/std/Build.zig#L547-L558). It will make the module available to other packages which depend on this package.

You may also like to know that a _pair of dependency name + the module_ can be represented as [`std.Build.ModuleDependency`](https://github.com/ziglang/zig/blob/fc48467a97021cb872ff2a947f96e882274c39c1/lib/std/Build.zig#L560-L563) now.

We've just gone for an initial 1:1 translation in our code, but adoption of the package manager will likely mean structuring your code a bit differently than the above, and the package manager is still a work-in-progress.

## Thanks for reading

As we work towards Mach v0.2, we're getting more serious about what _stability_ means for us. Our intent is to enable us to move quickly, while also helping you to update your code. We will be achieving this through articles like this which help you understand & update your code to the latest APIs. Hopefully this has helped you! You can find other _zig: Tips_ [here](/categories/zigtips/).

<img align="left" style="max-height: 150px;" src="https://user-images.githubusercontent.com/3173176/187348488-0b52e87d-3a48-421c-9402-be78e32b5a20.png"></img>
Be sure to join the [Mach engine Discord](https://discord.gg/XNG3NZgCqp) where we're building the future of Zig game development.
<br><br>
You can also [sponsor my work](https://github.com/sponsors/slimsag) if you like what I'm doing! :)
