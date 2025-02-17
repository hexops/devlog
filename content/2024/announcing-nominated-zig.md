---
author: "Emi Stein"
title: "Announcing Mach nominated Zig versions"
date: "2024-01-07"
draft: false
categories:
- mach
- zig
- gamedev
description: "Today we're announcing Mach nominated Zig versions, a sweet-spot between stable Zig and nightly Zig which offers a different balance of latest-and-greatest features and fixes, and less of a moving target."
images: ["/img/2024/nominated-zig-versions.png"]
---

Today we're announcing Mach nominated Zig versions, a sweet-spot between stable Zig and nightly Zig which offers a different balance of latest-and-greatest features and fixes, and less of a moving target.

If you are in the Zig community, you likely fall into one of two categories:

* You target Zig nightly, a target which moves every day
* You target Zig stable, which may be released once or twice per year.

## The challenge of using Zig stable

In recent years, Zig stable has about 2 releases per year. There are great benefits to using stable Zig:

* It is generally documented how to update/migrate your code to the new Zig version, rather than it being an ad-hoc process of discovery.
* The release is at a point where the Zig core team feels it is solid and ready to go (Zig releases are done when they are done, not usually timed.)
* These versions get nice numbers associated with them, package authors, distributions, etc. can easily target them.

But, for a language that is developed quite quickly, and has yet to reach v1.0 stability.. when you find that an awesome new feature like the package manager, incremental compilation, or something else you care about has just landed in Zig nightly.. are you going to wait 6 months or more to get that change?

## The challenge of using Zig nightly

Zig nightly is an ever-moving target. Although it is often nearly as stable as stable releases, that is not neccessarily the case during large refactors - such as the migration to the self-hosted compiler.

There are benefits to using nightly, though! It means you are testing the latest version of Zig, and your project can exist in a sort of symbiotic relationship with the Zig project where you test new functionality, help provide feedback on it, discover new issues, and have a greater chance of getting them fixed/addressed while that code is on everyone's mind.

Unlike stable Zig, you're not integrating 6+ months of breaking changes into your codebase all at once (quite painful!) but rather doing so as the breaks happen. Since many others in the Zig community do target nightly Zig, there are often people around who can help you with upgrading your code.

One major downside, aside from death by a thousand paper cuts, is that everyone targets a different Zig nightly version. Often, it's difficult to coordinate with others and keep your code compatible with theirs.

## The challenge Mach has using Zig nightly

We use Zig nightly, but we only periodically update the version.

Updating Mach's Zig version involves updating a dependency tree of over [40+ Zig repositories](https://github.com/hexops/mach/issues/1135), first updating the Zig code itself and testing manually, then updating their CI pipelines, then going and updating anyone who depends on that repository - from the bottom of the tree to the top!

We've built some serious automation to help with this process, but it is still painful - and it is also the case that every time we update our Zig version, our users need to do the same: we're not just updating Zig for us, we're updating Zig for every user of Mach.

This process has worked _pretty_ well for us, but the frequency of updates has always been ad-hoc, and when we do update it is a bit chaotic.

## We need our ecosystem of 40+ packages to be compatible

If we haven't updated our Zig version for a bit of time, then we end up with tens of outstanding pull requests by people who maybe don't use _all of Mach_ but rather just parts of it, and they can become (rightfully!) fairly frustrated that getting their pull-request merged takes us a while.

We can't just merge a one-off pull request to one repository, we have to update all 40+ to be compatible with the same Zig version.

<a href="/img/2024/so-many-pull-requests.png"><img alt="so many pull requests" src="/img/2024/so-many-pull-requests.png"></a>

## Coordinating outside Mach

Although Mach provides a lot of libraries, there are still many important aspects of gamedev we do not have yet. Some folks in the Mach community will pull in third-party Zig projects, like those from zig-gamedev, introducing another challenge in ensuring their code works with the same version.

## Announcing Mach nominated Zig versions

Today we're formalizing the process we've (generally) been following. This formalization will make it easier for others to understand what we're doing and when, and also make it easier for other projects to align their Zig version with Mach's if they desire.

Throughout the year (aiming for the 4th day of the month), we will pick the latest Zig nightly version at that time and nominate it for use:

| When      | What                    | 🚀 Other notable event                             |
|-----------|-------------------------|----------------------------------------------------|
| January   |                         | 🚀 Mach version release                            |
| February  |                         | 👋 Anticipated influx of new Machanists / Ziguanas |
| March     | ⚡ Zig version nominated |                                                    |
| April     |                         |                                                    |
| May       | ⚡ Zig version nominated |                                                    |
| June      | ⚡ Zig version nominated |                                                    |
| July      |                         | 🚀 Mach version release                            |
| August    |                         | 👋 Anticipated influx of new Machanists / Ziguanas |
| September | ⚡ Zig version nominated |                                                    |
| October   |                         |                                                    |
| November  | ⚡ Zig version nominated |                                                    |
| December  |                         | 👋 Anticipated influx of new Machanists / Ziguanas |

These versions will be noted as e.g. '2024.1.0-mach', and will correspond to a specific Zig nightly version from that month.

The exact versions, which nightly version they map to, and the whole process (which is more involved), is [documented in full here](https://machengine.org/about/nominated-zig/).

At the time of writing this, you'll see that [2024.1.0-mach](https://machengine.org/about/nominated-zig/#202401) is marked as 'in progress' - we will make sure that the Zig version we intend to nominate is at least compatible with all Mach projects before finalizing the nomination.

### A sweet spot between nightly and stable

Mach's nominated Zig versions provide a different set of tradeoffs, we believe it is a sweetspot between the two extremes of nightly and stable. You can benefit from the changes in Zig 2-3x faster than if you were using stable, and suffer less from the never-ending game of catch-up and incompatibilities between projects that nightly necessarily requires.

Other projects can target the same Zig version if they wish to be compatible with Mach Zig packages. For example, zig-gamedev is aiming to target the same versions. We encourage other gamedevs using Zig to do the same.

Projects that target nightly Zig can often be coincidentally compatible, too, since they possibly had a compatible Zig version around the time we nominated a Zig version for use.

## Final thoughts

You can read more about the specifics of everything in the [Mach documentation](https://machengine.org/about/nominated-zig).

We're currently working on nominating the first version, which will likely be finalized in the next week or so.

## Thanks

<div style="display: flex; flex-direction: row; align-items: center;">
    <img align="left" style="max-height: 12.5rem;" src="https://user-images.githubusercontent.com/3173176/187348488-0b52e87d-3a48-421c-9402-be78e32b5a20.png"></img>
    <ul>
        <li>Join the <a href="https://discord.gg/XNG3NZgCqp">Mach Discord server</a> (check #discuss for this article)</li>
        <li>Checkout <a href="https://machengine.org">machengine.org</a></li>
        <li>Consider <a href="https://github.com/sponsors/emidoots">sponsoring development</a> so we can do more of it!</li>
    </ul>
</div>
