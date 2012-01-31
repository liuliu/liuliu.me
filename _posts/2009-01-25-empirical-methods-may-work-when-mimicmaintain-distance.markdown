---
date: '2009-01-25 06:41:02'
layout: post
slug: empirical-methods-may-work-when-mimicmaintain-distance
status: publish
title: Empirical Methods may Work when Mimic/Maintain Distance
wordpress_guid: http://jsms.me/?p=450
wordpress_id: '450'
categories:
- 随感
tags:
- boost
- crawler
- local feature descriptor
- local sensitive hash
- similarity measurement
- visual vocabulary
- wavelet
---

Local feature descriptors have overwhelming advantage in comparing different images against cropping, resizing, affine trans, light vars. However, to compare local feature descriptors between two images needs O(MlogN) time where M is the LFD number of 1st image and N is the LFD number of 2nd image. Having in mind that classic LFD involves 128d or more, one can theoretically conclude that the comparing process will take almost 10ms in a commodity computer. Time becomes a severe problem when scale up to compare across hundreds of thousands images.

Most approaches nowadays try to solve the problem by reduce the complexity of comparing LFDs. LSH/Tree-based methods attack the problem with classical data structure which could reduce the time complexity of search process to O(1) or O(logN). Visual vocabulary introduced classic IR model to visual search which makes the time complexity irrelevent to number of LFD indexed. O(logN) complexity is not good enough for billions of features. Visual vocabulary takes too much time to cluster all LFDs. LSH seems like one good choice though indexing billion features needs cluster computer  to support.

All three methods above ignore the factor of sparsity and redundancy. Empirically, classic method can discover 500~600 LFDs from 800*600 images. In wavelet trans or fourier trans, discard 95% dimensions can still maintain most part of the images, which means that most information can be recovered from 24,000 dimension space. LFDs support a 64,000~76,800 dimension feature space which is far more enough for feature comparision. In the other hand, by a simple observation, sparsity can be proved. The classic comparing process find one to one correlation in two images. If we simply think other's similarity is zero, a 1/500 sparse similarity matrix are obtained. The idea of setting the similarity of two points that are too far from each other to zero is because when measuring in high dimensional euclidean space of random distributed points, their distances from each other tend to be very close. The measurement observation suggests that in high-dimensional euclidean space, far distance is useless. As here we only observe two image comparision, in actual case, because only few images have correlate points, the distance matrix should be much more sparse.

[My last article](http://jsms.me/?p=436) already suggested a method to reduce the indeterminate dimension to fewer dimension, for say, 30d. In that way, we are hoping the 30d data can still maintain good approximation of non-metric distances. After read several MDS methods, the computation cost scared me. Considering many MDS methods are examplar-based, it may be improper to apply to very large database.

The thought of introducing empirical methods because empirical methods, most of them, are fast and robust. A proper combination of classic global image features, such as gist, color histogram, wavelet and so on, maybe, can mimic a good approximation to LFD comparision. BTW, we don't expect a 90% correct approximation. a 10% hit rate is enough because that can also dramatically reduce the computation cost in refine stage.

All above are based on theoretical analysis. I programmed a crawler to capture some pics from flickr to verify my theory, but there are some serious memory leaks in the crawler framework I use.
