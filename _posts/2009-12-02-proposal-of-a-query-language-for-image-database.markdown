---
date: '2009-12-02 11:57:49'
layout: post
slug: proposal-of-a-query-language-for-image-database
status: publish
title: Proposal of a Query Language for Image Database
wordpress_id: '712'
categories:
- eyes
---

**Introduction**

In past decades, the maturity of hardware and design of large-scale system incubated number of state-of-art image retrieval systems. Early days' research includes query-refine-query model which people believe would eventually approximate the desired picture through this kind of "boosting" process. Later researches are more focus on the accuracy of image retrieval. Instead of naive global features such as color histogram or global momentum, a class of local feature descriptors are proved to be more accuracy and robust for deformation and lumination change. Some research show scale does matter as they reveal that the result on 10 million scale is much better than it on 100,000 scale.

Recent years, the scale problem becomes the central interests in CBIR system. M-trees, local sensitive hashing are named few that showed successful application in consumer market. However, all the efforts are paid to improve the recall rate of similarity measure itself is not flawless. The class of local feature descriptors are intended to capture identical objects in the scene which has little ability to derive similarity model for an object. Some recent local feature descriptors such as self-similarity descriptor shows its potential to reveal visual structure of object and ability to capture correct object with hand-sketch. But these additional abilities only make it more vague about what is the intention of querist for ''similar'' image if only one image or even chain of images (like what we do in old days) are provided.

Full-text search system has this fuzzy too. Usually, people have to iterate several times in order to get better results. But for structured text, there is no such problem. Once text is formalized, the meaning is clear and can be calculated. Any formalized language, such as structured query language, can analyze formalized text.

Because the lacking of image database architecture that sophisticated enough for interesting applications. Many existing applications that ultimately utilize image database end up to create the image database from scratch.

Developing a new language for query and manipulate image database will strength the image database to fit more challenging problems. Further more, as a language, it can liberate developers from the details of implementation and focus on interesting applications.

**Targeting Database**__

_Database definition_

I narrow the database that the language operates on to image database. Especially, it has no relations between whatsoever. Image database contains several image objects. These objects has a full list of properties, such as EXIF header, and global/local feature descriptors. There are mutable and immutable properties. For most text/number fields (such as EXIF header), they are mutable. All feature descriptors are immutable. You can certainly fork a new image that has different feature descriptors which will describe later.

_Database discussion_

Because there is no relations between image objects, a SQL-like query will downgrade to combination of binary operators. Only queries on text/number fields is not good enough for image database. After all, content-based methods are hard to embedded into the SQL-like query language.

**Language Objective**

The language for query should have minimum syntax. It should easy to query, and easy to manipulate and output human-readable result set. It has fewer keywords (possibly no keywords) and extensible properties. Property fields should be scoped and protected to minimize developer's mistakes. It should be as powerful as any query language on text/number fields. There should be no magic, every syntax is explainable by underlying mechanism and no special methods that requires specific knowledge to interpreter. It is not a general purpose language, so that Turing-complete is not important. If it is possible, make the language have restricted data dependency that benefits parallelism. By all means, it can really reduce the workloads for developing real-world applications based on image database.

**Language Examples**

Before dig more into the details of the language, I'd like to present several examples to show why it has to be in this particular form.

_Estimating geographic information from single image_

This is a research project in CMU. Here I will rephrase their main contribution with the new language.

    
    
    q(gist(#OwkIEjNk8aMkewJ) > 0.2 and gps != nil) (
    	oa = nil
    	a = ^.reshape(by=[gist(#OwkIEjNk8aMkewJ), gps.latitude, gps.longitude])
    	while (a != oa) (
    		oa = a
    		a = a.foreach() <$> (
    			r = ^.q(gps.latitude < $[1] + 5 and
    				gps.latitude > $[1] - 5 and
    				gps.longitude < $[2] + 5 and
    				gps.longitude > $[2] - 5)
    			? (r.size < 4) (return yield nil)
    			ot = r.gist(#OwkIEjNk8aMkewJ)' * r.gist(#OwkIEjNk8aMkewJ)
    			yield [ot,
    				r.gist(#OwkIEjNk8aMkewJ)' * r.gps.latitude / ot,
    				r.gist(#OwkIEjNk8aMkewJ)' * r.gps.longitude / ot]
    		)
    	)
    	return [a.sort(by=@0)[0][1],
    		a.sort(by=@0)[0][2]]
    )
    


The first impression is that it is similar to functional programming language. Some syntax are very similar to Matlab. The language natively support matrix/vector operation and hope to speed up these operations. Though syntax are similar, while and foreach are not keywords any more. They are ordinary methods.

The script above first takes images that similar to \#OwkIEjNk8aMkewJ (image identifier) on gist feature with threshold 0.2 and have gps information. The while loop is to calculate windowed mean-shift until it converged. Then return the location with maximal likelihood.

It is an interesting case to observe how the philosophy behind interact with real-world case.

_Sketch2Photo: Internet Image Montage_

It is a research project in Tsinghua University which presented on SIGGRAPH Asia 2009. It uses a rather complex algorithm for image blending, here I will only use poisson edit technique instead.

    
    
    q() (
    	sunset_beach = ^.q(tag="sunset" and tag="beach").shuffle()[0].image
    	wedding_kiss = ^.q(ssd(#Le6aq9mkj38fahjK)).sort(by=ssd(#Le6aq9mkj38fahjK))[0].image(region=ssd(#Le6aq9mkj38fahjK))
    	sail_boat = ^.q(ssd(#ewf_kefIwlE328f2)).sort(by=ssd(#ewf_kefIwlE328f2))[0].image(region=ssd(#ewf_kefIwlE328f2))
    	seagull = ^.q(ssd(#94xJ9WEkehR82-3j)).sort(by=ssd(#94xJ9WEkehR82-3j))[0].image(region=ssd(#94xJ9WEkehR82-3j))
    	sunset_beach.poisson(at=(120, 100), with=wedding_kiss)
    	sunset_beach.poisson(at=(50, 50), size=50%, with=sail_boat)
    	sunset_beach.poisson(at=(240, 10), size=20%, with=seagull)
    	return sunset_beach
    )
    


The code is very straightforward. Here I ignore the fact that the real system has user-interactive part and just stick everything together.

**Language Syntax (Draft)**

_Type_

There are seven types in this language: nil, Boolean, Number, String, Image, Object, Array. All six types are very intuitive, the reason why makes Image as a basic type is to satisfy the human-readable philosophy. Though every image can be represented by a 3-D array, it is not feasible to output a huge 3-D array to end-user. We human need a more readable case for image output. Only support in language level will give such flexibility.

_Keyword_

There are two keywords in this language: \emph{yield} and \emph{return}. But I am intended to eliminate these two in the future. There are several special characters however that is useful. \^\ is the same as \emph{this} in traditional language. $<$ will refer to last condition statement. array.@n where n is an integer, is the same as array[n] which will serve as identifier for indices.

_Function_

Function is in the core aspect of the language. However, it is arguably if it is useful to enable user-defined functions in such a light-weight language. Functions in this section are all about built-in functions.

For built-in function, there are two parts: \emph{condition} and \emph{action}. Form a call to function is flexible, you can ignore the \emph{action} part or both. A function call take this form:

\emph{function} (\emph{condition}) $<$\emph{self}$>$ (\emph{action})

For common case, \emph{action} part will executed when the function ends. But it solely depends on how the function's decision about what to do with action. However, one thing is certain, with \emph{return} keyword in \emph{action}, it will immediately return whatever value to the caller. The \emph{yield} keyword will, on the other hand, return value to the function itself. \emph{self} part is optional, it specifies how the script in \emph{action} part refer to the result value of function itself, by default, it is \^\ .

_Control Structure_

There exist basic control structures in this language such as while, foreach and if. However, they are all functions now. In the two examples provided before, you can see how it functions. Since all functions are taking \emph{condition} as parameter, it is very natural to make if and while as a function. foreach is not a global function, it is scoped to only result set. Usually \emph{yield} are coupled with foreach to generate a new result array. You can use keyword \emph{return} to jump out of a loop of if statement if you want. To jump out of a nested function call, you need to nest the \emph{return} statement.

_Script Structure_

A simple usage of this language can only take advantage of its query feature, call q function.

q(tag="rose")

More sophisticated operation requires to extend the q (like what I do in the first two examples). It looks much like C's main function statement. But for outside observer, the only input and output can achieved through return value and calling condition instead of standard i/o stream.
