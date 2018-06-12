---
layout: post
title: JSR 133 FAQ的翻译和扩展（3）—— 关键定义（一）
key: 20160514
tags: 架构 并发与并行 开发语言
---  

## 前言  

如果真正对Java 语言和 JVM 的很多底层细节和原理有兴趣，那么，阅读官方的 Java 语言规范 （*The Java Language Specification*  简称 JLS ）和 Java 虚拟机规范 （*The Java Virtual Machine Specification*  简称 JVMS ） ，无疑是最有帮助的，也最原汁原味。但是，对于很多研发人员来说，阅读这样的文档，无疑是一种挑战 。

<!--more-->   

## ▸JSR 133 是讲什么的？【译】    
  
[原文][What is JSR 133 about? ](http://www.cs.umd.edu/~pugh/java/memoryModel/jsr-133-faq.html#jsr133 )  

>翻译注：  
>JSR 133 里的内容和思想，最终是要在 **Java 语言规范**  和  **Java 虚拟机规范** 中体现。[^JSR133]     
  
1997 年以来，定义在 Java Language  Specification 第 17 章（ Chapter 17 ）的 Java 内存模型已经被发现了一些严重的缺陷（ flaws ）。这些缺陷会允许一些令人困惑的（ confusing ）行为发生 ( 例如 final 字段（ fields ）被发现改变了它们值 ) ，同时也削弱了编译器执行常见优化的能力。     
   
Java 内存模型是一个雄心勃勃的尝试；它第一次从一个编程语言规格说明（ specification ）层面来体现一个可以在跨越不同架构时，仍可保持对并发的语义上一致的内存模型。不幸的是，定义一个既能保持一致性，又直观无二义的内存模型，已经被证明要远比想像的更困难。   
   
完整的语义表述可参见：http://www.cs.umd.edu/users/pugh/java/memoryModel ,   但是正式的语义不是为小心翼翼者准备的。它以令人吃惊和清醒的方式揭示了像同步这样看上去简单的概念，实际上会多么的复杂。幸运的是，你不需要去了解正式语义中的全部细节 ——JSR133 的目标，是创造一整套正式语义来为 **volatile** ，**synchronized** 和 **final**   的运行提供一套直观无二义的框架。  
  
具体来说， JSR133 的目标包括：   

  - 保留已有的安全保障，如，类型安全（ type-safety ），同时强化一些其它保障。例如，变量的值不能无中生有（ out of thin air ）的生成：某个线程能观察（ observed ）到的一个变量的每个值，必须是被其他某线程合理设置的值。 
  - 正确同步的程序的语义应该尽可能的简单、直观。   
  - 不完整或者不正确的同步程序的语义，也应该被定义出来，以便把潜在安全威胁降到最低。   
  - 使编程人员能够确信地推断出多线程的程序是如何与内存进行交互的。   
  - 使设计出正确、高性能，同时能广泛的跨越多个流行硬件架构的 JVM （具体）实现成为可能。   
  - 提供一个新的叫做 **初始化安全** （initialization safety ） 的安全保障。如果一个对象被正确的构建了（即，对它的引用不会在构造期间暴露（ do not escape during construction ）），那么所有对这个对象的引用可见的线程，同样对它在构造器中设定的 **final** 字段（ final fields ）的值可见，并且，这个过程不需要同步。   
  - 对已有代码的影响应该最小化。     
  
  
## JSR 133中的关键定义（一）

>说明：JSR 133 的第 5 部分 Definations 为一些概念做了更详细的定义。这些定义还是偏抽象理论。这部分会结合JLS 和 JVMS 的具体内容来进一步解读。       
  

### 1. 共享变量／堆内存 (Shared variables/Heap memory ）    

> JSR 133 原文翻译：  
> 能够在线程间共享的内存被称为 “**共享**（shared 
> ）“内存 或者 “**堆**（Heap ）”内存 。所有的实例字段（instance fields ），静态字段（static fields ）以及数组元素都存储在 **堆**内存中。(翻译注：为便于表述 ) 我们使用 **变量(variable )**这个词来表示字段和数组元素。 方法中的局部变量永远不会在线程间共享且不受内存模型影响。    
   
本定义中涉及的部分关键词，在JLS 和JVMS 中的具体落地包括 ：  
       
#### ∙  **共享变量 （Shared Variables ）** 及 **冲突（Conflicting ）** 【JLS §17.4.1 】 [^ShaVar]  
  在JSR 133 明确三类共享变量。JLS中，进一步明确 **冲突** 概念 。  
  所谓 **冲突** ，就是同时对同一个恭喜变量，同时有两个（或以上）的 “读 ” ， “写 ”访问， 且其中至少存在一个 “ 写 ”访问。 换言之，如果所有的访问都是“读 ”，那么就不存在冲突了。
                    
#### ∙  **实例字段** 和 **静态字段 （static fields ）** 【JLS §8.3.1.1 】[^StaFie]  
结合这两个文档的说明：

  - 实例字段（或称：实例变量，或：非静态 non-static 字段 ）： 如果一个字段没有被声明为*static*， 当它的类被实例化为对象时，这些字段会生成独立的字段变量拷贝，这些变量被称为实例变量。它们属于不同的实例对象，保存在不同的内存地址（注：是保存在 **“堆”**内存内），可以拥有各自不同的值。 实例对象生成的数量决定了实例变量在内存中的数量。  
              
  -  静态字段（或称：类变量 ）：加上了 *static* 声明修饰的类字段。每个类的实例共享它们的类变量，也可以去尝试改变它们的值。类变量仅与类相关，与该类生产实例的数量无关（甚至可以是 0 个），唯一的保存在一个固定内存地址（注：**“堆”**内存中），不需要通过类实例化后再去操作（manipulate）它。    
                
 下面这张图做了比较好的示意。    
    ![字段示意图](http://p1wq6a0o4.bkt.clouddn.com/20160514fields.png-m640?v=160514 "字段示意图")  
    
    原图出处：https://cdn.crunchify.com/wp-content/uploads/2013/04/Java-Instance-Field-Crunchify-Tips-768x395.png   
         
  在【JLS §8.3.1.1 】[^StaFie] 中的几个例子，对于理解包括这两类变量在继承状态下处理原理等，也很有帮助。 

  也可以参考： [Oracle，Java Tutorial：Classes and Objects](https://docs.oracle.com/javase/tutorial/java/javaOO/classvars.html ) 中的表述。  

  此处不再进一步展开。  
     

#### ∙  **本地变量 （Local Variables ）**等 【JLS §14.4 】 [^LocVar]    
  在JLS 在明确共享变量的同时，也明确了 **本地变量、方法的形式参数以及例外（ exception ）处理参数** 这三类，不会在线程间共享，也不会受内存模型影响。 

  本地变量是指定义在程序块( Block ）中的变量，比如两个“ { }”括起来的部分 。程序块的严格定义略繁琐，不太严格的说，就是一段形成闭合结构的程序片段。 对 Java 语言而言，这里有一个嵌套的概念。比如：if 外面套 for 循环，for 外面再套 switch。我们可以说，每一个都是 Block 。它们组合的每一层，也可以称为 Block 。  
      
  回到主题，对于本地变量等而言，每个线程都在执行时有一套自己的变量拷贝，不存在共享问题。【JVMS §2.6.1】 [^JvmsLoc] 。  进一步从JVM机制上说，是因为每个线程在调用方法时，会其形成一个不共享的运行帧（Frames，或译作栈帧），而局部变量【注：这里指的变量，不是具体的变量的对象，对象仍然在“堆”内存中 】也被包含在其中。     
    
#### ∙  数组变量 （ Array Variables ） 和  数组 Arrays 【JLS §10.2 】[^JlsArr]  和 【JVMS §3.9 】 [^JvmsArr]
   数组变量大家都很熟悉。 数组的本质是一组指向对象的地址引用。它有一旦创建，就长度固定的特点。所以，在多线程共享状态下，可能产生以下两种的不安全状态：  

  ∙ 初始化的冲突。常见的indexOutOfBoundsException。    
  ∙ 对特定数组指向的对象的值写入和可见问题（共享变量的共同问题）。      
     
#### ∙  **堆（Heap ）** 和 **运行时数据区（Run-Time Areas ）** 【JVMS §2.5.3 】 [^JvmsHeap] ，【JVMS §2.5】[^JvmsRTA]  
  关于 **堆** 和 **栈** 技术文章很多。 

  >需要快速直观认识这两个概念，可阅读 [Java Stack and Heap: Java Memory Allocation Tutorial](https://www.guru99.com/java-stack-heap.html)    

  这里强调以下几点：      
  ∙ **堆**中保存所有对象【包括局部变量指向的对象】和数组。     
  ∙ 类和方法体代码都 **不在** 堆内，而是在 **方法区（ method area ）**中 。  
  ∙ **方法区（ method area ）** 虽然逻辑上也是堆的一部分。但通常不进行垃圾回收，所以一般不作为 我们通常讨论的 **堆**的范围指向。      
  ∙ **堆**里的每个对象都和 **方法区（ method area ）** 的对应的类相关联 。  

  
  下图较清晰的表现多线程态下的JVM 内存运行时数据区（ Run-Time Data Areas ）逻辑划分：   

  ![Run-Time Data Areas](http://p1wq6a0o4.bkt.clouddn.com/20160514runtimeRev.png-m640?v=160514 "Run-Time Data Areas")  
     
    原图复刻自：http://slideplayer.com/slide/12666692/   （原图字体过小）   

### 2. 线程间动作 (Inter-thread Actions ）  

>JSR 133 原文：线程间的动作是由某一线程执行，能被另一线程探测或直接影响的动作（action）。

我们并不需要真正关心线程内动作（ Intra-thread Actions ）———因为理所当然的是遵循Java语言的语义。   

相对于JSR 133中的表述（此处不再赘述），【JLS §17.4.2 Actions】 [^JlsAct]的表述述更加直接一些。归纳来说，有如下若干类线程间动作：  

  - Read (normal, or non-volatile)。 读取一个变量。     
  - Write (normal, or non-volatile)。写入一个变量。
  - 同步（Sychronization ）动作，包括：  
      ∙ Volatile 读。  Volatile方式读取一个变量（注：关于 Volatile 会在另外系列表述）。  
      ∙ Volatile 写。  Volatile方式写入一个变量。  
      ∙ 锁（Lock ）。 锁定一个监视器（或译作：管程）[^ Monitor] 
      ∙ 解锁（Unlock ）。 解锁一个监视器（或译作：管程）
      ∙ 生成线程的第一动作和最后一个动作。 
      ∙ 启动一个线程或者检测线程是否已经中止。 
  - 外部动作（External Actions ） 。   
      外部动作是可以从外部观察到的（程序）执行动作，结果取决于（程序）执行的外部环境。  
      > JSR 133 ：与外部世界交互的动作。   
      似乎都有点语焉不详，那么到底是指哪些动作呢？    
      确切的说，是指：**JIT 编译器无法评估执行效果和影响的，与执行环境相关的动作**  
      具体来说，包括：**JNI 指令、与外部的socket 通信、对文件系统的操作、与控制台的交互（非排他方式）**等。   
      这些动作的共同特点是涉及动作在Java 虚拟机进程的完全控制边界之外（从而无法预先评估结果影响），所以，也 **不会被重排序**。  

      > JSR 133 ：针对外部动作的参数（例如：准备写入 socket通信的字节流），不属于外部动作的一部分。属于线程的内部动作语义的结果范畴。  
       
  - 线程分离动作（Thread divergence actions ）。 
     > JSR 133 ：让线程进入无限循环的动作，且这个动作不涉及内存（操作）、同步或外部动作。 
     典型的线程分离动作代码片段：  

     {% highlight java %}

       void  thread1（）{
         while (true){ // 线程分离动作 
           abc=123;     
           //to-do lists#1
           Thread.sleep(1000); 
           //to-do lists#2
         } 
       }

      void  thread2（）{
         if（abc==123){
           //TODO somethings 
         }else{
           //TODO some other things 
         } 
       }

     {% endhighlight %}
 
  按照上述例子，由于thread1 加了线程分离动作 ```while (true);``` , 循环中的```abc=123 ```和 其他 ``` //to-do lists#1 ``` 代码行中的语句不会被 **重排序**【重排序概念可参见本系列（2）】，即便代码块中, 线程主动插入 ```sleep（）```。  

  上述thread1的代码中，如果没有```sleep()``` 动作，则很可能出现 JLS §17.4.4所说的导致其他线程停止或挂起（没办法继续执行）的情况。  

  >Thread divergence actions are introduced to model how a thread may cause all other threads to stall and fail to make progress.   
       
  线程分离动作的应用场景可能有两个：  
    ∙  确保某些线程沿着预期的分支走（比如在测试的时候）。   
    ∙  在某种状态下，让一部分线程完全进入阻塞状态 。  
       
  关于线程分离还需要明确的一点： 例子里的 abc 的字段，无论是否采用 atomic 类型，都不能确保线程间相互的“可见”完全如预期。换言之，thread2中的两个分支的执行顺序，并不能做确定的假设。除非在程序中加入更强约束的，    
    
### 3. 程序顺序 (Program Order ）和线程内语义（Intra-thread semantics）  
这两个定义，都是为了界定一个线程，在隔离或孤立运行状态下，正确的执行顺序和执行语义。
此处不再重复 JSR 133 中的表述。 
  
这里需要了解的一个重要的概念是在 【JLS §17.4.3 】[^JlsSeq] 中提到的 顺序一致性 （ Sequential consistency ） 问题 。顺序一致性是保证可见性和有序性的强保证。换言之，运作环境可以把多线程的行为，进行全局排序执行，且每个动作都是原子的，就像单线程那样，那么，就不会存在数据竞争 （ Data Race）了。
  
实现严格顺序一致性处理的代价是很大的，尤其对于多核系统而言。 所以，我们显然需要更 **自由** 顺序的内存模型，不同线程间不需要假定相对顺序，而是接下来要说的同步动作和同步顺序来进行竞争与协作。    

## 未完待续  

接下来，我们就要借着原文 FAQ 的中相关问题，继续聊一聊同步动作和同步顺序，以及 happen-before 等问题了。   



>参考链接汇总:  
>  -[Oracle， Java Tutorial：Classes and Objects](https://docs.oracle.com/javase/tutorial/java/javaOO/classvars.html )   
>  -[Java Tips: Never Make an Instance Fields of Class Public](https://crunchify.com/java-tips-never-make-an-instance-fields-of-class-public/ )  
>  -[CS432:Compiler Construction lecture 15](http://slideplayer.com/slide/12666692/)    
>  -[Java Stack and Heap: Java Memory Allocation Tutorial](https://www.guru99.com/java-stack-heap.html)  


---   
**参考索引** 

[^JSR133]:[JSR-133: JavaTM Memory Model and Thread Specification ](http://www.cs.umd.edu/~pugh/java/memoryModel/jsr133.pdf )  
[^JSR133cn]:[JSR-133 非官方中文翻译 ](http://ifeve.com/wp-content/uploads/2014/03/JSR133%E4%B8%AD%E6%96%87%E7%89%881.pdf )  
[^ShaVar]:[ JLS §17.4.1 Shared Variables ](https://docs.oracle.com/javase/specs/jls/se10/html/jls-17.html#jls-17.4.1 )    
[^StaFie]:[JLS §8.3.1.1 static fields ](https://docs.oracle.com/javase/specs/jls/se10/html/jls-8.html#jls-8.3.1.1 )  
[^LocVar]:[ JLS §14.4 Local variables ](https://docs.oracle.com/javase/specs/jls/se10/html/jls-14.html#jls-14.4 )    
[^JvmsLoc]:[JVMS §2.6.1 Local variables](https://docs.oracle.com/javase/specs/jvms/se10/html/jvms-2.html#jvms-2.5.3)   
[^JlsArr]:[JLS §10.2 Array Variables ](https://docs.oracle.com/javase/specs/jls/se10/html/jls-10.html )    
[^JvmsArr]:[JVMS §3.9 Arrays ](https://docs.oracle.com/javase/specs/jvms/se10/html/jvms-3.html#jvms-3.9 )  
[^JvmsHeap]:[JVMS §2.5.3 Heap ](https://docs.oracle.com/javase/specs/jvms/se10/html/jvms-2.html#jvms-2.5.3  )    
[^JvmsRTA]:[JVMS §2.5  ](https://docs.oracle.com/javase/specs/jvms/se10/html/jvms-2.html#jvms-2.5  )   
[^JlsAct]:[ JLS §17.4.2 Actions ](https://docs.oracle.com/javase/specs/jls/se10/html/jls-17.html#jls-17.4.2)   
[^JlsSeq]:[ Sequential consistency ](https://docs.oracle.com/javase/specs/jls/se10/html/jls-17.html#jls-17.4.3)   
[^Monitor]: [wiki：监视器（管程）](https://zh.wikipedia.org/zh-cn/%E7%9B%A3%E8%A6%96%E5%99%A8_(%E7%A8%8B%E5%BA%8F%E5%90%8C%E6%AD%A5%E5%8C%96))  

*<end>*

