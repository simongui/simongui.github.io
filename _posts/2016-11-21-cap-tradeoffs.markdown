---
layout: post
title: Thoughts on CAP tradeoffs
draft: true
---

Since the introduction of the [CAP theorem]() by Eric Brewer in 1999 and later [formally proven](http://groups.csail.mit.edu/tds/papers/Gilbert/Brewer2.pdf) by Seth Gilbert and Nancy Lynch of MIT in 2002 the CAP theorem has been at times a useful way to define and discuss the **consistency**, **availability** and **partition tolerant** properties distributed systems can choose from.

Over time we learn more about the subtleties that exist in the relationship between these properties. For example, C, A and P are often discussed equally but [we've learned](https://www.infoq.com/articles/cap-twelve-years-later-how-the-rules-have-changed) that the statement __you can pick any 2 of the 3__ is incorrect. This is because **partition tolerance** isn't optional. In reality any 2 computer systems can suffer a network partition which makes the ability to choose to be partition intolerant (**CA**) impossible. We are left with the only choice between **CP** or **AP**. Systems like Zookeeper or Etcd choose consistency over availability. Systems like Amazon Dynamo (Cassandra and Riak) choose availability over consistency.

# The properties are a spectrum
17 years later (at the time of this post) system designs primarily treat the CP and AP properties in a way that is black and white. The CAP theorem discusses the tradeoffs available during a network partition but it doesn't indicate you must make these tradeoffs at all times. Existing AP systems like [Amazon Dynamo](http://s3.amazonaws.com/AllThingsDistributed/sosp/amazon-dynamo-sosp2007.pdf) clones Cassandra and Riak handle operations exactly the same way whether there is a network partition or not. This means even when the cluster is healthy and performing optimally there's still potential for a user to observe inconsistency.

A system could choose to offer strongly consistent operations when healthy and eventually consistent operations during a network partition to increase availability. As always there is a tradeoff to make which affects the spectrum of consistency and availability guarantees offered once you do this. If stronger consistency is introduced we are sacrificing some temporary availability during a network partition before the system detects the failure exists. The strongly consistent operations will fail until the system adjusts to using weaker consistency guarantees. Likewise observing inconsistency is more probable as a network partition heals until the system detects the healing and adjusts back to using strongly consistent operations.

# Yahoo PNUTS
[Yahoo PNUTS](https://people.mpi-sws.org/~druschel/courses/ds/papers/cooper-pnuts.pdf) is a research paper published by Yahoo about a geographically distributed data store. PNUT's doesn't necessarily adapt to the health of the system on the fly but it does allow operations to sacrifice consistency for availability if they choose. Users can control whether they require the latest version or can accept a potentially stale version of a record. PNUT's allows the user to sacrifice a little consistency to gain availability but doesn't take it as far as Dynamo's weak eventual consistency.
