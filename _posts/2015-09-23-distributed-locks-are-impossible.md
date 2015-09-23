---
layout: post
title: Distributed locks are impossible
date: 2015-09-23 00:00:00
categories: CAP Distributed-Systems Distributed-Locks Locks
---

Distributed locks are implemented for all kinds of uses in distributed systems. Some examples include synchronizing workers, master/slave failovers, record level protection or even to implement distributed transactions. Distributed locks are used for similar purposes as lock primitives you find in many programming languages. You can have mutually exclusive locks, read/writer locks and several other kinds of distributed locks. 

Typically I would rather solve something without a distributed lock because they are very hard to make reliable due to the difficult edge cases. When they fail they cause very weird and unexpected results. It's very difficult to detect and understand what happened unless your system is excellent at logging its operations and you can trace the causality. Most of the time people find out from customers calling regarding an anomaly that you cannot explain with evidence.

When I said distributed locks are very hard to make reliable I actually meant *__impossible__* to make reliable. A lock primitive in a programming language uses hardware support for atomic operations like `compare-and-swap` that exists on a computer processor to ensure safety and eliminate race conditions. Unlike an atomic operation on a computer processor, it is far more common in a distributed system to lose a lock that you thought you acquired and own but don't. Because of this distributed locks are fundamentally impossible.

# Asynchronous networks and non-transactional services
Distributed systems typically communicate over asynchronous networks using protocols like TCP or UDP. Asynchronous networks don't offer the same guarantees that atomic operations on a computer processor offer. If `Process1` acquires a lock from a lock service and proceeds to issue requests to other non-transactional services like Redis, memcached or a REST service it's impossible to guarantee that those requests won't be duplicated by `Process2` when the `Process1` lock expires before it has received responses due to network congestion. 

Attempting to build distributed transactions across non-transactional service instances is impossible because of the inability to open, commit or abort a transaction across multiple parties using the lock service as the coordinator.   

If you're building a billing system for example you cannot guarantee you won't bill someone more than once. You cannot offer `exactly-once` guarantees under all posible failure cases.

# Timeouts, pauses and the critical section
Distributed systems experience varying degrees of latency and typically implement timeouts. A distributed lock usually has an expiration timeout defined to prevent deadlocks in the case the lock owner dies or gets partitioned. Any timing skew like large pauses in the network or the threads themselves can cause failures to the safety of the locking algorithm. JVM/Go/CLR/Ruby/any garbage collector could pause all threads and process a stop-the-world GC pause. This pause can take an unknown length of time. If you use a programming language or runtime without a garbage collector you're not free from this problem either. The paper [SuperMalloc: A Super Fast Multithreaded malloc for 64-bit Machines](http://delivery.acm.org/10.1145/2760000/2754178/p41-kuszmaul.pdf?ip=216.191.105.146&id=2754178&acc=OPENTOC&key=4D4702B0C3E38B35%2E4D4702B0C3E38B35%2E4D4702B0C3E38B35%2E9F04A3A78F7D3B8D&CFID=715610675&CFTOKEN=45790005&__acm__=1442957843_7706f3a0f8696873b3fe7493296c835d) mentions the following.
> I have seen JEmalloc causing a 3 s pause about once per day on a database server

The following sequence can occur:

```
Process1                      Process2                   Lock service
   +                             +                             +     
   | Acquiring lock request      |                             |     
   | +-------------------------------------------------------> |     
   |                             |                             |     
   |                             |      Lock acquired response |     
   | <-------------------------------------------------------+ |     
   |                             |                             |     
   |                             | Acquiring lock request      |     
   |                             | +-------------------------> |     
   |                             |                             |     
   | Critical section executing  |                             |     
   | +----+                      |                             |     
   |      |                      |                             |     
   |      |                      |                             |     
   |      |                      |                             |     
   |      |                      |                             |     
   |      |                      |                             |     
   | +----+                      |                             |     
   |                             |                             |     
   | GC/malloc pause             |                             |     
   | +----+                      |                             |     
   |      |                      |                             |     
   |      |                      |                             |     
   |      |                      |                             |     
   |      |                      |       Process1 lock expires |     
   |      |                      |                             |     
   |      |                      |      Lock acquired response |     
   |      |                      | <-------------------------+ |     
   |      |                      |                             |     
   |      |                      | Critical section executes   |     
   |      |                      | +----+                      |     
   | +----+                      |      |                      |     
   |                             |      |                      |     
   | Critical section continues  |      |                      |     
   | +----+                      |      |                      |     
   |      |                      |      |                      |     
   |      |                      |      |                      |     
   |      |                      |      |                      |     
   |      |                      |      |                      |     
   |      |                      |      |                      |     
   | +----+                      | +----+                      |     
   |                             |                             |     
   | Detected lock lost          |                             |     
   | Fail stop                   |                             |     
   +                             +                             +     

```


