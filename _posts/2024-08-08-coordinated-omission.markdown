---
layout: post
title: Coordinated Omission
draft: true
---

I find measuring system performance whether it's a database, an API or anything else really interesting and fun. Every time I measure something new I feel like I'm always learning about 
how hardware interacts with software architecture and how distributed systems interact with each other. I've measured a lot of things over my career. Here are a few things I'm proud of working on.

- One of the worlds first `1,000 node` Cassandra+Solr clusters a decade ago.
- [The first 1+ million CPU core Kubernetes cluster with 55,000 nodes that I'm aware of](https://vmblog.com/archive/2018/06/28/univa-leverages-aws-to-deploy-more-than-one-million-cores-in-a-single-univa-grid-engine-cluster.aspx).

If there's one thing I've learned when operating systems of this scale it's that you need to become a black belt at finding a needle in a haystack and accurate results are critical.

One of the primary challenges when measuring a system is correctly accounting for [Coordinated Omission](https://redhatperf.github.io/post/coordinated-omission/) which is a subtle but impactful effect
that commonly skews benchmark results without testers noticing. The topic was originally raised by [Gil Tene](https://www.azul.com/leadership/gil-tene/) in his [how not to measure latency](https://image.slidesharecdn.com/untitled-160328112522/75/How-NOT-to-Measure-Latency-14-2048.jpg) 
talk. He works on a proprietary Java runtime and garbage collector known to be one of the fastest and lowest latency runtimes and GC's in the industry. He knows a thing or two about measuring latency.

# What is Coordinated Omission
Coordinated Omission is a side effect where a testing tool is accidentally coordinating with the system being measured causing anomalies in the `request start times` which reports invalid response times. 

Unfortunately most popular testing tools incorrectly implement how to capture the `request start time` which makes these tools untrustable for reporting correct results.
This subtle bug makes the testing tool vulnerable to Coordinated Omission and the target system being measured will affect the testing tool in a way that makes response times look deceptively optimistic. 
This is extremely common in many tools and hides that real response times much worse than reported.

If you're generating traffic but unable to reproduce production like results that match the complaints from your customers, its possible the tool you're using isn't measuring correctly and your results
are being skewed by Coordinated Omission. Your response time percentiles like `p75`, `p90`, `p95` and `p90` will report much better than they actually are. **Sometimes the customers raising the most hell are in that 
1-25% of traffic that's being reported incorrectly.**

This can cause you the following pain.
- Before you release you have a false sense of comfort and experience unforseen issues on launch.
- You may start to not trust the customer complaints because you are unable to reproduce the issue.
- You may have a test workload created correctly to reproduce the issue you're investigating but you don't know it so you keep experimenting wasting time.
- You may measure a bug fix, believe you fixed it for your customers and then they still complain.

![](https://github.com/user-attachments/assets/b39b32db-34ce-4c3b-86ac-5886165047c1)  
_Figure 1. Corrected for Coordinated Omission vs Uncorrected Data._

The code bug that causes this vulnerability are subtle and common but as you can see in _Figure 1_ the results are not subtle at all and have a signifigant impact on the results 
and will influence how you make your decisions.

```mermaid
flowchart LR
    JMeter
    Client_A[Client A]
    Client_B[Client B]
    Thread_1[Thread 1]
    Thread_2[Thread 2]
    Request_1[Request 1
    2 seconds]
    Request_2[Request 2
    1 second]
    Web_API["Web API"]
    JMeter --> Client_A --> Request_1 --> Request_2
```
In many tools the `p50` would be reported as `1.5 seconds` but what actually happened was that `Request 2` had to wait `2 seconds` before it could dispatch. 
In many testing tools the `start time` isn't captured until the request begins but does not account for queueing wait times caused by external forces. The `p50` should be 2.5 seconds``



