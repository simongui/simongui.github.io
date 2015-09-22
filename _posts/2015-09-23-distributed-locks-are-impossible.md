---
layout: post
title: Distributed locks are impossible
date: 2015-09-23 00:00:00
categories: CAP Distributed-Systems Distributed-Locks Locks
---

Distributed locks are implemented for all kinds of uses in distributed systems. Some examples include synchronizing workers, master/slave failovers or record level protection to name a few of them. Distributed locks are used for similar purposes as lock primitives you find in many programming languages. You can have mutually exclusive locks, read/writer locks and several other kinds of distributed locks. 

Typically I would rather solve something without a distributed lock because they are very hard to make reliable due to the difficult edge cases. When they fail they cause very weird and unexpected results. It's very difficult to detect and understand what happened unless your system is excellent at logging its operations and you can trace the causality. Most of the time people find out from customers calling regarding an anomaly that you cannot explain with evidence.

When I said distributed locks are very hard to make reliable I actually meant *__impossible__* to make reliable. A lock primitive in a programming language uses hardware support for atomic operations like `compare-and-swap` that exists on a computer processor to ensure safety and eliminate race conditions. Unlike an atomic operation on a computer processor, it is far more common in a distributed system to lose a lock that you thought you acquired and own. Because of this two things contribute to locks in a distributed system being fundamentally impossible.

# Asynchronous networks
Distributed systems typically communicate over asynchronous networks using protocols like TCP or UDP. Asynchronous  networks don't offer the same safety guarantees that atomic operations on a computer processor offer. If `Actor1` acquires the distributed lock, executes 10 lines of code there's no guarantee that `Actor1` still owns the lock. TCP timeouts, session timeouts on the lock service and even packet send/receive latency create a window of time where you do not know if you still own a lock previously acquired. What do you do if you cannot talk to the lock service? Do you assume you still own the lock? Do you fail stop? How do you fail stop? Do you check the lock every line of code you execute in the critical section? Do you implement client-side timeouts? (clock skew and clock sync is now a new problem for you). 

Unfortunately you can't solve these problems in a perfectly safe way.

# Timeouts, pauses and the critical section
Distributed systems include various latency and timeouts. How do you plan to protect the critical section from executing when an acquired lock gets removed at any point of the critical section? Are you going to implement a check between every line of code? Even if you did that you could issue a query to an RDBMS over the network (remember it's asynchronous) and you could lose the lock while the TCP packet is travelling over the network and then executed by the RDBMS. You can't prevent a duplicate request.

You can't guarantee lock ownership between the time you acquired the lock and the very next line of code because a distributed lock will expire using a TTL (time-to-live) to prevent deadlocks. Because this timeout exists you cannot guarantee anything.

A JVM/Go/CLR/Ruby/any garbage collector could pause all threads and process a stop-the-world GC pause. This pause can take who knows how long. The following sequence could happen:

1. Your program could acquire the distributed lock.
2. The GC pauses all threads including the one acquiring the distributed lock and pause for an unknown period of time.
3. GC resumes all threads.
4. Lock server expires the lock because the TTL expired or the TCP connection died. This process is no longer the lock owner.
4. Your program executes the next line of code while it is not the owner of the lock.

If you use a programming language or runtime without a garbage collector you're not free from this problem either. The paper [SuperMalloc: A Super Fast Multithreaded malloc for 64-bit Machines](http://delivery.acm.org/10.1145/2760000/2754178/p41-kuszmaul.pdf?ip=216.191.105.146&id=2754178&acc=OPENTOC&key=4D4702B0C3E38B35%2E4D4702B0C3E38B35%2E4D4702B0C3E38B35%2E9F04A3A78F7D3B8D&CFID=715610675&CFTOKEN=45790005&__acm__=1442957843_7706f3a0f8696873b3fe7493296c835d) says the following.
> I have seen JEmalloc causing a 3 s pause about once per day on a database server
