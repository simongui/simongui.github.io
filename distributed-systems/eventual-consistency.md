---
layout: default
title: Eventual Consistency
---

# {{ page.title }}

Eventual consistency is a liveness property. Eventually consistent trade-offs are chosen to offer higher availability and/or performance (in CAP terminology AP biased trade-offs not CP) by sacrificing consistency guarantees. Strong consistency (linearizability, serializability, sequential, etc) can require coordination which reduces availability because it is not tolerant to failures and can reject operations for periods of time. To achieve high availability some systems allow writes and reads to happen on any side of a partition. The challenge is what to do with conflicting changes when the system heals.

## Casual history
Convergence requires metadata. One example is storing casual history with a `vector clock` or a `dotted version vector` `logical clock`. This comes at a cost. Some systems choose to provide weaker correctness and avoid recording and merging casual history by reducing metadata and providing LWW (last write wins) guarantees using timestamps.

There has been a lot of research and innovations around causality and how to make storing the casual history more efficient and how to prune history that is unneccessary after convergence using distributed garbage collection.

<!--
## Anti-entropy
TODO...
-->

## References
[Amazon Dynamo](dynamo.html)    
[Convergent replicated data types](crdt.html)    
[Logical clocks](http://research.microsoft.com/en-us/um/people/lamport/pubs/time-clocks.pdf){:target="_blank"}   
[Vector clocks](http://en.wikipedia.org/wiki/Vector_clock){:target="_blank"}    
[Dotted version vectors](http://arxiv.org/pdf/1011.5808v1.pdf){:target="_blank"}    
[Distributed garbage collection](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.30.7337&rep=rep1&type=pdf){:target="_blank"}     
