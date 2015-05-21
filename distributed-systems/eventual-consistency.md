---
layout: default
title: Eventual Consistency
---

# {{ page.title }}

There has been a lot of research and innovations around eventually consistent data stores, data structures and protocols in the last 10 years. This material focuses on improving availability at the cost of consistency. This should not be confused with correctness because there has been a lot of research around convergence.

[Amazon Dynamo: Amazon’s Highly Available Key-value Store](http://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf){:target="_blank"}

Amazon Dynamo discusses an approach to building a key-value store that offers high availability and makes explicit trade-offs (in CAP terminology AP biased trade-offs not CP) to increase the availability of the system while offering mechanisms for convergence. Dynamo is a DHT (distributed hashtable) that handles sharding, replicating and repairing the keyspace. Dynamo does this by combining epidemic protocols like gossip with transational concepts of quorums, sloppy quorums, anti-entropy using read repairs, and merkle tree's.

Implementations of Amazon Dynamo include Apache Cassandra (Google BigTable data model with Dynamo inspired distribution, Riak, Apache Voldemort). Some of the Dynamo implementations deviate from the original Amazon paper. Some of these implementations have added the optional ability to execute strongly consistent operations such as Cassandra and Riak's paxos inspired transactions.

Dynamo can scale to cluster sizes of thousands of nodes that span multiple data centers.

## CRDT: Convergent Replicated Data Types

Convergent (or commutative) replicated data types (also known as conflict-free replicated data types) is a category of research around distributed data structures that can survive network partitions while offering high availability by not requiring coordination or strongly consistent operations. CRDT's track changes using metadata and as the system heals values converge to the correct values.

Some data types that have been designed to be convergent are counts, sets, 

[Designing a commutative replicated data type](http://arxiv.org/pdf/0710.1784v1.pdf){:target="_blank"}