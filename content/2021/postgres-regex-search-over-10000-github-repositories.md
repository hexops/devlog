---
author: "Stephen Gutekanst"
title: "Postgres regex search over 10,000 GitHub repositories (using only a Macbook)"
date: "2021-02-17"
draft: false
categories:
- search
- trigrams
description: "In this article, we share empirical measurements from our experiments in using Postgres to index and search over 10,000 top GitHub repositories using pg_trgm on only a Macbook."
---

In this article, we share empirical measurements from our experiments in using Postgres to index and search over 10,000 top GitHub repositories using `pg_trgm` on only a Macbook.

This is a follow up to ["Postgres Trigram search learnings"](https://devlog.hexops.com/2021/postgres-trigram-search-learnings), in which we shared several learnings and beliefs about trying to use Postgres Trigram indexes as an alterative to Google's [Zoekt](https://github.com/google/zoekt) ("Fast trigram based code search").

We share our results, as well as [the exact steps we performed, scripts, and lists of the top 20,000 repositories by stars/language on GitHub](https://github.com/hexops/pgtrgm_emperical_measurements) so you can reproduce the results yourself should you desire.

## TL;DR

**This article is extensive and more akin to a research paper than a blog post.** If you're interested in our conclusions, see [conclusions](#conclusions) instead.

## Goals

We wanted to get empirical measurements for how suitable Postgres is in providing regexp search over documents, e.g. as an alterative to Google's [Zoekt](https://github.com/google/zoekt) ("Fast trigram based code search"). In specific:

* How many repositories can we index on just a 2019 Macbook Pro?
* How fast are different regexp searches over the corpus?
* What Postgres 13 configuration gives best results?
* What other operational effects need consideration if seriously attempting to use Postgres as the backend for a regexp search engine?
* What is the best database schema to use?

## Hardware

We ran all tests on a 2019 Macbook Pro with:

* 2.3 GHz 8-Core Intel Core i9
* 16 GB 2667 MHz DDR4

During test execution, few other Mac applications were in use such that effectively all CPU/memory was available to Postgres.

## Corpus

We scraped [lists of the top 1,000 repositories from the GitHub search API](https://github.com/hexops/pgtrgm_emperical_measurements/tree/main/top_repos) ranked by stars for each of the following languages (~20.5k repositories in total):

* C++, C#, CSS, Go, HTML, Java, JavaScript, MatLab, ObjC, Perl, PHP, Python, Ruby, Rust, Shell, Solidity, Swift, TypeScript, VB .NET, and Zig.

Cloning all ~20.5k repositories in parallel took ~14 hours with a fast ~100 Mbps connection to GitHub's servers.

### Dataset reduction

We found the amount of disk space required by `git clone --depth 1` on these repositories to be a sizable ~412G for just 12,148 repositories - and so we put in place several processes for further reduce the dataset size by about 66%:

* Removing `.git` directories resulted in a 30% reduction (412G -> 290G, for 12,148 repositories)
* Removing files > 1 MiB resulted in another 51% reduction (290G -> 142G, for 12,148 repositories - note GitHub does not index files > 384 KiB in their search engine)

## Database insertion

We [concurrently inserted](https://github.com/hexops/pgtrgm_emperical_measurements/blob/main/cmd/corpusindex/main.go) the entire corpus into Postgres, with the following DB schema:

```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE TABLE IF NOT EXISTS files (
    id bigserial PRIMARY KEY,
    contents text NOT NULL,
    filepath text NOT NULL
);
```

In total, this took around ~8 hours to complete and Postgres's entire on-disk utilization was 101G.

## Creating the Trigram index

We tried three separate times to index the dataset using the following GIN Trigram index:

```
CREATE INDEX IF NOT EXISTS files_contents_trgm_idx ON files USING GIN (contents gin_trgm_ops);
```

* **In the first attempt, we hit an OOM after 11 hours and 34 minutes.** This was due to a rapid spike in memory usage at the very end of indexing. We used a [fairly aggressive](https://github.com/hexops/pgtrgm_emperical_measurements#configuration-attempt-1-indexing-failure-oom) Postgres configuration with a very large max WAL size, so it was not entirely unexpected.
* **In the second attempt, we ran out of SSD disk space after ~27 hours**. Notable is that the disk space largely grew towards the end of indexing, similar to when we faced an OOM - it was not a gradual increase over time. For this attempt, we used the excellent [pgtune](https://pgtune.leopard.in.ua/#/) tool to reduce our first Postgres configuration as follows:

```
shared_buffers = 4GB → 2560MB
effective_cache_size = 12GB → 7680MB
maintenance_work_mem = 16GB → 1280MB
default_statistics_target = 100 → 500
work_mem = 5242kB → 16MB
min_wal_size = 50GB → 4GB
max_wal_size = 4GB → 16GB
max_parallel_workers_per_gather = 8 → 4
max_parallel_maintenance_workers = 8 → 4
```
* **In our third and final attempt, we cut the dataset in half and indexing succeeded after 22 hours.** In specific, we deleted half of the files in the database (from 19,441,820 files / 178GiB of data to 9,720,910 files / 82 GiB of data.) The Postgres configuration used was the same as in attempt 2.

## Indexing performance: Memory usage

In our first attempt, we see the reported `docker stats` memory usage of the container grow up to 12 GiB (chart shows MiB of memory used over time):

<img width="981" alt="image" src="https://user-images.githubusercontent.com/3173176/107313722-56bbac80-6a50-11eb-94c7-8e13ea095053.png">

In our second and third attempts, we see far less memory usage (~1.6 GiB consistently):

<img width="980" alt="image" src="https://user-images.githubusercontent.com/3173176/107314104-350ef500-6a51-11eb-909f-2f1b524d29b2.png">

<img width="980" alt="image" src="https://user-images.githubusercontent.com/3173176/107315387-ce3f0b00-6a53-11eb-886c-410f000f73bd.png">

## Indexing performance: CPU usage

Postgres' Trigram indexing appears to be mostly single-threaded (at least when indexing _a single table_, we test multiple tables later.)

In our first attempt, CPU usage for the container did not rise above 156% (one and a half virtual CPU cores):

<img width="982" alt="image" src="https://user-images.githubusercontent.com/3173176/107313915-cc277d00-6a50-11eb-9282-62159a127966.png">

Our second attempt was around 150-200% CPU usage on average:

<img width="980" alt="image" src="https://user-images.githubusercontent.com/3173176/107314168-507a0000-6a51-11eb-8a18-ec18752f7f16.png">

Our third attempt similarly saw an average of 150-200%, but with a brief spike towards the end to ~350% CPU:

<img width="980" alt="image" src="https://user-images.githubusercontent.com/3173176/107315239-8324f800-6a53-11eb-9a5b-fcc61d1a7b59.png">

## Indexing performance: Disk IO

Disk reads/writes during indexing averaged about ~250 MB/s for reads (blue) and writes (red). Native in-software tests show the same Macbook able to achieve read/write speeds of ~860 MB/s with <5% affect on CPU utilization.

<small>Addition made Feb 20, 2021:</small> We ran tests using native Postgres as well (instead of in Docker with a bind mount) and found better indexing and query performance, more on this below.

<img width="599" alt="image" src="https://user-images.githubusercontent.com/3173176/106507903-ec6f9e80-6488-11eb-88a8-78e5b7aacfd6.png">

## Indexing performance: Disk space

The database contains 9,720,910 files totalling 82.07 GiB:

```
postgres=# select count(filepath) from files;
  count  
---------
 9720910
(1 row)

postgres=# select SUM(octet_length(contents)) from files;
     sum     
-------------
 88123563320
(1 row)
```

**Before indexing**, we find that all of Postgres is consuming 54G:

```
$ du -sh .postgres/
 54G	.postgres/
```

After `CREATE INDEX`, Postgres uses:

```
$ du -sh .postgres/
 73G	.postgres/
```

Thus, the index size for 82 GiB of text is 19 GiB (or 23% of the data size.)

## Database startup times

From an operational standpoint, it is worth noting that if Postgres is starting clean (i.e. previous shutdown was graceful) then startup time is almost instantaneous: it begins accepting connections immediately and loads the index as needed.

However, if Postgres experienced a non-graceful termination during e.g. startup, it can take a hefty ~10 minutes with this dataset to start as it goes through an automated recovery process.

## Queries executed

In total, we executed 19,936 search queries against the index. We chose queries which we expect give reasonably varying amounts of coverage over the trigram index (that is, queries whose trigrams are more or less likely to occur in many files):

| Regexp query                          | Matching # files in entire dataset |
|---------------------------------------|------------------------------------|
| `var`                                 | unknown (2m+ suspected)            |
| `error`                               | 1,479,452                          |
| `123456789`                           | 59,841                             |
| `fmt\.Error`                          | 127,895                            |
| `fmt\.Println`                        | 22,876                             |
| `bytes.Buffer`                        | 34,554                             |
| `fmt\.Print.*`                        | 37,319                             |
| `ac8ac5d63b66b83b90ce41a2d4061635`    | 0                                  |
| `d97f1d3ff91543[e-f]49.8b07517548877` | 0                                  |

<details>
<summary>Detailed breakdown</summary>
<div markdown="1">

| Query                                 | Result Limit     | Times executed |
|---------------------------------------|------------------|----------------|
| `var`                                 | 10               | 1000           |
| `var`                                 | 100              | 1000           |
| `var`                                 | 1000             | 100            |
| `var`                                 | unlimited        | 4              |
| `error'`                              | 10               | 2000           |
| `error'`                              | 100              | 2000           |
| `error'`                              | 1000             | 200            |
| `error'`                              | unlimited        | 18             |
| `123456789`                           | 10               | 1000           |
| `123456789`                           | 100              | 1000           |
| `123456789`                           | 1000             | 100            |
| `123456789`                           | unlimited        | 2              |
| `fmt\.Error`                          | 10               | 1000           |
| `fmt\.Error`                          | 100              | 1000           |
| `fmt\.Error`                          | 1000             | 100            |
| `fmt\.Error`                          | unlimited        | 2              |
| `fmt\.Println`                        | 10               | 1000           |
| `fmt\.Println`                        | 100              | 1000           |
| `fmt\.Println`                        | 1000             | 100            |
| `fmt\.Println`                        | unlimited        | 2              |
| `bytes.Buffer`                        | 10               | 4              |
| `bytes.Buffer`                        | 100              | 4              |
| `bytes.Buffer`                        | 1000             | 4              |
| `bytes.Buffer`                        | unlimited        | 2              |
| `fmt\.Print.*`                        | 10               | 1000           |
| `fmt\.Print.*`                        | 100              | 1000           |
| `fmt\.Print.*`                        | 1000             | 100            |
| `fmt\.Print.*`                        | unlimited        | 2              |
| `ac8ac5d63b66b83b90ce41a2d4061635`    | 10               | 1000           |
| `ac8ac5d63b66b83b90ce41a2d4061635`    | 100              | 1000           |
| `ac8ac5d63b66b83b90ce41a2d4061635`    | 1000             | 100            |
| `ac8ac5d63b66b83b90ce41a2d4061635`    | unlimited        | 2              |
| `d97f1d3ff91543[e-f]49.8b07517548877` | 10               | 1000           |
| `d97f1d3ff91543[e-f]49.8b07517548877` | 100              | 1000           |
| `d97f1d3ff91543[e-f]49.8b07517548877` | 1000             | 100            |
| `d97f1d3ff91543[e-f]49.8b07517548877` | unlimited        | 2              |

</div>
</details>

## Query performance

In total, we executed 19,936 search queries against the database (linearly, not in parallel) which completed in the following times:

| Time bucket | Percentage of queries | Number of queries |
|-------------|-----------------------|-------------------|
| Under 50ms  | 30%                   | 5,933             |
| Under 250ms | 41%                   | 8,088             |
| Under 500ms | 52%                   | 10,275            |
| Under 750ms | 63%                   | 12,473            |
| Under 1s    | 68%                   | 13,481            |
| Under 1.5s  | 74%                   | 14,697            |
| Under 3s    | 79%                   | 15,706            |
| Under 25s   | 79%                   | 15,708            |
| Under 30s   | 99%                   | 19,788            |

## Query performance vs. planning time

The following scatter plot shows how 79% of queries executed in under 3s (Y axis, in ms), while Postgres's query planner had planned them for execution in under 100-250ms generally (X axis, in ms):

<img width="1252" alt="image" src="https://user-images.githubusercontent.com/3173176/107848471-ef379100-6db0-11eb-8396-4d156a179aae.png">

If we expand the view to include all queries, we start to get a picture of just how outlier these 21% of queries are (note that the small block of dots in the bottom left represents the same diagram shown above):

<img width="1250" alt="image" src="https://user-images.githubusercontent.com/3173176/107848517-3cb3fe00-6db1-11eb-9652-e65d7d88fe36.png">

## Query time vs. CPU & Memory usage

The following image shows:

* (top) Query time in milliseconds
* (middle) CPU usage percentage (e.g. 801% refers to 8 out of 16 virtual CPU cores being consumed)
* (bottom) Memory usage in MiB.

<img width="1255" alt="image" src="https://user-images.githubusercontent.com/3173176/107848716-efd12700-6db2-11eb-8e8b-a8141a6bdb0b.png">

Notable insights from this are:

* The large increase in resource usage towards the end is when we began executing queries with no `LIMIT`.
* CPU usage does not exceed 138%, until the spike at the end.
* Memory usage does not exceed 42 MiB, until the spike at the end.

We suspect `pg_trgm` is single-threaded within the scope of a single table, but with [table data partitioning](https://www.postgresql.org/docs/10/ddl-partitioning.html) (or splitting data into multiple tables with subsets of the data), we suspect better parallelism could be achieved.

## Investigating slow queries

If we plot the number of index rechecks (X axis) vs. execution time (Y axis), we can clearly see one of the most significant aspects of slow queries is that they have many more index rechecks:

<img width="1036" alt="image" src="https://user-images.githubusercontent.com/3173176/107849660-fc0cb280-6db9-11eb-9c10-cb7e74366ab7.png">


And if we look at [the `EXPLAIN ANALYZE` output for one of these queries](https://github.com/hexops/pgtrgm_emperical_measurements/blob/main/query_logs/query-run-3.log#L3-L24) we can also confirm `Parallel Bitmap Heap Scan` is slow due to `Rows Removed by Index Recheck`.

## Table splitting

Splitting up the search index into multiple smaller tables seems like an obvious approach to getting `pg_trgm` to use multiple CPU cores. We tried this by taking the same exact data set and splitting it into 200 tables, and found numerous benefits:

### Benefit 1: Incremental indexing

If indexing fails after 11-27 hours, as happened to us twice in the non-splitting approach, all progress is not lost.

### Benefit 2: Parallel indexing

Unlike our first non-splitting approach, which showed we were only able to utilize 1.5-2 virtual CPU cores, with multiple tables we are able to utilize 8-9 virtual CPU cores:

<img width="1143" alt="image" src="https://user-images.githubusercontent.com/3173176/108118901-3b5a2e00-705c-11eb-86d1-a7828517b2e8.png">

### Benefit 3: Indexing is 84% faster

Unlike our first attempt which took 22 hours in total, parallel indexing completed in only 3h27m.

### Benefit 4: Indexing uses 69% less memory

With non-splitting we saw peak memory usage up to 12 GiB. With the same exact Postgres configuration, we were able to index with only 3.7 GiB peak memory usage:

<img width="1140" alt="image" src="https://user-images.githubusercontent.com/3173176/108119244-bd4a5700-705c-11eb-949b-69828acd7c7c.png">

## Benefit 4: Parallel querying

Previously, we saw CPU utilization of only 138% (1.3 virtual CPU cores), with table splitting we see CPU utilization during queries of 1600% (16 virtual CPU cores) showing we are doing work fully in parallel:

<img width="1144" alt="image" src="https://user-images.githubusercontent.com/3173176/108114005-79078880-7055-11eb-9f55-bc4ca65c4808.png">

Similarly, we saw memory usage average around ~380 MiB, compared to only ~42 MiB before:

<img width="1143" alt="image" src="https://user-images.githubusercontent.com/3173176/108115193-04354e00-7057-11eb-9782-8d3125c122e1.png">

## Benefit 5: Query performance

We reran the same exact set of search queries, but a smaller number of times overall (350 queries, instead of 19.9k - which we found to still be a representative enough sample.)

As we can see below, table splitting in general led to a 200-300% improvement in query time for heavier queries that previously took 20-30s, now taking only 7-15s thanks to parallel querying (top chart is before, bottom is after, both in milliseconds):

<img width="1143" alt="image" src="https://user-images.githubusercontent.com/3173176/108156053-3d90ac80-709d-11eb-8f81-b93456b54a41.png">

We also grouped queries based on the `LIMIT` specified in the query and placed them into time buckets ("how many queries completed in under 50ms?") - comparing the two shows that less complex queries and/or queries for fewer results were negatively affected slightly, while larger queries were helped substantially:

| Change (positive is good) | Results limit | Bucket | **Queries in bucket before** | **Queries in bucket after** |
|------|-------|-----|------|-----------|
| -33% | 10   | <50ms  | 33% | 0%   |
| +13% | 10   | <250ms | 44% | 57%  |
| +33% | 10   | <1s    | 77% | 100% |
| -29% | 100  | <100ms | 29% | 0%   |
| +20% | 100  | <500ms | 50% | 70%  |
| +19% | 100  | <10s   | 80% | 99%  |
| -12% | 1000 | <250ms | 12% | 0%   |
| -13% | 1000 | <2.5s  | 77% | 64%  |
| +23% | 1000 | <20s   | 77% | 100% |
| +4%  | none | <20s   | 0%  | 4%   |
| +18% | none | <60s   | 0%  | 18%  |

Detailed comparisons are available below for those interested:

<details>
<summary>Queries with `LIMIT 10`</summary>
<div markdown="1">

| Time bucket | Percentage of queries (before) | Percentage of queries (after splitting) |
|-------------|--------------------------------|-----------------------------------------|
| 50ms | 33.00% (2999 of 9004) | 0% (0 of 100) |
| 100ms | 33.00% (2999 of 9004) | 1.00% (1 of 100) |
| 250ms | 44.00% (3999 of 9004) | 57.00% (57 of 100) |
| 500ms | 55.00% (4999 of 9004) | 79.00% (79 of 100) |
| 1000ms | 77.00% (6998 of 9004) | 80.00% (80 of 100) |
| 2500ms | 77.00% (7003 of 9004) | 80.00% (80 of 100) |
| 5000ms | 77.00% (7004 of 9004) | 80.00% (80 of 100) |
| 10000ms | 77.00% (7004 of 9004) | 100.00% (100 of 100) |
| 20000ms | 77.00% (7004 of 9004) | 100.00% (100 of 100) |
| 30000ms | 99.00% (8985 of 9004) | 100.00% (100 of 100) |
| 40000ms | 99.00% (9003 of 9004) | 100.00% (100 of 100) |
| 50000ms | 100.00% (9004 of 9004) | 100.00% (100 of 100) |
| 60000ms | 100.00% (9004 of 9004) | 100.00% (100 of 100) |

</div>
</details>

<details>
<summary>Queries with `LIMIT 100`</summary>
<div markdown="1">

| Time bucket | Percentage of queries (before) | Percentage of queries (after splitting) |
|-------------|--------------------------------|-----------------------------------------|
| 50ms | 29.00% (2934 of 10000) | 0% (0 of 100) |
| 100ms | 29.00% (2978 of 10000) | 0% (0 of 100) |
| 250ms | 39.00% (3975 of 10000) | 31.00% (31 of 100) |
| 500ms | 50.00% (5000 of 10000) | 70.00% (70 of 100) |
| 1000ms | 59.00% (5984 of 10000) | 79.00% (79 of 100) |
| 2500ms | 79.00% (7996 of 10000) | 80.00% (80 of 100) |
| 5000ms | 80.00% (8000 of 10000) | 80.00% (80 of 100) |
| 10000ms | 80.00% (8000 of 10000) | 99.00% (99 of 100) |
| 20000ms | 80.00% (8000 of 10000) | 100.00% (100 of 100) |
| 30000ms | 99.00% (9999 of 10000) | 100.00% (100 of 100) |
| 40000ms | 100.00% (10000 of 10000) | 100.00% (100 of 100) |
| 50000ms | 100.00% (10000 of 10000) | 100.00% (100 of 100) |
| 60000ms | 100.00% (10000 of 10000) | 100.00% (100 of 100) |

</div>
</details>

<details>
<summary>Queries with `LIMIT 1000`</summary>
<div markdown="1">

| Time bucket | Percentage of queries (before) | Percentage of queries (after splitting) |
|-------------|--------------------------------|-----------------------------------------|
| 50ms | 0% (0 of 904) | 0% (0 of 100) |
| 100ms | 0% (1 of 904) | 0% (0 of 100) |
| 250ms | 12.00% (114 of 904) | 0% (0 of 100) |
| 500ms | 30.00% (276 of 904) | 21.00% (21 of 100) |
| 1000ms | 55.00% (499 of 904) | 41.00% (41 of 100) |
| 2500ms | 77.00% (700 of 904) | 64.00% (64 of 100) |
| 5000ms | 77.00% (704 of 904) | 77.00% (77 of 100) |
| 10000ms | 77.00% (704 of 904) | 98.00% (98 of 100) |
| 20000ms | 77.00% (704 of 904) | 100.00% (100 of 100) |
| 30000ms | 88.00% (804 of 904) | 100.00% (100 of 100) |
| 40000ms | 99.00% (901 of 904) | 100.00% (100 of 100) |
| 50000ms | 99.00% (903 of 904) | 100.00% (100 of 100) |
| 60000ms | 100.00% (904 of 904) | 100.00% (100 of 100) |

</div>
</details>

<details>
<summary>Queries with no limit`</summary>
<div markdown="1">

| Time bucket | Percentage of queries (before) | Percentage of queries (after splitting) |
|-------------|--------------------------------|-----------------------------------------|
| 50ms | 0% (0 of 28) | 0% (0 of 50) |
| 100ms | 0% (0 of 28) | 0% (0 of 50) |
| 250ms | 0% (0 of 28) | 0% (0 of 50) |
| 500ms | 0% (0 of 28) | 0% (0 of 50) |
| 1000ms | 0% (0 of 28) | 0% (0 of 50) |
| 2500ms | 0% (0 of 28) | 0% (0 of 50) |
| 5000ms | 0% (0 of 28) | 0% (0 of 50) |
| 10000ms | 0% (0 of 28) | 0% (0 of 50) |
| 20000ms | 0% (0 of 28) | 4.00% (2 of 50) |
| 30000ms | 0% (0 of 28) | 16.00% (8 of 50) |
| 40000ms | 0% (0 of 28) | 16.00% (8 of 50) |
| 50000ms | 0% (0 of 28) | 18.00% (9 of 50) |
| 60000ms | 0% (0 of 28) | 18.00% (9 of 50) |

</div>
</details>

## Postgres-in-Docker vs. native Postgres

<small>Addition made Feb 20, 2021</small>

In our original article we did not clarify the performance impacts of running Postgres inside of Docker with a volume bind mount. This was raised as a potential source of IO performance difference to us by [Thorsten Ball](https://twitter.com/thorstenball).

We ran all tests above with Postgres in Docker, using a volume bind mount (the osxfs driver, not the experimental FUSE gRPC driver.)

We additionally ran the same table-splitting benchmarks on a native Postgres server ([reproduction steps here](https://github.com/hexops/pgtrgm_emperical_measurements#native-postgres-tests)) and found the following key changes:

### CPU usage & memory usage: approximately the same

CPU and memory usage was approximately the same as in our Docker Postgres tests.

We anticipated this would be the case as the Macbook does have VT-x virtualization enabled (default on all i7/i9 Macbooks, and confirmed through `sysctl kern.hv_support`)

### Indexing speed was ~88% faster

Running the statements to split up the large table into multiple smaller ones, i.e.:

```sql
CREATE TABLE files_000 AS SELECT * FROM files WHERE id > 0 AND id < 50000;
CREATE TABLE files_001 AS SELECT * FROM files WHERE id > 50000 AND id < 100000;
...
```

Was much faster in native Postgres, taking about 2-8s for each table instead of 20-40s previously, and taking only 15m in total instead of 2h before.

Parallel creation of the Trigram indexes using e.g.:

```sql
CREATE INDEX IF NOT EXISTS files_000_contents_trgm_idx ON files USING GIN (contents gin_trgm_ops);
```

Was also much faster, taking only 23m compared to ~3h with Docker.

### Query performance is 12-99% faster, depending on query

We re-ran the same 350 queries as in our earlier table-splitting benchmark, and found the following substantial improvements:

1. Queries that were previously very slow noticed a ~12% improvement. This is likely due to IO operations needed when interfacing with the 200 separate tables.
2. Queries that were previously in the middle-ground noticed meager ~5% improvements.
3. Queries that were previously fairly fast (likely searching only over a one or two tables before returning) noticed substantial 16-99% improvements.



<details>
<summary>Exhaustive comparison details (negative change is good)</summary>
<div markdown="1">

| Change | Time bucket | Queries under bucket **before** | Queries under bucket **after** |
|--------|-------------|---------------------------------|--------------------------------|
| 0%     | 500s        | 350 of 350                      | 350 of 350                     |
| -12%   | 100s        | 309 of 350                      | 350 of 350                     |
| -12%   | 50s         | 309 of 350                      | 350 of 350                     |
| -12%   | 40s         | 308 of 350                      | 350 of 350                     |
| -12%   | 30s         | 308 of 350                      | 349 of 350                     |
| -7%    | 25s         | 307 of 350                      | 330 of 350                     |
| -7%    | 25s         | 307 of 350                      | 330 of 350                     |
| -8%    | 20s         | 302 of 350                      | 330 of 350                     |
| -8%    | 20s         | 302 of 350                      | 330 of 350                     |
| -5%    | 10s         | 297 of 350                      | 311 of 350                     |
| -26%   | 5s          | 237 of 350                      | 319 of 350                     |
| -7%    | 2500ms      | 224 of 350                      | 240 of 350                     |
| -9%    | 2000ms      | 219 of 350                      | 240 of 350                     |
| -9%    | 1500ms      | 219 of 350                      | 240 of 350                     |
| -16%   | 1000ms      | 200 of 350                      | 237 of 350                     |
| -14%   | 750ms       | 190 of 350                      | 221 of 350                     |
| -23%   | 500ms       | 170 of 350                      | 220 of 350                     |
| -59%   | 250ms       | 88 of 350                       | 217 of 350                     |
| -99%   | 100ms       | 1 of 350                        | 168 of 350                     |
| -99%   | 50ms        | 1 of 350                        | 168 of 350                     |

</div>
</details>

## Conclusions

We think the following learnings are most important:

* `.git` directories, even with `--depth=1` clones, account for 30% of a repositories size on disk (at least in top 10,000 GitHub repositories.)
* Files > 1 MiB (often binaries) account for another 51% of the data size on disk of repositories.
* On only a Macbook Pro, it is possible to get Postgres Trigram regex search over 10,000 repositories to run most reasonable queries in under 5s - and certainly much faster with more hardware.
* `pg_trgm` performs single-threaded indexing and querying, unless you split your data up into multiple tables.
* By default, a Postgres `text` colum will be compressed by Postgres on disk out of the box - resulting in a 23% reduction in size (with the files we inserted.)
* `pg_trgm` GIN indexes take around 26% the size of your data on disk. So if indexing 1 GiB of raw text, expect Postgres to store that text in around ~827 MiB plus 279 MiB for the GIN trigram index.
* Splitting your data into multiple tables if using `pg_trgm` is an obvious win, as it allows for paralle indexing which can be the difference between 4h vs 22h. It also reduces the risk of an indexing failure after 22h due to e.g. lack of memory and uses much less peak memory overall.
* Docker bind mounts (not volumes) are quite slow outside of Linux host environments (there are many other articles on this subject.)

If you are looking for fast regexp or code search today, consider:

* [Sourcegraph](https://sourcegraph.com) (disclaimer: the author works here, but this article is not endorsed or affiliated in any way)
* [Zoekt](https://github.com/google/zoekt)
* [Ripgrep](https://github.com/BurntSushi/ripgrep)

Follow this devlog for updates as we continue investigating faster ways to do regexp & ngram search at large scales.
