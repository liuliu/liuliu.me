---
date: '2011-05-22 01:02:44'
layout: post
slug: ccv-new-in-the-summer
status: publish
title: 'CCV: New in the Summer'
wordpress_id: '1277'
categories:
- Eyes
---

The progress of ccv during the spring is not as good as I expected. During the spring, I've finished a new test framework for ccv known as CASE, and moved most ccv ad-hoc unit tests into this framework. I've implemented SWT (Stroke Width Transform) for text detection, though the result on test dataset is not as good as the original paper claims to be. The DPM (Deformable-Parts Model, a.k.a. Latent SVM) implementation was started, but quickly got stalled.

But finally I would have some spare time to work on ccv codebase. And I would love to implement these things during the summer:

1). I finally figured out appropriate HTTP interface for RPC to ccv;

2). A new cache back-end thus you don't have to call ccv_garbage_collect every time;

3). Long discussed DPM implementation in ccv;

4). Web/Android support with favor of Actionscript (for webcam), Javascript and HTTP-RPC (I am deadly serious, I plan to implement a semi-realtime detector/tracker with webcam support).
