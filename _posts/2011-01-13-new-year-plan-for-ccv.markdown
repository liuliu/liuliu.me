---
date: '2011-01-13 13:59:28'
layout: post
slug: new-year-plan-for-ccv
status: publish
title: New Year Plan for CCV
wordpress_id: '1148'
categories:
- eyes
---

I've developed ccv for about a good of year. In the beginning of September, I've set out the goal to finish 4 important applications in ccv: 1). a key-point detection/matching algorithm; 2). an object detection algorithm; 3). text detection algorithm; 4). 3d reconstruction algorithm. Now, the first two have already finished and released, and the development since has been stagnated for about two months, it is a good time to recap some of my thought on this project and what's the direction it will end up to.

ccv suppose to solve several problems in OpenCV. That means, modern/clear API interface, new compiler support/benchmarking, new architecture support, application-oriented development (only implement winning algorithms) and the dedication to open platform. Thus, ccv doesn't support Windows OS at all, and only works/targeting at 64-bit *nix OS. It also branded with new memory management routine which would "reuse" intermediate computations. All that are good.

But there are several problems to ccv, too. Since ccv was designed with server-side usage in mind, it is really not fit into portable device despite the small code-base. ccv's default behavior tends to use more memory (for caching intermediate result). Also, it was optimized for Intel x64 architecture and have a large dependency on various softwares (FFTW etc.). The dedication to open platform such as OpenCL is causing problem too, because no one in GPGPU community seriously work on applications with OpenCL, and the performance variation is even bigger than CUDA.

I will stop reflection here and list the new year's goal for ccv.

1). It will support RPC over HTTP, thus, enable users to use ccv just as easy as compile/running/making HTTP requests;

2). It will support portable devices (iOS and Android), using less memory with it (disable cache on-fly), with graceful dowgraded performance when depended library is not available (work, but slow without FFTW or cBLAS);

3). Migrating to CUDA/C interface instead of OpenCL;

4). Support of MPI/Hadoop;

Overall, ccv will keep its focus on server-side computation while expects to have modest performance on portable devices.
