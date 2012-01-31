---
date: '2009-12-19 05:59:45'
layout: post
slug: revising-basic-ideas-of-good-interesting-point-detector
status: publish
title: Revising Basic Ideas of Good Interesting Point Detector
wordpress_guid: http://jsms.me/?p=720
wordpress_id: '720'
categories:
- eyes
tags:
- interesting point detector
- local feature descriptor
---

It is always a good idea to revise some basic settings for your work after years. I was motivated by the work of revising basic ideas of AI research ([http://web.mit.edu/newsoffice/2009/ai-overview-1207.html](http://web.mit.edu/newsoffice/2009/ai-overview-1207.html)) and thinking about revising some basic concepts in my research related area (much smaller). One of my interested area in computer vision is local feature descriptor. Though you can examine local feature descriptor densely, it is always more economically to use an interesting point detector as a preprocess step.

The problem of using interesting point detector or not boils down to two fundamentally different paths for object detection some researchers will describe as feature-centric and window-centric detection. It is a curious case to investigate that we human beings are more likely to use feature-centric detection for observation. The feature-centric solution usually gives a reasonable good (where the object is obvious) result within much less time. I'd like to avoid the word of "superior" in this case since window-centric method is usually better for dedicate object detection (the case you have tens of thousands positive examples).

Interesting point detector is important if we attempt to gain cheap speed up with some loss of accuracy. The problem is so true in Internet age that the daily uploaded photo is about 0.2 million in Flickr which makes the densely examination nearly impossible. For many widely adopted local feature descriptors, authors themselves purposed their own interesting point detectors (local maxima in DoG for SIFT, local maxima in approximate hessian pyramid for SURF etc.). There are combinations and cross examinations to test which interesting point detector works better with which local feature descriptor. Few work analysed why certain interesting point detector works better with certain local feature descriptor other than provided empirical results.

Because most interesting point detectors are actually corner point detector, it tends not work well with small objects, objects with large plain indistinguishable surface or objects with complex 3D structure. However, the ambitious current state-of-art descriptors are hoping to have a general good performance in every case. Thus, I believe with careful designed experiments, some improvements can be done for repeatability and representativeness of interesting point detector based on sampling local feature descriptor.
