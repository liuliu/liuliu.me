---
date: '2012-09-08 22:36:00'
layout: post
slug: a-thought-experiment-of-modern-mobile-os
status: publish
title: A Thought Experiment of Modern Mobile OS
categories:
- eyes
---

Two weeks ago, after the Android training, I am starting a thought experiment on, given the state of current mobile hardware, what would be if we start to design a mobile OS (or more accurately, a mobile OS framework) from ground up? It is 2012, a mobile OS doesn't merely mean an abstraction above the hardware layer. It is a whole package specially designed to streamline the interaction of its application with graphic and user input systems. You are not required to redesign the file IO, or the network interface or the hardware interrupt. From an application developer's point of view, and the feasibility of the current mobile hardware, what's the most crucial parts of a new mobile OS framework? The set of choices presented here ranging from the language features, to the interaction with graphic system, and to the interaction between applications.

1). Animation as the First-class Citizen
========================================

Animation shouldn't be the after-thought. The new mobile OS framework should design its graphic / object system based on animations. In the language level, animation should be easily in-lined (a good example about what means to be in-lined is what LINQ to C#). Animation, fundamentally, is per-transition based, in a typical MVC arrangement, managing transition is a job of controller. It doesn't make sense to have a separate animation presentation in another language (CSS or XML). To say that later we can swap a better animation if we separate the two is an ignorance to the relationship between animation and interaction. There is only one or very small subset of animations make sense for a given interaction, the thinking of swapping different animations for one transition is fundamentally flawed.

**Implicit animation (as in iOS CALayer) v.s. explicit animation**: explicit. Having implicit animation while given developers the ability to specify animation means you have to have a mechanism to disable the implicit animation. That's just digging hole for the framework designer. Besides, for any quality interaction design, implicit animation is, always getting in the way. Whenever possible, framework designer should always give back the flexibility on choosing animations back to application developer. No implicit animation would avoid the confusion, and providing cleaner interface for both the framework designer and the application developer.

2). Message-passing, Multithreading and Event Loop
==================================================

Go with multiple event loops and message-passing. In the language level, parallel execution is formalized with channels and message-passing rather than locks and shared-memory. There will be no such thing as main thread throughout the UI framework. Every execution is asynchronous. It does impose problems when quick responses to touch events are required.

3). Key-Value-Observing and Well-defined Behavior
=================================================

Every property of every component, from UI to IO, can be observed. Even many UI events are delivered through KVO rather than a event callbacks. In the language level, these KVOs are asynchronously delivered by its message-passing system.

4). A Single True Persistent Object Management System
=====================================================

Framework shouldn't hide the performance implications of the persistent object system (thus, shouldn't provide a transparent persistent object system). But it should provide some kind of performance-explicit persistent object management system. This persistent object management system is the single recommended way to persist data, thus, free developers from dealing with different persistent object systems, and most importantly, developers can now make safe assumptions about the underlying data persistent mechanism to simplify their work on upper layers such as custom views etc.

5). Easy Interaction with C/C++
===============================

It would be an atrocity to not provide a way to compile with C/C++ libraries, in language level, this should be as simple as linking to the generated object file.

6). Garbage-collection
======================

This is controversial, but I believe that for any non-trivial parallel programs, memory / object life-cycle management is very difficult. Especially for a complex message-passing based system, managing the life-time of objects that passing around, potentially in a processing queue is a recipe to disaster. But all have been said, rolling out a half-baked garbage-collector is not an option for mobile OS because the real constraint on both memory and CPU. It does need careful profiling to see how far a garbage-collector can go even in current hardware environment.
