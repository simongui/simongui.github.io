---
layout: post
title: CAP Consistency and ACID Isolation
date: 2015-05-15 23:00:00
categories: CAP ACID Distributed-Systems
---

The C (consistency) in CAP and ACID can cause confusion because both share the same word however both use it for different definitions. The C in ACID refers to data integrity guarantees that span the entire database whereas the C in CAP refers to the ordering guarantees of operations on a data item. Gilbert and Lynch formalized CAP defined the C in CAP as a specific form of [Consistency Model](http://simongui.github.io/distributed-systems/consistency-models.html) called Linearizability. 

The consistency (C) guarantees defined in CAP relate more closely to the isolation (I) guarantees defined in ACID. Both define operation order guarantees and how changes in one transaction affect and become visible to concurrent and following transactions. 
