---
layout: post
title: Improving performance of lockless data structures
draft: true
---

As multi-core computer systems grow in number of processors and processor cores one of the largest challenges in software is to scale the performance of data structures to properly utilize the potential concurrent performance that modern hardware offers. To tap into this potential performance data structures need to be able to execute multiple operations at the same time with minimal synchronization between threads or else the hot path of the software risks stalling processor usage and dropping operations per second throughput. Below is probably the most common solution you'll find.

```
class Foo {
private:
    std::shared_mutex rw_lock;    // Read-write lock.
    std::vector<int> data;        // Some data to protect.

public:
    void broadcast(int index) {
        // Shared lock.
        std::shared_lock<std::shared_mutex> shared(rw_lock);
        return data[index];
    }

    void Change(int change) {
        // Exclusive lock
        std::unique_lock<std::shared_mutex> exclusive(rw_lock);
        data.push_back(change);
    }

```
_Figure 1. Example C++ read-write lock._

Traditional locking strategies including read-write locking doesn't scale beyond a couple threads. It's hard to find servers that have less than 4 cores and most data structures scale poorly beyond 4.

<a target="_blank" href="http://moodycamel.com/blog/2014/a-fast-general-purpose-lock-free-queue-for-c++"><img src="http://moodycamel.com/blog/2014/heavy32.png"/></a>
_Figure 2. <a target="_blank" href="http://moodycamel.com/blog/2014/a-fast-general-purpose-lock-free-queue-for-c++">A fast general purpose lock-free queue for C++</a> by Cameron._

Even using modern lock-free data structures performance starts to show signs of degrading significantly after 4 threads. `Figure 2` shows less than 1 million operations/second at 20 threads. Even the Intel(R) Threading Building Blocks library optimized for Intel(R) processors written by Intel(R) themselves suffer from scalability degradation.

In a [previous post](http://simongui.github.io/2016/12/30/octane-json-benchmarks.html) I published benchmark results showing my [octane](https://github.com/simongui/octane) project reaching `18.4 million` plaintext requests/second and `15.6 million` JSON requests/second. Octane is able to achieve this kind of throughput by reducing inter-thread synchronization so that each thread runs independently. Since main system memory (RAM) is a shared resource across all CPU cores one way octane does this is by reducing calls to `malloc()` that may under the hood trip on lock contention during allocating, resizing or freeing of memory. The less memory octane allocates and the more it reuses the less threads will stall each other. This is extremely key at this high of throughput.

<a target="_blank" href="/images/2016-12-30-throughput.png"><img src="/images/2016-12-30-throughput.png"/></a>
_Figure 3. Octane benchmark results on a 20 hardware thread Intel(R) Xeon(R) CPU E5-2660 v3 at 2.60GHz system._

Using efficient data structures will be extremely important to keep the performance shown in `Figure 3` as I add more features to octane.

# Multiversion concurrency control
Whether it's an on-disk or in-memory data structure the same challenges exist. Some storage libraries and database engines implement handling multiple concurrent transactions using a technique called multiversion concurrency control (MVCC). Each transaction gets a point in time consistent view of the database from the time the transaction began. The database internally manages multiple versions of the database to support this ability and cleans them up when there are no longer running transactions associated with a version. Some examples of databases or storage libraries with MVCC implementations are Berkeley DB, Couchbase, CouchDB, HBase, LMDB, MariaDB (XtraDB), MemSQL, Microsoft SQL Server, MongoDB (WiredTiger), MySQL (InnoDB), Oracle v4+, PostgreSQL and SAP HANA to name a few.

Instead of locking the access of the data itself you can give copies of the pointers to the data to each reader or writer. Readers can proceed to read without having to acquire a lock. The pointer pointing to the master record only needs to be updated by swapping it's pointer to a new pointer that has all the changes already written. This reduces a lot of inter-thread synchronization.

# Reclamation
http://preshing.com/20160726/using-quiescent-states-to-reclaim-memory/
http://www.rdrop.com/users/paulmck/RCU/hart_ipdps06.pdf
https://github.com/datacratic/soa/blob/master/gc/gc_lock.h
https://github.com/khizmax/libcds
https://github.com/rmind/libqsbr
