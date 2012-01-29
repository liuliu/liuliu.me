---
date: '2009-07-24 05:31:56'
layout: post
slug: image-self-similarity-everywhere
status: publish
title: Image Self-Similarity Everywhere
wordpress_id: '593'
categories:
- Eyes
tags:
- ensemble tracking
- self-similarity
- tracking algorithm
---

Recent days I came across a paper which describe a tracking algorithm called ensemble tracking. It is an interesting reading and really easy to implement. Actually, I spent 30 hours to finish this algorithm.

One funny part of this algorithm is that whether intention or not, the author took advantage of self-similarity feature to make his algorithm useful.

The feature used for ensemble tracking is per-pixel based, in that way, a linear classifier can be gathered with any boosting algorithm. By applying the classifier to all pixels on the image, a probability image can be generated. On the probability image, all the traditional tracking algorithm can be applied.

My first thought of the algorithm is the rising doubt about the segmentation ability of the simple linear classifier (a linear combination of ~5 weak classifier). The result is not comparable to well-trained detector, but it shows the attempt of the algorithm to classify very similar pixels (in color perspective), and that is a success.

![Ensemble Tracking 1](http://jsms.me/wp-content/uploads/2009/07/Screenshot-2.png) ![Ensemble Tracking 2](http://jsms.me/wp-content/uploads/2009/07/Screenshot.png)

(notice how the classifier improved over time)

Without the property that most of the images are sparse and shares many common parts (self-similarity), it cannot gain any knowledge through the per-pixel feature. In fact, if an image fulfill with noise, the per-pixel features within a rectangle are mostly self-contradiction.

We can grab more fruits with the nice the sparsity and self-similarity of images.

(Update about NDQI: it is not dead. the command tool & parser seems to be a time sinker but luckily, now it is in test phase)
