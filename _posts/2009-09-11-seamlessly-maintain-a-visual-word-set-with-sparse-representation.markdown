---
date: '2009-09-11 15:12:32'
layout: post
slug: seamlessly-maintain-a-visual-word-set-with-sparse-representation
status: publish
title: Seamlessly Maintain a Visual Word Set with Sparse Representation
wordpress_guid: http://jsms.me/?p=641
wordpress_id: '641'
categories:
- eyes
tags:
- sparse representation
- visual word
---

One problem about nowadays visual word based image retrieval system is to generate visual word set. Visual word set tend to be big (~50k), different cluster methods such as approximate k-center, affinity propagation are applied. However, the process is somehow periodical. Imagine you are an engineer in Flickr who roll out the new image retrieval system, and you generated visual word set based on yesterday's Flickr photo set; but today, there are 200k more photos just uploaded to Flickr, at least you have to monthly generate visual word set in order to avoid mis-classify new visual word in new photo.

In today's real-time world, the method looks like old fashion. The need is to add new visual word as soon as new photo is uploaded. The problem is, how we know a visual word is "new"? Though it is very obvious for us or computer to judge if a text word is new or not (or not so obvious for computer ie. wrong spelling?), visual word is hard. Maybe we can put some threshold for the k-NN visual word search and claim that the new query which similarity to the nearest neighbor below certain bar is a new visual word. But it is very unlikely the naive method can work in real world. The similarity measurement is very tricky part, maybe the query is not a new visual word at all, maybe it is just an old visual word under very different illumination. If that is the case, another naive method can be suggested. We may want to measure the difference between the 1st nearest neighbor and 2nd nearest neighbor. If the similarity of the query to 1st and 2nd NN are relatively the same, we may argue it is a new visual word because in high-dimensional Euclidean space, two random points tend to have relatively the same distance. The 2nd naive method is still not so persuasive because one can argue that the approximation method is very arbitrary.

Though I cannot provide more evidence to support this, but it may work to apply sparse analysis to filter out new visual words. Let's imagine that we try to get a representation from exist word list A: y=Ax and minimize ||x||_{L1}. And the SCI (sparsity concentration index) can give good indication about if a query is new visual word or not.
