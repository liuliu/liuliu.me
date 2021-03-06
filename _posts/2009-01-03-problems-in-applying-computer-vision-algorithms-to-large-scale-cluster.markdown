---
date: '2009-01-03 14:03:43'
layout: post
slug: problems-in-applying-computer-vision-algorithms-to-large-scale-cluster
status: publish
title: Problems in Applying Computer Vision Algorithms to Large-scale Cluster
wordpress_guid: http://jsms.me/?p=421
wordpress_id: '421'
categories:
- 随感
tags:
- computer vision
- failure
- framework
- parallel
---

Research in computer vision put the reliability problem aside and hold the principle that computer never goes wrong. Even with limited computing ability, let's say, the embedded system, although the memory resource and CPU were very limited, the system is reliable by default.

Distributing computer vision problem to multi-computer is not new. For many systems, to fulfill the real-time requirement, several computers are used. The three-tier NASA robotic framework was first implemented in three computers. Stanford winning Stanley vehicle was utilize three PCs to perform the decision-making, environment detecting and computer vision tasks. The small clusters (typically under 10 PCs) do not worry about system reliability because under that situation, the probability of system failure is ignorable.

For large-scale cluster, it is simply not true. The large-scale cluster, which contains at least 1000 commodity PCs. At that scale, single fatal failure happens all the time. In best wish, the system failure should be taken care by lower facilities. Many training process can be implemented by MapReduce like mechanism which should not be worried about. But as large part of computer vision algorithm concern about real-time task, taking account of the system failure in local implementation is inevitable.

A desirable low facility for real-time computer vision task has to be very flexible. It can be reorganized quickly after a single-node failure. The two phase design of MapReduce may be still in use, but the algorithm applying to the two phase procedure need to be reconsidered. Many algorithms just simply are not fit to the two phase idea.

When the highly reorganizable facility is accomplished, the problems are left for in the algorithm layer. The paralleled version of SVM was published in 2006. The parallelization of many well-known algorithms was just happened few years ago. But considering the good fact of offline parallel structure, the tricky part would not be the parallelization of algorithms. Contrarily, how to online all these offline algorithms can be a very challenged task. Even the famous metrics-tree (or best-bin-first search tree) cannot easily perform insert/delete node, how such as PCA/LLE become online algorithms?

As there are so many unknowns exist, all these problems forms the very bright future for distributed computer vision framework.
