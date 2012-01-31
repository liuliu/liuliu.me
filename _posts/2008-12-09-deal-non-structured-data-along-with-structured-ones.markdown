---
date: '2008-12-09 11:11:05'
layout: post
slug: deal-non-structured-data-along-with-structured-ones
status: publish
title: Deal Non-structured Data along with Structured Ones
wordpress_guid: http://jsms.me/?p=399
wordpress_id: '399'
categories:
- 随感
tags:
- fuzzy set
- structure
---

Several attempts have been made to make semi-structured information more structuralize. The central concept of semantic web is about universal form of the knowledge we have. Freebase is a highly structured information base, however, Wikipedia, the world largest encyclopedia, only have semi-structured data. In CIKM 2008, the awarded paper is about extract structured information from Wikipedia database. Basically, there are more semi-structured information than full-structured one. Another problem is about the massive, poorly organized data, for instance, the photos. Flickr made a good attempt in exploit human resource to organize data. However, there are less photos are tagged. Luckily, camera manufacturer came up with EXIF which can embed combined camera sensor's information into a photo. But time-dimension and geo-dimension is too vogue to fit in specific usage. Overall, with years efforts, we have pretty much structured or semi-structured data in hand.

The mixed data structure is organized in key-value form. An element can be described with several properties. These properties can be structured or non-structured. Here we recognize semi-structured property as non-structured, too. There are several questions remain unclear, for example, how to form a query in mixed data structure? How to slice data based on its mixed properties? In this article, we simply ignore these questions. So, we directly jump to how to fulfill a query. Once a query was made, firstly we break up the structured information. The break up process, was described in fuzzy set area for years. We used one assumption in this process: any data relation can be illustrate with similarity. It is a very big hypothesis, besides, we leave difficulties here for ourselves which I will discuss later. However, illustrate data with only similarity can simplify the problem. To fulfill query, we only have to sort based on the similarities.

I have to suggest several considerations in this process. First, in many cases, the structured data cannot be simply measured by one similarity method. For example, to fuzzy datatime field, we can only measure the time span between each other. Then, how we compare May 11, 2008 and May 14, 2006? The two date definately share some common, they are all mother's day. The second problem is about computing time. However, the similarity matrix was very spare, thus, it should reduce some calculation time.

The idea of fuzzy is not new. It came from multi-value logic and soon adapted to computer science. The idea I suggest here is about to form query and retrieve in database where data is poorly organized.
