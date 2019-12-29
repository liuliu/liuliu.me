---
date: '2019-08-02 18:53:00'
layout: post
slug: reduce-another-70-memory-usage-for-deep-neural-network-training-over-mixed-precision-with-tensor-compression
status: publish
title: Reduce Another 70% Memory Usage for Deep Neural Network Training over Mixed-Precision with Tensor Compression
categories:
- eyes
---

To train large deep neural network, you need a lot of GPU and a lot of memory. That is why a Titan RTX card cost more than 3 times of a RTX 2080 Ti with just a bit more tensor cores. It has 24GiB memory and that makes a lot of models much easier to train. More memory also means bigger batch size. Many GPU kernels run faster with larger batch size. If somehow we can reduce memory footprint at training time, we can train bigger models, and we can train with larger batch size faster.

There are methods to reduce memory footprints. It is no-brainer nowadays to use [fp16](https://en.wikipedia.org/wiki/Half-precision_floating-point_format) for training. Other than that, many of today's memory reduction techniques are derivatives of [binomial checkpointing](https://openreview.net/forum?id=BkYYXJ9i-), a well-known technique in automatic differentiation community. Specific details need to be considered that [cheap operations such as batch normalization or RELU results can be dropped and then recomputed later](https://arxiv.org/pdf/1604.06174.pdf). [The paper](https://arxiv.org/pdf/1604.06174.pdf) suggested a 30% more time required for DNN-tuned binomial checkpointing for roughly 80% reduction in memory usage. In practice, people often see 10% more time with 50% reduction in memory usage thanks to optimizations in forward pass over the years.

In the past a few days, I've been experimenting with another type of memory usage reduction technique.

It is common today in operating systems to do something called [virtual memory compression](https://en.wikipedia.org/wiki/Virtual_memory_compression). It uses data compression techniques to compress under-utilized pages, and on page fault, to decompress these pages back. These are lossless compressions. It doesn't make sense to revisit some memory and suddenly an 'a' becomes a 'z'. However, in another world, lossy compression does used to reduce memory usage.

In computer graphics, a full-blown 32-bit texture could take a lot of memory. People exploited more effective texture representation for ages. Formats such as [PVRTC or ETC](https://en.wikipedia.org/wiki/Texture_compression) rely on heavy compression schemes (many involve search a space for better representations) to find perceptually similar but much smaller texture representation. For example, PVRTC2 could spend less than 15% memory for visually the same result as a full-blown 32-bit texture. These compression schemes are also very light and predictable to decompress.

There are certain similarities between textures and tensors for convolutional neural networks. They both have spatial dimensions. Convolutional neural networks traditionally have more precisions, but nowadays we are exploring 4-bit or 8-bit tensors for convolutional neural networks too. For a tensor compression algorithm to work in practice, it needs to be fast at both compression and decompression on GPU, and hopefully, has high fidelity to the original.

I've devised a very simple, very easy-to-implement adaptive quantization algorithm for this purpose. The past a few days, I've been experimenting on ResNet-50 models to confirm its effectiveness.

At batch size 128x4 (4 GPUs, 128 per GPU), the baseline ResNet-50 model trained on ImageNet reached single crop top-1 accuracy 77.6% with 20.97GiB memory allocated across 4 GPUs. The ResNet-50 model with tensor compression trained on ImageNet reached accuracy 75.8% with 6.75GiB memory allocated.

On each feature map, within a 4x4 patch, we find the max value and the min value. With these, we have 4 values {min, max - min) / 3 + min, (max - min) * 2 / 3 + min, max}. Each scalar within that 4x4 patch can be represented with one of the 4 values. Thus, we use 2 bits per scalar. That totals 64 bits per patch, 25% of the original (assuming fp16). This is super easy to implement on GPU, in fact, I am surprised my simple-minded implementation on GPU this fast. It incurs less than 10% runtime cost during training (throughput reduced from 1420 images per second to 1290 images per second).

It is also simple to update the computation graph for tensor compression. For each convolution layer's output tensor, if it is used during backpropagation, we compress it immediately after its creation in forward pass, and decompress it before its use in backpropagation. If the backpropagation of the convolution layer uses a input tensor, we compress it immediately after its creation in forward pass, and decompress it before its use in the backpropagation. This simple scheme covered all tensors potentially have spatial redundancy.

Is this algorithm useful? Probably not. As long as there are accuracy loss, I am pretty certain no one will use it. At this moment, it is unclear whether 2-bit is too little or this whole scheme inherently doesn't work. Some more experiments are required to determine whether adaptive quantization is good enough or the spatial redundancy plays a role (by adaptive quantize across feature maps rather than within a feature map). Nevertheless, I'd like to share these early results to help the community determine whether this is a worthy path to explore.

You can find the CUDA implementation of the above adaptive quantization algorithm in: <https://github.com/liuliu/ccv/blob/unstable/lib/nnc/cmd/compression/gpu/ccv_nnc_lssc_gpu_ref.cu>