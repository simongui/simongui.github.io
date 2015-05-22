---
layout: default
title: Convergent Replicated Data Types
---

# {{ page.title }}

CRDT's, convergent (or commutative) replicated data types (also known as conflict-free replicated data types) is a category of research around distributed data structures that can survive network partitions while offering high availability by not requiring coordination or strongly consistent operations. CRDT's track changes using metadata and as the system heals values converge to the correct values.

CRDT's can use [vector clocks](http://en.wikipedia.org/wiki/Vector_clock){:target="_blank"} or [dotted version vectors](http://arxiv.org/pdf/1011.5808v1.pdf){:target="_blank"} for tracking object update causality using [logical time](http://research.microsoft.com/en-us/um/people/lamport/pubs/time-clocks.pdf){:target="_blank"} rather than chronological time (timestamps which are subject to clock-skew and increased data loss). They use these for causality-based conflict resolution when you merge updates existing on multiple replica nodes (which can happen during a partition).

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

#### Implementations
Carlos Baquero has a C++ implementation of these [on GitHub](https://github.com/CBaquero/delta-enabled-crdts){:target="_blank"}

Cassandra and Riak (Dynamo implementations) both have PNCounter support. PNCounters allow writes and reads to occur at all times no matter how many nodes have failed or which side of a partition you are on. When the system heals the changes converge by merging all the changes together into a correct final value.

#### References
[Summary of different types of CRDT's and how they work](https://github.com/pfraze/crdt_notes){:target="_blank"}    
[Designing a commutative replicated data type](http://arxiv.org/pdf/0710.1784v1.pdf){:target="_blank"}    
[CRDTs: Consistency without concurrency control](http://arxiv.org/pdf/0907.0929v1.pdf){:target="_blank"}    
[A comprehensive study of Convergent and Commutative Replicated Data Types](https://hal.inria.fr/inria-00555588/document){:target="_blank"}    
[An Optimized Conflict-free Replicated Set](http://arxiv.org/pdf/1210.3368v1.pdf){:target="_blank"}    
[Scalable Eventually Consistent Counters over Unreliable Networks](http://arxiv.org/pdf/1307.3207v1.pdf){:target="_blank"}    
[Efficient State-based CRDTs by Delta-Mutation](http://arxiv.org/pdf/1410.2803.pdf){:target="_blank"}    
[Making Operation-based CRDTs Operation-based](http://haslab.uminho.pt/ashoker/files/opbaseddais14.pdf){:target="_blank"}