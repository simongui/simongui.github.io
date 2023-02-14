---
layout: post
title: Visualing Postgres EXPLAIN ANALYZE to rapidly troubleshoot queries
draft: true
---
I work with a lot of database types and while I love my terminal, at times I find visualizations can make things clearer faster especially when I sometimes don't remember all the details involved. I was working with an extremely large query in a Postgres database and I decided to hunt for any visualization tools that might exist to help me understand this problematic query much faster.

I found a really helpful visualizer called [PEV2](https://github.com/dalibo/pev2/blob/master/index.html). PEV2 is a single HTML page you can download and open in your browser without the need of a web server. The page prompts you to paste both the query and the `EXPLAIN ANALYZE` output of the query. 

![](https://blog.dalibo.com/img/202101_pev2_diagram.png)  
_Figure 1. PEV2 left panel._

In _Figure 1_ on the left hand side of the UI you get the sequence tree of operations that executed with their duration shown.

![](https://blog.dalibo.com/img/202101_pev2_highlight.png)  
_Figure 2. PEV2 right panel._

On the right side showh in _Figure 2_ of the screen we get a visual of the operation tree.

As always I like to thank people for the tools they open source. Thank you DALIBO for your work on PEV2 I've really enjoyed how easily PEV2 makes all the information I need available to troubleshoot queries.

# References
1. [PEV2](https://github.com/dalibo/pev2/blob/master/index.html)
2. [DALIBO blog about PEV2](https://blog.dalibo.com/2021/02/09/pev2_whats_new_en.html)
