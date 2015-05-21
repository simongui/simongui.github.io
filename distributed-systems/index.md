---
layout: default
title: Distributed Systems
---

# {{ page.title }}

This page is the home page for information on distributed systems and transaction theory because both are fairly inter-related.

## Introduction material
[Fallacies of distributed computing](http://en.wikipedia.org/wiki/Fallacies_of_distributed_computing){:target="_blank"}

[The CAP FAQ](http://henryr.github.io/cap-faq/){:target="_blank"}

[Notes on distributed systems for young bloods](http://www.somethingsimilar.com/2013/01/14/notes-on-distributed-systems-for-young-bloods/){:target="_blank"}

[Byzantine Fault Tolerance](http://the-paper-trail.org/blog/barbara-liskovs-turing-award-and-byzantine-fault-tolerance/){:target="_blank"}

[Time, Clocks, and the Ordering of Events in a Distributed System](http://research.microsoft.com/en-us/um/people/lamport/pubs/time-clocks.pdf){:target="_blank"}

## Categories
[Consistency Models](consistency-models.html)    
Read Uncommitted, Read Committed, Linearizability, Serializability, Sequential, etc.

[Consistency Test Results](consistency-test-results.html)    
Jepsen tests of Mongo, Elastic Search, Etcd, Consul, Postgres, Kafka, Aerospike, etc.

[Consensus Protocols](consensus-protocols.html)    
Paxos, Multi-Paxos, Raft, Corfu, etc.

[Eventual Consistency](eventual-consistency.html)    
Dynamo, CRDT's, etc.

[Epidemic Protocols](epidemic-protocols.html)    
Gossip, Self-Healing Spanning Tree's (PlumTree), etc.

[Post Mortems](post-mortems.html)    
Post mortem (or root cause analysis) write-ups in the industry we can learn from.
