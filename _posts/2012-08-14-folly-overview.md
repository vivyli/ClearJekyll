---
layout: post
title: "Folly Overview"
description: ""
category: c++
tags: [C++]
---
{% include JB/setup %}

本文从整体上对Folly做一个简单的介绍，主要参考GitHub上Folly的文档，以及我读代码下来的感觉。

**Folly是什么**

它是Facebook Open-source Library的简称，是一个在今年的6月份开源的基于C++11的基础库，作为对STL和boost的补充和增强，并非是替代，也就是说是实现了一些STL和boost中没有的组件，也对STL和boost中的部分组件做了性能上的增强。

<!-- more -->

值得一提的是它的主要作者Andrei Alexandrescu，写过《Modern C++ Design》，精于模板编程，目前已经43岁，还奋战在coding的第一线，我想这样的人国内少有了吧。Folly代码写的非常漂亮，逻辑非常清晰，也很少有令人费解的写法，读起来比较舒服。

**Folly的特点**

高效

一方面在通用的数据结构结构中对算法速度和内存上做优化，相比于STL和boos中的组件（如string、vector）有较大的提高；

另一方面实现了在特定场合（例如容器中的元素个数较少或者已知的前提下）具有较好的性能。

易用

组件相比以前的更加易用，如ThreadLocal；另外有借鉴了其他语言语法上的优势，如实现了类似于java的synronized关键字的SYNCHRONIZED宏，python的格式化Format工具等。

**Folly中有什么**

Folly的代码量并不多，50几个头文件，10几个cpp文件，多的2000多行，少的也就几行。

主要包含了40几个组件，大致包括：

内存管理：如Arena

高性能的数据结构：如string、vector

有用的数据结构：如直方图、延时队列

多线程相关的优化：如线程本地内存、旋转锁

工具：如Hash的实现

我对组件做了一个简单的分类，接下来就根据这个顺序来逐个写

- 内存管理：Arena ThreadCachedArena

- 字符串：FBString Range Malloc

- 向量：FBVector Traits small_vector sorted_vector_types

- 多线程数据结构：ThreadLocal AtomicHashArray AtomicHashMap ThreadCachedInt ProducerConsumerQueue ConcurrentSkipList

- 多线程锁：SmallLocks RWSpinLock

- 整型编码：DiscriminatedPtr GroupVarint PackedSyncPtr

- 有用的数据结构：dynamic Histogram TimeoutQueue

- 工具：Bits Conv Format String Unicode Synchronized json Random Hash

- 简化编程：eventfd Foreach IntrusiveList Likely Preprocessor ScopeGuard StlAllocator