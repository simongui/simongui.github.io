---
layout: default
title: Eventual Consistency
---

# {{ page.title }}

There has been a lot of research and innovations around eventually consistent data stores, data structures and protocols in the last 10 years. This material focuses on improving availability at the cost of consistency. This should not be confused with correctness because there has been a lot of research around convergence.

[Amazon Dynamo: Amazon’s Highly Available Key-value Store](http://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf){:target="_blank"}

Amazon Dynamo discusses an approach to building a key-value store that offers high availability and makes explicit trade-offs (in CAP terminology AP biased trade-offs not CP) to increase the availability of the system while offering mechanisms for convergence. Dynamo does this by combining epidemic protocols like Gossip with transational concepts of quorums, sloppy quorums, anti-entropy approaches using read repairs, and merkle tree's.

Implementations of Amazon Dynamo include Apache Cassandra (Google BigTable data model with Dynamo inspired distribution, Riak, Apache Voldemort) 