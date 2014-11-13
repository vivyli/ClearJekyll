---
layout: post
title: "Folly源码分析(5) – 多线程锁"
description: ""
category: 
tags: [C++]
---
{% include JB/setup %}

Folly对多线程方面考虑地较多，串讲时候被告知其实现在每个产品都是这样的，对多线程的要求都非常高，Folly中的多线程中对于自旋锁（spinlock）特别实现了一下，实现了如下的自旋锁：

SmallLocks(MicroSpinLock、PicoSpinLock)：两个非常小的互斥量类型（1byte和1bit），主要用于highly memory-constrained环境

RWSpinLock(RWSpinLock、RWTicketSpinLock)：两个读写锁


**MicroSpinLock**

用一个byte表示锁（uint8_t），锁有两种状态，分别是FREE和LOCKED。

调用lock时，用一个while循环不断地尝试去获得锁（即调用try_lock），看是否可以将锁的状态置为LOCKED，对于获得锁的策略是

前4000次每次失败都调用pause，之后每次失败sleep 0.5ms

这样降低线程的上下文切换的开销。

<!-- more -->

try_lock的实现是使用了汇编的cas操作，即compare and swap，在效率上比普通的做比较会高很多，代码如下

{% highlight c++ %}
  bool cas(uint8_t compare, uint8_t newVal) {
    bool out;
    asm volatile("lock; cmpxchgb %2, (%3);"
                 "setz %0;"
                 : "=r" (out)
                 : "a" (compare), // cmpxchgb constrains this to be in %al
                   "q" (newVal), // Needs to be byte-accessible
                   "r" (&lock_)
                 : "memory", "flags");
    return out;
  }
 {% endhighlight %}
 

**PicoSpinLock**

这也是一个自旋锁，提供了自旋锁的基本函数lock和unlock，另外它可以保存一个整型变量，提供了getData和setData来获得/设置保存在锁中的整数，前提是这个整数的最高位必须为0，以使用这个最高位作为锁的表示，即最高位是0表示FREE，为1表示LOCKED，如图示：

<img width ="800px" src="/assets/images/2012/09/22/pico_spinlock.png" alt="pico_spinlock" />

其他实现与MicroSpinLock相同，如try_lock用汇编的原子cas实现，尝试获得锁的策略。

这个锁的好处是在内存占用上几乎0消耗，因为一般无符号整型变量的最高位都是用不到的

 

**RWSpinLock**

一个读写锁，比pthread_rwlock快并且低开销，主要特点是：

当线程数量小于CPU数量的时候性能较好
线程数量大的时候会导致写饥饿（写锁获得必须在读锁都释放之后才能被获得）
最多支持2^30 – 1个reader
在数据结构上，使用了一个32位的整型来表示这个锁，并有3个状态标记：写锁、更新锁、读锁，图示如下

<img width ="800px" src="/assets/images/2012/09/22/RWSpinLock.png" alt="RWSpinLock" />

写锁（Lockable）：只有在读、写、更新锁都没有的时候才能获得

在实现上，不断尝试申请写锁，多次失败（1000次），主动让出CPU，并且，以后的每次申请失败都让出CPU。

这样的优化的好处在于避免了频繁地切换上下文，但是却加剧了写饥饿，即在读者很多的情况下，写锁就非常难申请到，产生饥饿。

大致代码如下
{% highlight c++ %}

  void lock() {
    int count = 0;
    while (!LIKELY(try_lock())) {
      if (++count > 1000) sched_yield(); // 1000次申请失败，调用sched_yield() 主动让出CPU
    }
  }
  bool try_lock() {
    int32_t expect = 0;
    return bits_.compare_exchange_strong(expect, WRITER,
      std::memory_order_acq_rel); // 原子操作，判断lock(bits_)是否是0，
                                  //只有在是0的时候也就是没有被获得任何锁的情况加才申请到写锁
  }
 
 {% endhighlight %}

读锁（SharedLockable）：没有写锁的时候可以获得

同样是不断尝试申请锁，多次申请失败让出CPU，当申请成功过的时候给lock加4，即在高30位的计数器上加1，当释放锁的时候减1。

大致代码如下

{% highlight c++ %}

  void lock_shared() {
    int count = 0;
    while (!LIKELY(try_lock_shared())) {
      if (++count > 1000) sched_yield();
    }
  }
  bool try_lock_shared() {
    // fetch_add 比 compare_exchange 快100%
    int32_t value = bits_.fetch_add(READER, std::memory_order_acquire);
    if (UNLIKELY(value & WRITER)) {
      bits_.fetch_add(-READER, std::memory_order_release);
      return false;
    }
    return true;
  }
 {% endhighlight %}

更新锁（Upgradable）：与读锁一样，但可以升级成写锁，或者降级成读锁，用于那种需要先做判断，才决定是否需要加写锁的场景

申请更新锁，同样多次申请失败（已经拥有更新锁或者写锁）让出CPU，能够被升级成写锁或者读锁，做法是先释放更新锁，再加锁，使用原子操作完成。

大致代码如下

{% highlight c++ %}

 void lock_upgrade() {
    int count = 0;
    while (!try_lock_upgrade()) {
      if (++count > 1000) sched_yield();
    }
  }
 bool try_lock_upgrade() {
    int32_t value = bits_.fetch_or(UPGRADED, std::memory_order_acquire);
    // 申请锁失败，不回滚UPGRADED位，因为已经有更新锁或者写锁了，如果只是写锁，当unlock的时候这个位会被清除
    return ((value & (UPGRADED | WRITER)) == 0);
  }
{% endhighlight %} 

**RWTicketSpinLock**

RWSpinLock的问题是读写不平衡，在读者太多的时候，写锁就没办法获得了，RWTicketSpinLock一定程度上解决了这个不平衡的问题，解决方法为：

当有一个线程在等待写锁的时候，读锁是无法再被加上的，直到所有读锁被释放的时候，写锁就可以被加上，一定程度平衡了读写不平衡。

在实现上，使用了一个ticket，即32或者64位长度的union类型来做读、写锁的获得判断条件，如图示

<img width ="800px" src="/assets/images/2012/09/22/RWTicketSpinLock.png" alt="RWTicketSpinLock" />

如代码

{% highlight c++ %}
  union RWTicket {
    FullInt whole; // 用于整个数据的拷贝
    HalfInt readWrite; // 代表read和write数据
    __extension__ struct {
      QuarterInt write; // 用于加锁限制的标记数据
      QuarterInt read; 
      QuarterInt users; 
    };
  } ticket;
 {% endhighlight %}

读锁、写锁获得的条件过程大致如下（比较绕）

初始化

user=0、read=0、write=0

读锁

只有当users=read的时候，读锁才可以被加上，否则申请锁失败。
读锁被加上后，read++，users++，封锁了写锁的申请，但是读锁还是可以申请
Unlock的时候， write++
写锁

只有当users=write的时候，写锁才可以被加上，否则申请锁失败。
写锁被加上后，users++，封锁了读锁和写锁的申请
Unlock的时候，read++，write++
写锁等待

给users++，封锁了读锁的申请，然后等待user++之前的users==write的时候获取写锁。

自制算法示范举例图：

<img width ="800px" src="/assets/images/2012/09/22/RWTicketSpinLock_Example.png" alt="RWTicketSpinLock_Example" />

在这个锁中做的优化大致有：

使用并行的SSE2并行指令，对多个连续地址的整数一个指令完成++操作
使用Pause指令让出，一方面可以降低CPU能耗，另外可以帮助CPU优化指令。
减少分支预测，令old.user=old.read，判断user和read是否相等的逻辑延迟到CAS操作。
 

**总结**

Folly中的SpinLock中MicroSpinLock和PicoSpinLock缩短了锁需要表示的长度，以节省内存空间，由于cas的使用在效率上也有一定的提升。

RWSpinLock在效率上由于上下文切换的开销降低，有一定的提高，但是却产生了饥饿问题，RWTicketSpinLock在获得写锁的时候封锁读锁的获得，一定程度上解决了读写不平衡的问题。