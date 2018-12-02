---
layout: post
title: JSR 133 FAQ的翻译和扩展（4）—— 关键定义（二）
key: 20160528
tags: 架构 并发与并行 开发语言
---  

## 前言  

本篇，着重结合 JSR 133 FAQ 中的同步相关 FAQ ，以及JSR 133 中的其余的关键定义，围绕同步 sychronization ，happen before 等进行展开。   
<!--more-->   

## ▸ 不正确的同步意味着什么？【译】    
  
[原文][What do you mean by “incorrectly synchronized”? ](http://www.cs.umd.edu/~pugh/java/memoryModel/jsr-133-faq.html#incorrectlySync )       
  
不正确的同步处理代码（ incorrectly synchronized code ），对不同的人有着不同含义。 当我们在 java 内存模型语义下讨论不正确的同步代码时，是指任何代码中出现了以下现象：     

  1. 有某个线程对某个变量有写操作，   
  2. 另一个线程对同一个变量有读操作，   
  3. 且，读和写没有通过同步来实现有序。 
 
当有这些规则被违背的时候，我们就说存在对于这个变量的数据竞争（ data race ）。一个有数据竞争的程序，就是不正确地同步的程序。  

## ▸ 同步会做些什么？【译】        
[原文][What does synchronization do? ](http://www.cs.umd.edu/~pugh/java/memoryModel/jsr-133-faq.html#synchronization )   

同步（ Synchronization ）包括若干个方面。最广为人知的是互斥（ mutual exclusion ） —— 即，只有一个线程在某个时刻拥有监视器（ monitor，或译作管程  ）[^Monitor]，所以，基于监视器的同步行为意味着一旦一个线程进入了被监视器保护的同步块（ synchronized block ），在这个线程退出前，没有任何其它线程再可以进入被这个监视器保护的块。       
   
但是，同步还有 “ 互斥 " 之外的更多含义。在基于同一个的同步监视器的前提下，同步确保了一块内存被一个线程写入前或写入中，该同步块对于其它线程的可见过程是处于可预测的状态下。当我们 **退出**（ exit ）同步块，我们通过 **释放**（ release ）监视器来达到 **刷新**（ flushing ）缓存数据到主存的效果，从而使本线程完成的写入能够对其它线程可见。在我们能 **进入**（ enter ）同步块之前，我们通过 **获取**（ acquire ）监视器来达到使本地处理器缓存失效，让变量重新从主存中 **载入**（ reloaded ）的效果。这样，我们就可以看到由之前的（其他线程的） **释放**（ release) 涉及到的所有写入。     
   
从缓存的角度讨论这个话题，似乎听起来这些问题只影响到多处理器机器（ multiprocessor machines ）。然而，重排序效果其实也可以很容易在单处理器（ single processor ）上见到。例如，让编译器在 **获取** 前，或者在 **释放**后去移动你的代码（ code ）是不可能的。 当我们谈到缓存的 **获取**和 **释放**动作时，我们简略掉了其中大量的可能影响。   
 
新内存模型创建了对于内存操作行为（读字段 read field ，写字段 write field ，锁 lock ，解锁 unlock ）和其它线程操作行为（开始 start ，阻塞等待 join 【翻译注：参见 官方文档中线程行为的表述[^ConJoin] 】）等实现部分有序的语义：即，要求某些操作，比其他一些操作要 **“先于发生”（ happen before ）**。如果一个行为是发生在另一个之前（ happens before ），那么意味着第一个是确保在前序发生（be  ordered before）的且对第二个可见。具体的有序规则如下：  

  ∙ 在一个线程中的每个动作（ action ），都要比该线程中，按代码顺序（ program order ）的每一个后续动作先发生 **“先于发生”（ happen before ）**。   
  ∙ 对一个监视器的解锁（ unlock ）动作 ，要比**该监视器**的每一个后续锁（ lock ）动作 ** “先于发生”（ happen before ）**。   
  ∙ 对于一个可变( volatile )字段的 **写**操作 ，都要比**该变量**的每一个后续**读**操作  ** “先于发生”（ happen before ）**。   
  ∙ 对于某个线程的 start() 方法调用，都要比**该线程**启动后的任何动作先发生。     
  ∙ 对于某个线程中的所有动作，都要比其它任何线程调用 join() 方法阻塞等待从这个线程返回** “先于发生”（ happen before ）**。     
   
这就意味着，在同一个监视器保护的同步块，对于某个线程在退出同步块前可见的任何内存操作，对于后续进入同步块的其它任何线程也是可见的。因为，所有的内存操作比释放（ release ）行为 **“先于发生”（ happen before ）**，所有释放（ release ）行为比获取（ acquire ）行为 **“先于发生”（ happen before ）**     
   
  >【翻译注：为更简洁和符合中文习惯，happen - before ，在后文部分中文句式中，我简称为 “前序” 关系】 。  
  
另有一层含义是，像下面这样，有些人通常用来进行强制内存隔离的模式，实际是无效的：  

  ```    
  synchronized (new Object()) {} 
  ```  

这实际上是一个空操作，编译器可以把它完全移除，因为它知道没有其它线程会基于这个监视器同步。你必须为线程建立好**“ 前序 （happen - before）”** 关系，才能使一个线程（正确地）看到其它线程的操作结果。     
   
**重要提示：** 为了建立恰当的 **“ 前序 ”** 关系，很重要的一点，是要让线程同步在 **同一个监视器**上。 A 线程同步在对象 X 时的所有可见，不会变成线程 B 之后对对象 Y 的可见。释放（ release ）和获取（ acquire ）必须相 匹配（例如，基于同一个监视器），来确保正确的语义。否则，代码就会有数据竞争（ data race ） 。     

  
## JSR 133中的关键定义（二）

>说明：JSR 133 的第 5 部分 Definations 中的第一部分概念，已在系列（3）中涉及。       
  
在继续往下前聊JSR133 （JSR133中英文原文链接 [^JSR133] [^JSR133cn] ） 相关内容前，有必要再来聊一下顺序一致模型。理论上来说，只要有一个全局优化器，就可以实现顺序一致执行了。同时也可以很大程度简化代码编写的复杂度。但，实际上，无论是对硬件还是运行时（runtimes），都很难有这样的全局优化器。原因显而易见。一个较之弱化的模型，不是对全局，而是只对部分特殊的行为来有序。 

### 1. po、so、sw和hb     
从开发层面去了解Java 内存模型的动作行为，实际是理解以下四个行为的逻辑：  

  - **程序顺序** (Program Order ）—— “关键定义（一）“中已经说明， $$\xrightarrow{\text{ po }}$$     
    这里需要重点强调是：从线程间动作的视角看，程序顺序不能也不会提供任何重排序的保证，它只是代表了最终执行和原始代码之间的关联。      
     

  - **同步顺序**（Sychronization Order），$$\xrightarrow{\text{ so }}$$    
    在某一次执行中，所有同步动作的完整执行顺序 。 从单线程的角度看，同步顺序 （so) 和 程序顺序 （ po ）是一致的。在同步顺序中的所有的“读” 都能看到最新的 “写” 。     


  - **Sychronizes-With 边界**（Sychronization-With edges），$$\xrightarrow{\text{ sw }}$$   
     每个同步动作的起始点或者结束点，就是动作的“Sychronizes-With “ edges （同步动作间的边界）。  
     如公式示意：  
     $$\ release(m) \xrightarrow{\text{ sw }}  acquire(m) $$   ，“Sychronizes-With “ 边界的起点是同步动作的结束（ release ） ，终点是同步动作的开始 （ acquire ）。   
       **【注：是同步动作引出了 “ Sychronizes-With “ 关系，从而形成了同步顺序。而不是相反  】**    
      
  - **“前序”**关系 （Happen - before ），$$\xrightarrow{\text{ hb }}$$    
    同步顺序中的前序关系的发生是由 程序顺序与“ Sychronizes-With “ 关系共同作用的结果。  
    **但是有前序关系，不等于执行就会按照这个关系进行重排序 ** 。  
    ∙ 如果 $$\ x \xrightarrow{\text{ sw }}  y $$ （即，动作 x 和后续的动作 y 有“ Sychronizes-With “ 关系） ,那么，也意味着 $$\ x \xrightarrow{\text{ hb }}  y $$ 关系的存在 。  

    ∙ $$\xrightarrow{\text{ hb }}$$  属于偏序关系[^PartOrd]，具有自反性，传递性和非对称性的特征。  

接下来，我们就从这几个定义出发来进一步理解内存模型。  
  

### 2.动作和执行( Actions and Executions )  

内存模型的一个作用，就是为了构建良好（ well- formed  ）的执行。所以，这里先要定义一下执行。一个执行E可以用元组< P,A,po,so,W,V,sw,hb> 来描述[^ExeTurp] 。出了前文提到的 po,so,sw,hb 四种行为逻辑之外，
还包括：  
    ∙ P - 程序 。  
    ∙ A - 一组动作 。 可以用 <t,k,v,u> 元组，即<执行线程，动作类型（前面已经表述），变量／监视器，唯一标识符>来定义。       
    ∙ W - “写-可见”（ write-seen ）函数 。对于 A 中的每个读操作 r，W(r) 代表在执行 E 中对 r 可见的写。    
    ∙ V - “值写入”（ value-written ）函数。对于 A 中的每个读操作 w，V(w)代表在执行 E 中 w写入的值 。


本文并没有计划重复和深入相关的数学推导，这里强调元组定义的目的，只是为了帮助后续讨论建立完整概貌。  

### 3.构建良好的执行( Well-Formed Executions )  

终于离问题的目标越来越接近了。前面的所有的铺垫和解释，现在到了一个思考的聚焦点：   
E=< P,A,po,so,W,V,sw,hb>。  现在的问题是，如果有一段程序摆在我们面前，我们如何合理的推断出一个正确、无二义的执行计划 E 。
不太确切的说，最后的 E 是元组中不同元素的相互约束，制约的结果，从多组可能的候选执行计划中，选出满足各个约束的最终交集。   

我们还是先直接跳到结论吧。  

  [![执行的生成](https://i.postimg.cc/rmR4YMrN/20160528eplan.png)](https://postimg.cc/tZy7s0Ds)     

除了上图已经明确表达的约束。构建良好的执行【JSR133 §7.3 以及 JLS § 17.4.7 】也包括对 W ，V 的约束。即，对与volatile的变量的写都要确保每个读取的可见。但这并不以意味着对 $$\xrightarrow{\text{ so }}$$ 约束要求的放松。因为，这里的每个读取可见的实质含义是每次的写都被刷新到了主存中，以便读取的不是过期的值。但是，如果有没有同步序 so 的约束，不能互斥。  

举个直观的例子来说明下：  

 {% highlight java %}
  public class VSC<T> {
    volatile T val;
    public synchronized void set(T v) {
     if(val==null) { val=v; }
     ...
    } 
   
    public T get() {
      return val; 
    }
  }
 {% endhighlight %}

 如上例： 
    ∙  set（） 方法前的，synchronized 修饰仍是需要的，否则不能实现互斥，保证 val 只被设置一次。
    ∙  另一方面，如果 val 前没有 volatile 修饰，在保持代码其他部分不变的情况下，在几个线程的 同步边界间（ sw ）发生的 val 的写入的值 ，未必一定对另一个线程可见。   
    ∙  get（）方法前， 加上 synchronized 的修饰 在这个场景下 **不是必须** 的。除了增加系统瓶颈外（因为读的线程数可能远远超过写的），没有额外作用。 


关于基于因果关系的可提交【 JSR133 §7.3 】的9条原则，以及 其中的一些测试例子，这里不打算再展开。因为，从开发角度来说，我们还是更关心，按照怎么样的语法和语义，可以让程序按照我们期望的方式运行。而不是相反的：如果不加以约束，JIT 编译器和运行时会有多少种可能的运行。这个问题，显然复杂的多，而且也更适合像 JVM 虚拟机编写等岗位去研究。  

所以，抛开可提交这个约束 ，构建良好的执行需要满足如下条件【JLS §17.4.7】[^ExeWell]：  

   1. **对于 volatile 变量，每个读都可以看到对这个变量的写的值 v。**  即，W(r).v =r.v 。  
   2. **hb （通过 po 和 sw 传递构成）是偏序关系 。** 即，符合偏序的自反、传递和反对称三个特性。  
   3. **遵循线程内一致性（Intra-Thread Consistency）。** 即 P 的执行遵循线程内语义。  
   4. **执行满足 hb 一致性 （ happens-before consistent ）。**  即，对于任何变量的读操作，不存在 hb(r, W(r))  ；也不存在满足 w.v = r.v ， hb(W(r), w) 和 hb(w, r) 的 w 。
   5. **执行满足 so 一致性 。** 即对于所有volatile变量的读操作，不存在 so(r, W(r))  ；也不存在满足 w.v = r.v ， so(W(r), w) 和 so(w, r) 的 w 。  


### 4.理解代码的执行结构。

以及涉及到了相对抽象的概念和定义（当然 JSR133 原文中还要更多）。接下来，我要说一下从我的理解角度，怎么比较快速的理解代码的在多线程状态下的执行结构（包括执行结果）————暂时仅围绕本章已经涉及的概念。

  ∙ 如果没有 volatile 修饰的 变量。我们先可以假设在不违背单线程语义的情况下，一切的重排序都是有可能的。   

  ∙ 如果有 volatile 修饰的变量，我们要先找到这些变量在多线程间的 release - acquire 关系，找到关键的 sw 关系，并结合 sw 前后代码序的，分析可见的影响范围。 

  ∙ 对第2点，关于写入的可见问题。我们可以认为，如果 volatile 变量可见了，按照代码序po “前序”的变量也对其他线程可见了。 在该代码序之后的写入，可能可见（不排除重排序）。 

  ∙ Synchronized 修饰的存在，意味着互斥（锁）的存在。 按照互斥隔离的思路考虑执行结果。  

举个例子来说明上述的的理解：   

  [![happen - before相关 ](https://i.postimg.cc/g2LXbsnm/20160528vsc.png)](https://postimg.cc/hXgP99L5)  

  如上例：  

  1. 在 ready 变量有 volatile 修饰的情况下，对 ``` ready = true ```的写，和``` ready == true ``` 之间形成了 sw （同步边界）关系。

  2. 同时 ，由于 po关系 ，```a=12``` 和 ``` ready = true ```  形成了 hb （先序）关系。  

  3. 又，a 变量，由于没有其他特定修饰，很有概率```a=13 ``` 在执行时被重排序。 
    
 **综上，Thread - 2 和 Thread - 3 会打印 12 或者13 。这两个线程输出的结果可能相同也可能不同。**  
    
  4.假设：ready 变量去掉 volatile 修饰。那么， 线程间实际上失去了明确的同步边界。也就意味着在不做执行顺序的假设的情况，任何执行结果都有可能，即，打印出0、11、12、13的可能。  

  5.在实际代码中，这样的线程间关系，也可能很容易制造死锁。需要根据情况加上必要的线程暂停保护,如 sleep(）. 
   

## 本篇小结  
   至此，除了更偏底层的理论，已经把内存模型的来龙去脉、主要概念和总体结构做了相对完整的全貌的整理复述。  
   而在JSR 133 FAQ中的问题中，主要就剩下 volatile 和 final 字段相关的了。由于 final 语义的问题，更多是出在内存模型本身的问题——一般开发者对其运用并无太多疑问。而 volatile 在实际开发运用中，经常有些似是而非的理解，所以，下一篇，将重点关注这部分内容。 


---   
**参考索引** 

[^JSR133]:[JSR-133: JavaTM Memory Model and Thread Specification ](http://www.cs.umd.edu/~pugh/java/memoryModel/jsr133.pdf )    
[^JSR133cn]:[JSR-133 非官方中文翻译 ](http://ifeve.com/wp-content/uploads/2014/03/JSR133%E4%B8%AD%E6%96%87%E7%89%881.pdf)     
[^ConJoin]:[Java官方文档：join操作等 ](https://docs.oracle.com/javase/tutorial/essential/concurrency/join.html)    
[^PartOrd]:[wiki 偏序关系 ](https://zh.wikipedia.org/wiki/%E5%81%8F%E5%BA%8F%E5%85%B3%E7%B3%BB)   
[^ExeTurp]:[执行（ Executions ）的定义 ](https://docs.oracle.com/javase/specs/jls/se8/html/jls-17.html#jls-17.4.6)     
[^ExeWell]:[well-formed Executions  ](https://docs.oracle.com/javase/specs/jls/se8/html/jls-17.html#jls-17.4.7)         
[^Monitor]:[wiki 监视器（管程）](https://zh.wikipedia.org/zh-cn/監視器_(程序同步化))   


*<end>*

