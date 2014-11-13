---
layout: post
title: "Folly源码分析(4) – 多线程数据结构"
description: ""
category: 
tags: [C++]
---
{% include JB/setup %}

Folly为多线程做了许多相关的组件和优化，例如之前介绍内存管理时的提到的ThreadCachedArena就为多线程的内存分配做优化，本文将会提到几个多线程相关的组件，如线程局部变量ThreadLocal、原子递增变量ThreadCachedInt、线程安全原子哈希AtomicHashMap、并发的跳表ConcurrentSkipList、高性能的生产者消费者队列ProducerConsumerQueue。


**ThreadLocal**

线程局部变量是指只有在本线程中才能被访问到的数据，对于其他线程是屏蔽的，类似于线程范围的全局变量，如pthread_setspecific/pthread_getspecific实现了类似的功能。但是相比于pthread_setspecific/pthread_getspecific，ThreadLocal更为易用，只要定义ThreadLocalPtr<XXX>，这个对象就会为每个线程分配一个私有的变量，当线程退出的时候自动执行释放函数。

当然线程局部变量的一种替代解决方案是让线程访问一个全局变量，使线程内部的各个函数就能共享一个变量，这样的缺陷主要有1）当一个线程挂掉时影响到其他线程；2）数据同步影响效率；因此线程局部变量是一个很有用的东西。

<!-- more -->

Folly对这个的类的优化主要有：

1. 定义了模版参数Tag，每个Tag只是用1个pthread_key_t，原因是一般来说pthread_key_t的个数是有限的，一般是1024个。

2. 开发人员可以指定线程退出时执行的函数和所有线程退出后执行的函数。

3. 提供了accessAllThreads来遍历所有线程中的局部变量，这个被用在ThreadCachedInt的获得较为准确的数据。

在Folly的文档中介绍了这个类的用法，例如

{% highlight c++ %}

{
  folly::ThreadLocalPtr w;
  w.reset(new Widget(0), Widget::customDeleterA);
  std::thread([&w]() {
    w.reset(new Widget(1), Widget::customDeleterB);
    w.get()->mangleWidget();
  } // Widget(1) is destroyed with customDeleterB
} // Widget(0) is destroyed with customDeleterA
// customDeleterB通过THIS_THREAD设定，当单个线程退出的时候调用
// custonDeleterA通过ALL_THREAD设定，当所有线程退出的时候调用

{% endhighlight %} 
使用accessAllThread遍历访问线程的私有数据

{% highlight c++ %}
class SimpleThreadCachedInt {

  class NewTag;  // Segments the global mutex
  ThreadLocal val_;

 public:
  void add(int val) {
    *val_ += val;  // operator*() gives a reference to the thread local instance
  }

  int read() {
    int ret = 0;
    // accessAllThreads acquires the global lock
    for (const auto& i : val_.accessAllThreads()) {
      ret += i;
    }  // Global lock is released on scope exit
    return ret;
  }
};
{% endhighlight %}
ThreadLocal实现的数据结构大致如下

<img width ="800px" src="/assets/images/2012/09/09/threadlocal.png" alt="threadlocal" />

使用一个全局静态元数据StaticMeta保存了线程数据ThreadEntity的头指针和一些用于管理的数据，当一个线程启动时需要将数据加入到线程私有变量中去的时候，在head的指向的私有数据链表中，而实际的数据时保存在elements的ElementWrap的数组中，其中的ptr指向于真正的数据，而每个ElementWrap都可以指定自己的资源释放函数Deleter。
 

**ThreadCachedInt**

这是一个整型的类，主要用于多线程同时操作一个自增的整型变量，在精度保证不丢失的情况下提供极高的性能，在测试获得的结果是，它比std::atomic_fetch_add快10倍。

提供了两种的对这个自增整型类的读取方式：

- readFast：速度很快（一个load指令得到的数据），但是数据有可能是以前的数据，即读到的数据比真是数据小，这种情况适用于对精度无太大要求，但是对效率要求很高的情况。

- readFull：速度相比于readFast慢，但是数据较为准确，做法事实上就是累加所有线程中的计数器和全局的计数器，在读取线程的数据的时候需要获得一个锁，因此效率比较差。

这个类的数据结构示意图

<img width ="800px" src="/assets/images/2012/09/09/threadcachedint.png" alt="threadcachedint" />

这个示意图也示意了这个类的实现过程，即保存一个类全局变量和线程私有变量val，是ThreadLocalPtr，这个类就是上面介绍的ThreadLocal，当线程中的val更新次数超过一定次数的时候，将本线程中的计数器val加到全局的targe_中，本身的val清零。

readFast直接读取target_的值
{% highlight c++ %}

  IntT readFast() const {
    return target_.load(std::memory_order_relaxed);
  }
 {% endhighlight %}
readFull则需要累加所有的值

{% highlight c++ %}
  IntT readFull() const {
    IntT ret = readFast();
    for (const auto& cache : cache_.accessAllThreads()) {
      if (!cache.reset_.load(std::memory_order_acquire)) {
        ret += cache.val_.load(std::memory_order_relaxed);
      }
    }
    return ret;
  }

{% endhighlight %} 

**AtomicHashMap**

AtomicHashMap是一个几乎无锁的原子HashMap，几乎无所的意思是只有少量的几个方法是有锁的。

这个类的文档和源代码的注释中解释了这个类的有点和缺点，如下

优点：

- 高性能，特别是在多线程要求非常高的环境中
相比tbb::concurrent_has_map快2-5倍
- 好的内存利用率，容量不过估的前提下
良好的分散性能？（ Good fragmentation properties ）
- 查找快速

缺陷：

- Key只支持int32、int64，原因是需要执行无所的cas（compare and swap）操作
- 必须指定empty、locked、erased的key，原因也是key的限制
- 性能会随着初始尺寸的容量的增大线性降低，原因是其数据结构的submap的查找
- 最大的size被限制在初始size的18倍
- 擦除的内存块不被释放，而是被标记擦除，为了效率

数据结构如下

<img width ="800px" src="/assets/images/2012/09/09/atomichashmap.png" alt="atomichashmap" />

一个AtomicHashMap中最多包含16个subMap，subMap定义为AtomicHashArray，用数组实现的存储数据的hash 表。在初始化的时候只初始化1个SubMap，其他都置为null，当一个SubMap被放满时再初始化下一个SubMap。

主要方法的实现大致如下

Insert

从第一个SubMap查找位置，使用线性的解决冲突，即有冲突，不断找相邻的下一个位置，若满则找下一个，若全满，则申请新的SubMap，大小为

第一个SubMap的capacity * 1.8 ^ (nextIdx – 1)

<img width ="800px" src="/assets/images/2012/09/09/atomichashmap_insert.png" alt="atomichashmap_insert" />

这里在查找key的时候使用了cas操作，以原子地对位置加锁，而在AtomicHashArray中每个key的状态有可能是Empty、Locked、Erased、真实的key，并且预留一定百分比的位置以缓冲，避免超过capacity。

find

过程如下：

计算keyToAnchorIdx(key)的值，定位到对应的单元

判断单元中存储的key

        Key=key_in：查找成功，返回单元中的元素

        key≠key_in：

            Key=Empty：查找失败

            否则：查找下一个单元，直到找到元素或者超过capacity

erase

操作与find一样，找到元素，标记为Erased

其中find和erase是无等待的。


**ConcurrentSkipList**

这是一个支持并发的跳表，跳表是一个随机的链表结构，盗用网上的一张示意图对跳表进行描述

<img width ="800px" src="/assets/images/2012/09/09/skiplist.png" alt="skiplist" />

每个节点都是随机高度，当查找的时候从高往低查找，当一层失败的时候就降低一层查找，知道找到或者找到最低层。

Folly中实现的跳表的特点和缺陷主要有

特点：

较小的内存开销：比std::set小了~40%
Read操作时无锁几乎无等待的
快速，多线程环境下比带有一个RWSpinLock的std::set快了10倍以上
支持GC

缺陷：

写操作比std::set慢30%
每一个跳表都需要一个4-word的头结点
当元素个数较小（<1000）时，比std::set慢
只支持x64
释放的节点不可被重用
数据结构大致如下

<img width ="800px" src="/assets/images/2012/09/09/ConcurrentSkipList.png" alt="ConcurrentSkipList" />

保存了内存回收类和跳表的头结点，每个结点的结构如头结点所示，使用了一个spinlock来对结点加锁，同时GC类中也记录了结点数据和这个类的引用数，每次被引用refs增加一次，当调用release时，refs减少一次，当refs=1时调用release的时候结点被析构。

 

**ProducerConsumerQueue**

另外Folly实现了一个队列长度确定的无等待的单producer单consumer的生产者消费者队列，这个类的实现就相对比较简单。

用一个定长的数组表示循环队列，当调用write的时候往队列尾写数据，若满则返回false

{% highlight c++ %}

  template
  bool write(Args&&... recordArgs) {
    auto const currentWrite = writeIndex_.load(std::memory_order_relaxed);
    auto nextRecord = currentWrite + 1;
    if (nextRecord == size_) {
      nextRecord = 0;
    }
    if (nextRecord != readIndex_.load(std::memory_order_acquire)) {
      new (&records_[currentWrite]) T(std::forward(recordArgs)...);
      writeIndex_.store(nextRecord, std::memory_order_release);
      return true;
    }

    // queue is full
    return false;
  }
  {% endhighlight %}
  
调用read读队列头的数据，若队列空则返回false

{% highlight c++ %}

  // move (or copy) the value at the front of the queue to given variable
  bool read(T& record) {
    auto const currentRead = readIndex_.load(std::memory_order_relaxed);
    if (currentRead == writeIndex_.load(std::memory_order_acquire)) {
      // queue is empty
      return false;
    }

    auto nextRecord = currentRead + 1;
    if (nextRecord == size_) {
      nextRecord = 0;
    }
    record = std::move(records_[currentRead]);
    records_[currentRead].~T();
    readIndex_.store(nextRecord, std::memory_order_release);
    return true;
  }
{% endhighlight %}

由于C++11中对于内存读写顺序的方法的使用，对一些数据的原子读写就方便许多。

**总结**

Folly的多线程数据大致如上，线程局部变量ThreadLocal、原子递增变量ThreadCachedInt、线程安全原子哈希AtomicHashMap、并发的跳表ConcurrentSkipList、高性能的生产者消费者队列ProducerConsumerQueue，这些数据结构一方面在内存效率以及易用上有较大的优化，另外一方面也带来限制，如map中key的限制，对于元素个数的预估等，所以在一定条件下使用这些数据结构能够带来收益。