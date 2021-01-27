<a href="https://hexops.com"><img align="right" alt="Hexops logo" src="https://raw.githubusercontent.com/hexops/media/main/readme.svg"></a>

# Postgres Trigram search learnings

In this article I talk about learnings I have from trying to use pg_trgm as the backend for a search engine, Tridex, which aims to be a competitor to Google's [Zoekt](https://github.com/google/zoekt) ("Fast trigram based code search")

<small align="right">By Stephen Gutekanst, Jan 26, 2021</small>

## Background

I work @ [Sourcegraph](https://sourcegraph.com), which provides code search, code intelligence, and other developer tooling. (If you're one of my Sourcegraph co-workers, hey! Hexops is the name of my GitHub organization for after-hours experiments and what I hope will one day in the distant future become a successful game development company.)

Over the past ~8 months I have been exploring the intersection between game development and my work at Sourcegraph, and finding interesting overlap between the two. I have been working on a competitor to Google's [zoekt](https://github.com/google/zoekt) ("Fast trigram based code search"), which is one of the search backends used by Sourcegraph with a few goals in mind:

1. Produce a search backend I can use for Hexops, to provide search functionality for a user-voice type platform (think GitHub issues, but with duplicate issue removal and upvote/downvote capability), and other social-network type features in games I hope to one day create. i.e. not just code search
2. Learn more about search in hopes of being able to bring some of those learnings to Sourcegraph (I really want our commit/diff search to be indexed, and wish we could index more things in general.) It would also be cool to solve the ominous and difficult [Zoekt memory usage problem](https://github.com/google/zoekt/issues/86) we have had at Sourcegraph for as long as I can remember.
3. Provide a foundation for more bleeding-edge "here's a whacky idea" type experiments that are cool surrounding developer tools, but that are not necessarily guaranteed wins / anything I could reasonable pitch elsewhere.

## Indexing every character is different than regular FTS

First, it's important to note that my usage of pg_trgm is not the same as general FTS (Full Text Search) usage of pg_trgm in general. My usage (and the use case of code search) cares about every character being indexed, and being able to do regex searches - this is different than traditional FTS where only words matter.

## pg_trgm indexes apply to all data in that column

This sounds obvious, but in practice has interesting/weird implications when trying to build a code search engine like Zoekt.

For example, Zoekt builds indexes of repositories code in chunks (from what I understand) and then concurrently, in an unordered fashion, searches those repository code chunks (inverted trigram indexes). This plays a key role in the strategy of pagination that Zoekt implements: you can search over those chunks and give up searching further chunks after you've found enough results.

With pg_trgm, a naive approach would be to have a `file_contents` column with a pg_trgm GIN index over it, and put every file from every repository into that column. But that index would apply to _all_ file contents across every repository, so when you want to LIMIT search over that column you are searching over one giant trigram GIN index instead of many smaller ones. It's faster if your aim is to search the entire index, but if your aim is "get enough results and then get out" it's much slower, because you have to deal with the entire index instead of multiple smaller indexes. But, I have not tested this empirically - so take this statement with a grain or two of salt. It's possible I am wrong here.

There is an obvious way to counteract this effect, though: use one pg_trgm GIN index (i.e. a distinct table or column) per repository. You now have a distinct GIN trigram index per repository chunk. Of course, when there are thousands of repositories this introduces major complexity in schema management, query execution, etc. as you might imagine having thousands of tables/columns not exactly being great either.

It is possible that Postgres' [table data partitioning](https://www.postgresql.org/docs/10/ddl-partitioning.html) could interoperate with pg_trgm nicely to solve this problem, but I didn't explore this in-depth and found no information on the subject. Importantly, you would need to partition tables based on repository (or better, file-size-based chunks.) It is worth exploring this approach more.

## Naive usage of pg_trgm is competitive!

The good news is that even with the naive approach previously described, pg_trgm turns out to be approximately competitive with Zoekt, I assume due to it using a GIN index for trigram matching instead of an inverted index like Zoekt:

I don't have empirical measurements of this that I can share, you'll just have to take my word for it, but approximately on a 2020 Macbook pro with several thousand source code repositories:

* Query time performance is roughly the same for needle-in-the-haystack and haystack-full-of-needles queries over all repositories file contents.
* Pagination can be quite slow towards the tail end of the table, each subset being fetched requires a full search of the index to find the results at the end of the table. A streaming approach rather than traditional SQL pagination would be ideal.
* On-disk data size is quite small compared to Zoekt's, Postgres trigram GIN indexes appear to be quite small and its on-by-default data compression works really well with text.
* Postgres uses MUCH less memory than Zoekt. Like several orders of magnitude less.

Interestingly, however, Postgres uses MUCH less memory. The choice of using an inverted trigram index in Zoekt, [as I understand it](https://github.com/google/zoekt/issues/86), is also one of the reasons that its memory usage is so large (among other things, like Go being fairly relaxed about returning memory to the OS.) I also suggest reading [these Hacker News comments from 2018](https://news.ycombinator.com/item?id=18584294) and the linked article from Russ Cox about Google Code Search, from which Zoekt was ultimately born.

## Horizontally scaling pg_trgm is hard

Once your data no longer fits into a single machine / Postgres instance, things get tricky. How do you scale pg_trgm across multiple machines?

Postgres [supports a nauseating amount of complex High Availability deployment options](https://www.postgresql.org/docs/current/high-availability.html) for scaling horizontally, and ideally for a search engine you would want something like data partitioning where data is split across multiple hosts but also with the possibility of replication across multiple hosts (for the event a host goes down.)

One of the options it supports is horizontal data partitioning through splitting tables into multiple smaller tables, and then using a foreign data wrapper (postgres_fdw) to execute queries that access all of those tables across the network. This is described in a bit more depth [in this blog post](https://www.highgo.ca/2019/08/08/horizontal-scalability-with-sharding-in-postgresql-where-it-is-going-part-2-of-3/). This could be a good approach, but I decided not to explore this option further.

Ultimately I decided to go with a multiple-table approach, with each table representing a type of data (e.g. repository code) and performing horizontal sharding and scaling at the application layer outside of the DB entirely. I will explain why I took this approach in the next section.

## Deploying and tuning Postgres configuration is hard

In stark contrast to Zoekt, which uses a ridiculous amount of memory, with Postgres I was left with a different problem: I could not get it to use all available memory/CPU to perform search queries faster.

Raising `shared_buffers = 128MB` (default `32MB`) helped a fair amount, but still a similar issue. Ultimately I believe the majority of query time is spent not on CPU latency, but rather a combination of RAM lookups / L3 cache misses and IO latency.

Nonetheless, this introduced a new problem for me: I wanted this search engine to be as simple to deploy as possible, and the idea of having Postgres tuning being a requirement did not appeal to me. I have also seen in the field how many use Amazon RDS, which does not allow for tuning (and is often a several-years-outdated Postgres version anyway.)

With all of this in mind, I ended up going with deploying Docker containers with my own Postgres binary and configuration built-in and managing Postgres on behalf of the user. This ended up being interesting for other reasons I won't get into (think automatic zero-downtime upgrades from Postgres 12 -> 13.)

## Ultra large scales

Although using pg_trgm is competitive (much better than?) Zoekt - it's still not enough to be able to efficiently scale up to a massive scale such as Google. The index is still relatively large (in the hundreds of MB for thousands of repositories) well outside the bounds of CPU caches and that makes it kind of slow at truly large scales where the index grows near-linearly.

Ultimately.. Once you add in deployment pains, configuration tuning, trigram index splitting, horizontal scaling, etc. it's a lot less like using Postgres to build a search engine - and a lot more like using Postgres as a trigram index provider. It's interesting, and works, but there may be better options.

## Future exploration

A more fruitful direction may be to explore effectively the same architecture (i.e. roll-your-own-search-engine), but replacing pg_trgm and Postgres entirely with a custom ngram index built on top of the bloom-filter successor which is more L3-cache-friendly, [xor filters](https://lemire.me/blog/2019/12/19/xor-filters-faster-and-smaller-than-bloom-filters/).

I believe with this approach you could achieve scales/performance similar to Google, Bing, etc. while providing full regex search and more. This idea is not completely unfounded, it has been [suggested for indexing in ripgrep, for example](https://github.com/BurntSushi/ripgrep/issues/1518) (although I don't think they've noticed its opportunity yet and [appear to be going with an inverted trigram index similar to Zoekt](https://github.com/BurntSushi/ripgrep/issues/1497) instead.)

## Closing thoughts

In Tridex (the search engine I am working on), we're planning on exploring this avenue by replacing Postgres and pg_trgm with a custom trigram index based on xor-filters, and will likely write it in Zig. I only realized the opportunity here in a late-night conversation with a coworker who has an affinity for bloom filters, so perhaps I am misguided and this will turn up no fruit.

Follow this devlog for updates.

---

[![RSS Follow](https://shields.io/badge/RSS-follow-green?logo=RSS)](https://github.com/hexops/devlog/commits/main.atom) [![License: CC BY 4.0](https://img.shields.io/badge/License-CC%20BY%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by/4.0/)

<a href="https://hexops.com"><img height="50px" alt="Hexops logo" src="https://raw.githubusercontent.com/hexops/media/main/logo_whitebg.svg"></a>
