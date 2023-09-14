---
layout: post
title: Postgres v16 drastically lowers risk of change data capture and analytic pipelines
draft: false
---
Today on September 14th 2023 [Postgres version 16 was released](https://www.postgresql.org/docs/16/release-16.html) which includes many great improvements to logical replication, joins, parallel execution and new observability features that will help monitor Postgres in ways not possible previously. Many of these improvements are exciting but this post focuses on the changes to logical replication and its impact on data pipelines.

# What is Change Data Capture
Change Data Capture is a method of extracting data from data stores in a streaming fashion. Instead of running bulk extraction queries, the down stream pipelines can stream in near real-time all of the changes directly from the database. Typically this is served by the databases transaction log (WAL). In many databases the transaction log is used for streaming changes to secondary replicas from the primary so it is natural for CDC tools to use the same replication ports when available. In Postgres this is done with a `logical replication slot`. CDC tools will connect like a Postgres replica to a replication port and stream all of the database changes.

![](https://media.striim.com/wp-content/uploads/2021/04/06232959/Option-1.png)
_Figure 1. Example of change data capture (CDC) topology (source: [striim.com](https://www.striim.com/blog/change-data-capture-cdc-what-it-is-and-how-it-works))._

It is fairly common for these CDC events which are typically represented as insert/update/delete events to be published into a message queue such as Kafka, NATS, etc.

# The risk of Change Data Capture
Each database applies how CDC is serviced differently, but in Postgres the transaction log (WAL) is used to serve the CDC events and log position offsets are managed just like any other Postgres replica. This has the following risks.
1. The Postgres primary cannot prune and reclaim diskspace for transaction log entries until all consumers have read those entries. This means once all consumers progress past a certain point the old transaction log entries can be deleted and disk space can be reclaimed.
2. If the CDC pipeline is stalled for any reason and stops consuming events from the Postgres replication slot the disk space will grow until disk space is exhausted.
3. The Postgres primary will go down.

Unfortunately with Postgres v15 and below this is a high risk for some organizations especially since CDC tools don't seem as reliable as a Postgres replicas yet and will risk critical production primary instances. The keys to mitigate this risk are the following.
1. Plan for the worst case scenario. Measure and capacity plan to have enough disk space to troubleshoot any CDC outage you may have so that the Postgres primary can keep accepting writes even while your CDC pipeline is down.
2. You _must_ fix the CDC pipeline before the disk space is empty or else your Postgres primary will go down.
3. Put observability in place to monitor lag between the Postgres primary and your CDC queue and alert if you're running out of disk space to get ahead of the problem with sufficent time to fix your CDC pipeline.

# How Postgres v16 is a major improvement in Change Data Capture production risks
The risks of CDC pipelines on Postgres v15 or below as mentioned earlier is very high due to the fact you can only attach to the primary in production but with v16 the ability to enable logical replication on a secondary has been added! This change may fly under the radar with all the exciting improvements to performance but I cannot state how important this change is to the stability and risk of production systems who want to have near real-time change data capture but are uncomfortable with the risks to the primary.

# References
1. [Postgres Release 16](https://www.postgresql.org/docs/16/release-16.html).
