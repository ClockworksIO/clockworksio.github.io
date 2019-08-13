---
title: Two Kafka Superpowers
tags: Kafka kplex
language: EN
author:
  - team
---

We discuss partitioning — arguably the highest impact decision in any
Kafka setup — and introduce kplex, a tool for repartitioning Kafka
topics consistently and on-the-fly, allowing you to unlock concurrency
and perform correct stateful processing across partitions.

<!--abstract-->

Like any technology considered core infrastructure, Kafka forces its
users to make certain trade-offs. What sides we take in these trades
becomes increasingly harder to change as data accumulates. This is
exacerbated by Kafka's distinguished role as an immutable historic
record. Immutability, we might say, cuts both ways. 

In the following we look at partitioning — arguably the highest impact
decision in any Kafka setup — from two different perspectives: the
physical (concerned with scalability) and the logical (concerned with
correctness). We notice that an optimal, correctness-preserving
partitioning strategy *depends on what consumers will do* with the
data, thus impeding later use cases. We introduce
[kplex](https://www.clockworks.io/kplex/), a tool for repartitioning
Kafka topics consistently and on-the-fly, allowing you to unlock
consumer concurrency and perform correct stateful processing across
partitions.

## Partitioning - The Physical Perspective

Clusters are happy as long as each machine is only asked to bear load
in proportion to its relative capabilities. I.e., for a cluster made
up of identical machines, we want data to be distributed
uniformly. Machines that are asked to do more than their fair share,
tend to act up, or worse, get in the way of their peers.

In an ideal setting, taking into account only the physical
perspective, we would have the freedom to distribute data exclusively
according to the relative capabilities of each machine in our
cluster. Doing so prevents hotspots and ensures smooth scaling as we
add machines.

Unfortunately this requirement is often at odds with the constraints
imposed by the (arguably much more important!) need to produce correct
outputs.

## Partitioning - The Logical Perspective

Partitions are both the unit of concurrency and of consistency in
Kafka. The more partitions we have, the more consumer instances we can
bring to bear in parallel, increasing throughput. On the other hand,
records that need to be consumed in aggregate, in a specific order, or
both must go to the same partition. Kafka would like us to keep the
number of partitions within reasonable limits[^partition-limit] for
various performance-related reasons[^partition-performance]. However
it is the ordering guarantees provided by partitions that impose much
more stringent constraints, as the following two extremes will
highlight.

**Fully Sequential** We need to persist a database transaction log in
Kafka, i.e. a sequence of transaction data annotated with logical
transaction times `t0`, `t1`, `t2`, and so forth. For such a topic,
any downstream consumer will have to see *all* records up to some `t*`
and — crucially — see them *in transaction time order*. We therefore
have no choice but to use a single partition for this topic.

**Embarrasingly Parallel** We need to compress a stream of image data
and upload them to a blob store. Here, the processing time for each
record is comparatively high, but any individual record is fully
self-contained and can thus be processed by a stateless consumer. In
this scenario, we are free to choose the number of topics entirely
based on the throughput that we want this system to achieve.

Many Kafka use cases fall somewhere closer to the center of this
spectrum, exhibiting more granular "consistency domains" within which
records must be presented in order. Examples for such domains are the
subset of records affecting an individual user or those originating
from a specific geographic region.

In an ideal setting, from a logical point of view, we therefore want
the freedom to assign each consistency domain to its own physical
partition for maximum throughput — while still making sure that a
consumer sees all records for any specific key, and in the exact order
they were produced in.

## Partitioning Decoupled

We have seen now how throughput and skew considerations alone would
lead us to using a great many physical partitions and distribute
records between them uniformly. The correctness guarantees demanded by
our use case on the other hand, force us to partition according to
attributes of the data itself. This raises an uncomfortable issue,
because correctness is a joint property of producer, storage, and
consumer. Therefore, no matter how carefully we choose a partitioning
scheme, it will always interfere with some valid use cases later on.

> [...] the partitioning strategy for your producers depends on what
> your consumers will do with the data.
>
> - Amy Boyle, "Effective Strategies for Kafka Topic Partitioning"[^newrelic]

In order to satisfy the trimuvirate of throughput, skew, and
consistency, we will have to *decouple physical from logical
partitioning*, while <u>preserving ordering guarantees</u> in the
process. We built [kplex](https://www.clockworks.io/kplex/) to do
precisely that.

Specifically, we encounter two types of mismatch: (1) strong physical
guarantees supporting weak logical requirements — here the physical
distribution *limits the concurrency* that could otherwise be applied
to the computation, and (2) weak physical guarantees thwarting strong
logical requirements — here the physical distribution *makes it
impossible* to do consistent, stateful processing.

`kplex` solves both of these.

## Repartitioning a topic on-the-fly

`ksql-datagen`[^datagen] is a handy tool for generating synthetic
Kafka topics. We use this to populate three partitions of a
`pageviews_by_page` topic, each containing records not unlike the
following sample:

``` json
{"viewtime":1565606197468,"userid":"User_6","pageid":"Page_62"}
{"viewtime":1565606197566,"userid":"User_9","pageid":"Page_60"}
{"viewtime":1565606197821,"userid":"User_4","pageid":"Page_98"}
{"viewtime":1565606198058,"userid":"User_7","pageid":"Page_14"}
{"viewtime":1565606198622,"userid":"User_1","pageid":"Page_63"}
```

We initially choose `pageid` as the partitioning key, thus making sure
that all pageview records for any specific page end up on the same
partition and remain in the order they were produced in (which
corresponds to the `viewtime` order).

While this suited our initial use cases well, we might want to write a
consumer that looks at the history of pageviews *of each individual
user*. This is problematic, because pageview records for any
individual user are strewn across all of the physical partitions. We
therefore want to consume all partitions in parallel, reshuffling
records by `userid` as we go, all the while preserving `viewtime`
order. This is captured by the following `kplex` job:

``` json
{
  "workers": 3,
  "kafka": { "broker": "localhost:9092" },
  "topics": {
    "pageviews_by_page": {
      "max_delay_ms": 30000,
      "polling_interval": {"secs": 1, "nanos": 0}
    }
  },
  "derive": {
    "pageviews_by_user": {
      "from": "pageviews_by_page",
      "key": {"Pointer": "/userid"},
      "timestamp": {"Pointer": "/viewtime"},
      "order": "TimeOrder",
      "output": {
        "VirtualPartitions": { "count": 9 }
      }
    }
  }
}
```

Let's try it out first.

![repartition pageviews](/assets/blog/kplex/repartition_pageviews_extended.gif)

What you see in the above GIF is `kplex` repartitioning the three
physical partitions (each respecting `viewtime` order) of
`pageviews_by_page` into nine virtual partitions by user (each
respecting `viewtime` order). Each virtual partition feeds a fifo
pipe, waiting to be consumed (in this case by `cat` writing into a
file). No intermediate Kafka topics are created in the process.

``` shell
# A common pattern combining kplex with xargs.
kplex <config> | xargs -P9 -n2 <consumer>
```

`kplex` takes a job configuration (like the one above) as input, which
should provide four pieces of information:
 
 `workers` | How many cores to use.
 `kafka`   | How to talk to your Kafka cluster.
 `topics`  | Meta-data about the physical topics you want to process.
 `derive`  | Virtual topics to derive from those physical topics.

As you can see, the basic version of `kplex` is designed for use on
individual machines with multiple cores. Any program that works with
input streams can be a `kplex` consumer. For larger use cases we offer
a distributed version of `kplex`.

## Reading Consistently From Multiple Partitions

In the previous example we changed not only the partitioning key, but
also chose a higher number of virtual partitions. Going from `n`
physical to `m > n` virtual partitions can be useful even *without*
changing the partitioning key, because it can unlock concurrency for
I/O-heavy consumers.

The other extreme however, going from `n` to a single partition is
interesting as well, because it corresponds to reconstructing a
consistent timeline of events for an entire topic. There is much more
to it[^3df], but this is a first step to processing things like
distributed transaction logs — which traditionally have been
constrained to single partition setups.

To illustrate this we again make use of `ksql-datagen`, this time
populating six partitions of an `orders` topic. Here is a sample,
taken from one of those partitions:

``` json
{"ordertime":1504776415695,"orderid":5,"itemid":"Item_366","orderunits":6.6560792199065375,"address":{"city":"City_72","state":"State_76","zipcode":59809}}
{"ordertime":1501571188542,"orderid":11,"itemid":"Item_401","orderunits":8.026309881360708,"address":{"city":"City_71","state":"State_52","zipcode":29263}}
{"ordertime":1497015695172,"orderid":15,"itemid":"Item_276","orderunits":2.6758133491091525,"address":{"city":"City_11","state":"State_36","zipcode":67176}}
{"ordertime":1506027510309,"orderid":17,"itemid":"Item_572","orderunits":3.140245079869947,"address":{"city":"City_66","state":"State_78","zipcode":97471}}
{"ordertime":1494713135481,"orderid":21,"itemid":"Item_517","orderunits":5.044951208919925,"address":{"city":"City_41","state":"State_61","zipcode":44006}}
```

Of particular interest is the `ordertime` column, which this time
around does *not* correspond to the ingestion order on this partition
(you can see `1497015695172` appearing after `1501571188542`). So even
consuming from a single partition we will not observe a consistent
timeline. Consuming from all partitions will leave us with a
near-arbitrary interleaving of events. The following `kplex` job
reconstructs a consistent global timeline.

``` json
{
  "workers": 6,
  "kafka": { "broker": "localhost:9092" },
  "topics": {
    "orders": {
      "max_delay_ms": 30000,
      "polling_interval": {"secs": 1, "nanos": 0}
    }
  },
  "derive": {
    "orders": {
      "timestamp": {"Pointer": "/ordertime"},
      "order": "TimeOrder",
      "output": "Stdout"
    }
  }
}
```

Notice that this job is similar to the repartitioning job above. What
we have done now is first to leave out the `key` declaration in the
derivation of `orders`. It is redundant, as we don't want to change
the partitioning key in this scenario. Second, we have changed the
`output` declaration from `VirtualPartitions` to `Stdout`, as now with
only a single virtual partition we do not need to deal with multiple
pipes, and can produce to standard out directly. Lastly, we are using
six `kplex` threads now, in order to be able to consume all input
partitions in parallel[^threads].

[^partition-limit]: Previously in the hundreds, nowadays [in the thousands](https://www.confluent.io/blog/apache-kafka-supports-200k-partitions-per-cluster).
[^partition-performance]: [Jun Rao, "How to choose the number of topics/partitions in a Kafka cluster?"](https://www.confluent.io/blog/how-choose-number-topics-partitions-kafka-cluster)
[^newrelic]: [Amy Boyle, "Effective Strategies for Kafka Topic Partitioning"](https://blog.newrelic.com/engineering/effective-strategies-kafka-topic-partitioning/)
[^datagen]: [ksql-datagen](https://docs.confluent.io/current/ksql/docs/tutorials/generate-custom-test-data.html)
[^3df]: There is more to it, but [we are working on that as well](https://github.com/comnik/declarative-dataflow).
[^threads]: At some point we will start hitting diminishing returns here and should stick to a number of worker threads that is proportional to the number of physical cores available.
