---
date: '2009-04-05 15:03:17'
layout: post
slug: why-facool-failed-and-what-i-learned-from-it
status: publish
title: Why Facool failed and what I learned from it
wordpress_guid: http://jsms.me/?p=499
wordpress_id: '499'
categories:
- 随感
---

For those who don't know what Facool is, there is a video about it: [http://www.vimeo.com/1925998](http://www.vimeo.com/1925998)

It has been 3 years since the close of Facool in 2006. After working on serveral minor startup things, I still occasionally heard people's ask about why Facool failed at that time. I spend a lot spare time to think about it. Today it still seems to be a cool idea to put face retrieval technique online and there are many startups working on this (such as [face.com](http://www.face.com), [riya.com](http://www.riya.com) etc.). And now I think that I have a good perspective of why Facool failed.

Facool rolled out as an academic research result. It took me while to realize the economic potential and then I started to run it as an actual product. The year of 2005 is the time when everyone believes that search is the coolest stuff as SNS in 2007 and twitter in 2009. The idea is simple: to index all faces in the web and find it instantly. The missing point here is that the goal is too ambitious and the resource I could use is limited.

The shortage of resource can explain many negative facts that Facool encountered. First is the shortage of images. At 2005, Facebook just launched. There is no much good structural representation of personal information here on the Internet. By scraping 100,000 images, the detector found about 10,000 faces and most of them were low-resolution ones. You have to dig the deep net in order to find more useful information and due to the lack of structural information about person, I even have to develop a new algorithm to determine a person's name!

**Lesson 1 learned: start with a small thing, and evolve along the way.**

When Facool came out as a web service, I coded a web server from scratch which made me spend more time to take care of socket error, concurrency problem etc. To make a web server is a big time sinker, and even if it could take few percents advantage, it is not a convenient thing to start with. I actually spent 2 months to code the web server, comparing with now pile up a web service in 3 days with Django, I wasted too much time on unimportant stuff.

Contrarily, I was not a huge fan of opensource community at that period of time. In 2005, I only heard of OpenCV and never put real use of it. Without trying the power of opensource, I trained the face detector by my own. Which, no doubt, cost another 3 months to get a satisfactory result.

**Lesson 2 learned: saving time and avoiding reinvention of wheels, taking the power of opensource.**

When finally finished the beta version of Facool, I just about ran out of money. I spent about $5,000 to buy server and rent the bandwidth, left few bulks for living. It is hard to recall that just 3 years ago, there is no slicehost, no Amazon S3 and you have to startup with $2,000 server.

At June, I don't have one extra penny to pay the bandwidth, and that pretty much about it.

**Lesson 3 learned: startup with cheap stuff and save at least half of your money before the release day.**

Sometimes, I appreciate that I was failed so young and I have so much time to start over.
