---
layout: post
title: JSR 133 FAQ的翻译和扩展（5）—— volatile相关
key: 20160619
tags: 架构 并发与并行 开发语言
---
## 前言  
valotile 关键字对很多 java 开发人员来说并不陌生。 但，也许因为它看上去过于简单，便有了一些似是而非的认识。 
  
本部分，从FAQ中的 “What does volatile do ” 问题开始，聊一聊 volatile 的能做和不能做的事情以及在并发编程中的相关主题。  
<!--more-->   
   

## ▸volatile是用来做什么的？【译】  

[原文][What does volatile do ?](http://www.cs.umd.edu/~pugh/java/memoryModel/jsr-133-faq.html#volatile )  

>注1：本段在2018年初，重新进行了修订。 
  
volatile 字段是用来线程间状态通信（ communicating state ）的特殊字段。每次对一个 volatile 字段的读取都可看到其它线程对这个字段的写入（的最新值）；实际上，它们是编程人员设定用来保证不接收 “ 过期 ” （ stale ）值的。这些过期值可能是重排序或者缓存的结果。编译器和运行时会被禁止把它们分配入寄存器。同样，它们也必须确保在被改写后，被从缓存中刷新到主存，以便可以让它们被其它线程立即可见。类似的，在一个 volatile 字段被读取之前，缓存必须先失效，以保证读取到的值是来自主存，而不是在本地处理器缓存。在重排序访问 volatie 变量时，也会有一些额外的限制。  
  
在旧的内存模型里，对几个 volatile 变量的访问不能相互间重排序，但是它们可以与 **非 volatile （ nonvolatile ）**变量的访问顺序进行重排序。这就削弱了 volatile 字段作为线程间信号条件（ signaling conditions ）手段的作用。   
   
在新的内存模型中，对几个 volatile 变量的访问依旧不能相互间重排序。差别是，对它们周围的普通字段的访问，也不再那么容易重排序了。写入一个 volatile 字段，会有监视器释放（ monitor release ）相同的内存效果，而读取一个 volatile 字段，则有监视器获取（ monitor acquire ）同样的内存效果。实际上，因为新的内存模型设置了对 volatile 字段访问和其它类型字段访问的更加严格的重排序限制，无论（字段） volatile 与否，任何线程 A 在写入 volatile 字段 f 时可见的（周围）字段，对于线程 B 在读取 f 时候，都会变成可见。     
   
下面的简单例子展现了 volatile 字段怎么使用： 
{% highlight java %}    
class VolatileExample { 
  int x = 0; 
  volatile boolean v = false; 
  public void writer() { 
    x = 42; 
    v = true; 
  } 
 
  public void reader() { 
    if (v == true) { 
      //uses x - guaranteed to see 42. 
    } 
  } 
} 
 {% endhighlight %}   
  
假设有个线程正在调用 “writer()” 方法，而另一个正在调用 “reader()” 方法。则，在 writer 中对 v 的写入时， x 的写入也会被释放 （ release ）到内存，因而对 v 的读取，可以从内存中获取 (acquire) 该值。这样，如果可以在 “reader()” 中看到 v 的 true 值，也可确保之前对 x 写入的值 42 可见。在旧的内存模型中，情况未必会这样。（旧模型中）如果 v 不是 volatile 类型，那么编译器可以在 writer() 方法中对写入重排序，这样，在 reader() 方法中，可能会读到 x 的值为 “0” 。 

>注2：在本系列的第（4）篇中的“构建良好的执行”的例子中，其实更好解释了这一问题。这里有点语焉不详。  
  

对 volatile 类型语义的大幅度增强，几乎使之达到了同步的级别（ the level of synchronization ）。为了可见性的目的，对 volatile 字段的每个读、写，就像扮演了 “ 半 ” 同步的操作。  

 
重要提示：为了正确地建立前序（ happens-before ）关系，重要的一点是要让所有线程访问相同的 volatile 变量。否则，诸如当线程 A 写入的是 volatile 字段 f ，而在线程 B 读取的却是 volatile 字段 g ，那么所有的可见情况，未必会如预期。 释放（ Release ）和 获取（ acquire ）必须要 “ 匹配（ match ） ” （比如：要针对同一个 volatile 字段），以获得正确的语义。        

  
## volatile 能做和不能做的   

  1. volatile 可以保证可见性，即 每次读取时都从主存中读，每次修改时，都将修改后的值重新写入了主存。  
  2. volatile 可以防止了容易产生超出预期结果的指令重排序，这个影响可以扩散到程序顺序中前序的指令。【具体可参见 本系列第 (4) 篇】。
  3. volatile 不能保证原子性（所以，不能用做多线程下的计数器）。
  4. 凡事都有代价。volatile 防止一部分指令重排序的特性，削弱了JVM中的运行时优化，同时强制的缓存刷新和失效的操作，都会降低运行效率。 所以不要过度使用。  


## volatile 的用与不用

### 不适合的场景
   1.常量（ final ）字段 **不能**用 volatile 修饰，否则会编译出错。

   2.只有单线程访问的变量，**不需要**用 volatile 来保证可见（好像有点废话 ）。

   3.存在复杂的 “锁” 的操作需求的场景（比如对变量值修改期间，要阻止读取——即，读取值锁定），**不适合** 用 volatile 来实现。 

### 适合的场景  
   作为类似 “信号量” 变量 来使用，保证每个线程都可以看到当前的最新“信号”标志，且信号值的改变与当前值无关，并且不关心改变的历史量。 

>但，如果需要管理大量类似的 “信号量” ，使用 java.util.concurrent 包中的一些数据结构，显然是效率和性能更好的选择。  

## 为什么 GO 语言中没有 volatile 
  “Do not communicate by sharing memory; instead, share memory by communicating.” 这句“ 名言 ”，很好的概括了这一设计思想。[^GoComm]  

## volatile 和  Atomic 类型变量      
2004年，就在这篇 FAQ 发布后的半年后，JDK1.5 （JDK5 ）也发布了。JDK5 包含了全新
的并发工具包，就是大家熟知的 **java.util.concurrent** 包。 concurrent 包很大程度
地简化了并发应用的开发，后续影响深远。其中包括：    

  - **Executor 执行框架**，更好的管理线程池的方式。（较之于很多人自己手动管理多线程任务，
  可能更稳定，也能更好的管理资源的消耗。）   

  - **更好支持并发的集合类型**，比如 **BlockingQueue（ 阻塞队列 ）接口**，**ConcurrentMap 接口**及引用此接口的 **ConcurrentHashMap类** 等。（后续的版本里，这部分集合被持续的丰富了。也是很多面试官的考题来源）。  

  - **Atomic 工具类**，提供了无锁（lock-free）、线程安全（thread-safe）的变量操作。  
  
  **Atomic类型是对volatile修饰的进一步扩展** ， 

  - （对Atomic 变量的）*get* 操作（如：incrementAndGet ）等同于对一个 *volatile*变量的读取（等同的内存操作效果）。 类似的，*set* 操作，相当于对一个 *volatile* 的写入效果。  
    
  - weakCompareAndSet 操作，相当于先进行了一次 Atomic读取，然后再决定是否写入。但是这个过程不与周边代码形成强的“ happen-before（先序）”，所以只确保本身的可见性，不会确保周边变量的可见性。【关于这一点，之前的几篇的篇幅里，有例子提到过】  
    
  - JDK 1.6 以后，又引入了lazyset[^LazySet]。典型场景是应用在垃圾收集时，清零（nulling out ）一个永远不会再次访问的引用。采用的是 store-store 的内存栅栏方式，即保证这个写入能在后续其他的写入前，被其他线程可见。[^Store-Store] .  
    
  - Atomic 使原本非线程安全的数组（Array ），具有了线程安全的替代（AtomicLongArray ）。 
    

  
## 小结 
volatile 是个相当古老的编程语言关键词了。有不少经典的文章,如 *volatile vs. volatile*[^VvsV]。今天来聊这个话题，似乎显得有点过时。 也可以预见，对它的运用也会逐渐的淡出。

不过，深入到底层，我们还是可以发现，原理性的东西，其实并没有想象中的变化那么大————较之于眼花缭乱的时髦新技术名词而言————依然有助于我们理解和思考并发状态下的一些问题。  



**其他参考链接**
---
>  -[Stack overflow:atomicinteger-lazyset-vs-set](https://stackoverflow.com/questions/1468007/atomicinteger-lazyset-vs-set)   
>  
>  -[Oracle:High Level Concurrency Objects](https://docs.oracle.com/javase/tutorial/essential/concurrency/highlevel.html)  
>  
>  -[Concurrency in C++11](https://www.classes.cs.uchicago.edu/archive/2013/spring/12300-1/labs/lab6/)  
>  
 

**参考索引**   
--- 
[^GoComm]:[Go Blog :Share Memory By Communicating ](https://blog.golang.org/share-memory-by-communicating)  

[^LazySet]:[JDK-6275329 : Add lazySet methods to atomic classes](https://bugs.java.com/bugdatabase/view_bug.do?bug_id=6275329)  

[^Store-Store]:[Multithreading相关术语总结](http://dreamrunner.org/blog/2014/07/05/multithreadingxiang-guan-zhu-yu-zong-jie/)  

[^VvsV]:[volatile vs. volatile](http://www.drdobbs.com/parallel/volatile-vs-volatile/212701484?pgno=1)  



<end>

