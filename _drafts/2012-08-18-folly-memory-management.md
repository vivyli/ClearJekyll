---
layout: post
title: "Folly源码分析(1) – 内存管理"
description: ""
category: 
tags: [C++]
---
{% include JB/setup %}

Folly中内存管理相关的类主要有Arena和ThreadCachedArena，另外针对内存申请做优化的是Malloc.h这个工具，本文主要对这三个类/工具做实现上的分析。

**Arena**

Arena是一个类似于内存池的类，不同的是一般内存池都管理的被释放的内存，但是Arena无法释放内存，直到Arena本身被析构的时候才释放所有的内存块，这个类主要目的是作为ThreadCachedArena的实现。

它提供了三个接口：

- void* allocate(size_t size); //申请内存一块size大小的内存块，加入Arena的内存链表中

- void merge(Arena&& other); //将另外一个Arena中的内存块合并到当前的Arena中

- void deallocate(void* p); //为接口兼容，实际上不做任何操作

<!-- more -->


数据结构

内存的存放方式是使用了一个单向链表存储，cache了末端的节点

{% highlight c++ %}

typedef boost::intrusive::slist<
    Block,
    boost::intrusive::member_hook<<Block, BlockLink, &Block::link;>,
    boost::intrusive::constant_time_size<false>,
    boost::intrusive::cache_last<true>> BlockList;
{% endhighlight %}

存储的结构如下

image

以竖线为分界线，右边的都是定长（默认为4M）的内存块，左边为大于定长的内存块，ptr_和end_分别指向了链表末端的空闲块的开始和结束地址

分配内存

Arena的内存申请算法如下：

1.当申请的内存块的size小于链表末端的空闲空间（即end_-ptr_），则直接插入到链表末端的空间（以加快内存分配），否则分配内存块

2.分配内存块：

- 当申请的内存块的大小大于minBlockSize（4M），则直接插入到链表的前面

- 当申请的内存块的大小小于或者等于minBlockSize（4M），则插入到链表的末端，并调整ptr_和end_的指向，指向于(minBlockSize – size)

释放内存

调用deallocate不做任何操作，在调用~Arena()时，逐个析构链表中的内存块

 

**ThreadCachedArena**

ThreadCachedArena封装了Arena，用作多线程中的内存分配，主旨是在多线程中，每个线程都有自己的一个Arena来做内存分配和管理，同时线程共用成为zombie的Arena，当线程退出时，这个线程中的内存不做释放操作，而是merge到zombie中，直到所有的线程已经退出，ThreadCachedArena被析构时，zombie中存放的内存才被释放。

这样做都是为多线程环境的速度做考虑，线程申请内存不需要考虑线程同步，也减少了内存的释放操作，对性能有很大的提升。

其数据的示意图大致如下，每个线程中的Arena用Folly中的ThreadLocalPtr存放，其作用与pthread_getspecific类似，每个线程都是独立拥有一个对象。

image

 

**Malloc.h**

这是一个包装glibc的malloc、calloc、realloc的工具，它的主要优化是：

1. 对size做了goodMallocSize过滤，将size做64B/256B/4KB/4MB对齐，以减少内存碎片
2. 实现smartRealloc函数，判断是否使用jemalloc，再做优化

	2.1 当使用了jemalloc的时候，用jemalloc的rallocm代替std的realloc方法（jemalloc是FreeBSD上默认的分配器，在多线程并发环境下表现更好）
    
    2.2 当没有使用jemallc，需要realloc的时候，根据内存块使用率，分情况处理处理，如示意图

image


2.2.1 当内存使用率低于50%的时候，放弃使用realloc，因为realloc可能会拷贝整块内存，会导致效率低下，使用malloc-copy-free的方法，也就是先malloc一块大的内存，再把原内存中的数据copy过去，释放原内存

2.2.2 否则使用realloc，因为realloc有一定的可能性是会在当前内存之后扩展出一块内存，则而不需要拷贝整块内存；若realloc扩展内存失败，则会做malloc一块新的内存，把原内存块整块拷贝过去（包括使用的和未使用的）

**小结**

Folly中的内存相关的类就是如上，Arena是针对了多线程环境中的内存分配做优化，块内空闲段的利用、减少内存释放次数、以及内存局部变量的方法提高内存分配/释放的速度。

Malloc.h主要是针对了glibc中realloc的缺陷做了优化，原因是在容器中往往是需要做realloc操作的，如vector、string等，一方面也是为多线程的性能考虑。

以上都是个人理解，如有错误，请及时想告。