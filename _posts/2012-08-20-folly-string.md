---
layout: post
title: "Folly源码分析(2) – 字符串"
description: ""
category: 
tags: [C++]
---
{% include JB/setup %}

本文主要介绍了Folly中两个字符串相关的类FBString和Range，主要是介绍FBString中的优化，Range不做具体介绍。


**FBString**

Folly实现了一个自己的字符串类fbstring，关于fbstring我之前看源代码的时候参考了博客园里的[一篇文章](http://www.cnblogs.com/promise6522/archive/2012/06/05/2535530.html)，讲的非常不错，所以我相当于是重复性劳动了。

fbstring兼容了std::string的接口，属于重新造轮子，但是做了性能上的优化，主要有如下优化：


内存模型

- 数据存储

- 内存分配

常用算法的优化

- find算法

- 其他优化

<!-- more -->


**数据存储**

根据存储的string的size，分成三种存储方式：

Small：0-22个字符，存储到栈中，char[24]

Medium：23-254个字符，使用malloc分配内存，使用eager-copy方式，即字符串的复制总是重新分配并拷贝内容

Large：大于254个字符，存储方式与medium相同，在字符串之前存储了引用计数，使用lazy-copy，即copy-on-write，也就是说一般的引用只增加引用计数，只有在需要修改或者取下标操作的时候才对字符串进行copy，以减少内存拷贝的次数。

这样做的好处在于当字符串长度较小的时候，直接做栈操作，效率上会高很多，当字符串较长的时候，由于代价的考虑，不能做频繁的拷贝操作，因此使用了copy-on-write，但同时由于copy-on-write本身的缺陷，如多线程的时候会增加其内存拷贝次数等等，在中等长度的时候则不适用copy-on-write。（事实上高版本的string类已经不再使用copy-on-write了）

代码上的数据结构表示
{% highlight c++ %}

  struct MediumLarge {
    Char * data_;
    size_t size_;
    size_t capacity_;

    size_t capacity() const {
      return capacity_ & capacityExtractMask;
    }
  };
  union {
    mutable Char small_[sizeof(MediumLarge) / sizeof(Char)];
    mutable MediumLarge ml_;
  };
 {% endhighlight %}
ml_的大小正好为24，在实际存储中，small_的最后一位存储字符串的长度，倒数第二位保留给’\0′，所以small的时候最长能保存22个字符，而medium和large使用了相同的数据结构，不同点事large之前会保存一个RefCount，它是一个std::atomic<size_t>的原子整型。
 

**内存分配**

内存分配方面的优化使用了上篇中讲的Malloc.h，即一方面做了64B/256B/4KB/4MB的内存申请的对齐，令一方面优化了realloc操作，在内存扩张的时候先判断是否使用了jemalloc，若未使用则根据内存占用率判断是否使用realloc，具体可见上篇Folly相关的文章的Malloc.h部分。

**find算法优化**

std::string中的find算法使用了KMP查找算法，而在Folly中使用了一种简化的Boyer-Moore算法，测试显示当查找成功时相比string::find提高了30倍，失败时提高了1.5倍。

这个算法过程如下，举例说明

例如：ABCDEFABCDEFGEFG中查找DEFGE

1. 目标字符串（DEFGE）的最后一个字符是E，lastNeedle=’E'，找到源字符串中第一个lastNededle的位置，ABCDEFABCDEFGEFG

2. 从后往前比较，发现D!=G，因此不需要往前找，因为之前肯定不存在这个紫川，需要找下一个可能出现的地方。

ABCDEFABCDEFGEFG

DEFGE

3. 在这一步不是逐个找，而是跳过目标串中距离lastNeedle与lastNeedle相同的两个字母的距离，此处为两个E的距离，skip=3

4. 跳过skip（3）的距离ABCDEFABCDEFGEFG，即从B开始找下一个lastNeedle（E），为ABCDEFABCDEFGEFG

5. 不断循环查找，直到找到或者失败

ABCDEFABCDEFGEFG
DEFGE


**‘\0′的优化**

对于字符串预留末尾的’\0’，只在调用c_str()或者data()的时候添加，在一般的操作中保留着一位，而不再做添加和删除，减少了操作无用字符的开销，提高一定的性能。

 

**word-wise copy**

当是类型是small的时候，每次拷贝64bits数据，以提高拷贝的效率，如下
{% highlight c++ %}

        if (byteSize > 2 * sizeof(size_t)) {
          // Copy three words
          ml_.capacity_ = reinterpret_cast(data)[2];
          copyTwo:
          ml_.size_ = reinterpret_cast(data)[1];
          copyOne:
          ml_.data_ = *reinterpret_cast(const_cast(data));
        } else if (byteSize > sizeof(size_t)) {
          // Copy two words
          goto copyTwo;
        } else if (size > 0) {
          // Copy one word
          goto copyOne;
        }
      }
{% endhighlight %}
**小结**

以上就是string类的几个主要的优化，总结一下主要的优化点是1）根据string的长短优化其存储方式；2）在内存realloc的时候做优化；3）常用算法上的优化


**Range**

另外一个跟字符串有关的类是Range.h，这个类就没什么可写的，主要是封装了可随机访问的字符串StringPiece，与boost的Range相同，实现的目的是为了兼容接口。

实现方法是：

存储了两个指针b_和e_，分别为一个字符串片段的起始地址和结束地址，访问时访问地址偏移量