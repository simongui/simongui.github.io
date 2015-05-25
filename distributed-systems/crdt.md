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
> 
> -- <cite>[7 section 2.4] A comprehensive study of Convergent and Commutative Replicated Data Types</cite>

Some data types that have been designed to be convergent are:    

**G-Counter: State-based increment-only Counter**

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
> 
> -- <cite>[7 section 3.1.2] A comprehensive study of Convergent and Commutative Replicated Data Types</cite>

**State-based PN Counter**

> It is not straightforward to support decrement with the previous representation, because this operation would violate monotonicity of the semilattice. Furthermore, since merge is a max operation, decrement would have no effect.
> 
> Our solution, PN-Counter basically combines two G-Counters. Its payload consists of two vectors: P to register increments, and N for decrements. Its value is the difference between the two corresponding G-Counters, its partial order is the conjunction of the corresponding partial orders, and merge merges the two vectors.
> 
> -- <cite>[7 section 3.1.3] A comprehensive study of Convergent and Commutative Replicated Data Types</cite>

**LWW-Register: Last-Writer-Wins Register**

> Last-Writer-Wins Register (LWW-Register) creates a total order of assignments by associating a timestamp with each update. Timestamps are assumed unique, totally ordered, and consistent with causal order; i.e., if assignment 1 happened-before assignment 2 , the former’s timestamp is less than the latter’s. This may be implemented as a per-replica counter concatenated with a unique replica identifier, such as its MAC address.
> 
> -- <cite>[7 section 3.2.1] A comprehensive study of Convergent and Commutative Replicated Data Types</cite>

**MV-Register: Multi-Value Register**

> An alternative semantics is to define a LUB operation that merges concurrent assignments, for instance taking their union, as in file systems such as Coda [19] or in Amazon’s shopping cart [10]. Clients can later reduce multiple values to a single one, by a new assignment. Alternatively, in Ficus [34] merge is an application-specific resolver procedure.
> 
> To detect concurrency, a scalar timestamp (as above) is insufficient. Therefore the state- based payload is a set of (X, versionVector) pairs, as shown in Spec. 10, and illustrated in Figure 9 (the op-based specification is left as an exercise to the reader). A value operation returns a copy of the payload. As usual, assign overwrites; to this effect, it computes a version vector that dominates all the previous ones. Operation merge takes the union of every element in each input set that is not dominated by an element in the other input set.
> 
> As noted in the Dynamo article [10], Amazon’s shopping cart presents an anomaly, whereby a removed book may re-appear. This is illustrated in the example of Figure 10. The problem is that, MV-Register does not behave like a set, contrary to what one might expect since its payload is a set. We will present clean specifications of Sets in Section 3.3.
> 
> -- <cite>[7 section 3.2.2] A comprehensive study of Convergent and Commutative Replicated Data Types</cite>

**G-Set: Grow-Only Set**

> The simplest solution is to avoid remove altogether. A Grow-Only Set (G-Set) supports operations add and lookup only. The G-Set is useful as a building block for more complex constructions.
> 
> In both the state- and op-based approaches, the payload is a set. Since add is based on union, and union is commutative, the op-based implementation converges; G-Set is a CmRDT.
> 
> In the state-based approach, add modifies the local state by a union. We define a partial order on some states S and T as S ≤ T ⇔ S ⊆ T and the merge operation as merge(S, T) = S ∪ T. Thus defined, states form a monotonic semilattice and merge is a LUB operation; G-Set is a CvRDT.
> 
> -- <cite>[7 section 3.3.1] A comprehensive study of Convergent and Commutative Replicated Data Types</cite>

**2P-Set: Two-Phase Set**

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
> 
> -- <cite>[7 section 3.3.2] A comprehensive study of Convergent and Commutative Replicated Data Types</cite>

**LWW-element-Set: Last write wins element set**

> An alternative LWW-based approach,6 which we call LWW-element-Set (see Figure 12), attaches a timestamp to each element (rather than to the whole set, as in Figure 8). Con- sider add-set A and remove-set R, each containing (element,timestamp) pairs. To add (resp. remove) an element e, add the pair (e,now()), where now was specified earlier, to A (resp. to R). Merging two replicas takes the union of their add-sets and remove-sets. An element e is in the set if it is in A, and it is not in R with a higher timestamp: lookup(e) = ∃t,∀t′ > t : (e,t) ∈ A ∧ (e,t′) ∈/ R). Since it is based on LWW, this data type is convergent.
> 
> -- <cite>[7 section 3.3.3] A comprehensive study of Convergent and Commutative Replicated Data Types</cite>

**PN-Set**

> Yet another variation is to associate a counter to each element, initially 0. Adding an element increments the associated counter, and removing an element decrements it. The element is considered in the set if its counter is strictly positive. An actual use-case is Logoot-Undo [43], a (totally-ordered) set of elements for text editing.
> 
> However, as noted earlier (Section 3.1.3), a CRDT counter can go positive or negative; adding an element whose counter is already negative has no effect.
>
> Although the semantics are strange, PN-Set converges; thus if Replica 2 concurrent executes add(e) all replicas converge to state {e}.
>
> An alternative construction due to Molli, Weiss and Skaf [private communication] is presented in Specification 14. To avoid the above add anomaly, add increments a negative count of k by |k| + 1; however this presents other anomalies, for instance where remove has no effect.
>
> Both these constructs are CRDTs because they combine two CRDTS, a Set and a Counter.
> 
> -- <cite>[7 section 3.3.4] A comprehensive study of Convergent and Commutative Replicated Data Types</cite>

**OR-Set: Observed-Remove Set**

> The preceding Set constructs have practical applications, but are somewhat counter-intuitive. In 2P-Set (Section 3.3.2), a removed element can never be added again; in LWW-Set (Fig- ure 8) the outcome of concurrent updates depends on opaque details of how timestamps are allocated.
> 
> We present here the Observed-Removed Set (OR-Set), which supports adding and re- moving elements and is easily understandable. The outcome of a sequence of adds and removes depends only on its causal history and conforms to the sequential specification of a set. In the case of concurrent add and remove of the same element, add has precedence (in contrast to 2P-Set).
> 
> The intuition is to tag each added element uniquely, without exposing the unique tags in the interface. When removing an element, all associated unique tags observed at the source replica are removed, and only those.
> 
> Spec. 15 is op-based. The payload consists of a set of pairs (element, unique-identifier). A lookup(e) extracts element e from the pairs. Operation add(e) generates a unique identifier in the source replica, which is then propagated to downstream replicas, which insert the pair into their payload. Two add(e) generate two unique pairs, but lookup masks the duplicates.
> 
> When a client calls remove(e) at some source, the set of unique tags associated with e at the source is recorded. Downstream, all such pairs are removed from the local payload. Thus, when remove(e) happens-after any number of add(e), all duplicate pairs are removed, and the element is not in the set any more, as expected intuitively. When add(e) is concurrent with remove(e), the add takes precedence, as the unique tag generated by add cannot be observed by remove.
> 
> This behaviour is illustrated in Figure 14. The two add(a) operations generate unique tags α and β. The remove(a) called at the top replica translates to removing (a,α) down- stream. The add called at the second replica is concurrent to the remove of the first one, therefore (a,β) remains in the final state.
> 
> OR-Set is a CRDT. Concurrent adds commute since each one is unique. Concurrent removes commute because any common pairs have the same effect, and any disjoint pairs have independent effects. Concurrent add(e) and remove(f) also commute: if e ̸= f they are independent, and if e = f the remove has no effect.
> 
> -- <cite>[7 section 3.3.5] A comprehensive study of Convergent and Commutative Replicated Data Types</cite>

**Add-wins Replicated Sets**

> Considering the case of an add wins semantics we now recall [9] the CRDT design of an Observed Remove Set, or OR-Set, and then introduce an optimized design that preserves the OR-Set behaviour and greatly improves its space complexity.
> 
> -- <cite>[8 section 4] An Optimized Conflict-free Replicated Set</cite>

**Add-only monotonic DAG**

> In general, maintaining a particular shape, such as a tree or a DAG, cannot be done by a CRDT. Such a global invariant cannot be determined locally; maintaining it requires synchronisation.
>
> However, some stronger forms of acyclicity are implied by local properties, for instance a monotonic DAG, in which an edge may be added only if it oriented in the same direction as an existing path. That is, the new edge can only strengthen the partial order defined by the DAG; it follows that the graph remains acyclic.
> 
> Add-only Monotonic DAG is a CRDT, because concurrent addEdge (resp. addBetween) either concern different edges (resp. vertices) in which case they are independent, or the same edge (resp. vertex), in which case the execution is idempotent.
> 
> Generalising monotonic DAG to removals proves problematic. It should be OK to remove an edge (expressed as a precondition on removeEdge) as long as this does not disrupt paths between distinct vertices. Unfortunately, this is not live.
> 
> -- <cite>[7 section 3.4.1] A comprehensive study of Convergent and Commutative Replicated Data Types</cite>

**Add-Remove Partial Order data type**

> The above issues with vertex removal do not occur if we consider a Partial Order data type rather than a DAG. Since a partial order is transitive, implicitly all alternate paths exist; thus the problematic precondition on vertex removal is not necessary. For the representation, we use a minimal DAG and compute transitive relations on the fly (operation before). To ensure transitivity, a removed vertex is retained as a tombstone. Thus, our Spec. uses a 2P-Set for vertices, and a G-Set for edges.
> 
> We manage vertices as a 2P-Set. Concurrent addBetweens are either independent or idempotent. Any dependence between addBetween and remove is resolved by causal delivery. Thus this data type is a CRDT.
> 
> -- <cite>[7 section 3.4.2] A comprehensive study of Convergent and Commutative Replicated Data Types</cite>

**RGA: Replicated Growable Array**

> The Replicated Growing Array (RGA), due to Roh et al. [35] implements a sequence as a linked list (a linear graph), as illustrated in Figure 18. It supports operations addRight(v, a), to add an element containing atom a immediately after element v. An element’s identifier is a timestamp, assumed unique and ordered consistently with causality, i.e., if two calls to now return t and t′, then if the former happened-before the latter, then t < t′ [24]. If a client inserts twice at the same position, as in “addRight(v,a);addRight(v,b)” the latter insert occurs to the left of the former, and has a higher timestamp. Accordingly, two downstream inserts at the same position are ordered in opposite order of their timestamps. As in Add- Remove Partial Order, removing a vertex leaves a tombstone, in order to accommodate a concurrent add operation.
> 
> For example, in Figure 18, timestamps are represented as a pair (local-clock.client-UID). Client 3 added character I at time 30, then R at time 31, to the right of N. Clients 2 and 3 concurrently (at time 40) inserted an L and an apostrophe to the right of the beginning-of- text marker ⊢.
> 
> As noted above, RGA is a CRDT because it is a subclass of Add-Remove Partial Order.
>
> -- <cite>[7 section 3.5.1] A comprehensive study of Convergent and Commutative Replicated Data Types</cite>

**AWORSet: A add-wins optimized observed-remove set that allows adds and removes**

TODO...

**RWORSet: A remove-wins optimized observed-remove set that allows adds and removes**

TODO...

**MaxOrder: Keeps the maximum value in an ordered payload type**

TODO...

**RWLWWSet: Last-writer-wins set with remove wins bias (SoundCloud inspired)**

TODO...

**EWFlag:** Flag with enable/disable. Enable wins (Riak Flag inspired)

TODO...

**DWFlag:** Flag with enable/disable. Disable wins (Riak Flag inspired)

TODO...

## Implementations
Carlos Baquero has some C++ implementation of these [on GitHub](https://github.com/CBaquero/delta-enabled-crdts){:target="_blank"}.

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