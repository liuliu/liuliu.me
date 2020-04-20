---
date: '2020-01-29 17:11:00'
layout: post
slug: the-fun-part-of-programming
status: publish
title: The Fun Part of Programming
categories:
- eyes
---

Yesterday, I was listening to an interview by [Oxide Computer people with Jonathan Blow](https://oxide.computer/blog/on-the-metal-9-jonathan-blow/) on my way back to San Francisco. There were usual complaints about how today's programmers buried themselves into the pile of abstractions. As a game programmer Jonathan Blow himself, they also discussed some fond memories about programming basic games in his childhood. All that kept me thinking, how uninteresting today's programming books are! They start with some grand concepts. Compilers! Deep learning! GPGPU programming! SQL! Databases! Distributed systems! Or with some kind of frameworks. React! iOS! TensorFlow! Elasticsearch! Kubernetes! Is it really that fun to learn some abstractions people put up with? Seriously, what is the fun in that?

Over the years, I learned that there are two kinds of people. The first one loves to create programs that people can use. They take joy from people using the program they create. The magic satisfaction came from observing people using this thing. The second one loves to program. They take joy from solving the interactive puzzle through programming. The fact that the program can do non-obvious tasks by itself is enjoyable, no matter whether these tasks have practical use or not. As for myself, I love to solve puzzles, and understand every last detail about how these puzzles are solved. At the same time, I take pride in people using the software I built.

My earliest memories with programming came from Visual Basic and [Delphi](https://www.embarcadero.com/products/delphi). These were RAD tools ([Rapid-Application-Development](https://en.wikipedia.org/wiki/Rapid_application_development)) back in the late 1990s. They were integrated environments to shoot-and-forget when it came to programming. They were not the best to help understand computer architecture ins-and-outs. To some extent, they were not even that good at developing efficient programs. But there are two things they did really well: 1. it was really easy to jump in and write some code to do something; 2. things you made can be shared with others and ran on most Windows computers like the "real" applications would do. At that time, there were a lot of magazines and books that teach you how to make useful things. From a simple chat room, to a remake of the [Breakout game](https://en.wikipedia.org/wiki/Breakout_clone), you can type in the code and it would run! Then there were spiritual successors. Flash later evolved into [Flex Builder](https://en.wikipedia.org/wiki/Adobe_Flash_Builder), that meant to use Java-like syntax but preserves the spirit of RAD environment. As of late 2000s, you could build a SWF file and it would run almost everywhere. There were millions of amazing games built with Flash / Flex Builder by amateurs now live in our collective online memory.

Writing the iOS app in the 2010s somewhat gave me similar feelings. But the wheel moved on. Nowadays, we have MVVM / VIPER / RIB patterns. We have one-way data flow and React. We invented concepts to make programming more robust and productive in industrial settings with these abstractions. But the fun part was lost.

That is why this year, I plan to write a series to remind people how fun it is to program. It won't be a series about frameworks and patterns. We will pick the simplest tool available. We will write code in different languages if that is what’s required. We will maintain states in globals when that makes sense. We will write the simplest code to do fun things. It will work and you can show it to your friends, distribute it as if it was made by professionals.

I would like to cover a broad range of topics, but mostly, just practical things you can build. There certainly will be a lot of games. Some of the arrangements I have in mind, in this particular order:

 * A street-fighter like game. Introduce you to game loops, animation playback, keyboard events and coordinate system.
 * A remake of Super Mario 1-1 with a level editor. With physics simulation, mouse events and data persistence.
 * A chat room with peer-to-peer connection over the internet. Introduce the ideas of in-order message delivery and the need for protocols.
 * Remake Super Mario into a multiplayer side-scrolling game like Contra (NES). (this may be too much plumbing, I need to feel about it after the first 3 chapters).
 * Chess, and the idea of searching.
 * Online Chess with people or more powerful computers.
 * Secure communication through RSA and AES.
 * Why don’t implement your own secure protocols (show a few hacks and defenses around the protocols above).
 * Geometry and explore the 3D of DOOM. Introduce the graphics pipeline. I am not certain whether to introduce GPU or not at this point.
 * Face recognition with a home security camera. Introduce convolutional networks and back-propagation. Likely use a simple network trained on CIFAR-10, so everything will be on CPU.
 * Convolutional networks and Chess, a simple RL.

There are many more topics I’d like to cover. I would like to cover some aspects of natural language processing through machine translation, either RNN or Transformer models. It is however challenging if I don’t want to introduce GPGPU programming. I also would like to cover parsers, and a little bit of persisted data structures. But there are really no cool applications at the moment with these. Raytracer would be interesting, but it is hard to fit into a schedule other than it looks kind of real? Implementing a virtual machine, likely something that can run NES games would be fun, but that is something I haven’t yet done and don’t know how much plumbing it requires.

All the arrangements will be built with no external dependencies. We are going to build everything from scratch. It should run on most of the platforms with a very simple dependency I built, likely some kind of Canvas / Communication API. This is unfortunate due to several factors: 1. We don’t have a good cross-platform render API except HTML / JavaScript / TypeScript. 2. Most of our devices are now behind NAT and cannot talk to peers through IP addresses. The Canvas API would provide simple controls as well, such as text input boxes and scroll views. That also means the API will be pretty much in [retained mode](https://en.wikipedia.org/wiki/Retained_mode).

For the tool of choice, it has to be a language that professionals use. There are quite a few candidates nowadays. Python, Ruby, Julia, Swift and TypeScript are all reasonable choices. TypeScript has excellent cross-platform capability and I don’t really need to do much for the Canvas API. Python and Ruby all have libraries you can leverage to do both the Canvas API and Communication. However, I want to do a bit more raw numeric programming. For the speed, Python, Ruby and TypeScript are just not that great. Yes, there is numpy / numba, but what is the fun if I start to call numpy, PyTorch and millions of other Python packages do anything and everything for me? For Julia, I simply need to build too much myself to even get started.

There are many downsides with Swift too. For one, I still need to build a ton to support Windows and Linux. The language itself is too complicated especially with weak references and automatic reference count. Luckily, early on, Swift subscribed to the progressive disclosure philosophy. I will try to avoid most of the harder concepts in Swift such as generics, protocols and nullability. Will try to delay the introduction of weak reference as late as possible. Whenever there is a choice between struct and class, I will go with class until there is a compelling reason to introduce struct in some chapters. I also don’t think that I need to introduce threads or GCD. This probably depends on whether I can come up with an intuitive Communication API.

For the platform to run, I will prioritize macOS, Windows 10 and Ubuntu Linux on [Jetson Nano](https://developer.nvidia.com/embedded/jetson-nano-developer-kit). Keyboard and mouse will still be assumed as main input devices. Jetson Nano would be a particularly interesting device because that would be the cheapest to run with some GPGPU programming capability. I am not certain whether I want to introduce that concept. But having that flexibility is great.

Interested?
