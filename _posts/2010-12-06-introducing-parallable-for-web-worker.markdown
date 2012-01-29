---
date: '2010-12-06 11:37:34'
layout: post
slug: introducing-parallable-for-web-worker
status: publish
title: Introducing Parallable for Web Worker
wordpress_id: '1103'
categories:
- Eyes
---

I've spent sometime this afternoon to perfecting the web worker implementation in [ccv.js](https://github.com/liuliu/ccv/blob/current/js/ccv.js). Something interested me is that maybe, I can implement a thing that makes web worker painless. The current web worker flow is: 1). you partitioned the work into many pieces of jobs; 2). create a small js file that explicitly handle one job; 3). create bunch of workers from the small js file and run; 4). collect results. The workflow is great in a way that it explicitly specified the message-passing path. I am a big fan of MP model for parallel computing (my professor Andrew Grimshaw has a big rant on shared-memory thing just every time you asked him) and the web worker just hit the right taste.

However, it does purpose a hurdle for library authors who want to utilize web worker. Maybe they can use web worker to run computation expensive job in parallel, maybe they want to make the interface more responsive by moving computational part to background. However, current web worker infrastructure requires a separate js file explicitly written for a single job. Javascript is already notoriously bad at its package management, and scattered web worker code in the universe may make it worse.

Identified the problem with web worker, I created the tiny code snippet called "parallable". It is so small that you can copy & paste it to your js file and instantly, you can write code that runs in parallel. Well, not instantly, parallable suggested a code convention for writing parallable functions. I will show you a full code that basically compute sum of elements in an array with parallable ([sum.js](https://github.com/liuliu/parallable/blob/master/sum.js)):



To conform the convention of parallable, you have to separate the function into 3 parts - pre, core and post. It is a process chain, in a way that pre will split input into appropriate parts, and pass each part to core. The core will process each partitioned data, and return part of result. post will gather all results and generate the final one. Only the core part will be run on the web worker. They do share some information which you can specified in this.shared structure, but don't assume any consistency in this.shared data, it will never be synchronized.

Despite the convention, the real part of parallable is in the beginning of the code. It wrapped original function declaration with _parallable("sum.js", function (list) ..._ where sum.js is the file name and function structure is untouched. You can think it as a decorator to original function. For unnamed argument function (the traditional javascript way), parallable will append the arguments with two new parameters: the first is async and the second is worker_num. For named argument, it will append the two directly. So, to call sum in synchronous fashion, you can just call _sum(some list, false, 0)_, and it will return the result once it done with data. If you set async to be true, it will spawn some number of web workers to do the job, and what it returned immediately is a [continuable](https://github.com/creationix/do), thus, you can do _sum(some list, true, 4)(callback)_ to spawn 4 web workers and get result in the callback.

Checkout [parallable code snippet on github now](https://github.com/liuliu/parallable/).
