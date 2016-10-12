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
The Go in-memory Map implementations for Redcon and Redeo are sharded but each shard is protected by a writer lock. I've written a tool called [Tantrum](https://github.com/simongui/tantrum){:target="_blank"} to aid in automating the benchmark runs and visualizing the results. With the help of a script and Tantrum I've benchmarked various configurations of concurrent clients and pipelined requests to see how different workloads affect performance.

**Redis:** All disk persistence is turned off. I wrote a version of `redis-benchmark` that supports microsecond resolution instead of millisecond resolution because we were losing a lot of fidelity in some of the results.

# Go garbage collector
The Go garbage collector has received some nice performance improvements as of late but the Go 1.7 GC still struggles with larger heaps. This can surface with one or multiple large Map instances. You can read the [details here](https://github.com/golang/go/issues/15847#issuecomment-247453018){:target="_blank"}. Luckily in `master` there's a fix that reduces GC pauses in these cases by `10x` which can make a `900ms` pause down to `90ms` which is a great improvement. I've decided to benchmark against this fix because this will likely ship in Go version 1.8.

# CPU efficiency
```
----system---- ----total-cpu-usage---- -dsk/total- -net/total- ---most-expensive---
     time     |usr sys idl wai hiq siq| read  writ| recv  send|  block i/o process
07-10 03:39:01|  4   1  94   0   0   1|   0     0 |  56M 8548k|
07-10 03:39:02|  4   1  94   0   0   1|   0     0 |  56M 8539k|
07-10 03:39:03|  4   1  94   0   0   1|   0     0 |  56M 8553k|
```
_Figure 1: Redis CPU usage during 128 connection / 32 pipelined request benchmark._

Shown in `Figure 1` Redis used less CPU resources but it's single threaded design limits its ability to fully utilize all the CPU cores.

```
----system---- ----total-cpu-usage---- -dsk/total- -net/total- ---most-expensive---
     time     |usr sys idl wai hiq siq| read  writ| recv  send|  block i/o process
07-10 03:52:21| 35  11  51   1   0   1|   0     0 |  57M 8701k|
07-10 03:52:22| 35  12  51   1   0   1|   0     0 |  56M 8585k|
07-10 03:52:23| 33  12  52   2   0   1|   0     0 |  56M 8636k|
```
_Figure 2: Redcon and Redeo CPU usage during 128 connection / 32 pipelined request benchmark._

Shown in `Figure 2` Redcon and Redeo both utilized multiple CPU cores better than Redis and allow higher throughput per process however not as efficiently as Redis. This means that 1 Redcon or Redeo process can outperform 1 Redis process however if you ran multiple Redis processes you would experience higher throughput than Redcon or Redeo (at the cost of deployment complexity).

This is a Hyperthreaded machine which means `50% (usr + sys)` usage indicates near CPU saturation. This means the lack of free CPU cycles is getting in the way of greater throughput. I'm concerned that `Figure 2` shows IOWAIT delays.

# Benchmark passes
The combinations of the following configurations were used to record a total of `63` benchmark runs.

**Connections**  
`64, 128, 256, 512, 1024, 2048, 4096, 8192, 16384`    
**Pipelined requests**  
`1, 4, 8, 16, 32, 64, 128`

It's not uncommon in production environments I've seen to have thousands or tens of thousands of client connections to a single Redis instance. Each benchmark run lasts for `10 minutes` per service for a total duration of `30 minutes` of recorded results (`35 minutes` approximately with results processing).

There were `2` passes of those `63` benchmark runs recorded. Each pass is `32 hours` long for a total `64 hours` of recorded benchmarking (`74 hours` total including processing).

- **First pass**  
Redis, Redcon and Redeo are freshly restarted processes but warmed up before the benchmark begins.
- **Second pass**  
Redis, Redcon and Redeo services have been running for over a week having run over `80 hours` of benchmarking. This will tell us how a longer running process performs.

# Summary of results
You can see some differences between the first and second passes for Redcon and Redeo as the longer running Go processes show signs of wear. Latency in the higher percentiles seems to spike but more notably the throughput achieved in the first pass isn't seen in the second pass. In the first pass during some runs Redcon was able to peak at `1.2 million SET requests/second` while in the second pass `300,000 SET requests/second` was the maximum throughput.

Overall Redis has more predictable latency distribution with lower outliers in the `99-100%` range than Redcon or Redeo. Redcon and Redeo have higher throughput with lower latency throughout the `0%-99%` range but exhibits higher and sometimes excessive outliers in the `99%-100%` range. I suspect this is due to the GC pauses.

As client connections increase the single threaded design of Redis starts to show signs of weakness where Redcon and Redeo do not. As mentioned earlier, one solution for solving this is sharding multiple Redis processes however this article is benchmarking how each single process performs.

In many cases Redeo and/or Redcon out perform Redis both in throughput and latency up until the `99%` mark. This isn't surprising since they aren't single threaded.

# First pass results
[![30m46.145808132s      64/1](http://i.imgur.com/HNAbDOq.jpg)](http://i.imgur.com/HNAbDOq.jpg){:target="_blank"}
[![31m54.915331448s      64/4](http://i.imgur.com/M4hn5WW.jpg)](http://i.imgur.com/M4hn5WW.jpg){:target="_blank"}
[![33m18.954527192s      64/8](http://i.imgur.com/Sw7JdAN.jpg)](http://i.imgur.com/Sw7JdAN.jpg){:target="_blank"}
[![35m38.670499732s      64/16](http://i.imgur.com/cFv4jAT.jpg)](http://i.imgur.com/cFv4jAT.jpg){:target="_blank"}
[![35m57.70865066s       64/32](http://i.imgur.com/51uag28.jpg)](http://i.imgur.com/51uag28.jpg){:target="_blank"}
[![33m42.103557647s      64/64](http://i.imgur.com/wpsVRQm.jpg)](http://i.imgur.com/wpsVRQm.jpg){:target="_blank"}
[![35m58.991079487s      64/128](http://i.imgur.com/ZtHalpK.jpg)](http://i.imgur.com/ZtHalpK.jpg){:target="_blank"}
[![31m3.761905733s       128/1](http://i.imgur.com/vZhbsRx.jpg)](http://i.imgur.com/vZhbsRx.jpg){:target="_blank"}
[![32m27.354653488s      128/4](http://i.imgur.com/FpFLNXX.jpg)](http://i.imgur.com/FpFLNXX.jpg){:target="_blank"}
[![34m13.655440231s      128/8](http://i.imgur.com/BUIG5VK.jpg)](http://i.imgur.com/BUIG5VK.jpg){:target="_blank"}
[![35m50.602510961s      128/16](http://i.imgur.com/5lyeVsl.jpg)](http://i.imgur.com/5lyeVsl.jpg){:target="_blank"}
[![35m21.252212053s      128/32](http://i.imgur.com/LGMXiul.jpg)](http://i.imgur.com/LGMXiul.jpg){:target="_blank"}
[![35m42.410185798s      128/64](http://i.imgur.com/6IzYjbY.jpg)](http://i.imgur.com/6IzYjbY.jpg){:target="_blank"}
[![35m46.531760901s      128/128](http://i.imgur.com/sohFCQ8.jpg)](http://i.imgur.com/sohFCQ8.jpg){:target="_blank"}
[![30m45.271462955s      256/1](http://i.imgur.com/e8xKiyV.jpg)](http://i.imgur.com/e8xKiyV.jpg){:target="_blank"}
[![32m28.119180084s      256/4](http://i.imgur.com/GrTyert.jpg)](http://i.imgur.com/GrTyert.jpg){:target="_blank"}
[![33m27.796582097s      256/8](http://i.imgur.com/KxoCGl3.jpg)](http://i.imgur.com/KxoCGl3.jpg){:target="_blank"}
[![35m56.82829609s       256/16](http://i.imgur.com/aGgJG7c.jpg)](http://i.imgur.com/aGgJG7c.jpg){:target="_blank"}
[![33m42.667466329s      256/32](http://i.imgur.com/x8rWnPw.jpg)](http://i.imgur.com/x8rWnPw.jpg){:target="_blank"}
[![35m50.150773509s      256/64](http://i.imgur.com/GB5lsN7.jpg)](http://i.imgur.com/GB5lsN7.jpg){:target="_blank"}
[![34m3.364519318s       256/128](http://i.imgur.com/syqhCqL.jpg)](http://i.imgur.com/syqhCqL.jpg){:target="_blank"}
[![31m2.605489435s       512/1](http://i.imgur.com/Ny5MH1N.jpg)](http://i.imgur.com/Ny5MH1N.jpg){:target="_blank"}
[![32m27.317909709s      512/4](http://i.imgur.com/jjmpazc.jpg)](http://i.imgur.com/jjmpazc.jpg){:target="_blank"}
[![34m2.493518054s       512/8](http://i.imgur.com/daUPFKx.jpg)](http://i.imgur.com/daUPFKx.jpg){:target="_blank"}
[![36m10.265704985s      512/16](http://i.imgur.com/LRatVaf.jpg)](http://i.imgur.com/LRatVaf.jpg){:target="_blank"}
[![35m44.295602286s      512/32](http://i.imgur.com/WPSO6rb.jpg)](http://i.imgur.com/WPSO6rb.jpg){:target="_blank"}
[![35m59.046439084s      512/64](http://i.imgur.com/JXriX3U.jpg)](http://i.imgur.com/JXriX3U.jpg){:target="_blank"}
[![34m8.447420798s       512/128](http://i.imgur.com/gPStFwK.jpg)](http://i.imgur.com/gPStFwK.jpg){:target="_blank"}
[![31m4.847725062s       1024/1](http://i.imgur.com/NiSGTh8.jpg)](http://i.imgur.com/NiSGTh8.jpg){:target="_blank"}
[![31m34.371756627s      1024/4](http://i.imgur.com/r0k032B.jpg)](http://i.imgur.com/r0k032B.jpg){:target="_blank"}
[![34m27.924214506s      1024/8](http://i.imgur.com/qCEJHaD.jpg)](http://i.imgur.com/qCEJHaD.jpg){:target="_blank"}
[![36m45.18276848s       1024/16](http://i.imgur.com/K9sVqTh.jpg)](http://i.imgur.com/K9sVqTh.jpg){:target="_blank"}
[![37m35.898693978s      1024/32](http://i.imgur.com/96T5uvq.jpg)](http://i.imgur.com/96T5uvq.jpg){:target="_blank"}
[![35m51.042629847s      1024/64](http://i.imgur.com/PVR21oQ.jpg)](http://i.imgur.com/PVR21oQ.jpg){:target="_blank"}
[![36m16.456633136s      1024/128](http://i.imgur.com/Wm6eWQC.jpg)](http://i.imgur.com/Wm6eWQC.jpg){:target="_blank"}
[![30m52.305757298s      2048/1](http://i.imgur.com/XjbNCyw.jpg)](http://i.imgur.com/XjbNCyw.jpg){:target="_blank"}
[![32m25.956440667s      2048/4](http://i.imgur.com/ICG3i2D.jpg)](http://i.imgur.com/ICG3i2D.jpg){:target="_blank"}
[![33m53.535871481s      2048/8](http://i.imgur.com/KIUBzHq.jpg)](http://i.imgur.com/KIUBzHq.jpg){:target="_blank"}
[![35m10.484188444s      2048/16](http://i.imgur.com/IlHZ1br.jpg)](http://i.imgur.com/IlHZ1br.jpg){:target="_blank"}
[![35m23.073785143s      2048/32](http://i.imgur.com/d2Zo9Nw.jpg)](http://i.imgur.com/d2Zo9Nw.jpg){:target="_blank"}
[![35m17.670328868s      2048/64](http://i.imgur.com/pHyNNFj.jpg)](http://i.imgur.com/pHyNNFj.jpg){:target="_blank"}
[![35m5.750874151s       2048/128](http://i.imgur.com/nGVqP55.jpg)](http://i.imgur.com/nGVqP55.jpg){:target="_blank"}
[![30m52.137323909s      4096/1](http://i.imgur.com/zHHvQge.jpg)](http://i.imgur.com/zHHvQge.jpg){:target="_blank"}
[![31m55.711697359s      4096/4](http://i.imgur.com/BQsAsbi.jpg)](http://i.imgur.com/BQsAsbi.jpg){:target="_blank"}
[![32m48.11976969s       4096/8](http://i.imgur.com/7JxCQPY.jpg)](http://i.imgur.com/7JxCQPY.jpg){:target="_blank"}
[![35m17.842507084s      4096/16](http://i.imgur.com/fHlO6AP.jpg)](http://i.imgur.com/fHlO6AP.jpg){:target="_blank"}
[![36m57.763236335s      4096/32](http://i.imgur.com/YFVkfT6.jpg)](http://i.imgur.com/YFVkfT6.jpg){:target="_blank"}
[![35m14.315455611s      4096/64](http://i.imgur.com/r8zOqFs.jpg)](http://i.imgur.com/r8zOqFs.jpg){:target="_blank"}
[![33m44.700852885s      4096/128](http://i.imgur.com/R7SAIYr.jpg)](http://i.imgur.com/R7SAIYr.jpg){:target="_blank"}
[![31m3.304851101s       8192/1](http://i.imgur.com/9uG7aAv.jpg)](http://i.imgur.com/9uG7aAv.jpg){:target="_blank"}

# Second pass results
[![31m3.322475412s       64/1](http://i.imgur.com/cUG3z9k.jpg)](http://i.imgur.com/cUG3z9k.jpg){:target="_blank"}
[![32m42.38450521s       64/4](http://i.imgur.com/DpxTN8Q.jpg)](http://i.imgur.com/DpxTN8Q.jpg){:target="_blank"}
[![33m25.159167175s      64/8](http://i.imgur.com/C8b4IUC.jpg)](http://i.imgur.com/C8b4IUC.jpg){:target="_blank"}
[![34m58.887443776s      64/16](http://i.imgur.com/0bVoCug.jpg)](http://i.imgur.com/0bVoCug.jpg){:target="_blank"}
[![36m0.847855503s       64/32](http://i.imgur.com/IURWDNa.jpg)](http://i.imgur.com/IURWDNa.jpg){:target="_blank"}
[![35m18.62984506s       64/64](http://i.imgur.com/I2fxzes.jpg)](http://i.imgur.com/I2fxzes.jpg){:target="_blank"}
[![33m26.403964065s      64/128](http://i.imgur.com/O2fiBvc.jpg)](http://i.imgur.com/O2fiBvc.jpg){:target="_blank"}
[![30m52.935227999s      128/1](http://i.imgur.com/upjmVCC.jpg)](http://i.imgur.com/upjmVCC.jpg){:target="_blank"}
[![32m39.847335892s      128/4](http://i.imgur.com/9SNB9pY.jpg)](http://i.imgur.com/9SNB9pY.jpg){:target="_blank"}
[![33m22.998128944s      128/8](http://i.imgur.com/6PCYc5I.jpg)](http://i.imgur.com/6PCYc5I.jpg){:target="_blank"}
[![35m39.355654102s      128/16](http://i.imgur.com/RvaBNNc.jpg)](http://i.imgur.com/RvaBNNc.jpg){:target="_blank"}
[![37m6.338372132s       128/32](http://i.imgur.com/rJXdwoj.jpg)](http://i.imgur.com/rJXdwoj.jpg){:target="_blank"}
[![34m4.376775327s       128/64](http://i.imgur.com/4atTyLt.jpg)](http://i.imgur.com/4atTyLt.jpg){:target="_blank"}
[![35m12.063702525s      128/128](http://i.imgur.com/ZWvtUJi.jpg)](http://i.imgur.com/ZWvtUJi.jpg){:target="_blank"}
[![30m43.359264281s      256/1](http://i.imgur.com/ZNHkRDr.jpg)](http://i.imgur.com/ZNHkRDr.jpg){:target="_blank"}
[![31m38.929016957s      256/4](http://i.imgur.com/yA9xqId.jpg)](http://i.imgur.com/yA9xqId.jpg){:target="_blank"}
[![33m24.552355671s      256/8](http://i.imgur.com/GBsWepJ.jpg)](http://i.imgur.com/GBsWepJ.jpg){:target="_blank"}
[![36m51.827534508s      256/16](http://i.imgur.com/vU5qiEq.jpg)](http://i.imgur.com/vU5qiEq.jpg){:target="_blank"}
[![35m37.664602435s      256/32](http://i.imgur.com/srbQPV2.jpg)](http://i.imgur.com/srbQPV2.jpg){:target="_blank"}
[![33m31.277769482s      256/64](http://i.imgur.com/IH9mqOS.jpg)](http://i.imgur.com/IH9mqOS.jpg){:target="_blank"}
[![35m58.997174736s      256/128](http://i.imgur.com/3ysTqL8.jpg)](http://i.imgur.com/3ysTqL8.jpg){:target="_blank"}
[![30m51.599212418s      512/1](http://i.imgur.com/o5wtuTm.jpg)](http://i.imgur.com/o5wtuTm.jpg){:target="_blank"}
[![32m5.932652855s       512/4](http://i.imgur.com/VQJZM7O.jpg)](http://i.imgur.com/VQJZM7O.jpg){:target="_blank"}
[![33m24.46653593s       512/8](http://i.imgur.com/bnGBeVy.jpg)](http://i.imgur.com/bnGBeVy.jpg){:target="_blank"}
[![35m29.630088917s      512/16](http://i.imgur.com/kiiy5Dc.jpg)](http://i.imgur.com/kiiy5Dc.jpg){:target="_blank"}
[![37m4.720939476s       512/32](http://i.imgur.com/LqaKSoY.jpg)](http://i.imgur.com/LqaKSoY.jpg){:target="_blank"}
[![33m33.102139113s      512/64](http://i.imgur.com/En4V0sS.jpg)](http://i.imgur.com/En4V0sS.jpg){:target="_blank"}
[![34m59.409783875s      512/128](http://i.imgur.com/8IWmQ2F.jpg)](http://i.imgur.com/8IWmQ2F.jpg){:target="_blank"}
[![30m54.412707407s      1024/1](http://i.imgur.com/joD2p1Y.jpg)](http://i.imgur.com/joD2p1Y.jpg){:target="_blank"}
[![31m44.762055229s      1024/4](http://i.imgur.com/ut2ZYji.jpg)](http://i.imgur.com/ut2ZYji.jpg){:target="_blank"}
[![33m51.095726432s      1024/8](http://i.imgur.com/4k97LQz.jpg)](http://i.imgur.com/4k97LQz.jpg){:target="_blank"}
[![36m28.53817141s       1024/16](http://i.imgur.com/wJH95QS.jpg)](http://i.imgur.com/wJH95QS.jpg){:target="_blank"}
[![35m50.70489885s       1024/32](http://i.imgur.com/UC41UFN.jpg)](http://i.imgur.com/UC41UFN.jpg){:target="_blank"}
[![33m1.520941695s       1024/64](http://i.imgur.com/OuQsmXn.jpg)](http://i.imgur.com/OuQsmXn.jpg){:target="_blank"}
[![34m41.788418729s      1024/128](http://i.imgur.com/alCzH7R.jpg)](http://i.imgur.com/alCzH7R.jpg){:target="_blank"}
[![30m54.684691391s      2048/1](http://i.imgur.com/r33tKUY.jpg)](http://i.imgur.com/r33tKUY.jpg){:target="_blank"}
[![32m18.778053272s      2048/4](http://i.imgur.com/wER7iRE.jpg)](http://i.imgur.com/wER7iRE.jpg){:target="_blank"}
[![33m13.418346007s      2048/8](http://i.imgur.com/KTWtRGw.jpg)](http://i.imgur.com/KTWtRGw.jpg){:target="_blank"}
[![35m11.909186265s      2048/16](http://i.imgur.com/1VIhSsh.jpg)](http://i.imgur.com/1VIhSsh.jpg){:target="_blank"}
[![33m48.627801706s      2048/32](http://i.imgur.com/atI7pT0.jpg)](http://i.imgur.com/atI7pT0.jpg){:target="_blank"}
[![35m31.771586183s      2048/64](http://i.imgur.com/25t8Ejx.jpg)](http://i.imgur.com/25t8Ejx.jpg){:target="_blank"}
[![34m2.297144182s       2048/128](http://i.imgur.com/iJx5h1W.jpg)](http://i.imgur.com/iJx5h1W.jpg){:target="_blank"}
[![30m53.445688554s      4096/1](http://i.imgur.com/rEqj00F.jpg)](http://i.imgur.com/rEqj00F.jpg){:target="_blank"}
[![32m21.005815217s      4096/4](http://i.imgur.com/NRpMwvO.jpg)](http://i.imgur.com/NRpMwvO.jpg){:target="_blank"}
[![32m51.908892612s      4096/8](http://i.imgur.com/OTWNWe9.jpg)](http://i.imgur.com/OTWNWe9.jpg){:target="_blank"}
[![36m0.02284276s        4096/16](http://i.imgur.com/DPNzlr9.jpg)](http://i.imgur.com/DPNzlr9.jpg){:target="_blank"}
[![35m43.313700241s      4096/32](http://i.imgur.com/mnY7CbL.jpg)](http://i.imgur.com/mnY7CbL.jpg){:target="_blank"}
[![33m55.440627422s      4096/64](http://i.imgur.com/bKRcHG1.jpg)](http://i.imgur.com/bKRcHG1.jpg){:target="_blank"}
[![35m48.449855055s      4096/128](http://i.imgur.com/P3t7W7n.jpg)](http://i.imgur.com/P3t7W7n.jpg){:target="_blank"}
[![30m50.553462312s      8192/1](http://i.imgur.com/XTPPsXl.jpg)](http://i.imgur.com/XTPPsXl.jpg){:target="_blank"}
[![31m58.140338881s      8192/4](http://i.imgur.com/VseK0Jw.jpg)](http://i.imgur.com/VseK0Jw.jpg){:target="_blank"}
[![33m22.950393447s      8192/8](http://i.imgur.com/OciXQav.jpg)](http://i.imgur.com/OciXQav.jpg){:target="_blank"}
[![35m9.207854768s       8192/16](http://i.imgur.com/XhxIm0Z.jpg)](http://i.imgur.com/XhxIm0Z.jpg){:target="_blank"}
[![35m21.141004574s      8192/32](http://i.imgur.com/jjuwMqi.jpg)](http://i.imgur.com/jjuwMqi.jpg){:target="_blank"}
[![33m39.927007678s      8192/64](http://i.imgur.com/1peYpTD.jpg)](http://i.imgur.com/1peYpTD.jpg){:target="_blank"}
[![34m15.455992783s      8192/128](http://i.imgur.com/kJCC0Ie.jpg)](http://i.imgur.com/kJCC0Ie.jpg){:target="_blank"}
[![31m19.726389646s      16384/1](http://i.imgur.com/UvA467l.jpg)](http://i.imgur.com/UvA467l.jpg){:target="_blank"}
[![32m39.210737143s      16384/4](http://i.imgur.com/Nr9z5N3.jpg)](http://i.imgur.com/Nr9z5N3.jpg){:target="_blank"}
[![32m55.005394811s      16384/8](http://i.imgur.com/qLQF3kL.jpg)](http://i.imgur.com/qLQF3kL.jpg){:target="_blank"}
[![36m3.536117383s       16384/16](http://i.imgur.com/hPRxxga.jpg)](http://i.imgur.com/hPRxxga.jpg){:target="_blank"}
[![37m14.812024799s      16384/32](http://i.imgur.com/9EmgpDn.jpg)](http://i.imgur.com/9EmgpDn.jpg){:target="_blank"}
[![34m9.420295346s       16384/64](http://i.imgur.com/6jfxGzl.jpg)](http://i.imgur.com/6jfxGzl.jpg){:target="_blank"}
[![34m12.475975319s      16384/128](http://i.imgur.com/tPEZAy2.jpg)](http://i.imgur.com/tPEZAy2.jpg){:target="_blank"}
