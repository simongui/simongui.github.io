---
layout: default
title: Eventual Consistency
---

# {{ page.title }}

There has been a lot of research and innovations around eventually consistent data stores, data structures and protocols in the last 10 years. This material focuses on improving availability at the cost of consistency. This should not be confused with correctness because there has been a lot of research around convergence.

## Amazon Dynamo 

Amazon Dynamo discusses an approach to building a key-value store that offers high availability and makes explicit trade-offs (in CAP terminology AP biased trade-offs not CP) to increase the availability of the system while offering mechanisms for convergence. Dynamo is a DHT (distributed hashtable) that handles sharding, replicating and repairing the keyspace. Dynamo does this by combining epidemic protocols like gossip with transational concepts of quorums, sloppy quorums, anti-entropy using read repairs, and merkle tree's.

Implementations of Amazon Dynamo include Apache Cassandra (Google BigTable data model with Dynamo inspired distribution, Riak, Apache Voldemort). Some of the Dynamo implementations deviate from the original Amazon paper. Some of these implementations have added the optional ability to execute strongly consistent operations such as Cassandra and Riak's paxos inspired transactions.

Dynamo can scale to cluster sizes of thousands of nodes that span multiple data centers.

#### References
[Amazon Dynamo: Amazon’s Highly Available Key-value Store](http://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf){:target="_blank"}

## CRDT: Convergent Replicated Data Types

Convergent (or commutative) replicated data types (also known as conflict-free replicated data types) is a category of research around distributed data structures that can survive network partitions while offering high availability by not requiring coordination or strongly consistent operations. CRDT's track changes using metadata and as the system heals values converge to the correct values.

CRDT's can use vector clocks or dotted version vectors for tracking object update causality using [logical time](http://research.microsoft.com/en-us/um/people/lamport/pubs/time-clocks.pdf){:target="_blank"} rather than chronological time (timestamps which are subject to clock-skew and increased data loss). They use these for causality-based conflict resolution when you merge updates existing on multiple replica nodes (which can happen during a partition).

Some data types that have been designed to be convergent are:    
**GSet:** A grow only    
**2PSet:** A two phase set that supports removing an element for ever    
**GCounter:** A grow only counter    
**PNCounter:** A counter supporting increment and decrement    
**AWORSet:** A add-wins optimized observed-remove set that allows adds and removes    
**RWORSet:** A remove-wins optimized observed-remove set that allows adds and removes    
**MVRegister:** An optimized multi-value register (new unpublished datatype)    
**MaxOrder:** Keeps the maximum value in an ordered payload type    
**RWLWWSet:** Last-writer-wins set with remove wins bias (SoundCloud inspired)    
**LWWReg:** Last-writer-wins register    
**EWFlag:** Flag with enable/disable. Enable wins (Riak Flag inspired)    
**DWFlag:** Flag with enable/disable. Disable wins (Riak Flag inspired)   

Carlos Baquero has an implementation of these [on GitHub](https://github.com/CBaquero/delta-enabled-crdts){:target="_blank"}

#### References
[Designing a commutative replicated data type](http://arxiv.org/pdf/0710.1784v1.pdf){:target="_blank"}    
[CRDTs: Consistency without concurrency control](http://arxiv.org/pdf/0907.0929v1.pdf){:target="_blank"}    
[A comprehensive study of Convergent and Commutative Replicated Data Types](https://hal.inria.fr/inria-00555588/document){:target="_blank"}    
[An Optimized Conflict-free Replicated Set](http://arxiv.org/pdf/1210.3368v1.pdf){:target="_blank"}    
[Scalable Eventually Consistent Counters over Unreliable Networks](http://arxiv.org/pdf/1307.3207v1.pdf){:target="_blank"}    
[Efficient State-based CRDTs by Delta-Mutation](http://arxiv.org/pdf/1410.2803.pdf){:target="_blank"}    
[Making Operation-based CRDTs Operation-based](http://haslab.uminho.pt/ashoker/files/opbaseddais14.pdf){:target="_blank"}