---
date: '2009-03-31 19:14:03'
layout: post
slug: reflection-on-learning-distance
status: publish
title: Reflection on Learning Distance
wordpress_id: '492'
categories:
- eyes
tags:
- distance learning
- k-nn
---

The one major common sense shared in machine learning community is that euclidian distance is poor. To attack this problem, one way is to use another distance measurement, and the other is to learn a better distance representation. Mahanalobis distance is a good practice by linear tranform our data to a more suitable space. As it is only do one linear transformation, after the transformation, it is still a normal euclidian distance.

By finding a better linear space to retain NN, it may dramatically improve the result (>2x). However, it cannot dilute our concern to the imperfection of euclidian space. By simply turn to another "nonlinear" method cannot serve any good too. Turning a simply question to a space which has more degree of freedom and tuning a better result is a way to avoid harder and realistic problem. Stick to the linear way is not something too shy to say.

At the monment, we are still largely depended on lower-dimensional euclidian distance and hoping to find another unified way to do distance measurement.
