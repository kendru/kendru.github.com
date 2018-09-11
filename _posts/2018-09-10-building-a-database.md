---
layout: post
title: "Beginning a Database"
description: "The beginning of my journey building a next-generation distributed database"
category: Database
tags: ["database", "storage", "architecture", "distributed systems"]
---

For years, my dream project has been to build a database from scratch. For
most of my career, the most challenging questions have revolved around how to
ingest, store, and query data in a performant way, and while I have found a
number of really great solutions, there usually has not been a single product
that can meet all of the requirements.

For instance, at my current job, we receive potentially thousands of updates
per second, but we need to run ad-hoc analytical queries over terabytes of
data spanning many months. At a high level, we ingest the data into MySQL,
and at a certain point, we guarantee that no more data will be written to
particular rows, and we have a complex ETL process that moves them to Google
BigQuery. On the application side, we have to take user queries and distribute
them across both data sources and merge them on the way back. A lot of
engineering effort went into this system, and it works remarkably well, but in
the process, we ended up essentially building a distributed query engine, a
query optimizer, and a high-level storage manager. These are all normally
pieces of a database management system, but in our case, they were ad-hoc
components that we added to our system as necessary. In this case, the problem
that we are solving is needing both a write-heavy OLTP system and a performant
OLAP system, which do not normally go together.

I have been doing some reach into database systems, and I think that the
research and technology are finally at a point where it is feasible to build
a database that has acceptable performance for both transactional and analytics
workloads. Between advances in algorithms and datastructures for indexing data,
the low cost of DRAM, SSDs, and now, persistent memory, and the understanding of
the properties of distributed systems, I believe that we can now start to
develop database systems that act as both transactional engines and data
warehouses. Particularly, I believe that in-memory (or primarily in-memory)
databases are the key to unlocking this sort of system.

<script async src="//pagead2.googlesyndication.com/pagead/js/adsbygoogle.js"></script>
<ins class="adsbygoogle"
     style="display:block; text-align:center;"
     data-ad-layout="in-article"
     data-ad-format="fluid"
     data-ad-client="ca-pub-6265787006533161"
     data-ad-slot="3706397953"></ins>
<script>
(adsbygoogle = window.adsbygoogle || []).push({});
</script>

This is not a novel idea, and there are a number of databases out there that
do in fact have a similar design: [HyPer](https://hyper-db.de/), which is now
part of Tableau, and [Kudu](https://kudu.apache.org/) come to mind. There are
a couple of other OLAP databases out there that support inserts or streaming
ingestion but not updates, such as [Druid](http://druid.io/) and
[Google BigQuery](https://cloud.google.com/bigquery/). My understanding is that
Google's internal PowerDrill product is similar to Druid in that it keeps
recent data in memory for faster processing, but as far as I know, it does not
support OLTP functionality. Additionally, the trade-offs that these databases
make render them unsuitable for certain applications. For instance, Druid
(which we initially considered at my current workplace) favors accumulation of
highly-denormalized event-structured data. That is, in traditional data
warehouse terms, each entry is a fact along with all of its dimensions. While
Druid's compression is fantastic, it was not feasible for us to collect all
dimensions that we were interested in tracking with every insert. Additionally,
Druid is (or at least was at the time I evaluated it) quite painful to set up
and operate.

Ultimately, the database that I would like to build has a hybrid NSM
(row-oriented) and DSM (column-oriented) storage model that can seamlessly
distribute queries against both engines and handles migration of tuples between
each storage layout seamlessly. Also, in my experience, most users are more
interested in recent data, so the system should keep data in memory for as long
as possible but should still be able to process data that does not reside in
main memory, preferably in a parallel fashion. However, unlike a traditional
disk-based DBMS, it should not have the overhead of a buffer manager to page old
data into memory until it is evicted, since queries over old data are more
likely to be one-offs and should not evict the "hot" data set. This sort of
parallel processing of OLAP queries over old data is now readily available
with technologies like fast object storage, cloud computing, and container
orchestration, and it does not need to first be loaded into the database's
primary memory, especially since it is so simple to elastically spin up
worker processes to execute the queries over old data in parallel. This is
where I believe that a lot of in-memory and columnar databases go wrong. They
assume that processing is expensive and centralized. On the other side of the
coin, inherantly distributed systems like Hive and Spark SQL assume that
all data is far away - at least not already indexed and in memory - and that
access paterns are not predictable enough to optimize.

My plan is to begin building the components necessary for a complete DBMS in
Rust, partly because I do not have a lot of experience with systems programming,
and I like having a very picky compiler to help me from doing a lot of stupid
things and partly because I believe that it would be much quicker for me to
build a large-scale system in Rust than in C or C++. I plan to write about the
components as I build them, including the query parser, planner, and optimizer,
MVCC protocols, scheduler, a consensus protocol, distributed hash table,
cluster membership and failure detector, storage manager, system catalog,
indexes, user interfaces and tooling, and possibly vectorized execution for
small-scale paralellism. This is by far the largest project that I have ever
taken on, but I hope that through the process I can learn a lot and that the
posts to come will be useful to other database developers.
