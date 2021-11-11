---
layout: post
title: Measuring database changes using shadow instances
draft: true
---
These days I really enjoy how transparent engineering teams around the industry are with sharing their expertise and incident reports. Hearing about the environment, goals, constraints and tradeoffs these teams are working with provides a lot of insights. When I see these posts surfacing on hackernews there's a lot of arm chair architecting in the comments and often the original author replies by explaining additional constraints they had to account for that influenced their decisions.

I usually keep my own private personal notes about my thoughts when reading these types of posts because they are often loose thoughts while brainstorming but I've decided there's a way I can approach my notes in a blog style that 


The other day I was reading a post from the team at [heap.io](https://heap.io) who wrote about [how they saved millions in SSD costs (on Postgres instances) by upgrading their filesystem](https://heap.io/blog/how-we-saved-millions-in-ssd-costs-by-upgrading-our-filesystem). 

# REFERENCES

1. Multiversion concurrency control.  
[https://en.wikipedia.org/wiki/Multiversion_concurrency_control](https://en.wikipedia.org/wiki/Multiversion_concurrency_control)  
