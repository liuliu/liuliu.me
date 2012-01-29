---
date: '2009-04-27 14:39:16'
layout: post
slug: using-affinity-propagation-as-a-descriptor-reducing-tool
status: publish
title: Using Affinity Propagation as a Descriptor-Reducing Tool
wordpress_id: '517'
categories:
- eyes
tags:
- affinity propagation
- content-based image search
- image patch
- local feature descriptor
---

The fancinating part of high-dimensional descriptor is that it has two faces, one is the sparsity and the other is the density. The overall descriptor (in my case, image patch descriptor/local feature descriptor) data are lay in space with large variance. But by observing small group of descriptor data, there are some dense area in the space.  The duplicate detections of corner detector, similar objects and etc. all may cause the density. By reducing all the density descriptor cloud to one examplar, it will reduce the overall matching time. Especially, for CBIR, it would be nearly no after-effect of reducing descriptors for one image.

That is the time where affinity propagation comes in as a replacement for k-median. Affinity propagation is an amazingly fast and pretty good approximation to the optimal examplar result. Affinity propagation's property of using sparse matrix can largely reduce the computational cost. By using full neighborhood dissimilarity information and the mean of dissimilarity as preference, it reduced 1147 local patches in a image to 160 local patches. How ever, to compute the full dissimilarity matrix is time expensive,  for my experiment, a best-bin-first tree was used to speed up the k-nn search and only set the dissimilarity with top N (N=5, 10, 20) neighbors. In that case, the time cost was reduced from 30s to less than 1s and the number of local patches was reduced to 477(T20), 552(T10), 647(T5).

A coarse observation is that, by reducing the number of local patches, the accuracy of search over database is improved. The reduction of local patches leaves the more distinctive ones in the bank. More distinctive points reduced the false positive, and get the overall performance gains.

As the affinity propagation method shows many promising aspects, the new key word "EXAMPLAR" will be introduced in the implementation of non-structural data query lang.
