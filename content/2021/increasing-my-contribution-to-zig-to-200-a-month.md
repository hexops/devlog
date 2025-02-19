---
author: "Emi"
title: "My game development journey & why I'm increasing my contribution to Zig to $200/mo"
date: "2021-04-10"
draft: false
categories:
- zig
- gamedev
description: "Today I increased my monthly donation to Zig to $200 a month. Before Zig, I have not contributed financially to any open source project. Before I can explain why I am so extremely excited about the Zig programming language and its community, I need to explain where I come from."
images: ["https://user-images.githubusercontent.com/3173176/114259396-7ef95600-9982-11eb-9e32-3b8edff3e67f.png"]
---

Today, I increased my monthly donation to Zig to $200 a month. Before Zig, I have not contributed financially to any open source project.

Before I can explain why I am so extremely excited about the [Zig](https://ziglang.org/) programming language and its community, I need to explain where I come from.

- [I grew up playing Linux games like Mania Drive](#i-grew-up-playing-linux-games-like-mania-drive)
- [It wasn't long before I found that the Mania Drive game engine was open-source.](#it-wasnt-long-before-i-found-that-the-mania-drive-game-engine-was-open-source)
- [I was so infatuated with this game engine, I convinced my dad's coworkers to pay me to build them a virtual meeting world](#i-was-so-infatuated-with-this-game-engine-i-convinced-my-dads-coworkers-to-pay-me-to-build-them-a-virtual-meeting-world)
- [But the game kept crashing at random, and I had no idea why](#but-the-game-kept-crashing-at-random-and-i-had-no-idea-why)
- [Panda3D: Disney's Python/C++ game engine](#panda3d-disneys-pythonc-game-engine)
- [The Panda3D game engine opened new doors for me](#the-panda3d-game-engine-opened-new-doors-for-me)
- [I began to prevail](#i-began-to-prevail)
- [But my limited knowledge hit me again](#but-my-limited-knowledge-hit-me-again)
- [Learning C++](#learning-c)
- [Learning Go, writing my own game engine](#learning-go-writing-my-own-game-engine)
- [My game engine appeared on Hacker News (2014)](#my-game-engine-appeared-on-hacker-news-2014)
- [Joining Sourcegraph](#joining-sourcegraph)
- [Six and a half years later, I'm still at Sourcegraph.](#six-and-a-half-years-later-im-still-at-sourcegraph)
- [But I'm still a game developer at heart](#but-im-still-a-game-developer-at-heart)
- [C was easier for me as a beginner than C++](#c-was-easier-for-me-as-a-beginner-than-c)
- [Unity is the new Flash](#unity-is-the-new-flash)
- [Why do we encourage building, but not understanding?](#why-do-we-encourage-building-but-not-understanding)
- [One language to write your game and engine in](#one-language-to-write-your-game-and-engine-in)
- [Looking for the one language to rule them all](#looking-for-the-one-language-to-rule-them-all)
  - [Could Rust be it?](#could-rust-be-it)
  - [Could the V language be it?](#could-the-v-language-be-it)
  - [Could I build it?](#could-i-build-it)
- [Discovering Zig](#discovering-zig)
  - [Learning Zig](#learning-zig)
  - [Working in it](#working-in-it)
  - [The community is incredible](#the-community-is-incredible)
  - [My commitment to Zig](#my-commitment-to-zig)


## I grew up playing Linux games like Mania Drive

<iframe width="720" height="480" src="https://www.youtube.com/embed/7YFicbaXHw0" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

Mania Drive was an open-source clone of the popular Trackmania series. Me and my siblings in our early teens  easily spent hundreds, if not thousands, of hours in Mania Drive.

In retrospect it has quite bad graphics, physics, game-play mechanics, etc. But it was customizable! There was a simple tile-based level editor. We would spend days building the most confusing, crazy, impossible maps to beat so we could challenge each other. We would play it all night.

Obsession over this game led to even more modding: the discovery of [Blender](https://www.blender.org) meant we could create even more custom maps than in the limited tile-based map editor. Although the Blender UI was pretty rough back then:

<img class="color" src="https://user-images.githubusercontent.com/3173176/114266230-073f2180-99aa-11eb-9196-5c546fe71fb8.png">

## It wasn't long before I found that the Mania Drive game engine was open-source.

[Raydium](http://memak.raydium.org/index.php), the C game engine behind Mania Drive, is still around today - one of the beauties of open source software! At the time, the things about it that just blew my mind were:

* It supported scripting through PHP! I had used PHP a lot with LAMP stacks, so the idea that I could script the engine in PHP was _mind blowing to now 14-year old me._
* 2 years later, I got an iPod touch and the Raydium developers had just posted a demo video showing the engine running on the iPhone. 16 year old me thought this was _literally_ the coolest thing ever, albeit immensely disappointed I did not have a Mac to build it for my iPod:

<iframe width="720" height="480" src="https://www.youtube.com/embed/wcPfxr9BgA4" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

## I was so infatuated with this game engine, I convinced my dad's coworkers to pay me to build them a virtual meeting world

My dad was running one of his many startups at the time - it had some momentum behind it, basically a platform like Ebay but for selling services instead of goods. Several of his work friends were funding it with significant amounts of their own money.

Unfortunately for them, they spent most of their focus on business operations than actually getting a product out the door. Lucky for me, however, this meant they had came across Sun's [Project Wonderland](https://en.wikipedia.org/wiki/Open_Wonderland) - the delightfully terrible 3D virtual workplace of the future (or so Sun thought, before they had to sell to Oracle.) It was _terrible,_ barely a good demo:

<img class="color" src="https://user-images.githubusercontent.com/3173176/114258853-625b1f00-997e-11eb-959d-15a2cdafae9c.png">

It required something like 32 CPUs and 64G of memory to run the server for just 8 players - no small feat back in 2010! The client was laggy, there were virtual whiteboards you could draw on but everything was slow. Even its VOIP feature was glitchy - although quite novel at the time. It was all around _a terrible experience._

16-year-old me convinced my dad and his coworkers to instead pay me to build them a better version: one using Raydium, C - and PHP.

It wasn't long before I had some amateur copy of Project Wonderland - ironically better than Sun's in _many aspects_ - and surely worse in others. It even had a client auto-updater built with wxWidgets and Python (it just shelled out to an `svn` client to download the latest copy of the game, hah!)

## But the game kept crashing at random, and I had no idea why

The truth was literally a 16-year old script kiddy copying and pasting C code from various demos of Raydium, without a care in the world for freeing memory or avoiding stack corruption.

```
// Don't remove this print statement. Game will crash!
```

It was around this time that I began to really get into Python: it was simple, something I could really wrap my brain around, and it was powerful. I stumbled into cython and wrote OpenGL bindings - this time with more appreciation for memory management.

## Panda3D: Disney's Python/C++ game engine

Panda3D was the game engine Disney used to create Toontown Online and Pirates of the Carribean Online:

<img class="color" src="https://user-images.githubusercontent.com/3173176/114259007-b31f4780-997f-11eb-9741-db4507dd820f.png">

<img class="color" src="https://user-images.githubusercontent.com/3173176/114259061-0db8a380-9980-11eb-8991-1f8443388cc3.png">

It was written in C++, with automatic binding generation for Python. In fact, many portions of the engine were written in _just_ Python and not usable from C++ at all. [They revamped their website site recently, so I guess it's still around.](https://www.panda3d.org)

## The Panda3D game engine opened new doors for me

Discovering Panda3D opened new doors for me. At around 16-17 years old now, I was able to really get my first real taste of game development: I could write games in this -- in Python -- and _they wouldn't crash in ways that I couldn't understand._

Pretty soon, I had actual games in the works. I was starting to learn about why draw order matters - and how I had no understanding of mip-mapping:

<img class="color" src="https://user-images.githubusercontent.com/3173176/114259274-8e2bd400-9981-11eb-905d-cd675c582f5b.png">

<img class="color" src="https://user-images.githubusercontent.com/3173176/114259299-bb788200-9981-11eb-97df-ad00c29a54c2.png">

## I began to prevail

At this point I had several, actually working games - I was proud of what I was working on, had multiplayer functionality hooked up to a MySQL database even.

<img class="color" src="https://user-images.githubusercontent.com/3173176/114259396-7ef95600-9982-11eb-9e32-3b8edff3e67f.png">

## But my limited knowledge hit me again

For my game, I wanted nothing more than for my friends to be able to chat with me using a chat box. The problem was, Panda3D's Python GUI library, DirectGUI, was just too slow at rendering text.

I tried everything I could, and even got to the point where I was asking on the forums if it was possible to draw a TextNode with multi-threading:

> Calls to TextNode.generate() are very expensive.
>
> Is there a way for Panda to run all TextNode.generate() calls in a seperate thread? I’ve attempted doing it on my own using direct.stdpy.threading.Thread, only it causes dead locks, I would guess this is to my own lack of knowledge.
>
> could anyone help me?

I didn't get a response. I couldn't solve the issue. "I can't add a chat box to my games" became a problem _I could not solve._

## Learning C++

I was at a point where I had rewritten most of Panda3D's UI components myself in Python (mind you, theirs _are_ written in Python - you cannot use them from C++. I don't know why I did this.)

But I still needed a way to render text. I needed a way to make `TextNode.generate()` faster. Little did I know at the time that it was generating geometry from freetype and creating one draw call per text drawn - which is super slow, and did not help my naive usage of its API!

The harsh reality was that I didn't have anybody to teach me. I spent months trying to learn C++, but it is a beast (and "Disney game engine C++" is, of course, a flavor of C++ not found in books.) It wasn't something I could handle as 16-year-old kid without any real knowledge of low level languages.

In trying to learn C++, something became painfully clear to me:

> _Having part of my application written in Python and part of it written in C++, two very different languages, was only great until I realized I _had_ to dive into this large C++ code base and had no knowledge of it._

I gave up.

## Learning Go, writing my own game engine

When Google announced Go, I heard about it very early on. At this time, they were still advertising it as a low-level systems language, an alternative to C, _a better C_. But more forgiving, because it had a garbage collector.

Coming from a predominantly Python background at the time, this sounded incredible to me: I could write a game engine in this and understand my code _end to end_ and make sure there is no single piece that I do not understand.

I spent the next 4 years of my life, almost 100% full-time working on [Azul3D, a game engine in Go](https://azul3d.org) - and spent only minimal time attending online community college on the weekends.

There was _so much_ that I learned during this time, about software engineering, game engines, audio, input, math, image and audio codecs, blender plugins, file formats, physics, and working with other people (some cool things [like a NES emulator came out of that](https://github.com/nwidger/nintengo))

I learned an _immense_ amount, but I had nothing to show for it aside from [a funny looking website](https://azul3d.org) and some quite poor screenshots (to the dismay of every person I told.)

<img class="color" src="https://user-images.githubusercontent.com/3173176/114266918-f42e5080-99ad-11eb-8cff-f9376f3bf0bb.png">

## My game engine appeared on Hacker News (2014)

[Someone posted it on Hacker News](https://news.ycombinator.com/item?id=8151028), which was both exciting but also extremely depressing for me at the time. I took the feedback as statements that what I was doing _was wrong_, rather than as feedback about how to improve:

> The web site looks cool, but it sets off a whole bunch of red flags for me.

> the go programming language is not very suitable for games at all.

> if you truly need a performant graphics engine, it's going to be either C++, C or Rust anyway. 

> Azul3D is for programmers and doesn't provide GUI-editors.
>
> So, you write your levels using a text editor? That's not for programmers, that's for people who hate themselves. 

> No screenshots of the game at all?

> Garbage collector FAQ isn't necessarily reassuring, since it seems to say "go through the same hoops other GC gaming platforms push you through". Obligatory Rust gaming comment goes here.

> in Rust you have code without GC, but the compiler makes sure that everything is freed.

I learned so much from this interaction:

* Being transparent about project status is important.
* I shouldn't have "hidden" screenshots of the project. I was worried people would judge what the engine is capable of based on bad programmer artwork: instead, they judged it for having none.
* I should've talked about the interesting parts more:
  * Did you know there is a D* lite pathfinding algorithm that was used in one of the Mars rovers, is super simple, and handles dynamic terrains? Much nicer than A* and other variants.
  * What my vision for a game engine deeply integrated with Blender, and developer-first, would look like in practice.
* I frankly shouldn't have cared so much. I thought what I was doing was awesome, and I let others' viewpoints affect my own view of my work negatively.

## Joining Sourcegraph

It was around this time that I was basically deciding: _what would I do for a living?_

Luckily, someone in the Go community (whom I'd never talked to before) reached out to me and asked "hey, what are you doing?" - I told them I was in school, and left out the part where I was a college student living with parents, scraping by, and likely going grocery-store-part-time-job seeking soon.

<img class="color" src="/img/2021/bill-thanks.jpg">

I didn't come from a background that would lead me to believe I could make a living programming in Go, to the contrary my parents often warned me I couldn't and that I would need to go into Cisco network infrastructure instead.

I was told in blunt terms, I could scrape by doing what I love - or make a killing doing something I hate. My parents were mechanical engineers at aerospace companies (I'll let you guess which path they took.)

Bill's short ~20 minute conversation with me, quite literally changed my life in ways I couldn't have imagined. I often think about where I would be today had he not reached out to me, and I never quite knew how to reach back out and say thank you in a way that was as meaningful to him as what he did was for me.

## Six and a half years later, I'm still at Sourcegraph. 

I've learned _so much_ about startups, being a good engineer, management, business operations, cloud infrastructure, teamwork, communication, and so much more in the last six years I've spent at Soucegraph. There are so many stories I have, and so many great people I have had the opportunity to work with because of it.

We grew from awkward little startup without a clear product, a tiny team, an uncertain future - into a sprawling metropolis with massive amounts of happy users, customers, $50m i series C funding, and have grown the team to over a hundred people all over the world. I have played a key role in that, and continue to this day.

A passion for making games as a kid, turned into a passion for making developer tools all around better. I still have much to do here.

## But I'm still a game developer at heart

If there's one thing I return to _regularly_, _consistently_, and _frequently_ despite working a demanding job at a startup - it's game development. And you're going to hear a lot more about that soon.

Since March of last year, I began basically working two jobs: every day after I sign off from work at Sourcegraph, I spend around 8 hours working on game development.

I am more determined than ever before, and success or fail - _I will try._

## C was easier for me as a beginner than C++

Hacking together games in Raydium's C API taught me that C is hard, but also showed me in retrospect that if I had _just a little bit more guidance_, If C was just _slightly_ easier, if I only knew the tricks of how to debug C programs: I would have been immensely successful in working with it.

With Panda3D, writing some decent games in its Python API only to later find I needed to dive into this magical box of a complex C++ core made me believe that:

1. **C++ is less beginner friendly than C.** One major reason for this is due to the different C++ dialects: you're not going to understand Panda3D C++, or Unreal C++ - by going and reading books about the language or taking a class. They create their own dialects through the language. Today with different C++ versions, even the textbooks and classes you find will be using different dialects.
2. There are not good tutorials or explanations online about how game engines work and why. I regularly find that very experienced software engineers and even people who work in Unity or Unreal regularly, simply do not have a decent grasp of how game engines work. "What do you mean polygon count is not very important?!" are among the most basic questions that arise, with modern game engines abstracting away so many bits that your average developer merely says:

> "Game engines are just magical ultra-complex things I could never even begin to understand! Only the professional AAA studios and god programmers like Jonathan Blow should even try to do that!"

I do not subscribe to this belief - and believe that most game developers _have been robbed_ of the proper end-to-end understanding of game engines they deserve.

## Unity is the new Flash

You, dear reader, do not understand _just how far the bar for game development has been lowered._

<a href="https://www.youtube.com/watch?v=Nj8gt_92c-M"><img class="color" src="https://user-images.githubusercontent.com/3173176/114283707-f6b99600-99ff-11eb-82f8-5fd2e2139636.png"></a>

Putting together a game in Unity is so beyond ridiculously easy today with Unity that it is incredible, the game engine is truly the new Adobe Flash equivalent.

You could pick up that engine today, and have a silly little game you yourself put together the next.

Of course, with Unity, comes large problems for serious game developers:

* There are _so many_ people hacking together Unity games that the quality of the information out there is quite bad.
* The quality of what is on the Unity asset store is quite bad.
* Unity encourages hacking things together to get a quick demo running - and it shows. Game developers hide the fact that they use Unity, because it has such a negative connotation with players that Unity == low quality.

## Why do we encourage building, but not understanding?

Game engines today are the epitome of _large complex code-bases_:

* The people and companies working on them value features over quality.
* When there is a major issue, there are few people with an understanding of it to be found.
* Teaching people how to write good software is hard - and that's our customer base (I imagine Unity/Unreal say) - far easier to give them something akin to a scripting language. It's _good_ even if our users don't understand how all of this works.

Teaching is hard, but if done right is invaluable. There is a reason NeHe Productions' OpenGL tutorials are still revered today: they are incremental, and teach in the form of building blocks on top of what you previously learned.

There's a reason many AAA studios simply _throw out everything_ and start from scratch when working on their next title.

We encourage building new things, but not understanding existing things.

## One language to write your game and engine in

Scripting languages for game engines stem from multiple desires - the most common being some variant of:

* C++ is hard, but we need it for performance.
* My level designers can't write C++ code!
* I cannot understand C++, but do know C#/Python/Java/etc.

A lot of people have a _terrible_ experience from school where they were taught C or C++, had absolutely no understanding of what was going on - and were told "This is programming!"

I believe that in general, writing your game in a different language than the engine (Unity's C#/C++ core model, Panda3D's Python/C++ core model, and yes - perhaps even Unreal's [Blueprints](https://blueprintsfromhell.tumblr.com/)/C++ core model - which I admit is the better of the three)

<a href="https://blueprintsfromhell.tumblr.com"><img class="color" src="https://user-images.githubusercontent.com/3173176/114284178-025a8c00-9a03-11eb-8b22-3f7cd6324b31.png"></a>

Pictured: The Unreal character controller blueprint for a game called [Diacrisis](https://store.steampowered.com/app/1037260/Diacrisis/).

Whether you have good code or bad code, good blueprints or bad blueprints - the truth is that having one part of your application in a completely different language _creates a significant barrier to learning._ I believe that is a bad thing, and the long-term costs outweigh the benefits.

## Looking for the one language to rule them all

### Could Rust be it?

Initially, I spent a substantial amount of time considering Rust as that language. It's offer of memory safety guarantees is extremely compelling to me.

I even convinced us to adopt Rust at Sourcegraph in some form, our syntax highlighter is [a little Rust HTTP server](https://github.com/sourcegraph/syntect_server) that was basically write-and-forget. We haven't maintained it at all, and it's held up pretty well for over 5 years.

But maintaining it has been _brutal_. We mostly have Go developers there, and despite a strong desire from many of them to learn Rust really none of them have been able to successfully dive into the codebase and get started.

Rust's learning curve is _steep_. Steeper than C++ in my view, and definitely steeper than C (despite its many, massive flaws.)

I spent upwards of 6 months on-and-off trying to become proficient at writing Rust code, and I never really became productive: regularly stumbling across complex issues in downstream dependencies (often used by everyone, but maintained by no one in the rust-lang-nursery.)

**I _love_ the idea of Rust. I love what it promises. And I kept going back to it on-and-off for over 6 months _because I truly wanted to be able to be productive in it._**

It didn't work. "I'm just not smart enough to use this language" I often thought. And I fear this will be the takeaway of many who hear the promise of the language, only to discover another "I took a C++ class in school and it was terrible" experience, leading so many more developers to conclude "I'm not good enough for low-level programming, I should learn JavaScript instead".

### Could the V language be it?

**UPDATE:** The V language author [reached out over Twitter](https://twitter.com/v_language/status/1382171515924447234?s=20) and it would seem my memory was faulty about what happened here, this was due to a misunderstanding almost 100% on my side and I have falsely mis-characterized the V community here as being less friendly then they were in practice, and I am deeply sorry for that.

I believe my criticisms below about the controversy surrounding the project and the secretive nature _when it launched_ are still valid, and were ultimately major factors in why I chose to not further consider it.

At the same time, **I want to point out that V does not look the same as when it launched - and anybody who like me left due to those issues may do well [to reconsider it today](https://vlang.io/) as the project and details surrounding it appear to have changed substantially.**

What this section originally said was:

> When I heard about [the V programming language](https://news.ycombinator.com/item?id=25511073), it seemed right on the spot.
>
> I immediately jumped into the community to chat with the author, despite the controversy surrounding it - and tried to get more info about it, how he was thinking of the language, etc.
>
> I asked if there were plans to support raw multi-line string literals, like Go. I was struck by a firm 'No. Go doesn't have raw string literals either." - it was the unfriendly community I came across, the controversy surrounding it, and the _secretive nature of the project_ ("I have this, but I'm not going to share it yet") that made me lose faith in its promise.
>
> This wasn't a language whose community I could join and contribute to.

### Could I build it?

When the COVID-19 pandemic first hit, I thought to myself:

> If Go isn't it, Rust isn't it, the V language isn't it - could I build it? Could I create the "better C" I am looking for? What would it look like?

4 months later, I had a pretty good picture. I had an early stages compiler for the language in Go using LLVM, and knew what I wanted in a "better C". There was a _long_ road ahead, but I had a picture of it. Until..

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">*cat spills coffee on $2800 laptop, frying SSD with ~4 months of uncommitted work on EBNF parser generators* yeah.. no, that’s.. that’s okay, I wanted to rewrite all of that code. Yeah. This is fine.</p>&mdash; Emi (@emidoots) <a href="https://twitter.com/emidoots/status/1265452387453431808?ref_src=twsrc%5Etfw">May 27, 2020</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script> 

Obviously, I was an idiot and should've just `git push`d my code - or backed up my laptop - but nonetheless this was a setback.

## Discovering Zig

I continued to look for this mythical "better C" - and one name that kept arising in my sphere was [Zig](https://ziglang.org).

I didn't pay much attention to it, until I shared it with my brother for the 3rd time:

<img class="color" src="/img/2021/ando-thanks.jpg">

> "...I already shared this with you?"
>
> "I am really excited about this. It's literally the language I was trying to build before I think"

### Learning Zig

In trying to learn Zig, there were two things that struck me:

* I could be productive in Zig right away. Transitioning from Go at work to Zig after-hours every day _was easy._
* The community was so friendly, inviting, and helpful in answering my questions.
* I continuously saw a theme of "this is a decentralized community, there is no 'official' thing we'll ever push onto you, we want everyone to contribute and _truly be a part of this_"

Zig became the first open-source project I had _ever_ contributed to financially.

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">And if none of the above convinces you, let me tell you the following: <a href="https://twitter.com/ziglang?ref_src=twsrc%5Etfw">@ziglang</a> is the first language I have felt strongly I should try and contribute to, and the ONLY open source project I have ever donated to. No other has been so compelling</p>&mdash; Emi (@emidoots) <a href="https://twitter.com/emidoots/status/1319546299520200704?ref_src=twsrc%5Etfw">October 23, 2020</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script> 

### Working in it

Thus far, I've worked on two things in Zig:

* [an implementation of Xor Filters and Fuse Filters, which are faster and smaller than Bloom and Cuckoo filters and allow for quickly checking if a key is part of a set.](https://github.com/hexops/xorfilter)
* [Zig, Parser Combinators - and Why They're Awesome](https://devlog.hexops.com/2021/zig-parser-combinators-and-why-theyre-awesome)

I continue to work in Zig daily, with no plans to stop - mark my words, this is an amazing language to work in.

### The community is incredible

Over time, I watched and read more content from the Zig developers. It's been beautiful to see:

* Them constantly, proactively advocate against zealotry of the language.
* Them constantly advocate for new members of the community to actually help others.

Not only that, but I began to notice the Zig foundation actually _directly paying open source developers through donations_:

* [Please welcome Vexu to the core Zig team](https://www.reddit.com/r/Zig/comments/fvfguq/please_welcome_vexu_to_the_core_zig_team/)
* [Please welcome Jakub Konka to the Core Zig Team](https://www.reddit.com/r/Zig/comments/j2u1ww/please_welcome_jakub_konka_to_the_core_zig_team/)
* [Please welcome Alex Nask to the Core Zig Team](https://www.reddit.com/r/Zig/comments/ixvjsf/please_welcome_alex_nask_to_the_core_zig_team/)
* [Please welcome Frank Denis to the Core Zig Team](https://www.reddit.com/r/Zig/comments/mgluix/please_welcome_frank_denis_to_the_core_zig_team/)

> ZSF is a small organization and makes efficient use of monetary resources. The plan is to keep it that way, but we do want to turn our unpaid volunteers into paid maintainers to help merge pull requests and make swifter progress towards 1.0. The whole point of ZSF being non-profit is to benefit people. **We’re trying to get open source maintainers paid for their time.**

(from https://ziglang.org/zsf)

This is such a beautiful thing to see happening, and I hope that other open source communities take lessons from Zig here. The execution here is so important, and so far the Zig community's execution has been incredible here.

### My commitment to Zig

For me, Zig ticks all the boxes of a programming language that could fundamentally upend the way that video games are built for the better.

I want to see it succeed - and make it succeed at exactly that. Today, I raise my monthly contribution [on GitHub sponsors](https://github.com/sponsors/ziglang) to $200/mo. I would encourage anyone reading this to go and find ways to contribute (financially or not) to a vision you believe in.

In addition to the above, I am committed to building the following in Zig:

* A game engine for the future
* Better developer tools (not just for game developers)
* Several real video games, which I believe can be competitive with what AAA studios offer today.

Thanks for reading my journey, and I hope you'll consider following it in the future.
