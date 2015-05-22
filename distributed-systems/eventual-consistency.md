---
layout: default
title: Eventual Consistency
---

# {{ page.title }}

Eventual consistency is a liveness property. Eventually consistent trade-offs are chosen to offer higher availability and/or performance (in CAP terminology AP biased trade-offs not CP) by sacrificing consistency guarantees. Strong consistency (linearizability, serializability, sequential, etc) can require coordination which reduces availability because it is not tolerant to failures and can reject operations for periods of time. To achieve high availability some systems allow writes and reads to happen on any side of a partition. The challenge is what to do with conflicting changes when the system heals.

Convergence requires metadata like [logical clocks](http://research.microsoft.com/en-us/um/people/lamport/pubs/time-clocks.pdf){:target="_blank"} and storing casual history. This comes at a cost. Some systems choose to provide weaker correctness and avoid recording and merging casual history by providing LWW (last write wins) guarantees.

There has been a lot of research and innovations around causality and how to make storing the casual history more efficient and how to prune history that is unneccessary after convergence ([distributed garbage collection](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.30.7337&rep=rep1&type=pdf){:target="_blank"}).

## Amazon Dynamo 

Amazon Dynamo discusses an approach to building a key-value store that offers high availability and makes explicit trade-offs (AP biased trade-offs) to increase the availability of the system while offering mechanisms for convergence. Dynamo is a DHT (distributed hashtable) that handles sharding, replicating and repairing the keyspace. Dynamo does this by combining epidemic protocols like gossip with transational concepts of quorums, sloppy quorums, anti-entropy using read repairs, and merkle tree's.

Implementations of Amazon Dynamo include Apache Cassandra (Google BigTable data model with Dynamo inspired distribution, Riak, Apache Voldemort). Some of the Dynamo implementations deviate from the original Amazon paper. Some of these implementations have added the optional ability to execute strongly consistent operations such as Cassandra and Riak's paxos inspired transactions.

Dynamo can scale to cluster sizes of thousands of nodes that span multiple data centers.

### Implementations
Cassandra    
Riak    
Voldemort    

Comparions between Cassandra and Riak happen quite frequently because they are the 2 most popular Dynamo implementations. Both offer different guarantees. Both also offer strongly consistent operations using Paxos inspired transactions. Cassandra and Riak aren't only eventually consistent data stores anymore. Both Cassandra and Riak offer some CRDT's (convergent replicated data types, more on these below) but their implementations and guarantees differ.

#### CRDT    
Cassandra and Riak supports distributed counters in the form of a PNCounter but Cassandra differs from the actual PNCounter material. PNCounters store casual history on each node in the form of `NodeId:IncrementValue` and the sum of the history from all the nodes results in the final counter value. Cassandra deviates from the PNCounter design and each node stores it's count value versions and not the actual mutation increments in the form of `Version:Count`. Combining all the latest versions results in the counters final value.

#### Cassandra consistency levels    
Cassandra supports what they call consistency levels. The client on a read or write operation can choose what consistency level it wants the operation to require. Some examples of supported consistency levels are `Any`, `One`, `Two`, `Local Quorum`, `Quorum` and `All`. At first glance this appears to be safety guarantees but this is not the case. If the replication factor is 3 and you issue a write with consistency level quorum and 1 write succeeds and 2 writes fail the 1 write that succeeded is not rolled back. All it indicates to the client is the desired durability of 2 (majority) was not reached. The 1 succeeded write will eventually propagate to the other 2 nodes as the system heals using `anti-entropy` (more on anti-entropy below).

### References
[Amazon Dynamo: Amazon’s Highly Available Key-value Store](http://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf){:target="_blank"}

## Anti-entropy

## CRDT: Convergent Replicated Data Types

Convergent (or commutative) replicated data types (also known as conflict-free replicated data types) is a category of research around distributed data structures that can survive network partitions while offering high availability by not requiring coordination or strongly consistent operations. CRDT's track changes using metadata and as the system heals values converge to the correct values.

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