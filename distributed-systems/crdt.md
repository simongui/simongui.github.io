---
layout: default
title: Convergent Replicated Data Types
---

# {{ page.title }}

CRDT's, convergent (or commutative) replicated data types (also known as conflict-free replicated data types) is a category of research around distributed data structures that can survive network partitions while offering high availability by not requiring coordination or strongly consistent operations. CRDT's track changes using causal history and metadata. Reads and writes can happen on any node and as the system heals the values converge to the correct values.

CRDT's can use `vector clocks` or `dotted version vectors` for tracking object update causality using logical time rather than chronological time (timestamps which are subject to clock-skew and increased data loss). They use these for causality-based conflict resolution when you merge updates existing on multiple replica nodes (which can happen during a partition).

There are 2 types of CRDT's. State-based (CvRDT) and operation-based (CmRDT).

> State-based mechanisms (CvRDTs) are simple to reason about, since all necessary information is captured by the state. They require weak channel assumptions, allowing for unknown numbers of replicas. However, sending state may be inefficient for large objects; this can be tackled by shipping deltas, but this requires mechanisms similar to the op-based approach. Historically, the state-based approach is used in file systems such as NFS, AFS, Coda, and in key-value stores such as Dynamo and Riak.
> Specifying operation-based objects (CmRDTs) can be more complex since it requires reasoning about history, but conversely they have greater expressive power. The payload can be simpler since some state is effectively offloaded to the channel. Op-based replication is more demanding of the channel, since it requires reliable broadcast, which in general requires tracking group membership. Historically, op-based approaches have been used in cooperative systems such as Bayou, Rover, IceCube, Telex. 

-- <cite>[7] A comprehensive study of Convergent and Commutative Replicated Data Types</cite>

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

## Implementations
Carlos Baquero has a C++ implementation of these [on GitHub](https://github.com/CBaquero/delta-enabled-crdts){:target="_blank"}.

##### Distributed counters
Cassandra and Riak supports distributed counters in the form of a `PNCounter` but Cassandra differs from the actual `PNCounter` material. `PNCounters` store casual history on each node in the form of `NodeId:IncrementValue` and the sum of the history from all the nodes results in the final counter value. Cassandra deviates from the `PNCounter` design and each node stores the versions of count values and not the actual mutation increments in the form of `Version:Count` that you see in the CRDT research papers. Combining all the latest versions results in the counters final value.

##### Sets and maps
Riak supports sets and maps as `ORSWOT`.

## References
1. [Logical clocks](http://research.microsoft.com/en-us/um/people/lamport/pubs/time-clocks.pdf){:target="_blank"}   
2. [Vector clocks](http://en.wikipedia.org/wiki/Vector_clock){:target="_blank"}    
3. [Dotted version vectors](http://arxiv.org/pdf/1011.5808v1.pdf){:target="_blank"}    
4. [Summary of different types of CRDT's and how they work](https://github.com/pfraze/crdt_notes){:target="_blank"}    
5. [Designing a commutative replicated data type](http://arxiv.org/pdf/0710.1784v1.pdf){:target="_blank"}    
6. [CRDTs: Consistency without concurrency control](http://arxiv.org/pdf/0907.0929v1.pdf){:target="_blank"}    
7. [A comprehensive study of Convergent and Commutative Replicated Data Types](https://hal.inria.fr/inria-00555588/document){:target="_blank"}    
8. [An Optimized Conflict-free Replicated Set](http://arxiv.org/pdf/1210.3368v1.pdf){:target="_blank"}    
9. [Scalable Eventually Consistent Counters over Unreliable Networks](http://arxiv.org/pdf/1307.3207v1.pdf){:target="_blank"}    
10. [Efficient State-based CRDTs by Delta-Mutation](http://arxiv.org/pdf/1410.2803.pdf){:target="_blank"}    
11. [Making Operation-based CRDTs Operation-based](http://haslab.uminho.pt/ashoker/files/opbaseddais14.pdf){:target="_blank"}
12. [Implementations of CRDT's by Carlos Baquero](https://github.com/CBaquero/delta-enabled-crdts){:target="_blank"}    
13. [Cassandra 2.1: Better Implementation of Counters](http://www.datastax.com/dev/blog/whats-new-in-cassandra-2-1-a-better-implementation-of-counters){:target="_blank"}    