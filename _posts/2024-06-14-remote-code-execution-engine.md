---
title: "Distributed Remote Code Execution Engine"
date: 2024-06-14
header:
    image: "https://res.cloudinary.com/dkoypjlgr/image/upload/f_auto,q_auto:good,c_auto,w_1200,h_300,g_auto,fl_progressive/v1715952116/blog_cover_large_phe6ch.jpg"
tags: [scala,pekko,pekko-http,pekko-stream,pekko-cluster,docker,docker-compose,scala3]
excerpt: "Practical guide to building the distributed remote code execution engine in Scala and Pekko"
---

_by [Anzori (Nika) Ghurtchumelia](https://github.com/ghurtchu)_

## 1. Introduction

It's been a long time since my last article, but I am back with greater passion and energy to explore and learn more about the existing tools within Scala ecosystem by building something tangible and useful with them.

The greatest benefit of small side projects is the unique knowledge boost which can be useful later in our careers.

In this article we will see an attempt to build the remote code execution engine - the backend platform for websites such as [Hackerrank](https://hackerrank.com), [Leetcode](https://leetcode.com) and others.

As for tools, we will be using `Scala 3`, `Pekko` and `docker`.

There are many ways how such platform can be built, however, the main goal of this article and the project is to get familiar with `Pekko` and its modules such as `pekko-http`, `pekko-stream`, `pekko-cluster` and a few interesting concepts revolving around actor model concurrency, such as: 
- cluster nodes and formation
- cluster aware routers
- remote worker actors
- actor lifecycle and hierarchy
- actor supervision strategies
- actor location transparency
- message serialization
- and so on...

Let's get started then, shall we?

## 2. Project Structure




