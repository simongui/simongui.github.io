---
layout: default
title: Convergent Replicated Data Types
---

# {{ page.title }}

CRDT's, convergent (or commutative) replicated data types (also known as conflict-free replicated data types) is a category of research around distributed data structures that can survive network partitions while offering high availability by not requiring coordination or strongly consistent operations. CRDT's track changes using causal history and metadata. Reads and writes can happen on any node and as the system heals the values converge to the correct values.

CRDT's can use `vector clocks` or `dotted version vectors` for tracking object update causality using logical time rather than chronological time (timestamps which are subject to clock-skew and increased data loss). They use these for causality-based conflict resolution when you merge updates existing on multiple replica nodes (which can happen during a partition).

There are 2 types of CRDT's. State-based (CvRDT) and operation-based (CmRDT).

> State-based mechanisms (CvRDTs) are simple to reason about, since all necessary information is captured by the state. They require weak channel assumptions, allowing for unknown numbers of replicas. However, sending state may be inefficient for large objects; this can be tackled by shipping deltas, but this requires mechanisms similar to the op-based approach. Historically, the state-based approach is used in file systems such as NFS, AFS, Coda, and in key-value stores such as Dynamo and Riak.
>
> Specifying operation-based objects (CmRDTs) can be more complex since it requires reasoning about history, but conversely they have greater expressive power. The payload can be simpler since some state is effectively offloaded to the channel. Op-based replication is more demanding of the channel, since it requires reliable broadcast, which in general requires tracking group membership. Historically, op-based approaches have been used in cooperative systems such as Bayou, Rover, IceCube, Telex. 

-- <cite>[7 section 2.4] A comprehensive study of Convergent and Commutative Replicated Data Types</cite>

Some data types that have been designed to be convergent are:    

**State-based increment-only Counter (G-Counter)**

> A state-based counter is not as straightforward as one would expect. To simplify the problem, we start with a Counter that only increments.
>
> Suppose the payload was a single integer and merge computes max. This data type is a CvRDT as its states form a monotonic semilattice. Consider two replicas, with the same initial state of 0; at each one, a client originates increment. They converge to 1 instead of the expected 2.
> 
> Suppose instead the payload is an integer and merge adds the two values. This is not a CvRDT, as merge is not idempotent.
> 
> We propose instead the construct of Specification 6 (inspired by vector clocks). The payload is vector of integers; each source replica is assigned an entry. To increment, add 1 to the entry of the source replica. The value is the sum of all entries. We define the partial order over two states X and Y by X ≤ Y ⇔ ∀i ∈ [0,n−1] : X.P[i] ≤ Y.P[i], where n is the number of replicas. Merge takes the maximum of each entry. This data type is a CvRDT, as its states form a monotonic semilattice, and merge produces the LUB.
> 
> This version makes two important assumptions: the payload does not overflow, and the set of replicas is well-known. Note however that the op-based version implicitly makes the same two assumptions.
> 
> Alternatively, G-Set (described later, Section 3.3.1) can serve as an increment-only counter. G-Set works even when the set of replicas is not known.

-- <cite>[7 section 3.1.2] A comprehensive study of Convergent and Commutative Replicated Data Types</cite>

**State-based PN Counter**

> It is not straightforward to support decrement with the previous representation, because this operation would violate monotonicity of the semilattice. Furthermore, since merge is a max operation, decrement would have no effect.
> 
> Our solution, PN-Counter basically combines two G-Counters. Its payload consists of two vectors: P to register increments, and N for decrements. Its value is the difference between the two corresponding G-Counters, its partial order is the conjunction of the corresponding partial orders, and merge merges the two vectors.

-- <cite>[7 section 3.1.3] A comprehensive study of Convergent and Commutative Replicated Data Types</cite>

**Last-Writer-Wins Register (LWW-Register)**

> Last-Writer-Wins Register (LWW-Register) creates a total order of assignments by associating a timestamp with each update. Timestamps are assumed unique, totally ordered, and consistent with causal order; i.e., if assignment 1 happened-before assignment 2 , the former’s timestamp is less than the latter’s. This may be implemented as a per-replica counter concatenated with a unique replica identifier, such as its MAC address.

-- <cite>[7 section 3.2.1] A comprehensive study of Convergent and Commutative Replicated Data Types</cite>

**Multi-Value Register (MV-Register)**

> An alternative semantics is to define a LUB operation that merges concurrent assignments, for instance taking their union, as in file systems such as Coda [19] or in Amazon’s shopping cart [10]. Clients can later reduce multiple values to a single one, by a new assignment. Alternatively, in Ficus [34] merge is an application-specific resolver procedure.
> 
> To detect concurrency, a scalar timestamp (as above) is insufficient. Therefore the state- based payload is a set of (X, versionVector) pairs, as shown in Spec. 10, and illustrated in Figure 9 (the op-based specification is left as an exercise to the reader). A value operation returns a copy of the payload. As usual, assign overwrites; to this effect, it computes a version vector that dominates all the previous ones. Operation merge takes the union of every element in each input set that is not dominated by an element in the other input set.
> 
> As noted in the Dynamo article [10], Amazon’s shopping cart presents an anomaly, whereby a removed book may re-appear. This is illustrated in the example of Figure 10. The problem is that, MV-Register does not behave like a set, contrary to what one might expect since its payload is a set. We will present clean specifications of Sets in Section 3.3.

-- <cite>[7 section 3.2.2] A comprehensive study of Convergent and Commutative Replicated Data Types</cite>

**Grow-Only Set (G-Set)**

> The simplest solution is to avoid remove altogether. A Grow-Only Set (G-Set) supports operations add and lookup only. The G-Set is useful as a building block for more complex constructions.
> 
> In both the state- and op-based approaches, the payload is a set. Since add is based on union, and union is commutative, the op-based implementation converges; G-Set is a CmRDT.
> 
> In the state-based approach, add modifies the local state by a union. We define a partial order on some states S and T as S ≤ T ⇔ S ⊆ T and the merge operation as merge(S, T) = S ∪ T. Thus defined, states form a monotonic semilattice and merge is a LUB operation; G-Set is a CvRDT.

-- <cite>[7 section 3.3.1] A comprehensive study of Convergent and Commutative Replicated Data Types</cite>

**2P-Set**

> Our second variant is a Set where an element may be added and removed, but never added again thereafter. This Two-Phase Set (2P-Set) is specified in Specification 12 and illustrated in Figure 3. It combines a G-Set for adding with another for removing; the latter is col- loquially known as the tombstone set. To avoid anomalies, removing an element is allowed only if the source observes that the element is in the set.
> 
> State-based 2P-Set The state-based variant is in Specification 12. The payload is com- posed of local set A for adding, and local set R for removing. The lookup operation checks that the element has been added but not yet removed. Adding or removing a same element twice has no effect, nor does adding an element that has already been removed. The merge procedure computes a LUB by taking the union of the individual added- and removed-sets. Therefore, this is indeed a CRDT.
>
> Note that a tombstone is required to ensure that, if a removed element is received by a downstream replica before its added counterpart, the effect of the remove still takes precedence.
>
> Op-based 2P-Set Consider now the op-based variant of 2P-Set. Concurrent adds of the same element commute, as do concurrent removes. Concurrent operations on different elements commute. Operation pairs on the same element add(e)/add(e) and remove(e) ∥d remove(e) commute by definition; and remove(e) can occur only after add(e). It follows that this data type is indeed a CRDT.
> 
> U-Set 2P-Set can be simplified under two standard assumptions, as in Specification 13. If elements are unique, a removed element will never be added again.5 If, furthermore, a downstream precondition ensures that add(e) is delivered before remove(e), there is no need to record removed elements, and the remove-set is redundant. (Causal delivery is sufficient to ensure this precondition.) Spec. 13 captures this data type, which we call U-Set.
> 
> If we assume (as seems to be the practice) that every element in a shopping cart is unique, then U-Set satisfies the intuitive properties requested of a shopping cart, without the Dynamo anomalies described in Section 3.2.2.
>
> U-Set is a CRDT. As every element is assumed unique, adds are independent. A remove operation must be causally after the corresponding add. Accordingly, there can be no concurrent add and remove of the same element.

-- <cite>[7 section 3.3.2] A comprehensive study of Convergent and Commutative Replicated Data Types</cite>







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