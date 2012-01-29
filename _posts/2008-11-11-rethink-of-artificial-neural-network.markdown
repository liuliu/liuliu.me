---
date: '2008-11-11 05:30:38'
layout: post
slug: rethink-of-artificial-neural-network
status: publish
title: Rethink of artificial neural network
wordpress_id: '397'
categories:
- 随感
---

I discard artificial neural network idea long time ago since its over-fitting problem and the ugly expression of back-propagate algorithm. It is hard to say bp is an elegant algorithm. It directly magnifies the influence of error with the gradient, and the hidden layer structure is highly depended on empirical data.

People are easily convinced by SVM, HMM or manifolding methods. They look elegant with great mathematic skills. Other methods such as PCA and LFD, which in fact largely depends on linear hypothesis earn its credit, too. ANN method in a long time was only applied by engineers and ignored in science community.

There are some problems in existing statistic learning methods. Modern methods are expected longer execution time, in some case, it is unbearable. Applying nonlinear SVM which requires many support vectors is a painful experience. Successful applications nowadays largely rely on specific structure. In face detection application, it is a degenerated high-dimensional surface approximation. In general recognition problem, people much more rely on good "features" which is an indeterminate problem itself. Thus, nearly all the state-of-art methods in image recognition are empirical results more than formal mathematic proves.

Despite the over-fitting problem which can be tuned by carefully testing, nn algorithms have some advantages. They could be deployed in online learning problem where other statistic methods may need a holistic distribution of data for further calculation. Hence that, I am investigating some modern nn models such as RBM these days.
