---
date: '2009-05-25 12:18:41'
layout: post
slug: my-dear-it-is-two-different-problem-domain
status: publish
title: My dear, it is two different problem domains
wordpress_id: '555'
categories:
- Eyes
---

**1. How to make a query and get associated data?**

**2. How to assign data to a specific query instance?**

In the first glance, there seem no big differences between the two problems. One can mimic the solution for 2nd problem by making frequent query to database and get new assignment to current data. However, if we put the Real-time attribute into consideration, the problem becomes very diffcult. It means we cannot rely on lazy query performing/cache to ease the query load to database backend. Every data assignement has been done immediately as the data comes.

The optimization techniques are quite different. In the first scenario, we rely on indexing and shrinking the qualified database size to make the query faster. In the second scenario, the most natural optimization is the left-hand optimization which discards the data with first few conditions within a query. Until now, my research heavily addressed the first problem and ignored the second problem.

The second problem whether "realistic" or not remains unclear for most people. If the solutions to 2nd problem can be as efficient as the first one, our scheme of the overall Internet could experience a dramatic change. In many web apps, we don't deal with changing query, contrarily, we deal with changing data. If we can solve the second problem, btw, the more natural solution for changing data, we don't have to cache anything and deal with expire monster. Twitter took advantage of distributed queue system to deliver new messages other than query messages for different user with different query parameters. Since real-time streaming become the new bragging features for web apps, in foreseeable future, we have to solve the second problem.

A queue system is very primitive for 2nd problem. It only solves the problem of how to store the data's relationship with queries. How to check the data relation validity with millions queries is the real headache. We may utilize some common features between queries, however, for complicated query, I don't know how to do it well.

Any paper recommendations?
