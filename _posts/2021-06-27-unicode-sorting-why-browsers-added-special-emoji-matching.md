---
layout: post
title: "Unicode sorting is hard & why browsers added special emoji matching to regexp"
categories: unicode, regex, zig
author: "Stephen Gutekanst"
---

As I work on [Zorex, an omnipotent regexp engine](https://github.com/hexops/zorex) I have stumbled into a world of tales about why Unicode text sorting is so annoying in the modern day. Let's talk about that.

- [Why ASCII sorting is not enough](#why-ascii-sorting-is-not-enough)
- [Twitter's emoji problem, or when Unicode locale-aware sorting Really Matters™](#twitters-emoji-problem-or-when-unicode-locale-aware-sorting-really-matters™)
- [Browsers added special emoji matching to regexp](#why-browsers-added-special-emoji-matching-to-regexp)
- [Language comparison](#language-comparison)
    - [JavaScript Collator sorting is not guaranteed across browsers](#javascript-collator-sorting-is-not-guaranteed-across-browsers)
    - [Go sort.Strings is not locale aware](#go-sortstrings-is-not-locale-aware)
    - [Rust Vec sorting is not locale aware](#rust-vec-sorting-is-not-locale-aware)
    - [Swift's default is not locale aware, but unicode support is notable](#swifts-default-is-not-locale-aware-but-unicode-support-is-notable)
    - [Zig's ziglyph package](#zigs-ziglyph-package)
- [Why is localized text sorting hard?](#why-is-localized-text-sorting-hard)
- [WebAssembly may make things worse?](#webassembly-may-make-things-worse)

## Why ASCII sorting is not enough

Perhaps you are sorting strings in JavaScript like this:

```javascript
const words = ['Bears', 'Beetle', 'kiss', 'Similar', 'Apples'];
words.sort();
// [ "Apples", "Bears", "Beetle", "Similar", "kiss" ]
```

And that works pretty well, until someone translates it to German:

```javascript
const words = ['Bären', 'Käfer', 'küssen', 'Ähnlich', 'Äpfel'];
words.sort();
// [ "Bären", "Käfer", "küssen", "Ähnlich", "Äpfel" ]
```

The preferred alphabetical sorting would be `[ "Ähnlich", "Äpfel", "Bären", "Käfer", "küssen" ]` - `Array.sort` doesn't do that.

That is because it is sorting lexicographically by byte values in the string, and not taking into account locales.

## Twitter's emoji problem - or when Unicode locale-aware sorting Really Matters™

Twitter is [no stranger to issues with emojis](https://9to5google.com/2018/05/21/twitter-android-emoji-updates/), but have you ever thought about how they check if a hashtag contains only legal characters and emojis? Regexp, of course!

You might think one could just use a regexp unicode character class, like `[\u{1f300}-\u{1f5ff}]` - but that only covers a single codepoint! Emojis and other text rely on combining multiple Unicode codepoints to compose _grapheme clusters_ - and often what we see as a single visible character on our screen.

The full regexp needed to match all emojis with codepoints would be:

```regexp
(?:\ud83e\uddd1\ud83c\udffb\u200d\u2764\ufe0f\u200d\ud83d\udc8b\u200d\ud83e\uddd1\ud83c\udffc|\ud83e\uddd1\ud83c\udffb\u200d\u2764\ufe0f\u200d\ud83d
[102,816 characters omitted]
```

For your sake, I've omitted the other 102,816 characters of that regexp. You can view it here: https://regex101.com/r/2ia4m2/7

## Browsers added special emoji matching to regexp

Luckily for Twitter and others, ECMAScript's [TC39 proposal a few years back](https://github.com/tc39/proposal-regexp-unicode-property-escapes) extended the regexp engine to support Unicode property escapes for emojis and a few other Unicode properties so you can write e.g.:

```regexp
\p{Emoji_Presentation}
```

Without packing several thousand bytes of Unicode data tables or regexp into your JS bundle.

## Language comparison

As [Daniel Lemire said](https://lemire.me/blog/2018/12/17/sorting-strings-properly-is-stupidly-hard/): _sorting strings is stupidly hard_.

### JavaScript Collator sorting is not guaranteed across browsers

You may have found browser's [`String.prototype.localCompare`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/localeCompare) or [`Intl.Collator`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/Collator) and they **DO** fix the issue[[1]](https://ourcodeworld.com/articles/read/958/how-to-sort-an-array-of-strings-alphabetically-with-special-characters-properly-with-javascript):

```
const words = ['Bären', 'Käfer', 'küssen', 'Ähnlich', 'Äpfel'];
words.sort(Intl.Collator().compare);
// [ "Ähnlich", "Äpfel", "Bären", "Käfer", "küssen" ]
```

(note, however, you may wish to use `Intl.Collator('de').compare` instead to sort according to German language customs)

However, beware that if you look at [the ECMA spec](https://tc39.es/ecma402/#sec-collator-comparestrings) for this you will find:

> It is **recommended** that the CompareStrings abstract operation be implemented following Unicode Technical Standard 10, Unicode Collation Algorithm [...]
>
> Applications should not assume that the behaviour of the CompareStrings abstract operation for Collator instances with the same resolved options will remain the same for different versions of the same implementation. 

Although many browsers may produce similar sorting results - not all will. For one thing, not all locales are available across browsers.

Further, different browsers may choose to sort things differently. For example IE 11 sorting "co-op" after "coop" while other browsers do the opposite.[[2]](https://stackoverflow.com/questions/33919257/sorting-strings-with-punctuation-using-intl-collator-is-inconsistent-across-brow)

## Go sort.Strings is not locale aware

It may be interesting to note that Go's `sort.Strings` operates on byte comparisons, and has the same issue as JavaScript's `Array.prototype.sort`:

```Go
words := []string{"Bären", "Käfer", "küssen", "Ähnlich", "Äpfel"}	
sort.Strings(words)
// [Bären Käfer küssen Ähnlich Äpfel]
```

One can easily perform unicode code point (rune) sorting in Go, which would fix the above example - but note that rune sorting is not locale-aware, and importantly that [a Go rune is not the same as a visible character](https://www.reddit.com/r/golang/comments/o1o5hr/fyi_a_single_go_rune_is_not_the_same_as_a_single) and would not take into account grapheme clusters.

For proper Unicode locale-aware sorting in Go, you need to use the Unicode Collation Algorithm via [golang.org/x/text/collate](https://pkg.go.dev/golang.org/x/text/collate) but be sure to also apply normalization to your text first via [golang.org/x/text/unicode/norm](https://pkg.go.dev/golang.org/x/text@v0.3.6/unicode/norm)

### Rust Vec sorting is not locale aware

A Rust `Vec` of strings implements sorting[[3]](https://doc.rust-lang.org/std/primitive.str.html#impl-Ord) lexicographically by their byte values, consistent with Go's `sort.Strings` and JavaScripts `Array.prototype.sort`:

```
let mut vec = Vec::new();
vec.push("Bären");
vec.push("Käfer");
vec.push("küssen");
vec.push("Ähnlich");
vec.sort("Äpfel");
println!("{:?}", vec);
// ["Bären", "Käfer", "küssen", "Ähnlich", "Äpfel"]
```

Locale-aware sorting in Rust is provided [by ICU4C bindings by Google, google/rust_icu](https://github.com/google/rust_icu) (note however, there have been a number of [vulnerabilities in the ICU4C library](https://github.com/rust-lang/rust/issues/14656#issuecomment-45164318)) and there is ongoing work to implement internationalization in pure Rust as a safer alternative: [unicode-org/icu4x](https://github.com/unicode-org/icu4x).

### Swift's default is not locale aware, but unicode support is notable

Swift remains consistent with other languages in sorting strings lexicographically by byte value:

```swift
var words = ["Bären", "Käfer", "küssen", "Ähnlich", "Äpfel"]
words.sort()
// ["Bären", "Käfer", "küssen", "Ähnlich", "Äpfel"]
```

However, it is notable that Swift includes locale sensitive sorting out of the box[[4]](https://sarunw.com/posts/different-ways-to-sort-array-of-strings-in-swift/):

```swift
var words = ["Bären", "Käfer", "küssen", "Ähnlich", "Äpfel"]
let sorted = words.sorted { (lhs: String, rhs: String) -> Bool in    
    return lhs.localizedStandardCompare(rhs) == .orderedAscending
}
// ["Ähnlich", "Äpfel", "Bären", "Käfer", "küssen"]
```

It also seems quite notable just [how very unicode-aware the Swift documentation is on their String type](https://developer.apple.com/documentation/swift/string). Other languages could learn a thing or two here in educating developers.

## Zig's ziglyph package

Zig's standard library is still quite under development, however it seems likely that major unicode functionality will be outside the stdlib.

Luckily, [@jecolon](https://github.com/jecolon) in the Zig community is working on an excellent package for this: [ziglyph](https://github.com/jecolon/ziglyph).

I mention this because I'm a fan of the language and have recently begun contributing to that package; but otherwise Zig isn't any different than other languages listed here aside from there being no real "default" way to sort strings from what I know.

## Why is localized text sorting hard?

I believe there are a combination of factors at play:

* Most languages leave Unicode locale-aware text sorting as an afterthought.
* Most developers don't care enough to use Unicode, let alone implement locale-aware text sorting. Internationalization is always "that thing we'll do if somebody complains" or an afterthought.
* It's hard. It wasn't until recently that we got semi-decent support for it across browsers, and what is there still leaves a lot to be desired.
* Many are still running into dated software, like NodeJS versions from ~2019 ish that [didn't have full ICU support on by default](https://github.com/nodejs/node/issues/19214).

## WebAssembly may make things worse?

As a closing thought, I just want to hint at why I think WebAssembly will make things worse before they get better.

Whether your application is in Go and has it's own Unicode Collation Algorithm (UCA) implementation, or Rust and uses bindings to the popular ICU4C library - one thing is going to remain true: it requires large data files to work.

The UCA algorithm depends on two quite large data table files to work:

* [UnicodeData.txt](https://www.unicode.org/Public/9.0.0/ucd/UnicodeData.txt) for normalization, a step required before sorting can take place.
* [allkeys.txt](http://www.unicode.org/Public/UCA/12.0.0/allkeys.txt) for weighting certain text above others.
* And more, if you want truly locale-aware sorting and not just "the default" the UCA algorithm gives you.

Together, these files can add up to over a half a megabyte.

While WASM languages could shell out to JavaScript browser APIs for collation, I suspect they won't due to the lack of guarantees around those APIs. 

A more likely scenario is languages continuing to leave locale-aware sorting as an optional, opt-in feature - that also makes your application larger.

I think this a worthwhile problem to solve, so I am working on [compression algorithms for these files specifically](https://github.com/jecolon/ziglyph/issues/3) in Zig to reduce them to only a few tens of kilobytes.
