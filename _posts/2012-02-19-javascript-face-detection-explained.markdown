---
date: '2012-02-19 23:33:00'
layout: post
slug: javascript-face-detection-explained
status: publish
title: JavaScript Face Detection Explained
categories:
- eyes
---

The [Not-so-slow JavaScript face detector](http://liuliu.me/ccv/js/nss/) was written two years ago. Initially, it is a one-day-hacking to see if the state-of-art face detector technology is implementable at tolerable speed with JavaScript. That one day's hack lived up years with many extensions and applications spreading on the web: a [JQuery plug-in](http://facedetection.jaysalvat.com/), a [video face detector](http://wesbos.com/html5-video-face-detection-canvas-javascript/) and a [mustache demo](http://www.easymustache.com/). One interesting finding over years is that the JavaScript speed increased dramatically on both Google Chrome and Mozilla Firefox. When I was writing the face detector, a 800x600 image usually took more than 6 seconds on Firefox 3, but now with Firefox 10, it takes about 1 second. At around the same time, Google Chrome is improved from about 2 seconds to 1 second. This script alone witnessed the armed race between browsers and it is a good thing. But over years, although [the source code is out there](http://github.com/liuliu/ccv), how this worked is never explained. I did little comment in the source code, and the algorithm is not as well-known as HAAR classifier used in OpenCV.

The very basic instrument used in my implementation is called [control-point feature (renamed to brightness binary feature to reflect that the implementation in ccv works only on brightness value)](http://scholar.google.com/scholar?q=YEF*+Real-time+Object+Detection). For a given WxH image region, one feature consists of two sets of control points, a[1], a[2], ... a[n] and b[1], b[2], ... , b[m]. To classify the given image region, a feature examines the pixel values at control points in group a and group b in relevant images (at original size, half-size and quarter-size). The feature only answers "yes" if all pixel values in group a is greater / less than any pixel values in group b. The details can be found in the original paper [YEF: Real-time Object Detection](http://scholar.google.com/scholar?q=YEF*+Real-time+Object+Detection) and a follow-up [High-Performance Rotation Invariant Multiview Face Detection](http://scholar.google.com/scholar?q=High-Performance+Rotation+Invariant+Multiview+Face+Detection). Long story short, the training program bbfcreate will create several strong linear classifiers from control-point features using [AdaBoost](http://en.wikipedia.org/wiki/AdaBoost).

The control-point feature is simple enough that after the generation of the image pyramid (a series of images that downsized from original WxH size image to W/2xH/2, W/4xH/4 ...), there is no further image processing required. If the computation to generate such image pyramid can be negligible, for each control-point feature, it accesses fewer memory locations (n + m <= 5) than HAAR-like features (the one implemented in OpenCV, requires 6~9 memory accesses). This turns out to be a good improvement, and the ccv implementation in C achieved similar accuracy (82.97% with 12 false alarms V.S. 86.69% with 15 false alarms) comparing with OpenCV default face detector but 3 times faster (as a side note, this is still far from proprietary implementation which achieves ~90% with ~3 false alarms on the same data set, [read more details](http://github.com/liuliu/ccv/blob/stable/doc/bbf.md)). This is an even better news for the JavaScript implementation since the downsizing operation can be offloaded natively with HTML5 canvas' drawing method. That's the secret sauce in my not-so-slow face detector (implemented in [line 200](https://github.com/liuliu/ccv/blob/unstable/js/ccv.js#L200)).

Once the image pyramid is generated, the detection process is just following the paper. The algorithm sweep over the whole image at different resolutions to check if a face exists there with control-point feature ([line 290](https://github.com/liuliu/ccv/blob/unstable/js/ccv.js#L290)). I have no other tricks to improve speed-wise beyond this point. At the end of this process, it merges detected areas and returns that with confidence score.

OK, let's reconfirm how fast it is:

<http://liuliu.me/ccv/js/nss/#http%3A%2F%2Fmtlweb.mit.edu%2Fresearchgroups%2Ficsystems%2Fphotos%2Fpeople%2Flarge%2Fgroup_dec2011_large.jpg>

This 2808x1805 image takes 6 seconds on Firefox with Web Worker off, and 10 seconds with Web Worker on. It takes 4 seconds on Google Chrome (Web Worker doesn't work as smooth in Google Chrome).

Please let me know what else in this implementation you want to be explained in the comments.