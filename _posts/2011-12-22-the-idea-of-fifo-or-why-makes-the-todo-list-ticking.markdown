---
date: '2011-12-22 05:54:21'
layout: post
slug: the-idea-of-fifo-or-why-makes-the-todo-list-ticking
status: publish
title: The Idea of FIFO, or Why Makes the TODO List Ticking
wordpress_id: '1400'
categories:
- Eyes
---

Yesterday, I started to work on a new kind of TODO list, or a fancier way to call it, "a personal deadline scheduler".

TODO list is easy to write. The idea is simple. Having list of items, and assigning each of them with the status (completed, in-progress, abandon etc.). A fancier one may have participants, milestones and deadlines. These ones are called task trackers. TODO lists tend to be small, they cover things that can be done in few hours most. Task trackers tend to cover multi-day ones. But neither of them "tick". Thus, there is no way to prevent one item sit on your TODO list for a year and still have zero progress.

Speaking of progress, it is something subjective and in general hard to measure. For example, you can sit there for a whole day and still have zero line of code written. Many task trackers let you to estimate your daily progress and often suffer overshot or under. It is not a great way to meet deadline because it never saves you from be stalled. FIFO is different. FIFO aims at things that takes multi-hours but less than two or three days, such as a prototype of a feature, a test suite, a minor feature or few bug fixes. It doesn't measure your progress, but in general, it will help you make progress on ALL the items.

The secret sauce is "ticking". Once you started a FIFO item, the clock started to tick. For a item in FIFO, you needs to specify two things, the item name, and the estimate time required. In current implementation, you can specify the two with one sentence, like "CUDA on-device slab pool, 3 hrs". Once the item is entered, it will start to record the time spent. You can pause the timer at any time or resume, but still, there is a timer that records time spent.

The beautiful part comes when you have several items at hand. Whenever you don't want to work on the current item, you can click the "R" button, and that item will be put to the end of the list. Another way to move on is when the interval meets. The default interval is 1 hours. Thus, when you have spent more than 1 hours on this task, it will be automatically moved to the bottom. In other word, it works exactly like a deadline scheduler in your Operating System, which keeps you to make some progress on each of the items instead of having one or some of them sit there for a whole year.

At this time, I only have a demo at [http://fifo.me/](http://fifo.me/), but feel free to try it out and comment. Things like server-side persistence, Facebook frictionless sharing will come. Trust me, I will make some progress, because I am using FIFO now.
