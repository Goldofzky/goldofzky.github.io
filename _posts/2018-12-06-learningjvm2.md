---
layout: post
title:  "垃圾收集器和内存分配策略"
subtitle: "JVM笔记2"
header-img: "img/post-bg-universe.jpg"
tag: 
    - JVM笔记
---
**目录**
* content
{:toc}

程序计数器，虚拟机栈，本地方法栈这3个区域随线程而生，随线程而灭；栈中的栈帧随方法进入和退出有条不紊地执行出栈和入栈操作。每一栈帧中分配的内存基本上从类结构确定下来就已知。所以这几个区域就**不需要过多考虑回收**的问题，因为方法结束或者线程结束时，内存自然就跟随回收了。而**Java堆和方法区**则不一样，这部分的内存的分配与回收都是动态的,`垃圾回收器`重点关注的是这里的内存。

# 对象已死吗
垃圾回收器对堆进行回收前，第一时间就是要确定这些对象之中哪些还“存活”着。
## 引用计数算法
**方法**：给对象添加一个引用计数器，每当有一个地方引用它时，计数器值就加1；引用失效则减1；任何时刻计数器为0的对象就是不可能再被使用的。  
`缺点`：不能解决相互循环引用问题。
{% highlight ruby %}
public class Sample{
    public Object instance = null;
    private static final int _1MB = 1024*1024;
}
/**
 *这个成员的作用就是占点内存
 */
private byte[] BigSize = new byte[2 * _1MB];

public static void testGC(){
    Sample objA = new Sample();
    Sample objB = new Sample();

    objA.instance = objB;
    objB.instance = objA;

    objA = null;
    objB = null;
    
    System.gc();
    /**
    *事实上是，在这之后，objA和objB被回收了，
    *因此我们可以确定Java并不是用的引用计数算法
    */
}
{% endhighlight %}

## 可达性分析算法
现在的主流应用程序语言（包括Java）都是通过可达性分析（Reachability Analysis）来判定对象是否存活的。  
**方法**：通过一系列称为“GC Roots”的对象作为起始点，从这些节点开始向下搜索，搜索所走过的路径被称为`引用链`,当一个对象到GC Roots没有任何引用链相连时，证明此对象**不可用**。
![](\img\in-post\L-JVM\JVM2-1.PNG)
以下几种对象可以作为GC Roots:  
1. 虚拟机栈(栈帧中的本地变量表)中引用的对象。
2. 方法区中类静态属性引用的对象。
3. 方法区中常量引用的对象。
4. 本地方法栈中JNI（Native方法）引用的对象。

## 再谈引用
Java对引用的概念进行了扩充，将引用分为强引用（Strong Reference），软引用（Soft Reference），弱引用（Weak Reference），虚引用（Phantom Reference）4种，引用强度依次递减。  
* 强引用指程序代码中普遍存在的，类似“Object obj = new Object()”这类的引用，只要强引用还存在，垃圾回收器永远不会回收掉被引用的对象。
* 软引用用来描述一些还有用但非必需的对象。在系统快要发生`内存溢出`异常之前，将会将这些对象进行第二次回收，如果还没有足够内存，会抛出内存溢出异常。
* 弱引用关联对象只能生存到`下一次垃圾收集`发生之前。
* 虚引用是最弱的引用关系，一个对象是否有虚引用对自身完全不构成影响，也无法通过虚引用获得实例。仅用来能在引用虚引用的对象被收集器回收时受到一个系统通知。

## 生存还是死亡
真正宣告一个对象的死亡要经过`两次`标记过程：  
1. 如果对象在可达性分析后发现没有与GC Roots相连的引用链，那么它将会被第一次标记并且进行一次筛选。  
2. 筛选的条件是此对象是否有必要执行`finalize（）`方法。当对象没有覆盖finalize方法，或者finalize方法已经被虚拟机调用过，虚拟机将这两种情况都视为“没有必要执行”。
3. 如果一个对象被视为“有必要执行”finalize方法，那么对象将会被放置到一个叫做F-Queue的队列中。并在稍后由一个虚拟机自动建立、低优先级的Finalizer线程去执行它。这里的执行不代表会等待运行结束，如果一个对象执行缓慢或者发生死循环就是另外一种情况了。
4. 在执行finalize方法的过程中，是对象`最后一次`拯救自己的机会，对象如果在finalize方法中重新与引用链上的一个对象建立关联，则被移除出**即将回收**的集合。如果这个时候还没有建立关联，那么它就真的被回收了。需要注意的是，finalize方法对于任何对象来说都只会被系统自动调用`仅一次`。并且不建议在代码的编写中，用这个方法。

## 回收方法区
永久代主要回收两部分内容：**废弃常量**和**无用的类**。  
* 回收废弃常量的方法与Java堆中的对象类似。比如一个字符串“abc”已经进入常量池，而在当前系统没有任何一个String对象叫做“abc”，换句话说，没有任何一个String对象引用常量池中的“abc”常量，也没有其他地方引用这个字面量，这个时候进行内存回收，如果有必要，它将会被清理出常量池。
* 判定一个类是否为**无用的类**，条件就要苛刻了许多。要`同时满足`下面三个条件。
1. 该类所有的实例都已经被回收，也就是Java堆中不存在该类的任何实例。
2. 加载该类的ClassLoader已经被回收。
3. 该类对应的java.lang.Class对象没有任何地方被引用，无法在任何地方通过反射访问该类的方法。

# 垃圾回收算法
## 标记-清除算法
![](\img\in-post\L-JVM\JVM2-2.PNG)
**思路**：首先标记所有需要回收的方法，然后在标记完成后统一回收所有被标记的对象。  
`缺点`:标记和清除两个过程的效率都不高；标记清除后，会产生大量不连续的内存碎片，空间碎片太多会导致以后在程序运行过程中需要分配大的对象时，无法找到匹配大小的连续内存，从而不得不提前触发另一次垃圾回收操作。
## 复制算法
![](\img\in-post\L-JVM\JVM2-3.PNG) 
**思路**：将可用内存分为大小相等的两块，每次只使用其中的一块。当一块的内存用完了，就将还活着的对象复制到另一块上面，然后把已使用的内存空间清理掉。  
`注意`:  
* 当前的商业虚拟机都采用这种收集方法收集新生代。
* 实际运用中，并不是按照1：1的方法，而是将内存分为一块较大的Eden空间和两块较小的Survivor空间，每次使用Eden和其中一块Survivor。默认比例为8：1：1.
## 标记-整理算法
![](\img\in-post\L-JVM\JVM2-4.PNG) 
**思路**：标记清除的过程仍然与标记-清除算法一样，但后续是让所有存活的对象都向一端移动，然后直接清理掉边界以外的内存。  
`注意`：
* 在老年代一般选用这种算法
## 分代收集算法
当前商业虚拟机都采用**分代收集算法**（Generational Collection）算法，这种算法根据对象的存活周期将内存划分为几块。一般是将对象分为新生代和老年代。  
**新生代**：每次收集都发现有大批对象死去，只有少量存活，那就选用复制算法。  
**老年代**：存活率高，没有额外空间对它进行担保，就必须选择标记-整理或者标记-清理算法来进行回收。

# HotSpot的算法实现
* 可作为GC Roots的节点主要在`全局性的引用`(例如常量或类静态属性)与`执行上下文`(例如栈帧中的本地变量表中)。
* GC进行时必须`停顿`所有Java执行线程。
* 在HotSpot的是实现中，使用一组称为OopMap的数据结构来得知哪些地方存放着对象引用。
* 程序只在特定的地方停顿下来开始GC,只有在到达`安全点`时才能暂停.
* 安全点以程序**是否具有让程序长时间执行的特征**为标准进行选定的。例如方法调用、循环跳转、异常跳转等。
* 让线程都跑到安全点上停顿下来，有`抢先式中断`和`主动式中断`两种方法。
* 假如线程处于Sleep或者Blocked状态，就用`安全区域`来解决。

# 垃圾收集器
HotSpot虚拟机的所有收集器如图所示。如果两个收集器之间存在连线，就说明它们可以搭配使用。所处区域代表它是属于老年代收集器还是新生代收集器。
![](\img\in-post\L-JVM\JVM2-5.PNG)  
## Serial收集器
![](\img\in-post\L-JVM\JVM2-6.PNG)  
* 是一个`单线程`的收集器。
* 进行垃圾收集时，必须暂停其他所有的工作进程，直到它收集结束。
* 对于运行在Client模式下的虚拟机来说时一个很好的选择。

## ParNew收集器
![](\img\in-post\L-JVM\JVM2-7.PNG)  
* 其实就是Serial收集器的`多线程`版本
* 随着可用CPU的数量的增加，它对于GC时系统资源的有效利用很有好处。
* 是许多运行在Server模式下的虚拟机首选的新生代收集器。

## Parallel Scavenge 收集器
* 是一个新生代收集器，也是使用复制算法的收集，又是并行的多线程收集器。
* 设计的目标是为了达到一个可控制的`吞吐量`（CPU用于运行用户代码的时间与CPU总消耗时间的比值）。
* Parallel Scavenge收集器提供两个参数用于精确控制吞吐量。分别是控制最大垃圾收集停顿时间的`-XX：MaxGCPauseMillis`参数以及直接设置吞吐量大小的`-XX：GCTimeRatio`参数。
* `-XX:+UseAdaptiveSizePolicy`参数用于根据当前系统运行情况收集性能监控信息，动态调整新生代大小、Eden与Survivor区比例、晋升老年代对象大小等细节参数。这种调节方式被称为GC`自适应`的调节策略（GC Ergonomics）。

## Serial Old收集器
![](\img\in-post\L-JVM\JVM2-6.PNG)  
* 是Serial收集器的老年代版本，同样是一个单线程收集器，使用标记-整理算法。
* 主要意义在于给Client模式下的虚拟机使用。
* 如果在Server模式下主要有两大用途：  
1. 在JDK1.5以及之前的版本中与Parallel Scavenge收集器搭配使用。
2. 作为CMS收集器的后备预案，在并发收集发生Concurrent Mode Failure时使用。

## Parallel Old收集器
![](\img\in-post\L-JVM\JVM2-8.PNG)  
* 是Parallel Scavenge收集器的老年版本，使用多线程和标记-整理算法。
* 在JDK 1.6中才开始提供。
* 直到Parallel Old收集器出现后，“吞吐量优先”收集器终于有了比较名副其实的应用组合，在注重吞吐量以及CPU资源敏感的场合，都可以考虑Parallel Scavenge收集器加Parallel Old收集器。

## CMS收集器
![](\img\in-post\L-JVM\JVM2-9.PNG)  
* CMS（Concurrent Mark Sweep）收集器是一个以获取最短回收停顿时间为目标的收集器。
* CMS是基于“标记-清除”算法实现的，它的运作过程分为4个步骤：  
1. 初始标记。标记GC Roots能够直接关联到的对象，速度很快。
2. 并发标记。进行GC Roots Tracing的过程。
3. 重新标记。为了修正并发标记因用户程序继续运作而导致标记产生变动的那一部分对象的记录。
4. 并发清除。
* 其中，初始标记、重新标记这两个步骤，仍然需要暂停其他线程。由于整个过程中耗时最长的并发标记和并发清除过程收集器线程都可以与用户线程一起工作，因此从总体上来说，CMS收集器的内存回收过程是与用户线程一起`并发执行`的。
* 有**并发收集、低停顿**的优点。但是却有三个明显的缺点：  
1. 对CPU资源非常敏感。
2. 无法处理浮动垃圾，可能出现“Concurrent Mode Failure”失败而导致另一次Full GC的产生。不能像其他老年代在等到快要被填满了再收集。
3. 会有大量空间碎片产生，给大对象的分配带来很大麻烦。因此CMS收集提供了开关参数`-XX:+UseCMSCompactAtFullCollection`（默认开启），用于在CMS顶不住进行FullGC时开启内存碎片的合并整理过程，但停顿时间不得不变长。还有参数`-XX：CMSFullGCsBeforeCompaction`，这个参数用于设置执行多少次不压缩的Full GC后，跟着来一次带压缩的（默认为0，表示每次进入都进行碎片整理）。

## G1收集器
* 是当今收集器技术发展的最前沿成果之一，早在JDK1.7确立项目目标。
* 是一款面向服务端应用的垃圾收集器。特点如下：  
1. 并行与并发：G1能充分利用多CPU、多核环境下的硬件优势，使用多个CPU来缩短Stop-The-World的时间。
2. 分代收集：G1可以不需要其他收集器的配合就独立管理整个GC堆，但它能采用不同的方式区处理不同对象。
3. 空间整合：G1从整体上来看是基于“标记-整理”算法实现的收集器，从局部（两个Region之间）上来看是基于“复制”算法实现的。不会产生内存空间碎片。
4. 可预测的停顿：除了追求低停顿外，还能建立可预测的停顿时间模型，让使用者明确指定一个长度为M毫秒的时间片段内，消耗在垃圾收集上的时间不得超过N毫秒。
* 它将整个Java堆划分为多个大小相等的独立区域（Region），虽然还保留新生代和老年代的概念，但新生代和老年代不再是物理隔离了，而是一部分Region（不需要连续）的集合。

# 内部分配与回收策略
* 大多数情况下，对象在新生代Eden区中分配。当Eden区没有足够空间时，虚拟机将发起一次Minor GC（新生代GC）。
* `大对象`直接进入老年代。大对象是指需要大量连续内存空间的Java对象，比如很长的字符串以及数组。虚拟机提供`-XX：PretenureSizeThreshold`参数，令大于这个设置值的对象直接在老年代分配。这样做是为了避免在Eden区及两个Survivor区之间大量的内存复制。
* `长期存活的对象`将进入老年代。虚拟机给每个对象定义了一个对象年龄（Age）计数器。如果对象在Eden出生并经过一次Minor GC后仍然存活，并且能被Survivor容纳的话，将被移动到Survivor空间中，并且对象年龄设为1.每熬过一次Minor GC，年龄就加1岁当他的年龄增加到一定长度（默认15），将会被晋升到老年代。虚拟机提供`-XX：MaxTenuringThreshold`设置年龄阈值。
* **动态对象年龄判定**：为了更好适应不同程序的内存情况，虚拟机并不是永远地要求年龄必须达到MaxTenuringThreshold才进入老年代。如果在Survivor空间中相同年龄所有对象大小总和大于Survior空间的一般，年龄大于或等于该年龄的对象都可以直接进入老年代。
* **空间分配担保**：当Survivor空间不够用时，需要依赖其他内存（这里指老年代）进行分配担保（Handle Promotion）。在发生Minor GC之前会先检查老年代最大可用的连续空间是否大于新生代所有对象的总空间，如果这个条件成立，那么Minor GC可以确保是安全的。如果不成立，则虚拟机会查看HandlePromotionFailure设置是否允许担保失败。如果允许，那么会继续检查老年代最大可用连续空间是否大于历次晋升到老年代对象的平均大小，如果大于，将尝试进行一次Minor GC，尽管是有风险的；如果小于，或者设置不允许冒险，那这时也要改为进行一次Full GC。