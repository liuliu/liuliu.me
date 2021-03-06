---
date: '2009-08-27 07:34:35'
layout: post
slug: on-the-design-choices-of-ndqi
status: publish
title: On the Design Choices of NDQI
wordpress_guid: http://jsms.me/?p=520
wordpress_id: '520'
categories:
- eyes
---

In my [several](http://jsms.me/?p=450) [former](http://jsms.me/?p=484) [articles](http://jsms.me/?p=517), I mentioned a system from different degrees and now I think it is the time to bring the stealth project: NDQI (Non-structural Data Query Interface) to the spot light.

The decades study in content-based image search (CBIR) was focused on the accuracy. Some earlier researches in CBIR has shown that if the size of database expanded, the accuracy will be dramatically improved. Recent years, my research on CBIR has two directions, one is to scale out, the other is to make it more user friendly. Two years ago, the experimental software [ClusCom](http://www.slideshare.net/liuliu_1987/A-Scalable-Architecture-for-Distributed-Retrieval-System-in-High-Concurrency-Environment) tried to solve the first problem. Now, I believe that I have reached a point which I can solve the second problem, partially. Instead of pursuit "user friendly", I transformed the problem to "developer friendly".

NDQI promises to provide the same accessibility to multimedia content (currently, only still image is supported) as today's SQL system provided to structural data. It takes many good ideas from OSS and should be open-sourced in the future.****

**Basic Utilities
**

The idea of NDQI is to design a special-purpose database which can access multimedia data efficiently with simple, SQL-like language. The first concern is how the storage layer works. As for now, NDQI only works with still image, a 16-byte string is used to identify one image. A radix-tree like in-memory structure is the basic utility of NDQI which takes over the key-value storage scheme inside NDQI. The radix-tree like structure is designed for memory efficient situation and have a comparable performance with other in-memory data structure such as Google sparsehash/APR hashtable etc. However, the in-memory storage layer is not designed for storing images. Specially, it is designed to store the meta-data and other indexes. Where to store the image is really not my concern because there are already in-production solutions out there such as Facebook's Haystack.

The foundation of NDQI is a bunch of routines with c-style language. It is not really a c implementation because it depends on OpenCV lib. However, c-style interface is provided for the manually manipulation of database. Besides, it is natively thread-safe and take advantages of write-read lock internally. Thus, an upper layer (parser, indexer etc.) based on scripting language is possible.

**Database Types
**

Two types of database designed specially for multimedia content search is provided. The first type is called bags of words database. In this scenario, an input can be extracted to various fixed-length words. For example, a picture with N people can be explained as N words, each word stands for a people in the picture. Actually, for a wide range of image recognition problems, "bags of words" idea is a good generalization.

The second type is called fixed-length vector database. For still image, the fixed-length vector database can store some simple global features such as histogram/gabor filter etc. This kind of database provide a more superior speed performance than bwdb because it can be speed up with tree-based method or local sensitive hash. Actually, in reality, it outperforms bwdb by 100x speed up, both with indexes.

Other meta-data such as date, location, tags and camera types are stored within [Tokyo Cabinet](http://tokyocabinet.sourceforge.net/) which can do most things just like SQL-DB.

**Language Specs**

NDQI uses a set of SQL-like language for users to access the funtionalities provided by NDQI. It is SQL-like but unfortunately, not compatible with SQL. You can see more specifications on [http://limpq.com/docs/ndqi](http://limpq.com/docs/ndqi). Here I'll brief some of them. There is no real INSERT/REPLACE/UPDATE functionality as all images are uploaded and indexed automatically. INSERT and DELETE keyword can add/delete image and indexes from database immediately. SELECT keywords support nested SELECT natively which means query like "SELECT # WHERE lfd LIKE (SELECT # WHERE tag='tree')" can be performed efficiently with internal mechanism. # is a symbol in NDQI which represents the 16-byte string identifier of image. #F9IEkdfneI328jfek-3Et can define a specific image with identifier "F9IEkdfneI328jfek-3Et"(base64 encoded). Without point the table with FROM keyword, NDQI assume that the SELECT clause wants a global search. Actually, there is really only one big table and a smaller table can only be created runtime with SELECT clause.

User can modified several attributes with INSERT tag="whatsoever" INTO #F9IEkdfneI328jfek-3Et clause and delete with DELETE tag="whatsoever" FROM #F9IEkdfneI328jfek-3Et. You can also update attributes with WHERE clause: INSERT exif.gps.latitude=123.4 WHERE lpd LIKE #F9IEkdfneI328jfek-3Et.

**High-Level Architecture**

Though at first, I want to implement high-level functionality (parser, client-server app etc.) in scripting language, but in reality, I end up to program them all in C. The parser was implemented with lex and yacc; the server side was done by libevent. The workflow looks like this: the service was open through HTTP protocol. Once client side requests by "q=" parameter, server will try to parse the query into NQPARSERESULT PREQRY ">struct. Notably, it parses a SELECT clause into so-called PREQRY struct. A PREQRY may be nested that requires external knowledge to get through (contains references, subqueries).

**Embedded into Current Architecture**

NDQI is nothing more than a collection of database routine for multimedia content. Alter, the successor of ClusCom is responsible for scale NDQI to multi-machines and provide the network protocol access. Other than modern SQL-DB, the parser of QL is deployed in front. Because there is no join etc. functionality, it is really easy to scale. The front-end will parse a query into PREQRY, and "plan" the PREQRY into series of NQQRY struct with the help of nqclient.h. NQQRY is a stand-alone struct, which means that it relies on zero external knowledge. Front-end can just execute NQQRY on several computers and synthesize the returned results. To reimplement functions in nqclient.h is the most efficient way to embed with current scale architecture.

A new set of javascript APIs is also provided for users which can just "query" the server with a NDQI-valid query string. The functionality is called limpq.ndqi.Q.
