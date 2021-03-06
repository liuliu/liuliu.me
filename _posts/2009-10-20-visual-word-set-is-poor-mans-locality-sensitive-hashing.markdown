---
date: '2009-10-20 06:02:17'
layout: post
slug: visual-word-set-is-poor-mans-locality-sensitive-hashing
status: publish
title: Visual Word Set is Poor Man's Locality Sensitive Hashing?
wordpress_guid: http://jsms.me/?p=665
wordpress_id: '665'
categories:
- eyes
tags:
- locality sensitive hashing
- visual word
---

The constructing of visual word set is essentially a method to find clusters/exemplars in given feature space. The result by comparing a local feature vector with visual word set is the index of visual word. It compacted the reverse index to each image, but the underlying mapping is the same. It performs the calculation h(v) to convert one vector to an integer.

Though the mapping function of looking up visual word is very similar to the hashing function of locality sensitive hash scheme, the idea behind it is very different. LSH more cares about preserve the distance measurement and coverage to whole feature space. Visual word generating cares more about concentration of points. Thus, for L2 distance, visual words tend to use different diameter balls to cover existing points, but LSH uses more same diameter balls to cover the whole range.

It is easy to recognize that LSH encoded more information because it preserved the distance information through a set of hashing functions. It carved each point more precisely by providing several measurements. On the other hand, looking up visual word lost all the distance information, only yielded a categorical result. Comparing as one hashing function, visual word method is good because it obtained approximately best partition for the existing points. But the overall performance are restricted by the limited output.

I am looking into an algorithm that combines dynamic generated visual word set, the sparsity nature of self-similarity descriptor and locality sensitive hashing into an online local feature comparison.
