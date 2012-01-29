---
date: '2009-10-27 06:08:49'
layout: post
slug: photorealistic-3d-graphics-guided-object-detection-training
status: publish
title: Photorealistic 3D Graphics Guided Object Detection Training
wordpress_id: '671'
categories:
- eyes
tags:
- 3d render
- object detection
---

Today's robust object detection algorithm need large training dataset. For example, THU's high accuracy face detection system uses 30k positive faces, and the negative examples collected from background images are countless. It poses a very serious problem for researchers. Because good result relies on both novelty of algorithm and size of training dataset. Completeness of dataset will help the algorithm to generate good assumptions of the subject. But the collecting of dataset is heavily labored. It requires human input for every sample, which makes the collection process not scalable.

On other hand, the photo-realistic 3D graphics is very much mature. Nowadays' PC software can produce very real graphics. The only difference is the rendering is very computational expensive. However, one stage of effective training typically requires ten thousands positive images, it is not a small number, a 6 min length 3D movies roughly has 10,000 frames. However, only a very low-resolution image is something we really need in later training stage, a 32x32 resolution is enough for many tasks. The specific requirement makes the problem feasible.
