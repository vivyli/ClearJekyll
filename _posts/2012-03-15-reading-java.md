---
layout: post
title: "Java的一些读书笔记"
description: ""
category: java
tags: [java]
---
{% include JB/setup %}

Java的GC

**如何判断对象是否可以被GC？**

主要有两个算法：1）引用计数法；2）根搜索法；

引用计数法是指对象每被引用1次，对象中的引用计数器加1，引用失效时，计数器减1，直到计数器为0的时候，对象被标记为可回收。

根搜索法是指从GC Roots开始建立一个引用的图，当一个对象到GC Roots的路径不存在时，对象可以标记为可回收，JAVA中使用的是这种方法。

<!-- more -->


**对象GC的过程？**

当对象被标记为可回收时，JVM判断判断对象是否有必要执行finalize方法（只有当finalize方法未被执行过的时候才被认为需要执行finalize方法）。

如需要执行finalize方法，把对象加入到F-Queue中，稍后由虚拟机调用其finalize方法，如对象调用完之后仍旧是被标记为可回收，这块对象所占的内存空间就被虚拟机回收了。

**内存回收的基本算法？**

主要有四个算法：1）标记-清除算法；2）复制算法；3）标记-整理算法；4）分代收集算法。

标记-清除算法，是指直接回收被标记的内存块。会产生许多不连续的内存块碎片。

复制算法，把内存块分为大小相等的两个区，只使用其中1个区，当一块内存块用完时，把每块复制到另外一个区，并且使用的内存块处于连续的空间。

标记-整理算法，是指内存块标记完之后，让所有使用的内存块向一端移动，然后清理掉端界意外的内存。

分代收集算法，是指把JAVA堆根据对象存活周期分为新生代和老生代，根据每个代的特点采用适当的手机算法。

**各种垃圾收集器？**

主要有：1）Serial；2）ParNew；3）Parallel Scavenge；4）Serial Old；5）Parallel Old；6）CMS(Concurrent Mark Sweep)；7）G1(Garbage First)；

**Java的引用**

Java中引用主要有四类，分别是：1）Strong Reference；2）Soft Reference；3）Weak Reference；4）Phantom Reference；主要跟Java的GC有关。

可以从java.lang.ref包里可以看到有几个类：SoftReference、WeakReference、PhantomReference，分别对应后面三种引用。

Strong Reference，就是一般意义上的引用，类似于Object obj = new Object的引用，只要Strong Reference存在，GC就不会回收这个对象。

Soft Reference，被Soft Reference引用的对象，只有当系统将要发生内存溢出一场之前，才会被回收这块内存。

Weak Reference，被Weak Reference引用的对象，能生存到下次垃圾收集之前，可以使用一段代码解释。

{% highlight java %}
private static void checkNull(Object obj)
{
    if (obj == null)
        System.out.println("Dead");
    else
        System.out.println("Alive");
}
public static void main(String[] args) throws Throwable
{
    Object obj = new Object();
    WeakReference<Object> wr = new WeakReference<Object>(obj);
    checkNull(wr.get());
    obj = null;
    checkNull(wr.get());
    System.gc();
    checkNull(wr.get());
}
{%endhighlight%}

输出的是：

Alive

Alive

Dead

也就是说只有调用了GC只有，Weak Reference才失效，get不到任何东西，内存才被回收，而在调用GC之前，即使obj=null了，建立Weak Reference的对象还是能get到的。

Phontom Reference，对对象的生存时间产生任何影响，只是希望能在GC回收内存块的时候收到一个系统通知。

**犄角旮旯的关键字**

主要有3个犄角旮旯的关键字：strictfp、transient、native

strictfp

浮点数计算方法相关的关键词，可用于修饰类、接口和方法，在此关键词内的方法的浮点数计算都是符合IEEE 754标准的，可以保证可移植性。

transient

序列化相关的关键词，用于修饰用来Serializable类属性，这个属性在类序列化的时候不被序列化。如

{% highlight java %}
public class Foo implements Serializable { 

    public transient int a = 0; 
    public int b = 0; 
}
{% endhighlight %}
当改变a和b的值之后序列化，再从序列化文件中读取这个类之后，b的值为序列化之前的值，而a是0。

native

用于修饰Java类的方法，表示这个方法是原生的方法，是调用被编译后的库文件的。

{% highlight java %}
class Prompt {
    private native String getLine(String prompt);

    public static void main(String args[]) {

	Prompt p = new Prompt();
	String input = p.getLine("Type a line: ");
	System.out.println("User typed: " + input);

    }
    static {
	System.loadLibrary("MyImpOfPrompt");

    }

}
{%endhighlight%}
这些犄角旮旯的关键字目前开发中还没用到

参考：《深入理解Java虚拟机——JVM高级特性与最佳实践》