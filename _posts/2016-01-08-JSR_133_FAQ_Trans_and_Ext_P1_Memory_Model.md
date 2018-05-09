---
layout: post
title: JSR 133 FAQ 的翻译和扩展（1）——关于内存模型
key: 20160108
tags: 架构 并发与并行 开发语言
---
## 前言  
网上有很多JSR 133 FAQ 的中文翻译版本，一大半是大家为便于自己的学习理解，做的读书笔记似的翻译。这一点上，这个版本也不例外。  
  
不过, 本系列版本，尝试在不打破原文表达结构，更准确翻译表述的基础上，尝试对于一些关键概念，特别是随着时间推移已经有所变化的知识点，做一些拓展梳理。  
<!--more-->   

## 关于内存模型        
一句话表述：内存模型（Memory Model）描述了在进行计算时，**线程间**如何通过**内存**和**共享数据的使用**来进行**交互**。（Wikipedia）  

维基百科上关于内存模型的说明有一部分就引用自JSR 133，这里暂不再展开。[^WikiMM]     
  
## 关于JSR 133    
既然有“JSR 133 常见问题（FAQ ）”，顺理成章的思路，第一个问题似乎应该是：JSR 133是什么？   
  
但是这个FAQ 英文原文的组织方式，却不是这样（原文是作为JSR 133 系列文档的一部分，这样的组织也是没有问题的）。这里，为避免有人陷入缺少上下文的迷惑，这里先简述下JSR 133 ————更具体的内容，将在本系列（2）中说明。  

一句话描述JSR 133:  表述线程（threads）,锁（locks），volatile变量和数据竞争（data races）语义的Java语言建议规范，它包括了称为Java内存模型的内容。  

这一句话里的关键词看上去信息量够大，够技术，够诱人吧？  

## ▸为什么我要关注内存模型? 【译】  

[原文][why should I care?](http://www.cs.umd.edu/~pugh/java/memoryModel/jsr-133-faq.html#conclusion )  

>翻译注：  
>这本来是FAQ 原文的最后一个问题，但我觉得放在这里似乎更合适。   

为什么你应该关注？因为并发的 bug 是非常难调试的。这类 bug ，在测试的时候，通常不会被发现，直到程序跑在重负荷的状态时才会暴露；要重现和捕获也很困难。 
所以，最好把额外的功夫花在前面———确保你程序会被正确地同步；虽然，这也不太容易，但是比起花大量的时间去调试一个糟糕的同步程序，可能还是会简单很多。 
  

## ▸那么,内存模型到底是什么? 【译】   

[原文][What is a memory model, anyway?](http://www.cs.umd.edu/~pugh/java/memoryModel/jsr-133-faq.html#whatismm)  
  
在多处理器系统中，处理器通常有一个或多个内存缓存层（layers of memory cache ）来改善性能————比如，加速数据的访问（通过让数据离处理器更近），或者降低共享内存总线上的流量（因为许多内存操作可以通过本地缓存来满足）。 虽然内存缓存能极大改善性能，但也会成为新的挑战的根源。例如，当两个处理器在同一时间来查看同一个内存地址时，在什么情况下，它们会看到相同的值？   
   
在处理器层面，内存模型定义了充要条件来明确：   
其它处理器对内存的写入（如何）对当前处理器可见；   
当前处理器对内存的写入（如何）对其它处理器可见。     
  
一些处理器呈现的是强内存模型（strong memory model ），在这种模型下，所有处理器在任意相同时刻，看到该内存地址的值，是保证相同的。其它处理器提出了的一种弱内存模型（ weak memory model ），需要借助特定的指令，所谓内存壁垒（ memory barries，也有翻译“内存屏障” ），通过清空（ flush ）或者失效（invalidate ）本地缓存，来看到其他处理器的 “写” ，或让本处理器的 “写” 对其他处理器可见。 这些内存壁垒通常在执行锁（lock ）和解锁（unlock ）操作中体现（performed ）；而且，它们对于高级语言的编程者（high level programmers ）是不可见的。  

>翻译注：  
>关于强、弱内存模型模型的问题，本文后半部分有补充叙述。  
  
因为降低了对内存壁垒（内存屏障）处理的要求，所以，有时候，在强内存模型下写程序会简单一些。然而，即使在最强的内存模型（ strongest memory model ）里，内存壁垒也经常是必要的；更常见的是，它们的处置方式还会背离（我们）的直觉意图（counterintuitive ）。最近的处理器设计趋势，多倾向于弱一些的内存模型（weaker memory model ）。因为这样的（约束）放松，可以让缓存一致性（问题）（cache consistency ）在跨多处理器和更大内存的（场景下），有更大的可伸缩性（greater scalability ）。   
   
什么时候 “ 写 ” 操作（的结果）变成对于其他线程是可见的，这个问题取决于（ compounded ）编译器的代码重排序（ reordering of code ）结果；例如：编译器可能认定 “ 写 ” 放在程序的更后面（后移）会更有效率，只要这段代码的行为没有改变程序的语义，就可以被允许这样做。当编译器推迟执行一个操作时，在此操作被执行完之前，对另一个线程而言是不可见的。这也反映了缓存的作用。  
  
此外， “ 写 ” 内存也可以在程序中提前（前移）。在这种场景下，其他线程可能比实际程序发生（ actually "occurs" in the program ）更早的看到一个 “ 写 ” 内存（翻译注：可能会读到违背代码原意的值）。所有的这些灵活性都是被设计出来的 —— 在特定的内存模型的边界（ bounds ）内，通过给予编译器、运行环境或者硬件，在执行操作时，优化顺序执行的灵活性，来获得更好的性能。   
   
一个简单的例子：   

{% highlight java %}
Class Reordering {          
  int x = 0, y = 0;        
  public void writer() {       
    x = 1;    
    y = 2;         
  }        
              
public void reader() {         
    int r1 = y;        
    int r2 = x;        
  }        
}    
{% endhighlight %}  

比如说，这段代码在两个线程中并发执行，且对 y 读取到值 2 。因为程序中对 y 的写是在 x 之后，所以，编程人员可能会假设此时能读取 x 的值会是 1 。然而，实际上，这段写操作可能会被重排序（ reordered ）。如果这个情况真的发生了，那么有可能， y 先被写成了 2 ，然后对 x ， y 两个变量的读取接着进行，最后对 x 的写操作才会开始。此时的结果就可能变成了 r1 的值等于 2 ，而 r2 的值却还是 0 。     
   
java 内存模型描述了什么样的行为在多线程的代码中是合法的（legal in multithreaded code ），以及线程与内存的可能交互方式。同时，描述了程序代码中的变量与它们在真实计算机系统的内存或寄存器中的存、取的底层（low－level ）细节之间的关系。以便通过这种方式，保证代码在差异巨大的硬件和不同的编译器优化时，都能够正确的实现。     
   
java 包含了若干的语言结构，包括 volatile ， final 和 synchronized , 以帮助编程人员向编译器表述一段程序的并发（状态下的）需求。 java 内存模型不仅定义了 volatile 和 synchronized 的行为方式，而且，更重要的是，保证了（任意）一段（包含了）正确同步（ synchronized ）处理的java 程序代码在所有处理器架构下都能正确地运行。  
  

## ▸其他语言，比如C++ ，有内存模型么？【译】    

[原文][Do other languages, like C++, have a memory model?](http://www.cs.umd.edu/~pugh/java/memoryModel/jsr-133-faq.html#otherlanguages)  
  
  
>翻译注：  
>原文写于2004年，实际上很多情况近些年已经发生了变化。具体内容请见后文。  
  

大多数的其他编程语言，像 C 和 C++ ，都没有设计针对多线程（ multithreading ）的直接支持。这些语言提供的，对于发生在不同编译器和架构下的重排序（ reordering ）的各种保护，都严重依赖于所用编译器及代码运行所在平台使用的特定线程库（ threading libraries ）的保证（ guatantees ），例如 pthreads 。  

>翻译注：  
>这里的 pthreads 应该指的是 POSIX thread  库（ www.cs.cmu.edu/afs/cs/academic/class/15492-f07/www/pthreads .html ），而不是 php 的多线程库 pthread （参见： pthreads.org   
  
## 关于内存模型的补充   
 
自2004年2月，这篇著名的FAQ 发布以来，互联网世界已经发生了巨大的变化。例如，那时还没有Github，没有Go  语言，也没有双十一这样制造瞬间高并发的购物节；移动应用还处于wap ，stk 、短信这样的初级阶段；很多人在帮客户做技术方案时，还要向用户宣讲B/S 还是C/S 的优劣比较.... 等等 。  

还有，作为Java语言的发明者及第一个JVM的推出者的Sun公司，应该没有完全预料到自己会在短短5年后被Oracle收购。  
  
近10年里，主要的编程语言也是在相互借鉴中演进，语言的特性已经不像早期那样泾渭分明了。总体来说，多是为了适应计算和开发场景的需求，特别是越来越多服务端面对高性能、高并发的要求，以及普遍的分布式，大规模并行的架构特点。 
  
一些经典的语言（如C++ ）的新标准中，补充了这部分缺失的特性。而新的语言，如Go  ，在一开始就加入了这样的特性。 虽然语言不同，但从理论基础角度讲，差别并不大。  

### § 内存模型中的“内存”   

众所周知，Java语言当初诞生的一个核心理念是 “Write once，run anywhere”   ————当然，最后演变成了“Write once ，Debug everywhere” ————但是，为了支撑这一理念，必须抽象出一个与硬件架构相对无关的JVM规范。对上，作为应用开发者的规范，对下，作为开发不同架构下JVM的依据。所以，很明显，JVM 内存模型中的内存，实际是个抽象概念，对应到具体计算机系统，可能涵盖CPU寄存器、CPU 缓存（诸如，一级缓存，二级缓存等等）以及内存。  

对于绝对多数不会去参与JVM 开发的应用研发人员来说，对这个 “内存” 问题，目前了解到这一点，已经基本够了。

### § 处理器的内存模型  

前面译文中，有较多篇幅提到“处理器”的强、弱内存模型。这个话题涉及到计算机体系架构。随着并行运算框架的成熟和pc 服务器的流行，x86 以外的处理器架构，似乎已经渐渐淡出一般开发人员的视野了。实际上，现在的大部分处理器按照“放松的一致性”（Relaxed Consistency ）方式在运转，因为性能永远是处理器的第一追求。当然，问题的另一面是，几乎所有的处理器事实上都是可以支持严格的顺序一致性的————即所有的 “写” 的行为都要被从缓存中刷入主存以后，才能被读取。  

所以，了解还有很多不同内存模型的处理器[^MemPaul]，比如大量存在移动计算节点的ARM架构，对于我们开发一些特定应用时，或许会增加一些避免犯错的预见能力。  


### § C++ 的内存模型    

在C++标准1998(C++89 ) 年版发布了13年以后，新的C++标准2011版（C++11 )，终于增加了多线程特性的支持[^Cpp11] 。
  
在C++11 中，发布了的并发内存模型最终版本[^CppMM],原子类型和操作(Atomic Types and Operations )，线程本地存储（Thread-Local Storage ）等 。而在此之前，就像前述FAQ原文所说，需要严重依赖线程库的支持。 但是，利用POSIX这类库的来实现，往往不够优雅，而且存在性能的问题。这类类库对于把单线程的应用转换为多线程应用时候，也不是特别的有帮助。C++11 大致相同的实现了一些厂商（包括IBM ，Intel 等）实现的对线程本地存储这部分的语言扩展[^CppAtom] [^CppTLS]。

为了在C++代码中可以有下面的一句声明，背后的工作实际上是大量的。    

{% highlight C++ %}  
  #include <thread>
{% endhighlight %}  

大量支持高性能、高并发的C++ 的组件和项目支撑着当今互联网世界的核心骨架。目前最受瞩目的两大应用趋势，人工智能和区块链也毫不例外，其中包括流行的机器学习和神经网络库 tensorflow ，caffe；比特币和以太坊的核心实现等等。  

C++ 并发编程这个主题不再这里进一步展开了。有一本对C++11 标准相对实操的"名"著（目前已出第二版） *"C++ Concurrency in Action"* [^CppAction]，可在对相对抽象的官方文档了解地基础上延伸阅读。      

### § Go 的内存模型  

本段要讨论，不是Go语言的语法。只是作为获得越来越多社区支持，为并发编程而生的语言样本进行研究。
  
JSR 133 开创了从语言规范层面思考和定义内存模型的先河。所以在多年以后，仍然被开发人员不断拿出来解读，也成为其他语言在演进或者创造时，必然要思考的问题框架。

Go从语言层面定义了”goroutine " ————由Go运行环境管理的轻量级线程,确切的说，更接近于协程[^Corou]。一个程序在初始化时，只运行了一个goroutine 。 由这个goroutine，可以创建出其他并发运行的goroutine 。

先看几行的代码的例子：  
  
{% highlight Go %}  
  // 假设有一个函数 f(x,y,z) ,每增加一行下面的代码，可以启动一个新的并发执行的goroutine 
  go f(x,y,z) 
  // 如果我们把函数 f（）变成我们更加熟悉的两个函数名呢？   
  go produce(x,y,z)  
  go consume(x,y,z) 
{% endhighlight %}  
  
在其他一些语言，如Java ，可能需要用线程来实现的 “生产” 和 “消费” 例子，
这里Go语言中，可以用两个并发执行的goroutine 实现。更重要的是，goroutine 不依赖于CPU内核数量，不需要内核态的切换。它运行在线程之下，一个线程可能包含成百上千个goroutine 并发。每个goroutine 仅需4k～8K的内存。
  
现在，回到主题。Go 的内存模型，同样是为了表述在什么样的条件下，一个“线程” 的 “读” 操作可以保证 “看“ 到另一个 “线程” 对同一个变量 “写” 的值的结果。这里的 ”线程“ 在Go 语言中特指的是"goroutine" 。 如果用锁（lock ）这样的方式，来显式的控制同步，保证goroutine 共享内存的严格有序，显然也是可行的。但，无论从代码书写的优雅简洁，避免代码里的逻辑混乱（加锁就要解锁），还是性能角度（所有的锁，都意味着强行的串行化）————都可以有更恰当的方式。  

或许是基于此，Go语言设计了Channel 和 Channel Buffer （即，限定了缓存数量的Channel ），通过Channel 通信，是Go语言中goroutine 之间同步（sychronization）的主要方式，保证了多个goroutine 对共享变量 “写” 和 “读” 的严格有序。 在 Go 语言中，这个场景下的更常见表述方式是“send” 和 “reveive” 。   

“Do not communicate by sharing memory; instead, share memory by communicating.” 这句“ 名言 ”，很好的概括了这一设计思想。[^GoComm]  
  


## 本部分小结    
1. 选择C++ 和Go 作为两个参照样本进行补充展开，主要从Java 的演化轨迹看，正好同时并存，并且一个代表了经典且继续老当益壮的语言；另一个代表了近10年内编程语言的一些发展趋势。    
2. 由于很多内存模型常用概念，散落在FAQ 原文的其他部分，所以会在相应的部分进行必要的展开。  
3. 原文的FAQ ，重点在并发与同步，而不是Java 运行态中的内存管理————如内存的回收等等——这是另外一个话题。
 

>注：本文在2018年初，重新进行了修订和补充 
  
>推荐的一些相关链接：     
>- [https://tour.golang.org/concurrency/1 ](https://tour.golang.org/concurrency/1 )   
>- [Channel, ttps://tiancaiamao.gitbooks.io/go-internals/content/zh/07.1.html](https://tiancaiamao.gitbooks.io/go-internals/content/zh/07.1.html )   
>- [PopularitY of Programming Language, http://pypl.github.io/PYPL.html](http://pypl.github.io/PYPL.html)     
>- [C语言之由内存模型说起,http://binyanbin.github.io/2016/03/25/c-3/ ](http://binyanbin.github.io/2016/03/25/c-3/ )  
>- [Understanding Go Lang Memory Usage, https://deferpanic.com/blog/understanding-golang-memory-usage/ ](https://deferpanic.com/blog/understanding-golang-memory-usage/ )




---   
**参考索引** 

[^WikiMM]:[Memory Model（Progamming） ](https://en.wikipedia.org/wiki/Memory_model_(programming) )   
[^Cpp11]:[C++11 并发（Concurrency ）相关定义  ](https://gcc.gnu.org/projects/cxx-status.html#cxx11 )  
[^CppMM]:[C++11 内存模型（Memory Model） ](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2007/n2429.htm )  
[^CppAtom]:[C++ 原子化（Atomic ） ](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2007/n2427.html )
[^CppTLS]:[C++ 线程本地存储（Thread Local storage ）](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2008/n2659.htm  )  
[^MemPaul]:[Memory Ordering in Modern Microprocessors by Paul McKenney](http://www.rdrop.com/users/paulmck/scalability/paper/ordering.2007.09.19a.pdf )
[^CppAction]:[C++ Concurrency in Action ](http://www.bogotobogo.com/cplusplus/files/CplusplusConcurrencyInAction_PracticalMultithreading.pdf)  
[^Corou]:[协程 Coroutine](https://en.wikipedia.org/wiki/Coroutine )   
[^GoComm]:[Go Blog :Share Memory By Communicating ](https://blog.golang.org/share-memory-by-communicating)  

*<end>*
