---
author: "Emi Stein"
title: "Zig hashmaps explained"
date: "2022-01-29"
draft: false
categories:
- zig
- zigtips
description: "If you just got started with Zig, you might quickly want to use a hashmap. Zig provides good defaults, with a lot of customization options."
images: ["/img/hexops-opengraph.png"]
---

If you just got started with [Zig](https://ziglang.org), you might quickly want to use a hashmap. Zig provides good defaults, with a lot of customization options.

Here I will try to guide you into choosing the right hashmap type.

## 60-second explainer

You probably want:

```zig
var my_hash_map = std.StringHashMap(V).init(allocator);
```

Or if you do not have string keys, you can use an `Auto` hashmap instead:

```zig
var my_hash_map = std.AutoHashMap(K, V).init(allocator);
```

Where `K` and `V` are your key and value data types, respectively. e.g. `[]const u8` for a string.

You can then use these APIs:

### Insert a value

```zig
try my_hash_map.put(key, value);
```

### Insert a value, assert entry does not already exist

```zig
try my_hash_map.putNoClobber(key, value);
```

Note `putNoClobber` may be renamed to something like `putAssumeNoEntry` in the near future: [ziglang/zig#10736](https://github.com/ziglang/zig/issues/10736)

### Get a value

```zig
var value = my_hash_map.get(key);
if (value) |v| {
    // got value "v"
} else {
    // doesn't exist
}
```

### Get a value, insert if not exist

```zig
var v = try my_hash_map.getOrPut(key);
if (!v.found_existing) {
    // We inserted an entry, specify the new value
    // This is a conditional in case creating the new value is expensive
    v.value_ptr.* = "my value";
}

var value = v.value_ptr.*; // use the value
```

You can find more APIs [by going here](https://github.com/ziglang/zig/blob/master/lib/std/hash_map.zig#L342) and using your browser's builtin search for `pub fn`.

## About key data types

Zig hash map types start with the data type of the key:

* `std.StringHashMap` - uses a good default hashing function for string keys
* `std.AutoHashMap` - uses a good default hashing function for most data types
* `std.HashMap` - the "bring your own hashing function" option

Note: `AutoHashMap` does not support _slices_, such as `[]const u8` string slices, because that is a pointer to an array and it is ambiguous whether or not you intend to hash _the array elements_ or _the pointer itself_. You can use the generic `std.HashMap` for any slice type, you just have to provide your own hash functions.

## Hashmaps are also sets

A set in Zig is just a hashmap with a `void` value:

```zig
var my_hash_map = std.AutoHashMap(K, void).init(allocator);

try my_hash_map.put(key, {}); // `{}` is a value of type `void`
```

## Advanced usages

If you're just getting started with Zig, don't worry too much about the below. Just know that you have options available should you need to reduce memory usage or optimize your use of hashmaps in the future.

### Managed vs. unmanaged hashmaps

You can add `Unmanaged` to the end of a Zig hashmap data type, e.g. `std.StringHashMapUnmanaged` in order to get the _unmanaged_ version.

This merely doesn't carry an `allocator` internally, instead you must pass the allocator into every method of the hashmap. While only a few bytes, this can be a useful optimization if you're storing many hashmaps for example.

Managed:

```zig
var my_hash_map = std.StringHashMap(V).init(allocator);
```

Unmanaged:

```zig
var my_hash_map = std.StringHashMapUnmanaged(V){};
```

### Array hash maps

Zig actually provides [_two hashmap implementations_](https://github.com/ziglang/zig/pull/5999) in the standard library

`std.HashMap`, perfect for every-day use cases:

* Optimized for lookup times primarily
* Optimized for insertion/removal times secondarily

`std.ArrayHashMap`, useful in _some_ situations:

* Iterating over the hashmap is an order of magnitude faster (a contiguous array)
* Insertion order is preserved.
* You can index into the underlying data like an array if you like
* Deletions can be performed one of two ways, mirroring the `ArrayList` API:
  * `swapRemove`: swaps the target element with the last element in the list to remove it
  * `orderedRemove`: removes target element by shifting all elements forward, maintaining current ordering

### Hashmap context

If you choose to use `std.HashMap` or `std.ArrayHashMap` directly (without the `String` or `Auto` prefix), then you'll find it wants a _context_ parameter and _max load percentage_:

```zig
var my_hash_map = std.HashMap(K, V, std.hash_map.AutoContext(K), std.hash_map.default_max_load_percentage);
```

The _context_ parameter lets you embed some of your own data within the hash map type. This can be useful for [reducing the amount of memory that a hash map takes up when doing a string table](https://zig.news/andrewrk/how-to-use-hash-map-contexts-to-save-memory-when-doing-a-string-table-3l33).

### Pick your hashmap

Regular implementation:

| Key type | Managed?    | How to initialize                            |
|----------|-------------|----------------------------------------------|
| `String` | yes         | `std.StringHashMap(V).init(allocator)`       |
| `Auto`   | yes         | `std.AutoHashMap(K, V).init(allocator)`      |
| `String` | `Unmanaged` | `std.StringHashMapUnmanaged(V){}`            |
| `Auto`   | `Unmanaged` | `std.AutoHashMapUnmanaged(K, V){}`           |

`ArrayHashMap` implementation:

| Key type | Managed?    | How to initialize                            |
|----------|-------------|----------------------------------------------|
| `String` | yes         | `std.StringArrayHashMap(V).init(allocator)`  |
| `Auto`   | yes         | `std.AutoArrayHashMap(K, V).init(allocator)` |
| `String` | `Unmanaged` | `std.StringArrayHashMapUnmanaged(V){}`       |
| `Auto`   | `Unmanaged` | `std.AutoArrayHashMapUnmanaged(K, V){}`      |

### Learn more

The source code is very readable:

* [`std.HashMap`](https://github.com/ziglang/zig/blob/master/lib/std/hash_map.zig)
* [`std.ArrayHashMap`](https://github.com/ziglang/zig/blob/master/lib/std/hash_map.zig)

### Help improve this page

I wrote this article quickly because I needed to explain my choice of hashmaps in the ["Let's build an Entity Component System from scratch"](https://devlog.hexops.com/categories/build-an-ecs/) series and there was no better source of this info. I'm sure there are things that can be improved.

[Feel free to send a PR!](https://github.com/hexops/devlog/blob/main/_posts/2022-01-29-zig-hashmaps-explained.md)
