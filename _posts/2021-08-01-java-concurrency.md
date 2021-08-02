---
layout: post
title: "java concurrency"
date: 2021-08-01 12:31:00 +0800
---



### CPU缓存

cpu在载入数据的时候，不是从内存中直接加载，而是会先从cpu缓存中加载，如果缓存中没有需要的数据，才会到内存中加载。因为相对于cpu的运行速度，读取内存数据的速度是比较慢的。

主流的cpu架构中，cpu缓存有3个，分别是 L1 Cache、L2 Cache、L3 Cache。
