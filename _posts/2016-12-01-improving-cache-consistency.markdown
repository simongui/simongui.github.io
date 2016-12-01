---
layout: post
title: Improving cache consistency
draft: true
---

A typically web application introduces an in-memory cache like `memcache` or `redis` to reduce load on the primary database for reads requesting hot data. The most primitive design looks something like `Figure 1`.

```
+------------+        +----------------+
| web server +-------->     cache      |
+------------+        | memcache/redis |
                      +----------------+
```
_Figure 1_

Unfortunately this design is really common despite the many issues it introduces. I've seen some organizations with large scale applications still using this design and they maintain a bunch of hacks to overcome these issues which increases the systems operational complexity and sometimes surfaces as inconsistent data to end users.

## Issue 1. Pool of connections to the cache services per web server instance
In a large application sometimes thousands of web server instances (especially in slower languages like Ruby) are hosting the web application. Each one has to maintain connections to the infrastructure the web application code communicates with directly. This can include primary databases like MSSQL, MySQL, Oracle, Postgres and cache services like Memcache or Redis. Each web server instance would for example have a pool of connections for each database or cache service instance it communicates with.

This can be a strain on resources both on the web server but more importantly the database or cache service. This is why I included a `16,384` connection benchmark in my [benchmarks of Redis server libraries for Go](http://simongui.github.io/2016/10/24/benchmarking-go-redis-server-libraries.html) to see how they scaled. It's not uncommon to see `10,000` or `20,000` connections to a Memcache or Redis server in a large system designed like `Figure 1`.

## Issue 2. Many web app requests have to execute cache set operations
Similar to how a HTTP request may issue multiple SQL INSERT or UPDATE statements, multiple SET operations may be issued against the cache service. Even though these can be done asynchronously they still consume resources on the web server and it would be great if the web servers only had to be concerned with updating the primary database.

## Issue 3. No fault tolerance. Data loss if cache set operations fail
The typical sequence of operations of how `Figure 1` in a web application would be designed would be as follows.

1. Update the primary database (MSSQL, MySQL, Oracle, Postgres, etc).
1. If the transaction fails return a HTTP error.
1. If the transaction succeeds send SET operations to the cache server(s) (memcache, redis, etc).

Any SET operation could fail even after retrying which puts the cache service(s) inconsistent with the primary database which could result in users seeing incorrect information. Even worse depending how the application is designed you could experience _partial failures_ which results in users seeing partially correct and partially incorrect information after a change and a cache hit.

Some cache service protocols support sending multiple SET operations in one command but some do not. Not all web applications are smart enough to group SET operations that happen in different areas of the code into a single command either. If this is the case you could have _partial failures_ where some of the SET operations succeeded and some failed.

Outside of retrying there's not much the web application can do to eventually correct the missing cache SET operations. It has to retry and give up at some point. The cache will be serving cache hits that are inconsistent with the primary database until the cache key(s) invalidate via a TTL or some other process.

### Messaging middleware
Sometimes this gets solved by messaging middleware like Kafka where the web applications push SET operations into Kafka and consumers pull changes from Kafka and execute the SET operations on the cache service(s). This greatly increases the cache consistency and allows the caches survive failures and catch up after short or long failures.

This introduces latency in the system. Changes may not be seen right away to users. Some web applications solve this by doing sticky sessions and caching in-memory in the web application to hide that data is inconsistent. Stale results are still possible if the web server fails and requests route to a different web server instance. This introduces complexity in the request routing tier of the system.

This introduces a lot of operational complexity such as.

1. Deploy and operate a high throughput messaging system like Kafka with multiple brokers to survive broker failures.
1. Deploy and operate multiple consumer processes that consume messages in Kafka and execute SET operations to the cache service(s) to survive consumer failures.



## Issue 4. No sequential consistency with the primary database
Leslie Lamport describes [sequential consistency](http://research.microsoft.com/en-us/um/people/lamport/pubs/lamport-how-to-make.pdf) as follows.

> The result of any execution is the same as if the operations of all the processors were executed in some sequential order, and the operations of each individual processor appear in this sequence in the order specified by its program.

`Issue 3` describes the possibility of complete and partial failures and explains how a user could see partially up-to-date and partially stale results. Diving deeper operations could fail before following operations succeed. The order of visible changes could be out-of-order. Some web applications may be more sensitive to this kind of inconsistency than others. Some applications may require strict partial order. Even if order isn't critical providing sequential consistency is a better experience for users and less confusing.
