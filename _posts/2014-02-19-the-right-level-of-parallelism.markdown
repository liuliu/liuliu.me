---
date: '2014-02-19 22:03:00'
layout: post
slug: the-right-level-of-parallelism
status: publish
title: The Right Level of Parallelism
categories:
- eye
---

I was not a fan of low-level parallelism exploitation. The level of parallelism that OpenMP tries is too easy to get wrong. On the other hand, I'd prefer dispatch_apply in any cases for the precise reason to disfavor OpenMP: it is restrictive enough to be only useful for embarrassing parallel problems.

The belief I subscribed to as stated above comes from Amdahl's law; and in my simplified world, the more points you have dispatch / join, the more sequential execution portion you will end up with. Therefore, further limits what you can gain from parallel executions. In the same spirit, I'd prefer long-running processes with limited message-passing at any given time.

It's not just talking points. In past versions of ccv, I never exploited thread-level parallelism to speed up ccv's core functions (SIMD is not in the discussion, it is total awesomeness.) for the simple belief that as long as a given function can finish under a reasonable time, thread-level parallelism should be left for the upper layer of your application stack. When you are running ccv on iOS or Android, even though there are multiple cores, you probably don't want ccv_icf_detect_objects to occupy all of them while executing. On the other hand, if you deploy ccv to a server environment, you probably want to pin process per core, and don't have ccv to over-reaching other cores for parallelism (as hopping could be expensive).

At least that was what I believed until very recently.

Now looking back, the idea of that there exists a right level of parallelism is probably wrong. Until past three or four years, we have a very coarse parallelism levels (I am discounting SIMD again as it is for me a processor hack). We have processes, which can live in remote or local, but any communication between them are done explicitly via messaging. We have threads, which lives locally (or for most of threads), and probably no formal messaging channel; the communications are done implicitly via shared memory and synchronization. Most of the time, register, L1/L2 cache, and memory access latencies are the separated concerns and left for specific optimization. In that pretty coarse world, communications done with shared memory access, which assumes uniform latency, or with messaging, which holds to be expensive (high latency, and the cost of memory copy). The de-facto way to do parallelism in that world, as I previously said, is to go with embarrassing parallelism (avoid communication at all cost, and prefer high throughput / high latency alternatives over low throughput / low latency alternatives) and see how far you can get away with it.

But during the past few years, we have had a much finer-grain parallelism model, which specifies access latency with regards to different parallelism options. Even discounting the rising tide of GPGPU programming, we have much more CPU cores on a single machine (thread + shared memory) and the performance penalty of not aware of the non-uniform memory access pattern will be unforgivable.

I don't have a good plan in mind about how to adapt to this as reality is still very messy. However, there are some interesting observations: 1). in heterogeneous computing environment, we are still playing the old game of balancing out communication cost with computation cost, but in very different ways for different platforms; 2). thus, it is unlikely one well-crafted kernel will perform universally well; include a set of tunable kernels, as the case in libatlas or fftw3 should be the common practice when delivering performance centric software; 3). because it is impossible to have the right level of parallelism, exploit parallelism structure at every level, and let tuners / schedulers to figure out how to adapt to a specific computing environment is the better bet.

It would be quite some fun to experiment how to exploit parallelism at thread-level for functions like ccv_icf_detect_objects / ccv_dpm_detect_objects without exhaust CPU resources on mobile devices with ccv in the future.