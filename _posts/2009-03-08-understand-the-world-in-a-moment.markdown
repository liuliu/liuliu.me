---
date: '2009-03-08 18:53:25'
layout: post
slug: understand-the-world-in-a-moment
status: publish
title: Understand the World in a Moment
wordpress_id: '484'
categories:
- 随感
tags:
- computer vision
- MSER
- query language
- SQL
- SURF
- table engine
---

The holy grail of computer vision is to understand the scene, and output with proper language. In ideal senerio, it should be able to answer questions like "how many people visited our college this afternoon" or at least given result to a query like "SELECT COUNT(*) FROM Camera1|Camera2|Camera3->VideoStream, DateTime->VideoStream WHERE FaceDetector LIKE (SELECT FaceDetector WHERE Tag=Face) AND Time > '12:00:00' AND Time < '23:59:59'".

In this senerio, we extracted a goal that with existing technology could be achieved. Other than to distort structural data which introduced in [the article](http://jsms.me/?p=399), here we try to structuralize visual data. The result of the effort is a new structural query language for visual data. It should be a subset of existing SQL and more over, ideally, it should be able to collabrate with other SQL engines. In practice, we sacrified the compatibility with exisiting SQL frontface in order to get better performance. As a result, we yield a incompatible query syntax with SQL. In fact, for the current stage, I'd better describe it as a process to find similar visual data.

Visual data is processed with several different feature extractors when the first time put in database. The different feature representations become columns for every visual data. It also generate a unique fingerprnt in order to avoid duplicate visual data. To interact with text, tags are introduced again. Every piece of visual data can be tagged. With tags, one can write a nested query to do classification like the query in the beginning, it is actually a kNN classification. The feature extractor is atomic coomponent in the construction. Three feature extractors are provided in the start: face extractor, SURF extractor and MSER extractor. The structure is fleaxible, any extractor can be added later. Ideally, the extractors should provide a function to measure similarity between two visual data. In current case, it would be a huge performance penalty. To avoid these penalities, several helper extractors are introduced. The general extractors can have very different output, it can be variable length, binary data, or serialized data. a list, or a tree. However, the helper extractors output fixed length float sequence; they also have a implication that they can be measured by L1/L2 distance. In the system, we use helper extractors to [mimic](http://jsms.me/?p=450) the similarities that are output by general extractors. Three helper extractors are used: global histogram, local histogram and gabor filter.

Overall, it looks far from a query language. One may think it as a table engine like what MyISAM/InnoDB does. Giving it time, maybe it could be more powerful, who knows?
