---
date: '2011-03-05 03:57:18'
layout: post
slug: new-schedule-algorithm-for-my-life
status: publish
title: New Schedule Algorithm for My Life
wordpress_id: '1211'
categories:
- 随感
---

It turns out my old scheduler performed poorly recent days and things become hard to get done. The older one uses a FIFO queue with ad-hoc prioritize items, and the problem is, when an item takes really long time, it will block every consequent operations.

Without further due, I will present the new one. The new scheduler have a new ETC (Estimated Time to Complete) attribute which indicates the deadline for a particular job. It still has a FIFO queue structure, such that any new job will enter the bottom of the queue, with a ETC. Job on the top of the queue, if completed, will be removed; if the ETC reached, will request a new ETC, save the context, and put it in the bottom of the queue again.

The main take on the new scheduler is that, it will pursuit earliest start time rather than earliest completion time in order to be "fair". Assume no priority setting, each job will have a strict start time.

I will experiment the new scheduler in the next few days.
