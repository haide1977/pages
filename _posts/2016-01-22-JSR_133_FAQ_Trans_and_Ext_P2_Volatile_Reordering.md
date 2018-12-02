---
layout: post
title: JSR 133 FAQ的翻译和扩展（2）—— 内存模型的困扰
key: 20160122
tags: 架构 并发与并行 开发语言
---
## 前言  
多线程并发态下的内存模型经常带给开发者的困扰，概括来说就是反直觉（ counterintuitive ）。
看上去代码逻辑没有问题啊，但是结果却出乎了我们的预料。这类问题实际更像是“ 一个硬币的两面 ”。  
  
本部分，会结合FAQ原文中的“重排序（Reordering ）” 和“旧的内存模型的问题” 等相关问题来展开相关讨论。  
<!--more-->   

## 硬币的两面    
一面，在多数情况下，是开发者对该开发语言底层原理，包括与之相关体系架构的代码编译和运行时（Run time ）逻辑并不真正的了解，而仅仅是站在高级语言的角度去揣测低层运行行为。这一点，是高级语言的程序开发和书写汇编语言程序的一个重要的差别。  

另外，有些高些语言，为了提升业务逻辑的开发效率、平缓学习曲线，也会无意间向开发者传递这样的错觉。

在单线程态下的程序代码，开发者的上述不足，基本可以通过测试用例的设计来弥补，因为其中潜在问题的复现是相对稳定的。而在多线程态的程序代码中，由于存在更多的不确定性，这类BUG 往往会躲过不完善的测试【注1】，被发布到生产环境中。
  
>注1: 本系列会在覆盖其他有关知识点后，对这类场景下的“ 测试 ”问题 进行专题梳理。        

另一面，任何一个语言的编译器、运行时（Run time ）的优化，同样需要一个较漫长的过程。真正的稳定成熟，从Java ，
或者Go 来看，差不多需要10 年左右周期。2004 年，对JVM 而言，差不多到了这样的演进节点。对于Go 这样的新生语言来说，内存模型或许不再需要完全摸石头，但是新的场景同样意味着新的挑战[^ToGo2]【注2】。

>注2：本段在2018年初，重新进行了修订。    

## ▸旧的内存模型哪些问题？【译】  

[原文][What was wrong with the old memory model?](http://www.cs.umd.edu/~pugh/java/memoryModel/jsr-133-faq.html#oldmm )  

>翻译注：  
>本段提到的final，volatile等语言结构，会在系列的其他部分中单独讨论，此处暂不需展开。   

旧的内存模型存在一些严重的问题。由于理解起来困难，也存在大量的使用背离。例如，在很多场景下，旧的内存模型，不允许在 JVM 中发生多种的重排序。 这种旧内存模型的潜规则的困扰在 JSR-133 中被正式的公式化表达了出来。   
   
之前有一个被广泛确信的想法是，如果使用了 final 字段，那么线程间的同步时，就不需要去额外保证去确保另一个线程对这个字段值的可见。然而，这个貌似合理的假设且明智的行为模式，在旧的内存模型里，当我们真希望事情如此进展的时候，显然情况不是真的（如此）。在旧的内存模型里，对待 final 字段与其它任何字段，并无二致——这意味着，同步处理是保证所有线程可以看到被构造器写入后的 final 字段值的唯一途径。由于（在旧的内存模型里）一个线程既可能看到这个字段的默认值，也可以在之后看到它被构造后的值。也就是说，例如，一些不可变对象，如 String ，可以看上去它们在改变自己的值 —— 这着实是令人困扰的预期。     
   
旧的内存模型允许可变的（ volatile ）（字段）“ 写 ”与不可变的（ nonvolatile ）字段的“ 读 ”，“ 写 ” 进行重排序，这与大部分开发人员对于可变的（ volatile ）语义的直观理解不一致，并会由此产生困扰。    
   
最后，正如我们所见，编程人员对于他们的程序在不正确地同步时会发生什么的直觉，通常是错误的。 JSR-133 的其中一个目标，也是为了让这一事实引起注意。   

  
## ▸重排序的含义是什么? 【译】   

[原文][What is meant by reordering? ](http://www.cs.umd.edu/~pugh/java/memoryModel/jsr-133-faq.html#reordering)    
   
在很多场景里，访问程序变量（对象实例的字段，类静态字段及数组元素）的执行顺序可能不同于程序中确定的顺序。编译器可以根据优化的需要，自由地改变指令的执行顺序【注3】 。在某些特定的环境下，处理器会打乱执行指令的顺序。数据也可能不按照程序中设定的顺序在寄存器、处理器缓存和主存间移动。   

>注3：本文后半部分有对指令执行相关的补充。    

例如，如果一个线程先写字段 a ，然后写 b ，且 b 的值不依赖 a 的值的话，编译器可以自主地重排序这几个操作，而且缓存也可以自主地，在 a 之前，把 b 刷新到内存。存在许多潜在的重排序根源，比如，编译器、 JIT （翻译注：这里应该是指实时编译器，后文会有补充）和缓存等。   


编译器，运行时（runtime ）和硬件共同造成了一种 “ 看上去顺序化( as-if-serial) ” 的语义假象，即，在单线程的程序里，应该不能够观察到重排序的效果。然而，在不正确地同步的多线程程序中，重排序的效果就会显现出来。此时，一个线程能够感受到其它线程的影响，可能会发现对其它线程可见的变量访问（variable accesses ）的顺序，与程序内设定的或者执行的顺序是不一样的。     
   
大多数时候，一个线程不关注其它线程在做什么。但是，如果需要在意的话，就需要涉及到同步了（ that's what syschronization is for ）。     


## 与重排序有关的概念和语义

### § 指令周期与重排序
一般而言，CPU 的指令周期主要包括四个步骤：  

  1. 获取指令 Fetch the instruction   
  2. 解析指令 Decode the instruction 。解码指令的过程也会把指令所需数据等从主存中载入（ load ）  
  3. 执行指令 Execute the instruction   
  4. 写回（ Write- Back ）结果。 把指令产生的结果数据写回（ Store）主存等存储中。  
    
或者像类似下面的伪代码表述（这里没有包含例外处理过程）[^IssJvms]：    
  {% highlight text %}  
  do {
      // pc-- program counter 指令计数器 
      // opcode--  指令码 ，JVM的指令码长度为1个字节。
      atomically calculate pc  and fetch opcode at pc;  

      //operands   操作数，一条指令码可能需要 {0...n}个操作数。 
      if (operands) fetch operands;
      execute the action for the opcode;
   } while (there is more to do);  
 {% endhighlight %}  


不同的 CPU 有不同的指令集，也有不同的指令周期，具体载入和写回的策略也有不同，实际情况远远比这四个步骤更多，也更复杂。但总体不影响讨论重排序主题。   
    
由于2，4 两个步骤，涉及到数据与存储间的操作，相对与 CPU 有多个数量级的速度差，很容易成为阻塞指令的瓶颈。为了使 CPU 在每个时钟周期都不空闲，所以有了“指令管道（ Instruction Pipeline ）” 【详细内容可参见 wikipedia】[^PipeWiki]和 “分发队列 （ Dispatch Queue ）”。  

  - 指令管道 （ Instruction Pipeline ）   
    
    ![指令管道示意图（来自wiki）](https://upload.wikimedia.org/wikipedia/commons/thumb/6/67/Pipeline%2C_4_stage_with_bubble.svg/350px-Pipeline%2C_4_stage_with_bubble.svg.png "指令管道示意图")   

      >原图出处：[https://en.wikipedia.org/wiki/Instruction_pipelining](https://en.wikipedia.org/wiki/Instruction_pipelining  )    

    绿、紫、蓝、红分别代表了指令管道中并行的四条指令。   

    蓝色椭圆代表了该时钟周期（ Clock Cycle）出现的指令执行空闲。按照上面示意图来说，紫色这条指令，从“解析指令”开始，由于某种原因（也许是数据尚未准备好），堵塞了一个时钟周期。从而导致了所有四条指令的完成往后拖延了一个时钟周期。   
       
  - 分发队列 （ Dispatch Queue ）  
    分发队列从解析指令／写回指令的工作中，把Load 和 Store这两个动作相对松耦合的独立出来，从而进一步降低指令执行堵塞的可能性。这样的设计是很合理、自然的思路。显然对提高 CPU 的整体性能有利。

      >JVM spec v2中定义的 Load 和 Store 主要四类指令，可参考：  
      > [https://docs.oracle.com/javase/specs/jvms/se7/html/jvms-2.html#jvms-2.11.2](https://docs.oracle.com/javase/specs/jvms/se7/html/jvms-2.html#jvms-2.11.2  )  
      
有了上述提高性能的设计的描述补充，再来理解前文提到的重排序场景就比较容易了，比如：    

  - CPU 的指令调度算法，必然不会容忍大片时钟周期的空闲，会将当前具备运行条件的指令提高优先级，反之亦然。  
        
  - 多个 CPU + 多个指令管道的模式下，如果没有特定的指令语义约束，期望严格有序几乎是不可能的。      
    
  - 分发队列，为了保证其自身的分发性能，必然也会对不同读写速率的数据分发做必要的重排。   


### § JIT与重排序
[![JIT原理图](https://i.postimg.cc/6qGdTcTV/20160122jit.png)](https://postimg.cc/BP48VxTt)   
>原图出处：http://belajarjava-19.blogspot.jp/2011/05/jdk-dan-jre.html  

对于程序代码中的一个特定的方法而言，JIT（Just-In-Time ）编译 **当且仅当** 该方法第一次被调用，需要把它从字节码编译为本地可执行代码的时候会发生。   
  
JIT 这样的动态编译（ Dynamic Compliation ）[^JitWiki] 的区别与静态编译（ Static Compliation ）的一个明显好处是：可以根据实时运行态的状况，对代码进行简单的优化。这些优化包括：**优化循环嵌套、把堆栈操作变成寄存器操作、通过分配寄存器来减少内存访问、减少不必要的中间表达式等** 。这个过程，有点类似现在的 DBMS（ 数据库管理系统 ） 对 SQL 的优化，可以一定程度建立在对在表和数据的特征统计基础上，而不是单纯的依据规则。    
  
所以，对于这个过程中，重排序的产生，就很顺理成章了。  

不同的 VM 对 JIT 优化能力和结果的差别可以是巨大。  对于应用开发者而言，我们最后还需要清楚 **优化过程本身也需要消耗 CPU 和内存等资源**。 所以，不妨可以多看看诸如 *《java高性能代码编写》* 之类的总结文章。这是题外话 。

>更多关于JIT编译和优化的内容，可参考：  
>  - [https://www.oreilly.com/library/view/java-performance-the/9781449363512/ch04.html ](https://www.oreilly.com/library/view/java-performance-the/9781449363512/ch04.html)    
>  - [https://aboullaite.me/understanding-jit-compiler-just-in-time-compiler/](https://aboullaite.me/understanding-jit-compiler-just-in-time-compiler/)    
>  - [https://dzone.com/articles/performance-improvements-via-jit-optimization](https://dzone.com/articles/performance-improvements-via-jit-optimization)    
>  (新修订补充)  
  

### § as-if-serial 和 happen-before 语义  
前面都是讲的重排序场景出现的可能性。但是，需要强调的是，对单线程运行的一段程序而言，有一个限制指令重排的强约束，即，执行结果需要是稳定的、符合预期的———否则就变成了违背代码语义———没有了正确性，性能优化本身就失去了意义。  
  
as-if-serial 和 happen-before 本质含义是相同的。 即要遵循严格的事件偏序关系[^HBWiki]。 
简而言之，就要满足三个基本特性：**传递性、反自反性，反对称性**。  

另外，happen-before 的语义，会在本系列后续 涉及“同步”时，再进行讨论。


## 一沙一世界  
目前为止，本篇都在偏底层的角度讨论一些原理。对很多应用开发者而言，这些偏理论的细节与实际工作似乎很遥远。 其实，从单CPU 的并发、多处理器内核的并行，再到基于消息的分布式计算架构以及现在火热的微服务架构，思考的基本框架基本是一脉相承的。差别更多是在所聚焦问题粒度：是更微观的指令（ Instruction ）层面，还是更宏观层面的任务（ Tasks ）、服务 （ Services ）。    

下面的这个列表，几乎是现在分布式架构和开发工作中每天要思考的问题（实际上还远不止）：  

  - 分解。分解的策略和分解的粒度 （虽然我们不需要到指令级别）。  
  
  - 解耦。把非核心流程业务剥离到不同的事务队列管道中去。   
  
  - 通信。通信策略，通信协议。  
  
  - 有序和幂等。 消息的传递如何兼顾性能和准确性？  

  - 调度策略。 优先级／资源调度／迁入迁出。   
  
  - 流量控制和忙碌等待。本质上没有跑出阻塞／活锁／死锁等的范畴。  
  
  - 数据同步。永恒的焦点。  
  
 




---   
**参考索引** 

[^ToGo2]:[the Go Blog ：Toward Go 2 ](https://blog.golang.org/toward-go2 )  
[^PipeWiki]:[ Instruction pipelining ](https://en.wikipedia.org/wiki/Instruction_pipelining )    
[^JitWiki]:[Just-in-time_compilation ](Just-in-time_compilation )  
[^HBWiki]:[Happened-before ](https://en.wikipedia.org/wiki/Happened-before)  
[^IssJvms]:[JVM spec :Instruction Set Summary  ](https://docs.oracle.com/javase/specs/jvms/se7/html/jvms-2.html#jvms-2.11)   

*<end>*

