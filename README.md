# Look

**Look: The Generic Distributed Materialized Views Engine.**

_By engineers, for engineers, crowdsourced under the umbrella of the SysDesign Meetup._

## Product Vision

Look is the ultimate high-performance materialized view.

It aggregates data mutations from various sources, applies these mutations to a single large, holistic, mostly-in-memory data blob, and exposes interfaces to run queries against this data, at its most recent snapshot as well as against its past.

Look is designed to handle millions of mutations and read requests per second on a single node. At the same time, Look is distributed-first, durable, and infinitely scalable, and its design emphasises zero downtime in addition to high throughput.

The only requirement for the data sources is to provide their updates in a stream. This stream of updates should have the property of retaining each update message for a short period of time (single-digit seconds, minutes max), so that each update is guaranteed to be delivered to Look. Three most common use cases that immediately fit the bill are the Transactional Outbox RDBMS pattern, Kafka streams, or a Git branch with force-pushes disabled.

When multiple input data sources need to be joined, the user does not have to worry about the relative order of events in them. Specifically, this applies to feeding Look from a Kafka topic with multiple partitions. Look will never violate the order of updates that have been pushed to it via each individual channel, and the data in Look will always remain as consistent as it was at the point of entry. Look can further be used to increase the degree of consistency of the data served by it, by allowing the users to implement custom logic for the respective CQRS queries, so that this logic can validate user-defined invariants before sending the response to the query.

## Tech Philosophy

TL;DR specific to Look:

* Easy to run, test, and use on a single machine, both in dev and in prod setting.
* Interfacing with the constellation of nodes is identical to interfacing with a single node.
* Infinitely scalable means can add nodes on the fly, with no downtime.

For a more complete description, please refer to [this section](https://github.com/SysDesignMeetup/sdm#educational-designs) of SysDesignMeetup's top-level README on our crowdsourced designs.

## Implementation Challenges

There are four pillars to Look.

### Linearizing Vector Clocks

In terms of functional programming, the inputs to Look are monadic transormations. Transactional Outboxes, Kafka consumers, logical replication logs, CDC pipelines and Git commits can all be viewed as such.

The inputs do not have to be synchronized with one another; even Kafka partitions within the same topic are not synchronized. Thus, the spectrum of inputs to a logical instance of Look can be described by a vector clock. Inside Look, data updates are applied in a strictly sequential, totally ordered, linearized way.

Therefore, for each particular incoming event from any of the Logical Transactional Outbox, a unique sequence ID number (the epoch number) is stamped.

Inversely, each epoch number within a logical Look instance corresponds to one atomic update in the lifetime of this Look; most often an update to the data it stores, although system-level updates, such as changes to data schema, its processing logic, or deployment topology, also get their own unique epoch numbers, interleaved with data updates in a totally ordered fashion.

### In-Memory Event Store

Under the hood, Look maintains a complete event log of all the mutations that took place, both to the data and to the logic that handles this data. Thus, it is accurate to say that Look is based on an immutable, append-only event log.

The in-memory storage of Look follows the approach of [persistent data structures](https://en.wikipedia.org/wiki/Persistent_data_structure), the nodes of which are compactified and serialized to disk. The storage keeps the "dirtied" pages as well, on disk for the history, as well as in memory, for a configurable amount of wall time and epoch number; some 15 seconds and a million mutations by default.

To a certain degree, this implies that read queries that refer to a specific epoch number are fully idempotent, as long as they are performed against a replica that is configured to store enough of the past state. It is also possible to spin a dedicated replica that is configured to keep more of historic data in hot storage. The storage layer of Look also performs regular snapshots, so, for reconciliation reasons, it is possible to "roll it back", temporarily and on a dedicated node or set of nodes, to the state of data & code that was actual far away in the past.

Last but not least: adding Look replicas is a straightforward task. For read-only replicas, it is as trivial as spinning up a node and pointing it at the currently up and running Look cluster. For read-write replicas, a.k.a. potential leaders when it comes to the high availability and durability of the distributed system, the node is first added as read-only, and, as it has caught up, the record that registers it as such can be added into the cluster configuration namespace of the immutable append-only log of this Look deployment.

### High-Throughput Distributed Consensus

Extra effort is required to keep the system highly available and consistent in an environment where the nodes and/or the network links between them can fail. The throughput that Load can handle is far beyond what modern-day most mature leader elections solutions such as `etcd` can offer.

The solution is to build a custom high-throughput distributed consensus engine by leveraging a low-throughput one. What is elected, say, once every few seconds, is not the leader for the storage engine, but the leader for the front-facing gateway that is responsible for batching the stream of incoming requests so that the actual mutations take place not more than a ~50 times per second.

Latency-wise, in addition to having to wait for each mutation to be replicated to other nodes, this batching adds ~10ms of latency on average (~20ms at high percentiles) to the time it takes to confirm each mutation. As of 2023, 10ms of extra latency is negligible compared to the throughput that Look's materialized views offer in exchange. 

### JIT, and Code & Schema Safety

Compute-wise, Look is based on JIT-compiled CPU-bounded mutations and getters that support zero-downtime upgrades.

Broadly speaking, there are two scopes of tasks that require custom code: the logic that updates the state of the data in response to events from the inbound streams, and the logic that serves user queries from this data. Speaking in CQRS teams, it is the "C" and the "Q" parts, although Look is not really performing mutations in the first version, it just follows the "logical replication log" of external sources of data.

One of these sources (that is recommended to be a stream of commits of a certain branch of a linked Git repository) contains the versioned log of the logic for the mutations and reads (commands and queries) that supported by Look at a given point in time. The schema, both of the data and of requests & responses, is part of the code that implements this logic.

The code is implemented in a special language, and, for performance reasons and to avoid any downtime, it is JIT-compiled. The important part is that this special language is designed to guarantee a provable upper bound on the runtime of each function. This guarantees the stability of the high throughput that the system can sustain.

In the SaaS setting, Look makes it easy to account which commands and which queries, from which origins, are responsible for how much CPU usage. This allows for fine-grained billing down the road, analogous to "gas fees" in smart-contract-enabled Web3 ledgers.

Logically, the code deals with an infinite (2^64 bytes) set of "persisted RAM". Pragmatically, as this "RAM" is broken down into hierarchical pages for storage purposes, there are two constraints imposed on the code that implements CQRS: The "number of CPU commands" to execute, and the "number of randomly accessed" storage locations. Actions such as "scanning" the "storage RAM" are banned by design in the OLTP setting of Look.

(I have well-developed ideas for the multi-stage process for performing schema evolution by read-through-writing into a new "data structure" on the commands level while simultaneously running a "cleanup job" on a dedicated, one-off spun up "OLAP cluster", but this goes out of scope of the this document.)

Last but not least: The "execution model" for the "programming language" in which these CQRS commands are implemented includes the logic of de-duplicating strings, and overall arbitrarily long blobs of data of arbitrary length. Thus, operations such as finding duplicates or splitting a string, are O(1) CPU-wise. Technically, they truly are O(1) when it comes to executing them in the sequential, linearized setting, as the deduplication happens at the front-facing gateway level, before the command even reaches the Total Order Broadcast / epoch number stamping component.

(Moreover, internally, the very bodies of these CQRS commands also go through this deduplication engine. This not only checks their idempotency tokens, but also ensures that the bulk index stamping operation is as effective as possible, as what ultimately needs to be stamped is a flat, aligned array in memory, with clearly marked placeholders interleaved with unique 64-bit identifiers for the mutations to be applied. Preparing this "interleaved format" is also the job of the front-facing gateway of Look, in addition to participating in semi-frequent top-level leader elections, and in addition to orchestrating the read-only and then potentially mutable storage replicas.)

The above may sound nontrivial, but it is actually straightforward. The reference implementation contains several examples, most importantly:

* The set of strongly consistent Redis commands (everything but its pub-sub, that can lose messages, and the Lua engine),
* The way to define traditional RDBMS schemas, such as Northwind,
* A basic yet powerful implementation of the Task Queue, and
* A full-text search index that supports on-the-fly indexing and querying documents at 1M++ terms per second.

## Background

Obligatory links, included here for the sake of completeness of this V1 proposal.

* The original slides for Current, "The Pyramid", 2015: [in this repo](https://github.com/SysDesignMeetup/look/blob/main/.static/slides-2015.pdf), [on the Web](http://dima.ai/static/current.pdf).
* The "Current 2.0" proposal, 2021: [in this repo](https://github.com/SysDesignMeetup/look/blob/main/.static/doc-2021.md), [on the Web](https://github.com/dkorolev/Current/blob/current20/current20.md).
* The all-encompassing architectural diagram: [in this repo](https://github.com/SysDesignMeetup/look/blob/main/.static/diagram-2022.pdf), [Miro board](https://miro.com/app/board/uXjVOLKwJHU=/).
* The broader idea which was reduced to Look: [in this repo](https://github.com/SysDesignMeetup/look/blob/main/.static/slides-2023.pdf), [Google slides](https://docs.google.com/presentation/d/1fbBD6C4lfvd3A4fNUtT8QxLAZs6njA1VOL1_pdObyIc).
