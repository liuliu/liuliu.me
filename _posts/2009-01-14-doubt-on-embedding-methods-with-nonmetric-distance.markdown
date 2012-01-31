---
date: '2009-01-14 17:19:39'
layout: post
slug: doubt-on-embedding-methods-with-nonmetric-distance
status: publish
title: Doubt on Embedding Methods with Nonmetric Distance
wordpress_guid: http://jsms.me/?p=436
wordpress_id: '436'
categories:
- 随感
tags:
- k-nn
- linear embedding
- local feature descriptor
- machine learning
- visual vocabulary
---

I allocated some time last week to investigate tree-based and hash-based approximate k-NN methods. It seems that tree-based methods are more likely to be promising. As a result, I implemented the spill tree in OpenCV repository. In test, tree-based method suffers great pain from the curse of dimensionality. 32d is ok, but for 128d, it is beaten by naive search. However, to reduce the dimension is always the right choice, but even consider that, 4ms for a indexed 60k size database, it is far from practical.

My recent interests takes great concern to local feature descriptor (LFD) and view that as the most promising method for image recognition I've ever encountered. Don't forget that even for face recognition method which has been considered as partly solved, dividing to several parts is the common technique.

For most methods to extract LFD, the actually resulting features are lay in euclidean space. But for one image, the length of result LFD sequence can be hundreds or thousands. In single feature space, we have to perform thousands of queries in a billions of features database (million image database). Even scale to thousands computers and assume the network delay is negligible, the results are still poor (4s per thousand queries).

One practical method that was borrowed from full text retrieval is visual vocabulary. But there is one thing in my mind that keeps me walk away from the state-of-art method. How one can determine if 50k words is enough for describe any image? Even with a very convincing results (comparing 300k, 50k and so on size of vocabularies), it is still unknown how the word size grows along with the number of image indexed.

Another way to do this is to fall back to traditional content-based image retrieval which consider the whole image as a single element. Unsuccessful trials have been made with histogram, local histogram, color monument and so in 1990s. That is the time tree-based approximate k-NN methods developed as the dimension of histogram is manageable. After decades, certainly we should add something new to the old. One thing changed the most is, the similarity measurements are no longer in euclidean space. That means, the ideas of mean vectors, euclidean distance are not valid here.

That is why the filter-refine method comes out. The idea is mapping nonmetric space to a euclidean space, and then find the approximate nearest subset. Applying distance calculation on a subset is much easier. But question remains is how good the mapping proceduce would be. However, it worth to give a try.
