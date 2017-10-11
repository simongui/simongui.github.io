---
layout: post
title: Code documentation quality
draft: true
---

As I get older I start to value documentation more and more. Maybe it's because I'm getting involved with larger and more complicated projects than earlier in my career, or maybe I'm just getting older and I don't hold every possible thing in my head at once like I used to. Either way, I've come to really appreciate a well documented code base.

[Postgres](https://github.com/postgres/postgres) is a project I like to look at as an example of a really well kept code base. I like that even though Postgres is written in C (a difficult to master programming language) it has a really well maintained source repository. It has a few key things I think are great to strive for.

Let's look at this [GIN secondary index](https://github.com/postgres/postgres/tree/master/src/backend/access/gin) implementation in Postgres as an example.

1. Detailed [README](https://github.com/postgres/postgres/blob/master/src/backend/access/gin/README) created in it's sub-directory to describe how features are designed and how they work.
1. Detailed comments describing [what functions](https://github.com/postgres/postgres/blob/master/src/backend/access/gin/ginbtree.c#L65) do.
1. Comments within functions to [explain logic](https://github.com/postgres/postgres/blob/master/src/backend/access/gin/ginbtree.c#L99) of the code.

The Postgres documentation quality and code clarity is superb. There's no reason why we can't strive for similar quality in any project using any programming language.
