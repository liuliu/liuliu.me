---
date: '2008-09-04 02:22:00'
layout: post
slug: why-k-mean-is-wrong-in-hyperspace
status: publish
title: Why k-mean is wrong in hyperspace
wordpress_guid: http://jsms.me/?p=16
wordpress_id: '305'
categories:
- 随感
---

K-mean algorithm is a typical cluster algorithm when you have numbers of classes to be clustered. More than that, k-mean is a basic algorithm which foster many semi-supervised cluster and flexible cluster algorithms. Recently, k-mean and its derivatives (hierarchical k-mean, approximate k-mean, etc.) dominated the algorithm of visual word discovery. However, k-mean algorithm is a dead way when you try to reach higher accuracy. 

K-mean algorithm only applys to simple linear space. At least, in a space with following properties: continuous, defined operators of plus and multiply. The basic definitions only make k-mean executable. K-mean algorithm also implies that the space is isotropic in every direction. 

It is a strict assumption k-mean needed. In fact, most manually generated data to test k-mean are in Euclidean space which is well-known for its poor performance in high dimensional circumstance. And unfortunately, most real world problems lay in high dimensional space which we can hardly prove it is linear or has a center. 

An ideal cluster algorithm should: 

1. Resist to pseudo-random trap;  
2. With minimum assumption, like only define the distance between each sample;  
3. Executed in reasonable time, better than or at least equal to O(M*N), where M is the number of classes, N is the number of samples.   

