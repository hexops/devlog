---
layout: post
title: "Zig, Parser Combinators - and Why They're Awesome"
categories: zig, regex, parsers
author: "Stephen Gutekanst"
---

In this article we will be exploring what [parser combinators](https://en.wikipedia.org/wiki/Parser_combinator) are, what _runtime parser generation_ is - why they're useful, and then walking through a [Zig](https://ziglang.org) implementation of them.

- [What are parser combinators?](#why-are-parser-combinators-useful)
- [Going deeper: _runtime parser generation_](#going-deeper-runtime-parser-generation)
- [A note about traditional regex engines](#a-note-about-traditional-regex-engines)
- [Implementing the Parser interface](#implementing-the-parser-interface)
    - [Compile-time vs. run-time](#compile-time-vs-run-time)
    - [The parser interface](#the-parser-interface)
    - [Zig generics are provided via type parameters](#zig-generics-are-provided-via-type-parameters)
    - [Zig runtime interfaces](#zig-runtime-interfaces)
    - [Type parameters](#type-parameters)
    - [Errors the Parser interface can produce](#errors-the-parser-interface-can-produce)
- [Our first Parser](#our-first-parser)
    - [What actually is a "Reader"?](#what-actually-is-a-reader)
    - [A Parser that parses a literal string](#a-parser-that-parses-a-literal-string)
    - [Passing parameters to a parser implementation](#passing-parameters-to-a-parser-implementation)
    - [Understanding Zig's wild/confusing `@fieldParentPtr`](#understanding-zigs-wildconfusing-fieldparentptr)
    - [Implementing the rest of `parse`](#implementing-the-rest-of-parse)
- [Our first _parser combinator_](#our-first-parser-combinator)
- [Using our OneOf parser combinator](#using-our-oneof-parser-combinator)
- [Runtime parser generation](#runtime-parser-generation)
- [Closing thoughts](#closing-thoughts)

## What are parser combinators?

A parser parses some text to produce a result:

![image](https://user-images.githubusercontent.com/3173176/110372092-1c234080-800b-11eb-8095-654c3c81354d.png)

A [parser combinator](https://en.wikipedia.org/wiki/Parser_combinator) is a [higher-order function](https://en.wikipedia.org/wiki/Higher-order_function) which _takes parsers as input_ and _produces a new parser_ as output:

![image](https://user-images.githubusercontent.com/3173176/110372575-b4b9c080-800b-11eb-9ef8-58f3ee0e1f1d.png)

## Why are parser combinators useful?

Let's say we want to parse the syntax which describes a regular expression: `a[bc].*abc`

We can define some _parsers_ to help us parse this syntax (e.g. into tokens or AST nodes):

![image](https://user-images.githubusercontent.com/3173176/110375065-b1740400-800e-11eb-987e-b5a7c5a3381b.png)

Suppose that for `a[bc].*abc`:

* `RegexLiteralParser` can parse `a`, `b`, and `c`, but not `abc` (the string.)
* `RegexRangeOpenParser` can parse `[`.
* `RegexRangeCloseParser` can parse `]`
* `RegexAnyParser` can parse the `.` "any character" syntax.
* `RegexRepetitionParser` can parse the `*` repetition operator.

Now that we have these _parsers_, we can define _parser combinators_ to help us parse the full regular expression. First, we need something to parse a string `abc` which we can define as:

![image](https://user-images.githubusercontent.com/3173176/110413375-fa49ae00-804a-11eb-8311-64e737513000.png)

What is `OneOrMore`, though? That's our first parser combinator!

It takes a single parser as input (in this case, `RegexLiteralParser`) and uses it to parse the input one or more times. If it succeeded once, the parser combinator succeeded. Otherwise, it failed to parse anything.

Now if we want to parse the `[bc]` part of our regex, let's say it can only contain a literal like `bc` (of course, real regex allows far more than this) we can e.g. reuse our new `RegexStringLiteralParser`:

![image](https://user-images.githubusercontent.com/3173176/110413643-780db980-804b-11eb-8fe5-8ca97b2e96ca.png)

In this case, `Sequence` is a parser combinator which takes multiple parsers and tries to parse them one-after-the-other in order, requiring all to succeed or failing otherwise.

Building upon this basic idea, we can use parser combinators to build a full regex syntax parser:

![image](https://user-images.githubusercontent.com/3173176/110414508-2ebe6980-804d-11eb-9422-0888208fac19.png)

## Going deeper: _runtime parser generation_

From before, our _parser combinator_ `RegexSyntaxParser` is built out of multiple parsers (`Regex...Parser`) and ultimately produces an AST describing the syntax for a given regex.

We can use the same combinatorial principle here to introduce a new _parser generator_ called `RegexParser` which uses `RegexSyntaxParser` to create a _brand new parser at runtime_ that is capable of parsing the actual semantics the regex describes - forming a full regex engine:

![image](https://user-images.githubusercontent.com/3173176/110528627-94eecf00-80d5-11eb-8fb6-f6bb051d9394.png)

## A note about traditional regex engines

Popular regex engines are implemented using DFA (Deterministic Finite Automatons) or NFA (Nondeterministic Finite Automatons), which is described in great detail in [Russ Cox's article here](https://swtch.com/~rsc/regexp/regexp1.html) or using Henry Spencer's virtual machine approach, also described in great detail on [Russ Cox's website](https://swtch.com/~rsc/regexp/regexp2.html).

It's worth noting that combinatorial parsing / generating parsers at runtime is very much an _uncommon_ method of implementing a regular expression engine. This is _somewhat_ close to what [Comby](https://comby.dev) does in practice, although we use a runtime parser generator instead of parser parser combinators.

One could argue this makes what we're parsing not strictly _regular expressions_, although as Larry Wall (author of the Perl programming language) [writes](https://raku.org/archive/doc/design/apo/A05.html), neither are the modern "regexp" pattern matchers you are likely used to:

> "Regular expressions" […] are only marginally related to real regular expressions. Nevertheless, the term has grown with the capabilities of our pattern matching engines, so I'm not going to try to fight linguistic necessity here. I will, however, generally call them "regexes" (or "regexen", when I'm in an Anglo-Saxon mood).

## Implementing the Parser interface

Parser combinators _tend_ to be written in higher-level languages with much fancier type-systems such as Haskell and OCaml, which lend themselves well to higher-order functions like parser combinators.

We'll be implementing this in [Zig](https://ziglang.org), which is a new low-level language aiming to be a better C.

### Compile-time vs. run-time

Zig has very cool [compile-time code execution semantics](https://ziglang.org/documentation/master/#comptime) which help provide its generics. We'll be exploring these a bit, but since we want to ultimately _build parser generators at runtime_ (in order to execute a regexp) what we'll be looking at is mostly _runtime parser interfaces_ rather than _compile-time parser interfaces_ (which are very much possible!)

Since we'll be dealing with heap allocations, our parser will not be able to run at comptime for now. Once [Zig gets comptime heap allocations](https://github.com/ziglang/zig/issues/1291) this should be possible and opens up interesting new opportunities.

### The parser interface

We need an interface in Zig which describes a _parser_ as we previously mentioned:

![image](https://user-images.githubusercontent.com/3173176/110372092-1c234080-800b-11eb-8095-654c3c81354d.png)

Here it is - there's a lot to unpack here so we'll walk through it step-by-step:

```zig
pub fn Parser(comptime Value: type, comptime Reader: type) type {
    return struct {
        const Self = @This();
        _parse: fn(self: *Self, allocator: *Allocator, src: *Reader) callconv(.Inline) Error!?Value,

        pub fn parse(self: *Self, allocator: *Allocator, src: *Reader) callconv(.Inline) Error!?Value {
            return self._parse(self, allocator, src);
        }
    };
}
```

### Zig generics are provided via type parameters 

```Zig
pub fn Parser(comptime Value: type, comptime Reader: type) type {
    return struct {
        ...
    };
}
```

This is a Zig function which takes two arbitrary `type` arguments at `comptime`, named `Value` and `Reader`. Uppercase is used to denote the name of a type in Zig. Thes are:

* `Value` will be the type of the actual value that the parser will produce (e.g. a string of matched text, or an AST note.)
* `Reader` will be the type of the actual source of the raw text to parse (we'll cover this more later.)

The function itself _returns a new type_. 

What we're seeing here is the key way in which [Zig approaches generic data structures](https://ziglang.org/documentation/master/#Generic-Data-Structures): you merely pass around types as parameters - as if they were values - and you write functions which take types as parameters and return types as values. Some examples of valid calls to this function are:

* `Parser(u8, []u8)` where `u8` is an unsigned 8-bit integer and `[]u8` is a slice of unsigned 8-bit integers.
* `Parser([]const u8, @TypeOf(reader))` where `[]const u8` is describing a slice of UTF-8 text (a string) and `reader` is some reader type, such as `std.io.fixedBufferStream("foobar")`.

### Zig runtime interfaces

Now, since we're trying to define an interface whose actual implementation can be swapped out _at runtime_ - what we need is pretty simple:

* A `struct` type which has the methods we want every implementation to provide.
* Those methods to _call function pointers_ which are defined as _fields_ of our struct.

Basically, if someone wants to implement our interface they just need to create a new instance of `Parser` and populate the fields (callbacks) so their implementation is called when the interface is used.

This is the same pattern used by the Zig [`std.mem.Allocator` interface](https://sourcegraph.com/github.com/ziglang/zig/-/blob/lib/std/mem/Allocator.zig).

In our case here, the returned struct has a method that consumers of the interface would invoke called `parse` - and the function pointer field that implementors will set to get a callback is the `_parse` field:

![image](https://user-images.githubusercontent.com/3173176/110578739-ad390b00-8122-11eb-816c-09e1e281db9d.png)

### Type parameters

Let's look at some of the data types going around here:

![image](https://user-images.githubusercontent.com/3173176/110578578-60553480-8122-11eb-897b-e52e2d45eede.png)

A few other notes:

* `Error!?Value` is just describing the function can return an `Error` OR no value OR a `Value` type. See Zig's [error union types](https://ziglang.org/documentation/master/#Error-Union-Type) and [optional types](https://ziglang.org/documentation/master/#Optionals).
* `callconv(.Inline)` is just telling the compiler to inline the function call - since our function isn't doing a ton.

### Errors the Parser interface can produce

Our error type might start out looking something like this:

```zig
pub const Error = error{
    EndOfStream,
} || std.mem.Allocator.Error;
```

`error{...}` describes [a set of potential errors](https://ziglang.org/documentation/master/#Error-Set-Type) and `|| std.mem.Allocator.Error` merely says to _merge_ the allocator type's error set with ours - so our potential set of errors includes _ours and theirs_.

As we start performing different operations within parsers, it will become more complex to describe more potential sources of errors:

```zig
pub const Error = error{
    EndOfStream,
    Utf8InvalidStartByte,
} || std.fs.File.ReadError
  || std.fs.File.SeekError
  || std.mem.Allocator.Error;
```

Zig can often [infer error sets](https://ziglang.org/documentation/master/#Inferred-Error-Sets) but only in some contexts today.

## Our first Parser

All we need to do in order to implement a `Parser` is provide the `_parse` method, and define its return `Value` type and `Reader` input type:

```zig
const parser: Parser([]u8, @TypeOf(reader)) = .{
    ._parse = myParse,
};
```

In the above, the type `T` in `const parser: T` is denoting the type of the constant named `parser` - in this case it'll be the type returned by `Parser([]u8, @TypeOf(reader))`. And this:

```zig
something = .{
    ._parse = myParse,
}
```

Is the Zig syntax for populating a struct. We're setting the `_parse` field to `myParse`. Zig can infer the type of the struct if you write a `.{}` instead of `T{}` - which avoids the need for us to repeat the call to the `Parser()` function which is verbose.

### What actually is a "Reader"?

Up to this point, we've just talked about `Reader` as being _any type_.

Similar to our `Parser` interface, the Zig standard library [provides a `std.io.Reader` interface](https://sourcegraph.com/github.com/ziglang/zig@f2b96782ecdc9e2f8740eb7d294203b2a585ea52/-/blob/lib/std/io/reader.zig#L13-20) and there are [many implementors of it](https://sourcegraph.com/search?q=repo:%5Egithub%5C.com/ziglang/zig%24+file:%5Elib/std/+fn+reader%28&patternType=literal) including:

* `std.fs.File`
* `std.io.fixedBufferStream("foobar")`
* `std.net.Stream` (network sockets)

However, in contrast to our `Parser` type which invokes _function pointers_ at runtime, the `std.io.Reader` interface is a _compile time type_ - meaning calls to the underlying implementation do not involve a pointer dereference.

Today, Zig is in early stages (version 0.7) and does not have anything like an interface or trait type (although [it seems likely this will be improved in the future](https://github.com/ziglang/zig/issues/1268).)

This means that, for now, we cannot simply define our function as accepting _only_ an `std.io.Reader` interface - instead we must declare that we accept _any type_ which we'll call `Reader`, write our code _as if it is an `std.io.Reader`_ - and the compiler will just barf if anybody passes something in that _isn't_ an `std.io.Reader`. This can sometimes lead to confusing compiler error messages ("there's an error in the standard library code? Ah, no, I just needed to pass a `.reader()`!").

### A Parser that parses a literal string

If we want a `Parser` interface implementation that parses a specific string literal, one way to do that is to also make that a generic function which accepts _any_ reader type (so we're not restricted to e.g. just file inputs):

```zig
pub fn Literal(comptime Reader: type) type {
    return struct {
        // TODO
    };
}
```

This is pretty good - but we need some way to have the type we return _implement_ the `Parser` interface we defined. The way to do this is by defining a field in our struct:

```zig
pub fn Literal(comptime Reader: type) type {
    return struct {
        parser: Parser([]u8, Reader) = .{
            ._parse = parse,
        },
    };
}
```

Now a consumer can write the following to get a literal string parser:

```zig
const parser = Literal(@TypeOf(reader)).parser;
```

### Passing parameters to a parser implementation

If we want our `Literal` parser to accept a parameter -- the literal string to look for -- we need to give it a parameter.

In the case of merely passing it a string, we _could_ adjust the signature so that this is possible:

```zig
const parser = Literal("some string", @TypeOf(reader)).parser;
```

However, we'll define ours using an `init` method which is more common in Zig data structures:

```zig
pub fn Literal(comptime Reader: type) type {
    return struct {
        parser: Parser([]u8, Reader) = .{
            ._parse = parse,
        },
        want: []const u8,

        const Self = @This();

        // The `want` string must stay alive for as long as the parser will be used.
        pub fn init(want: []const u8) Self {
            return Self{
                .want = want
            };
        }
    };
}
```

In this case, `want` is the string literal we want to match - and `[]const u8` is Zig's string type. It describes a slice of immutable (non-modifiable) encoded UTF-8 bytes.

Unlike C, `[]const u8` being a slice means it is _a pointer to the string in memory and its length_ - so we don't have to pass around the length parameter separately or use a null-terminated string. In Zig, there are two ways to represent a string:

* `[]const u8` (unmodifiable string, most common)
* `[]u8` (modifiable string)

You can consider these two similar to Rust's `String` trait and immutable `&str` if[[1](https://www.ameyalokare.com/rust/2017/10/12/rust-str-vs-String.html)] you[[2](https://stackoverflow.com/questions/24158114/what-are-the-differences-between-rusts-string-and-str)] like[[3]](https://www.reddit.com/r/rust/comments/2bpenl/confused_by_the_purpose_of_str_and_string/), but Rust's `String` would actually be a _growable vector of bytes_ (well, technically a compile-time interface that is _probably_ a growable vector of bytes) which would be represented as an `std.ArrayList(u8)` in Zig.

As the Rust book says, ["Strings are not so simple [...] Let’s switch to something a bit less complex"](https://doc.rust-lang.org/book/ch08-02-strings.html#strings-are-not-so-simple) /shade/

### Understanding Zig's wild/confusing `@fieldParentPtr`

We're finally ready to actually have our `Literal` parser _parse_ something! We just need to implement our `parse` method:

```zig
pub fn Literal(comptime Reader: type) type {
    return struct {
        parser: Parser([]u8, Reader) = .{
            ._parse = parse,
        },
        want: []const u8,
        ...
        const Self = @This();

        fn parse(parser: *Parser([]u8, Reader), allocator: *Allocator, src: *Reader) callconv(.Inline) Error!?[]u8 {
            const self = @fieldParentPtr(Self, "parser", parser);
            ...
        }
    };
}
```

But wait a minute! In order for the `._parse = parse,` assignment to work the first argument to `parse` needs to be the `self` parameter for a `Parser([]u8, Reader)` - so how does _our_ `parse` implementation method get to access the `want` field of our struct?

This is where some Zig magic comes in: on obscure builtin function we can use inside of our `parse` method:

```
const self = @fieldParentPtr(Self, "parser", parser);
```

To understand this, first let's get a look at what these parameters are referring to:

![image](https://user-images.githubusercontent.com/3173176/110593977-7b7f6e80-8139-11eb-8ecc-41dff5766ec2.png)

We can see from the Zig documentation that this function operates as follows:

> Given a pointer to a field, returns the base pointer of a struct. 

So in our case:

* `Self` is the "parent struct" we're trying to acquire a reference to (our type)
* `"parser"` is the name of our struct's field.
* `parser` is the _pointer to our `parser` struct field_.

Hopefully you can start to see the link here: `parser` is a pointer to _our struct field_, so Zig has a little helper `@fieldParentPtr` which can rely on that fact to give us _our struct_ given a pointer to _our struct field_.

### Implementing the rest of `parse`

Our full `parse` method will look like this:

```
// If a value is returned, it is up to the caller to free it.
fn parse(parser: *Parser([]u8, Reader), allocator: *Allocator, src: *Reader) callconv(.Inline) Error!?[]u8 {
    const self = @fieldParentPtr(Self, "parser", parser);
    const buf = try allocator.alloc(u8, self.want.len);
    const read = try src.reader().readAll(buf);
    if (read < self.want.len or !std.mem.eql(u8, buf, self.want)) {
        try src.seekableStream().seekBy(-@intCast(i64, read));
        allocator.free(buf);
        return null;
    }
    return buf;
}
```

There are a few notable things here:

* We're trying to return a string from our `parse` function, i.e. the value it emits is a string (instead of an AST node).
* The `want` string we _got_ inside of our `init` method is agreed to only be valid while `parse` will still be called. We've decided to create a contract that all of our `Parser` implementations will either not hold onto memory given by others - or if they do, only do so until `parse` returns. Hence, we need to allocate a new string in our method.
* Normally we could use `defer` ("run at end of function") or `errdefer` ("run if an error is returned"), but since we've chosen to reserve the _none optional_ `null` as "we didn't parse anything" we need to manually free. A `nulldefer` and `somedefer` could be nice, maybe?

Putting it all together, you'll get something like this: [GitHub gist](https://gist.github.com/slimsag/8f098a13177b4bc008a7741505819f90).

## Our first _parser combinator_

To demonstrate how a _parser combinator_ would be implemented, we'll try implementing the `OneOf` operator. It will take any number of _parsers_ as input and run them consecutively until one succeeds or none do.

Let's first start by writing out the basic structure of our function:

```zig
pub fn OneOf(comptime Value: type, comptime Reader: type) type {
    return struct {
        parser: Parser(Value, Reader) = .{
            ._parse = parse,
        },

        ...
    };
}
```

You'll notice here that in contrast to our `Literal` _parser_ function from earlier, this function takes a second `comptime Value: type` parameter. This is because we want it to work with any existing `Parser` implementation, regardless of what type of value it produces.

We can start to fill in the type by adding our `init` method:

```zig
pub fn OneOf(comptime Value: type, comptime Reader: type) type {
    return struct {
        parser: Parser(Value, Reader) = .{
            ._parse = parse,
        },
        parsers: []*Parser(Value, Reader),

        const Self = @This();

        // `parsers` slice must stay alive for as long as the parser will be
        // used.
        pub fn init(parsers: []*Parser(Value, Reader)) Self {
            return Self{
                .parsers = parsers,
            };
        }
    };
}
```

As you can see here, we're going to simply take in a list of pointers to parsers. They'll all need to have the same return `Value` as was specified in the call to `OneOf`.

One reason for this is that [Zig does not support _return type inference_](https://github.com/ziglang/zig/issues/447). You can have a function which takes `anytype` as a parameter, but it cannot return an `anytype`. This just means we need to have a generic function (in this case, `OneOf`) which accepts a type parameter and then use that `Value` type later. In a language like Haskell or OCaml, this would not be true.

Finally, we can implement our `parse` method:

```zig
pub fn OneOf(comptime Value: type, comptime Reader: type) type {
    return struct {
        ...

        // Caller is responsible for freeing the value, if any.
        fn parse(parser: *Parser(Value, Reader), allocator: *Allocator, src: *Reader) callconv(.Inline) Error!?Value {
            const self = @fieldParentPtr(Self, "parser", parser);
            for (self.parsers) | one_of_parser | {
                const result = try one_of_parser.parse(allocator, src);
                if (result != null) {
                    return result;
                }
            }
            return null;
        }
    };
}
```

There are a few things to unpack here:

* `try one_of_parser.parse(allocator, src);` indicates that if parsing using `one_of_parser` returns an _error_ that our function should return immediately and not continue attempting to parse with other parsers.
* `if (result != null) {` is how you check if an Optional type in Zig is "None". I find this pretty interesting: it's not `null`, it's actually an optional "none" type - but it is called `null`. I'm not sure why, but can imagine this making the language friendlier to people unfamiliar with optional types.

## Using our OneOf parser combinator

Now for the cool part: we get to put both our `Literal` parser and `OneOf` parser combinator to _build a new parser_!

```zig
// Define our parser.
var one_of = OneOf([]u8, @TypeOf(reader)).init(&.{
    &Literal(@TypeOf(reader)).init("dog").parser,
    &Literal(@TypeOf(reader)).init("sheep").parser,
    &Literal(@TypeOf(reader)).init("cat").parser,
});
```

The above will parse one of `"dog"`, `"sheep"`, or `"cat"` from the input reader.

We're passing `@TypeOf(reader)` frequently above which makes the code a bit more cryptic than needed, and it would be possible to introduce a `OneOfLiteral` helper which makes the above instead read:

```zig
// Define our parser.
var one_of = OneOfLiteral([]u8, @TypeOf(reader)).init(&.{
    "dog",
    "cat",
    "sheep",
});
```

One thing to unpack here is this syntax for passing an array to `init`: `&.{...}`:

* The function takes a parameter `parsers: []*Parser(Value, Reader)`
* `.{...}` would give us _a fixed size array_ `[3]*Parser(Value, Reader)`
* `&.{}` gives us a pointer to an array, i.e. _a slice_ `[]*Parser(Value, Reader)`.

Since our list is known at compile time, we don't have to allocate or free memory for the slice. If our list was dynamic, we would need to do so.

Finally, we can actually use our parser above:

```zig
var p = &one_of.parser;
var result = try p.parse(allocator, &reader);
std.testing.expectEqualStrings("cat", result.?);
if (result) |r| {
    allocator.free(r);
}
```

## Runtime parser generation

You might be wondering how we would go from the `Literal` _parser_ and `OneOf` _parser combinator_ to actually _generating a parser at runtime that can parse the semantics defined in a regexp string_.

Since our `Parser` interface is a runtime interface (you can swap out the implementation at runtime) and since our parser combinator `OneOf` operates using that interface (only the return value must be known at compile time, it could be a generic AST node) it means that we can easily dynamically create slices of `[]*Parser(...)` at runtime based on the result of a parser combinator we have built - like our "dog, cat, sheep" parser from earlier.

The challenge left for you as a reader is to:

* Write _parsers_ like our `Literal` parser that can parse the components of our regexp `a[bc].*abc`:
    * `RegexLiteralParser` can parse `a`, `b`, and `c`, but not `abc` (the string.)
    * `RegexRangeOpenParser` can parse `[`.
    * `RegexRangeCloseParser` can parse `]`
    * `RegexAnyParser` can parse the `.` "any character" syntax.
    * `RegexRepetitionParser` can parse the `*` repetition operator.
* Write a _parser combinators_ like our `OneOf` parser, except have it parse a `Sequence` of parsers.
* Use our `Sequence` parser combinator and `RegexLiteralParser` to build a `RegexStringLiteralParser` - similar to how we built out "dog, cat, sheep" parser.
* Write a _new kind of function_ called a _runtime parser generator_ named `RegexParser` which will be super familiar:
    * Take in a _parser combinator_ called `RegexSyntaxParser` which can turn your regexp syntax into some intermediary like an AST.
    * Have your function _use parser combinators like OneOf, Sequence, etc._ to build a brand new parser at runtime based on that intermediary AST.
    * Return that new parser which parses the actual semantics described by the input regexp!

## Closing thoughts

I am sorry for not giving you a full (or even partial) regex engine :) I am still exploring this and it is a large undertaking, this blog post would be far too long if it was included.

You can find a copy of the final code with _parsers_ and _parser combinators_ [here](https://gist.github.com/slimsag/db2dd2c49aa038e23b654120e70c9b00). Just `zig init-exe` and plop them into your `src/` directory.

You may also want to check out [Mecha](https://github.com/Hejsil/mecha), a parser combinator library for Zig.

If anything was unclear or confusing, I'm happy to help: shoot me an email stephen@hexops.com or leave a comment on Hacker News / Reddit and I'll follow up.
