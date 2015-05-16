---
layout: post
title: CAP Consistency and ACID Isolation
date: 2015-05-15 23:00:00
categories: CAP ACID Distributed-Systems
---

The C (consistency) in CAP and ACID can cause confusion because both share the same word however both use it for different definitions. The C in ACID refers to data integrity guarantees that span the entire database whereas the C in CAP refers to the ordering guarantees of operations on a data item. The C in CAP is defined as Linearizability by Gilbert and Lynch who formalized CAP. Linearizability is a specific form of [Consistency Model](http://simongui.github.io/distributed-systems/consistency-models.html).

First let's give a brief summary of what these acryonyms mean.

### ACID

#### Atomicity
- All the changes will happen or none of them will happen.

#### Consistency
- Data integrity is preserved before and after a transaction.
- Constraints, foreign keys, etc.
