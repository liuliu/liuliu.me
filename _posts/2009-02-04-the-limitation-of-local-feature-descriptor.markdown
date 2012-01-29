---
date: '2009-02-04 07:29:04'
layout: post
slug: the-limitation-of-local-feature-descriptor
status: publish
title: The Limitation of Local Feature Descriptor
wordpress_id: '467'
categories:
- eyes
---

Local feature descriptor (LFD) is an overwhelming successful method for image comparision which is currently the best solution against in-plane-rotation, distortion and light variations. However, there are some assumptions should be noticed. LFD is a appearance feature. It is more stable than pixel feature, but after all, the property of appearance feature still disturbs the ability of LFD. First, the appearance feature is vulnerable to variations of light and the description ability is depended by the complexity of appearance. LFD by collecting features through the key areas counteract the interferences of image variations. You can view that as sort of extension to shape description. Still, for object with low complexity of appearance, LFD failed to achieve any thing. The test senerio will be a color ball with a complex background. For this senerio, LFD cannot capture any useful information of the ball because the distinguish of the ball is the shape not appearance. For that part, LFD can do little.

One way to solve the problem is to combine some sort of shape descriptors to our LFD machanism. As appearance features and shape features are total different in their domain, there is no simple way to do this. There are rare cases that shape is more powerful than appearance. A closer look at how shape affects the appearance we will notice that only smooth boundary is something LFD cannot describe well. For sharp boundary and turnning point (corner point), LFD algorithm can effeciently extract local descriptor from those key point. In our extreme case, LFD lost its most important function to keep track the percise location of key point. For a ball, there is just no way to judge which point is key point on the boundary of circle.

Another limitation that is heavily addressed is the computing complexity of LFD. In my [previous](http://jsms.me/?p=450) [articles](http://jsms.me/?p=436), I provided some insights of how to solve the problem.
