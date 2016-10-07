---
layout: post
title: Benchmarking go redis server libraries
draft: true
---

There has been a lot of discussions on Twitter and the Go performance slack channel recently about the performance of Go Redis protocol libraries. These libraries give you the ability to build a service of your own that supports the Redis protocol. There are 2 libraries that seem to be of interest in the community. [Redcon](https://github.com/tidwall/redcon){:target="_blank"} and [Redeo](https://github.com/bsm/redeo){:target="_blank"}.

I needed to support the Redis protocol in a soon to be announced project and I thought it might be useful to others if I published my benchmark findings while evaluating these libraries.

# Hardware setup
- Client and servers are on independent machines.
- Both systems have `20` physical CPU cores.
- Both systems have `128GB` of memory.
- This is _not_ a localhost test. The network between the two machines is `2x bonded 10GbE`.

# Software setup
The Go in-memory Map implementations for Redcon and Redeo are sharded but each shard is protected by a writer lock. I've written a tool called [Tantrum](https://github.com/simongui/tantrum){:target="_blank"} to aid in automating the benchmark runs and visualizing the results. With the help of a script and Tantrum I've benchmarked various configurations of concurrent clients and pipelined requests to see how different workloads affect performance. I've benchmarked the combinations of the following configurations.

**Connections:** `64, 128, 256, 512, 1024, 2048, 4096, 8192, 16384`    
**Pipelined requests:** `1, 4, 8, 16, 32, 64, 128`

It's not uncommon in production environments I've seen to have thousands or tens of thousands of client connections to a single Redis instance.

**Redis:** All disk persistence is turned off. I wrote a version of `redis-benchmark` that supports microsecond resolution instead of millisecond resolution because we were losing a lot of fidelity in some of the results.

# Go garbage collector
The Go garbage collector has received some nice performance improvements as of late but the Go 1.7 GC still struggles with larger heaps. This can surface with one or multiple large Map instances. You can read the [details here](https://github.com/golang/go/issues/15847#issuecomment-247453018){:target="_blank"}. Luckily in `master` there's a fix that reduces GC pauses in these cases by `10x` which can make a `900ms` pause down to `90ms` which is a great improvement. I've decided to benchmark against this fix because this will likely ship in Go version 1.8.

# CPU efficiency
Shown in `Figure 1` Redis used less CPU resources but it's single threaded design limits its ability to fully utilize all the CPU cores.

```
----system---- ----total-cpu-usage---- -dsk/total- -net/total- ---most-expensive---
     time     |usr sys idl wai hiq siq| read  writ| recv  send|  block i/o process
07-10 03:39:01|  4   1  94   0   0   1|   0     0 |  56M 8548k|
07-10 03:39:02|  4   1  94   0   0   1|   0     0 |  56M 8539k|
07-10 03:39:03|  4   1  94   0   0   1|   0     0 |  56M 8553k|
```
_Figure 1: Redis CPU usage during 128 connection / 32 pipelined request benchmark._

Shown in `Figure 2` Redcon and Redeo both utilized multiple CPU cores better than Redis and allow higher throughput per process however not as efficiently as Redis. This means that 1 Redcon or Redeo process can outperform 1 Redis process however if you ran multiple Redis processes you would experience higher throughput than Redcon or Redeo (at the cost of deployment complexity).

This is a Hyperthreaded machine which means `50% (usr + sys)` usage indicates near CPU saturation. This means the lack of free CPU cycles is getting in the way of greater throughput.

```
----system---- ----total-cpu-usage---- -dsk/total- -net/total- ---most-expensive---
     time     |usr sys idl wai hiq siq| read  writ| recv  send|  block i/o process
07-10 03:52:21| 35  11  51   1   0   1|   0     0 |  57M 8701k|
07-10 03:52:22| 35  12  51   1   0   1|   0     0 |  56M 8585k|
07-10 03:52:23| 33  12  52   2   0   1|   0     0 |  56M 8636k|
```
_Figure 2: Redcon and Redeo CPU usage during 128 connection / 32 pipelined request benchmark._

These system metrics were captured during the 128 connections with 32 pipelined requests run. We are even experiencing IOWAIT time on the CPU during the Redcon benchmark.

# Results
Redis has more predictable latency distribution with lower outliers in the `99-100%` range than Redcon or Redeo. Redcon and Redeo have higher throughput with lower latency throughout the `0%-99%` range but exhibits higher and sometimes excessive outliers in the `99%-100%` range. I suspect this is due to the GC pauses.

As client connections increase the single threaded design of Redis starts to show signs of weakness where Redcon and Redeo do not. As mentioned earlier, one solution for solving this is sharding multiple Redis processes however this article is benchmarking how each single process performs.

In many cases Redeo and/or Redcon out perform Redis both in throughput and latency up until the `99%` mark. This isn't surprising since they aren't single threaded.

Each benchmark run lasts for `10 minutes` per service for a total duration of `30 minutes`.

![31m3.322475412s       64/1](http://i.imgur.com/cUG3z9k.jpg)
![32m42.38450521s       64/4](http://i.imgur.com/DpxTN8Q.jpg)
![33m25.159167175s      64/8](http://i.imgur.com/C8b4IUC.jpg)
![34m58.887443776s      64/16](http://i.imgur.com/0bVoCug.jpg)
![36m0.847855503s       64/32](http://i.imgur.com/IURWDNa.jpg)
![35m18.62984506s       64/64](http://i.imgur.com/I2fxzes.jpg)
![33m26.403964065s      64/128](http://i.imgur.com/O2fiBvc.jpg)
![30m52.935227999s      128/1](http://i.imgur.com/upjmVCC.jpg)
![32m39.847335892s      128/4](http://i.imgur.com/9SNB9pY.jpg)
![33m22.998128944s      128/8](http://i.imgur.com/6PCYc5I.jpg)
![35m39.355654102s      128/16](http://i.imgur.com/RvaBNNc.jpg)
![37m6.338372132s       128/32](http://i.imgur.com/rJXdwoj.jpg)
![34m4.376775327s       128/64](http://i.imgur.com/4atTyLt.jpg)
![35m12.063702525s      128/128](http://i.imgur.com/ZWvtUJi.jpg)
![30m43.359264281s      256/1](http://i.imgur.com/ZNHkRDr.jpg)
![31m38.929016957s      256/4](http://i.imgur.com/yA9xqId.jpg)
![33m24.552355671s      256/8](http://i.imgur.com/GBsWepJ.jpg)
![36m51.827534508s      256/16](http://i.imgur.com/vU5qiEq.jpg)
![35m37.664602435s      256/32](http://i.imgur.com/srbQPV2.jpg)
![33m31.277769482s      256/64](http://i.imgur.com/IH9mqOS.jpg)
![35m58.997174736s      256/128](http://i.imgur.com/3ysTqL8.jpg)
![30m51.599212418s      512/1](http://i.imgur.com/o5wtuTm.jpg)
![32m5.932652855s       512/4](http://i.imgur.com/VQJZM7O.jpg)
![33m24.46653593s       512/8](http://i.imgur.com/bnGBeVy.jpg)
![35m29.630088917s      512/16](http://i.imgur.com/kiiy5Dc.jpg)
![37m4.720939476s       512/32](http://i.imgur.com/LqaKSoY.jpg)
![33m33.102139113s      512/64](http://i.imgur.com/En4V0sS.jpg)
![34m59.409783875s      512/128](http://i.imgur.com/8IWmQ2F.jpg)
![30m54.412707407s      1024/1](http://i.imgur.com/joD2p1Y.jpg)
![31m44.762055229s      1024/4](http://i.imgur.com/ut2ZYji.jpg)
![33m51.095726432s      1024/8](http://i.imgur.com/4k97LQz.jpg)
![36m28.53817141s       1024/16](http://i.imgur.com/wJH95QS.jpg)
