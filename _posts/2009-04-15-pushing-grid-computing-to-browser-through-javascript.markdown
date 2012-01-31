---
date: '2009-04-15 20:38:21'
layout: post
slug: pushing-grid-computing-to-browser-through-javascript
status: publish
title: Pushing Grid Computing to Browser through Javascript
wordpress_guid: http://jsms.me/?p=512
wordpress_id: '512'
categories:
- 随感
tags:
- grid computing
- javascript
- mapreduce
---

Half a year ago, I read an article about how to use simple javascript to perform MapReduce in browser. It is very interesting, but the author obviously ignored that the locality of MapReduce made it so good. It is not proper to introduce MapReduce to the scenario of browser because it solves data-intensive problem which bandwidth is critical (that is why Reduce part introduced).

However, the idea of making browser do some extra work is suitable for computing-intensive work which only requires little data. Someone is already on the track years ago by using Java applet or Flash. With Google Gears or even _setTimeout_, I believe it is very realistic now to introduce browser-based grid computing with Javascript.

More details about it will be revealed in July.
