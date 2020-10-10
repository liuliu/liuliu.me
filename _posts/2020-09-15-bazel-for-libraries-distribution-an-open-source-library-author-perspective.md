---
date: '2020-09-15 21:02:00'
layout: post
slug: bazel-for-libraries-distribution-an-open-source-library-author-perspective
status: publish
title: Bazel for Open-source C / C++ Libraries Distribution
categories:
- eyes
---

In the past a few days, I've been experimenting with Bazel as a library distribution mechanism for [ccv](https://github.com/liuliu/ccv).

I am pretty familiar with hermetic build systems at this point. My main knowledge comes from Buck dating 8 years back. At that time, it never occurred to me such a build system could eventually be a library distribution mechanism. During the same 8 years, NPM has taken over the world. New language-dependent package managers such as Go module, Cargo and Swift Package Manager popularized the concept of using the public repositories (GitHub) as the dependency references. Languages prior to this period, mainly C / C++ are moving to this direction, slowly.

[ccv](https://github.com/liuliu/ccv) has a simple autoconf based feature detection / configuration system. You would expect the package to work when `./configure && make`. However, it never made any serious attempt to be too smart. My initial experience with monorepos at companies strongly influenced the decision to have a simple build system. I fully expect that serious consumers will vendor the library into their monorepo using their own build systems.

This has been true for the past a few years. But as I am finishing up [nnc](https://libnnc.org) and increasingly using that for other closed-source personal projects, maintaining a closed-source *monorepo* setup for my personal projects while upstreaming fixes is quite an unpleasant experience. On the other hand, [nnc](https://libnnc.org) from the beginning meant to be a low-level implementation. I am expected to have high-level language bindings at some point. Given that I am doing more application-related development with [nnc](https://libnnc.org) in closed-source format now, it feels like the right time.

Although there is no one-true library distribution mechanism for C / C++, there are contenders. From the good-old apt / rpm, to Conan, which has gained some mind-share in the open-source world in recent years.

The choice of Bazel is not accidental. I've been doing [some Swift development with Bazel](https://liuliu.me/eyes/migrating-ios-project-to-bazel-a-real-world-experience/) and the experience has been positive. Moreover, the choice of high-level binding language for [nnc](https://libnnc.org), I figured, would be Swift.

## Configure

[ccv](https://github.com/liuliu/ccv)'s build process, as much as I would rather not, is host-dependent. I use autoconf to detect system-wide libraries such as libjpeg and libpng, to configure proper compiler options. Although [ccv](https://github.com/liuliu/ccv) can be used with zero dependency, in that configuration, it can sometimes be slow.

Coming from the monorepo background, Bazel doesn't have many utilities that are as readily available as in autoconf. You can write automatic configurations in Starlark as [repository rules](https://docs.bazel.build/versions/master/skylark/repository_rules.html), but there is no good documentation on how to write robust ones.

I ended up [letting whoever use ccv to decide how they are going to enable certain features](https://github.com/liuliu/ccv/blob/unstable/WORKSPACE#L25). For things like CUDA, such configuration is not tenable. I ended up copying over [TensorFlow's CUDA rules](https://github.com/liuliu/rules_cuda).

## Dependencies

Good old C / C++ libraries are notoriously indifferent to libraries dependencies v.s. toolchains. Autoconf detects both toolchain configurations as well as available libraries. These types of host dependencies make cross-compilation a skill in itself.

Bazel is excellent for in-tree dependencies. For out-tree dependencies however, there is no established mechanism. The popular way is to write a [repository rules to load relevant dependencies](https://github.com/protocolbuffers/protobuf/blob/master/protobuf_deps.bzl#L5).

This actually works well for me. It is versatile enough to handle cases that [have Bazel integrations](https://github.com/liuliu/ccv/blob/unstable/config/ccv.bzl#L103) and [have no Bazel integrations](https://github.com/liuliu/dflat/blob/unstable/deps.bzl#L17).

## Consume Bazel Dependencies

Consumption of the packaged Bazel dependencies then becomes as simple as adding `git_repository` to the `WORKSPACE` and call proper `<your_library_name>_deps()` repository rule.

After packaging [ccv](https://libccv.org) with Bazel, now [Swift for nnc can consume the packaged dependency](https://github.com/liuliu/s4nnc/blob/main/WORKSPACE#L3).

## Semantic Versioning Challenges

While the Bazel-provided library distribution mechanism works well for my case, it is simplistic. For one, there is really no good way to do [semantic versioning](https://semver.org/). It is understandable. Coming from a monorepo culture, it is challenging for anyone to dive into dependency hells of library versioning. A [slightly different story happened to Go](https://donatstudios.com/Go-v2-Modules) a while back as well.

It is messy if you want to pin a specific version of the library while your dependencies are not agreeing with you. This is going to be messy regardless in C / C++ world, unless you prelink these extremely carefully. Bazel's philosophy from what I can see, seems largely on *keeping the trunk working* side. It is working so far, but one has to wonder whether this can scale if more libraries adopted Bazel as the distribution mechanism.

## Closing Words

The past a few months experience with Bazel has been delightful. While I would continue to use language specific tools (pip, Go modules, Cargo, NPM) when doing development in that particular language, Bazel is a capable choice for me when doing cross-language development. Concepts such as `workspace`, `git_repository`, `http_archive` fit well within the larger open-source ecosystem. And most surprisingly, it works for many-repo setup if you ever need to.