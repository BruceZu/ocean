# JVM结构、GC工作机制详解
#### JVM结构
![JVM结构](https://github.com/yr0918/ocean/raw/master/doc/img/java.gc.jvm.jpg)
![类加载器](https://github.com/yr0918/ocean/raw/master/doc/img/java.gc.jvm.classloader.jpg)

![JVM结构](https://github.com/yr0918/ocean/raw/master/doc/img/java.gc.jvm.2.jpg)

内存区

![内存区](https://github.com/yr0918/ocean/raw/master/doc/img/java.gc.jvm.memory.jpg)

1. 类加载器（ClassLoader）:在JVM启动时或者在类运行时将需要的class加载到JVM中。
  （右图表示了从java源文件到JVM的整个过程，可配合理解）
2. 执行引擎：负责执行class文件中包含的字节码指令（执行引擎的工作机制，这里也不细说了，这里主要介绍JVM结构）
3. 内存区（也叫运行时数据区）：是在JVM运行的时候操作所分配的内存区。运行时内存区主要可以划分为5个区域
 - 方法区(Method Area)：用于存储类结构信息的地方，包括常量池、静态变量、构造函数等。虽然JVM规范把方法区描述
   为堆的一个逻辑部分， 但它却有个别名non-heap（非堆），所以大家不要搞混淆了。方法区还包含一个运行时常量池。
 - java堆(Heap)：存储java实例或者对象的地方。这块是GC的主要区域（后面解释）。从存储的内容我们可以很容易知道，
   方法区和堆是被所有java线程共享的。
 - java栈(Stack)：java栈总是和线程关联在一起，每当创建一个线程时，JVM就会为这个线程创建一个对应的java栈。在
   这个java栈中又会包含多个栈帧，每运行一个方法就创建一个栈帧，用于存储局部变量表、操作栈、方法返回值等。
   每一个方法从调用直至执行完成的过程，就对应一个栈帧在java栈中入栈到出栈的过程。所以java栈是现成私有的。
 - 程序计数器(PC Register)：用于保存当前线程执行的内存地址。由于JVM程序是多线程执行的（线程轮流切换），
    所以为了保证线程切换回来后，还能恢复到原先状态，就需要一个独立的计数器，记录之前中断的地方，可见程序计
    数器也是线程私有的。
 - 本地方法栈(Native Method Stack)：和java栈的作用差不多，只不过是为JVM使用到的native方法服务的
4. 本地方法接口：主要是调用C或C++实现的本地方法及返回结果

#### 内存分配
Java虚拟机是先一次性分配一块较大的空间，然后每次new时都在该空间上进行分配和释放，减少了系统调用的次数，节省了一定
的开销，这有点类似于内存池的概念；二是有了这块空间过后，如何进行分配和回收就跟GC机制有关了

![内存分配](https://github.com/yr0918/ocean/raw/master/doc/img/java.gc.allocation.jpg)

JVM内存申请过程如下：

1. JVM 会试图为相关Java对象在Eden中初始化一块内存区域
2. 当Eden空间足够时，内存申请结束；否则到下一步
3. JVM 试图释放在Eden中所有不活跃的对象（这属于1或更高级的垃圾回收）,释放后若Eden空间仍然不足以放入新对象，则试图将部分Eden中活跃对象放入Survivor区
4. Survivor区被用来作为Eden及OLD的中间交换区域，当OLD区空间足够时，Survivor区的对象会被移到Old区，否则会被保留在Survivor区
5. 当OLD区空间不够时，JVM 会在OLD区进行完全的垃圾收集（0级）
6. 完全垃圾收集后，若Survivor及OLD区仍然无法存放从Eden复制过来的部分对象，导致JVM无法在Eden区为新对象创建内存区域，则出现”out of memory”错误

#### 内存回收（GC）【理论】

###### What? -- 哪些内存需要回收？
程序计数器、虚拟机栈、本地方法栈是每个线程私有的内存空间，这3个区域的内存分配和回收都是确定的，无需考虑内存回收的问题

GC主要进行回收的内存是JVM中的方法区和堆；
涉及到多线程(指堆)、多个对该对象不同类型的引用(指方法区)，才会涉及GC的回收

###### When? -- 什么时候回收？
**1.堆回收时间**

`JVM在做垃圾回收的时候，会检查堆中的所有对象是否会被这些根集对象引用，不能够被引用的对象就会被垃圾收集器回收`

引用计数法：给一个对象添加引用计数器，每当有个地方引用它，计数器就加1；引用失效就减1。

可达性分析算法(补充互相引用)：以根集对象为起始点进行搜索，如果有对象不可达的话，即是垃圾对象。
这里的根集一般包括java栈中引用的对象、方法区常良池中引用的对象本地方法中引用的对象等。

>无论是引用计数器还是可达性分析，判定对象是否存活都与引用有关！那么，如何定义对象的引用呢？

我们希望给出这样一类描述：当内存空间还够时，能够保存在内存中；如果进行了垃圾回收之后内存空间仍旧非常紧张，则可以抛弃这些对象
- 强引用(Strong Reference):Object obj = new Object();只要强引用还存在，GC永远不会回收掉被引用的对象。
- 软引用(Soft Reference)：描述一些还有用但非必需的对象。在系统将会发生内存溢出之前，会把这些对象列入回收范围进行二次回收（即系统将会发生内存溢出了，才会对他们进行回收。）
- 弱引用(Weak Reference):程度比软引用还要弱一些。这些对象只能生存到下次GC之前。当GC工作时，无论内存是否足够都会将其回收（即只要进行GC，就会对他们进行回收。）
- 虚引用(Phantom Reference):一个对象是否存在虚引用，完全不会对其生存时间构成影响

**2.方法区回收时间**

`总而言之，对于堆中的对象，主要用可达性分析判断一个对象是否还存在引用，如果该对象没有任何引用就应该被回收。而根据我们实际对引用的不同需求，又分成了4中引用，每种引用的回收机制也是不同的。
 对于方法区中的常量和类，当一个常量没有任何对象引用它，它就可以被回收了。而对于类，如果可以判定它为无用类，就可以被回收了`

###### How? -- 如何回收？
**1.回收算法**

a.标记-清除（Mark-sweep）：标记所有需要回收的对象，然后统一回收。这是最基础的算法，后续的收集算法都是基于这个算法扩展的。

不足：效率低；标记清除之后会产生大量碎片

![](https://github.com/yr0918/ocean/raw/master/doc/img/java.gc.algorithm.mark_sweep.png)

b.复制（Copying）：此算法把内存空间划为两个相等的区域，每次只使用其中一个区域。垃圾回收时，遍历当前使用区域，把正在使用中的对象复制到另外一个区域中。
  此算法每次只处理正在使用中的对象，因此复制成本比较小，同时复制过去以后还能进行相应的内存整理，不会出现“碎片”问题。当然，此算法的缺点也是很明显的，
  就是需要两倍内存空间

![](https://github.com/yr0918/ocean/raw/master/doc/img/java.gc.algorithm.copying.png)

c.标记-整理（Mark-Compact）：第一阶段从根节点开始标记所有被引用对象，第二阶段遍历整个堆，把清除未标记对象并且把存活对象“压缩”到堆的其中一块，
  按顺序排放。此算法避免了“标记-清除”的碎片问题，同时也避免了“复制”算法的空间问题

![](https://github.com/yr0918/ocean/raw/master/doc/img/java.gc.algorithm.mark_compact.png)

`d.分代收集算法`(当前商业虚拟机常用：居于不同的对象的生命周期是不一样的)

![](https://github.com/yr0918/ocean/raw/master/doc/img/java.gc.algorithm.generation.jpg)

1.  在Young Generation中，有一个叫Eden Space的空间，主要是用来存放新生的对象，还有两个Survivor Spaces（from、to），它们的大小总是一样，它们用来存放每次垃圾回收后存活下来的对象。
2.  在Old Generation中，主要存放应用程序中生命周期长的内存对象。
3.  在Young Generation块中，垃圾回收一般用Copying的算法，速度快。每次GC的时候，存活下来的对象首先由Eden拷贝到某个SurvivorSpace，当Survivor Space空间满了后，剩下的live对象就被直接拷贝到OldGeneration中去。因此，每次GC后，Eden内存块会被清空。
4.  在Old Generation块中，垃圾回收一般用mark-compact的算法，速度慢些，但减少内存要求。
5.  垃圾回收分多级，0级为全部(Full)的垃圾回收，会回收OLD段中的垃圾；1级或以上为部分垃圾回收，只会回收Young中的垃圾，内存溢出通常发生于OLD段或Perm段垃圾回收后，仍然无内存空间容纳新的Java对象的情况。


#### 内存回收器（GC）【具体实现】

![](img/java.gc.collector.png)

1. Serial收集器：单线程收集器，表示在它进行垃圾收集时，必须暂停其他所有的工作线程，直到它收集结束。"Stop The World".
2. ParNew收集器：实际就是Serial收集器的多线程版本。
 - 并发(Parallel):指多条垃圾收集线程并行工作，但此时用户线程仍然处于等待状态；
 - 并行(Concurrent):指用户线程与垃圾收集线程同时执行，用户程序在继续运行，而垃圾收集程序运行于另一个CPU上。
3. Parallel Scavenge收集器：该收集器比较关注吞吐量(Throughout)(CPU用于用户代码的时间与CPU总消耗时间的比值)，保证吞吐量在一个可控的范围内。

4. CMS(Concurrent Mark Sweep)收集器：CMS收集器是一种以获得最短停顿时间为目标的收集器,CMS收集器是基于“标记-清除”算法实现的
 - CMS对收集器对CPU资源非常敏感。
 - CMS无法处理浮动垃圾
 - CMS 是基于“标记-清除”算法的收集器，会产生大量的空间碎片
5. Serial Old收集器：Serial Old收集器是Serial收集器的老年代版本，同样是一个单线程收集器，使用“标记-整理”算法。这个收集器的主要意义也是在于Client模式下的虚拟机使用
 - 一种用途是在JDK1.5以及之前的版本与Parallel Scavenge收集器搭配使用。
 - 另一种用途就是作为CMS收集器的后备预案，在并发收集发生Concurrent Mode Failure时使用
6. Parallel Old收集器：Parallel old 是Parallel Scavenge收集器的老年代版本，使用多线程和“标记-整理”算法。这个收集器是JDK6才出现的，在之前Parallel Scavenge一直处于尴尬状态
7. G1(Garbage First)收集器：从JDK1.7 Update 14之后的HotSpot虚拟机正式提供了商用的G1收集器，与其他收集器相比，它具有如下优点：并行与并发；分代收集；空间整合；可预测的停顿等

#### 实践
gc日志分析工具：http://mp.weixin.qq.com/s/eztcZn1S8LkylgZoMO5pHw