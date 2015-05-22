---
layout: default
title: Amazon Dynamo
---

# {{ page.title }}

Amazon Dynamo discusses an approach to building a key-value store that offers high availability and makes explicit trade-offs (AP biased trade-offs) to increase the availability of the system while offering mechanisms for convergence. Dynamo is a DHT (distributed hashtable) that handles sharding, replicating and repairing the keyspace. Dynamo does this by combining epidemic protocols like gossip with transational concepts of quorums, sloppy quorums, anti-entropy using read repairs, and merkle tree's.

Implementations of Amazon Dynamo include Apache Cassandra (Google BigTable data model with Dynamo inspired distribution, Riak, Apache Voldemort). Some of the Dynamo implementations deviate from the original Amazon paper. Some of these implementations have added the optional ability to execute strongly consistent operations such as Cassandra and Riak's paxos inspired transactions.

Dynamo can scale to cluster sizes of thousands of nodes that span multiple data centers.

#### Implementations
Cassandra    
Riak    
Voldemort    

Comparions between Cassandra and Riak happen quite frequently because they are the 2 most popular Dynamo implementations. Both offer different guarantees. Both also offer strongly consistent operations using Paxos inspired transactions. Cassandra and Riak aren't only eventually consistent data stores anymore. Both Cassandra and Riak offer some CRDT's (convergent replicated data types, more on these below) but their implementations and guarantees differ.

Riak supports `LWW` (last write wins) and `vector clock` casual history support for conflict resolution. Cassandra only offers `LWW` guarantees.

#### Cassandra consistency levels    
Cassandra supports what they call consistency levels. The client on a read or write operation can choose what consistency level it wants the operation to require. Some examples of supported consistency levels are `Any`, `One`, `Two`, `Local Quorum`, `Quorum` and `All`. At first glance this appears to be safety guarantees but this is not the case. If the replication factor is 3 and you issue a write with consistency level quorum and 1 write succeeds and 2 writes fail the 1 write that succeeded is not rolled back. All it indicates to the client is the desired durability of 2 (majority) was not reached. The 1 succeeded write will eventually propagate to the other 2 nodes as the system heals using `anti-entropy` (more on anti-entropy below).

#### References
[Amazon Dynamo: Amazon’s Highly Available Key-value Store](http://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf){:target="_blank"}