---
title: JVM垃圾回收
date: 2019-11-13 00:13:00
tags:
- java虚拟机
- 垃圾回收
categories:
- JVM
---

# 什么是垃圾回收

跟踪所有仍在使用的对象并将其余对象标记为垃圾的这一过程就叫做垃圾回收。

## 手动内存管理

在开始介绍现代Garbage Collection之前，快速回顾一下以前不得不手动和显式分配和释放数据存储空间的日子。如果你忘记释放它，则将无法重用它，但这块内存已经被声明了只是没有被使用，这种情况称为**内存泄漏**。

下面是一个用C语言编写的，使用手动内存管理的简单示例：

```c
int send_request() {
    size_t n = read_size();
    int *elements = malloc(n * sizeof(int));

    if(read_elements(n, elements) < n) {
        // elements未释放
        // 因为代码运行到此处就返回了，elements还来不及释放
        return -1;
    }

    // …
    // 如果运行到此处才会被释放
    free(elements)
    return 0;
}
```

如你所见，忘记释放内存是很容易的。内存泄漏曾经是比今天更常见的问题，你只能通过修复代码来真正打败“它们”，那有没有更优雅的解决方案呢？答案是肯定的，更好的方法是采取自动回收未使用的内存的策略，从而完全消除人为错误的可能性。这种自动化的过程称为垃圾收集（或简称GC）。

## 自动内存管理

在上面的C++代码中必须明确地知道何时需要进行内存管理，把这个工作给程序员，就意味着肯定有系统性风险，即人为忽略。如果把内存管理交给程序自己处理呢？这将非常方便，因为开发人员不再需要考虑自己清理。在程序运行时将自动了解不再使用某些内存并将其释放。换句话说，它会自动** 收集垃圾**。第一个垃圾收集器是在1959年为Lisp创建的，此后技术才有所发展。

### 引用计数（Reference Counting）

上面用C++的共享指针示例可以适用于所有对象，并且许多语言（例如Perl，Python或PHP）都采用这种方法。下面用图片说明：

![](/images/Java-GC-counting-references1.png)

绿云（GC ROOTS）表示程序员指向的对象仍在使用中。从技术上讲，这些可能是当前正在执行的方法中的局部变量或静态变量之类的东西。

蓝色圆圈是内存中的活动对象，其中的数字表示其引用计数。灰色圆圈表示没有被哪个仍在显式使用的对象中引用。因此灰色表示的是垃圾，可以由垃圾收集器清理。

这一切看起来真的很好，不是吗？但是这个策略有一个巨大的缺点。如果原先的引用不存在了，但可能它们之间仍然有相互引用，这种循环引用的问题，会导致它们的引用计数永远不为零。下面是一个例子：

![](/images/Java-GC-cyclical-dependencies.png)

红色圆圈表示的对象实际上是应用程序不使用的垃圾，可是由于引用计数的限制，仍然会存在内存泄漏问题。

有一些方法可以解决此问题，例如使用特殊的“弱”引用或应用单独的循环收集算法。上述语言（Perl、Python和PHP）都以某种方式处理循环，不过本方主要介绍JVM采取的方法。

#### 标记和扫描（Mark and Sweep）

在上面看到的模糊定义的绿色云被称为垃圾收集根（Garbage Collection Roots），下面有一组非常具体和明确的对象被当作GC根：

* 局部变量
* 活动线程
* 静态字段
* JNI引用

JVM用来跟踪所有可访问的（活动）对象，并确保不可访问的对象所占用的内存可以复用，这种方法称为标记和扫描算法。它包括两个步骤：

* 标记（Marking）是遍历所有可访问的对象，从GC根开始，并在本机内存中保留有关所有此类对象的分类记录。
* 扫描（Sweeping）确保了不可访问对象占用的内存地址可以在下一个分配中重用。

JVM中不同的GC算法，例如Parallel Scavenge，Parallel Mark + Copy或CMS，在实现这些阶段时略有不同，但是在概念上，该过程仍然类似于上述两个步骤。

关于此方法，至关重要的一点是不再发生内存泄漏问题：

![](/images/Java-GC-mark-and-sweep.png)

但不好的是必须**停止应用程序线程**才能进行GC，因为如果引用一直在变化，那么就无法真正计数引用。当应用程序暂时停止以便JVM可以沉迷于清理活动，这种情况称为**Stop The World**暂停。它们的发生可能有多种原因，但是GC是迄今为止最受欢迎的一种。

# Java中的垃圾回收

上面介绍的“标记和扫描”的垃圾收集策略主要是理论层面上的，在具体实现时需要进行大量调整以适应现实情况。

## 碎片和压缩

每当进行扫描时，JVM必须确保无法访问对象的区域可以被复用。这可能（并最终将）导致内存碎片，与磁盘碎片类似，会导致两个主要问题：

* 写操作变得更加耗时，因为找到下一个足够大的空闲块不再是微不足道的操作。
* 当创建新对象时，JVM在连续块中分配内存。因此，如果碎片升级到没有单个可用碎片足够大以容纳新创建的对象，就会发生分配错误。

为了避免此类问题，JVM确保碎片不会失控。因此，在垃圾回收过程中还会发生“内存碎片整理”过程，而不仅仅是标记和清除。此过程将所有可访问的对象彼此相邻放置，从而消除（或减少了）碎片。下图是一个说明让大家更方便理解：

![](/images/fragmented-vs-compacted-heap.png)

## 次要GC，主要GC与完整GC（Minor GC vs Major GC vs Full GC）

清除堆内存中不同部分的垃圾回收事件通常称为Minor，Major和Full GC事件。下面介绍这些事件之间的差异。

重要的是应用程序是否满足其[SLA](https://www.ibm.com/developerworks/cn/webservices/ws-sla/)，是否监控应用程序的延迟或吞吐量。这些事件的重要之处在于它们是否停止了应用程序以及花费了多长时间。

## 次要GC

从Young空间收集垃圾称为Minor GC。这个定义既清晰又统一。但是在处理次要垃圾回收事件时，仍然应该注意一些有趣的小知识：

1. 当JVM无法为新对象分配空间时（例如Eden变满），总是会触发次要GC。因此，分配率越高，次要GC发生的频率就越高。
1. 在次要GC事件中，有效地忽略了永久生成。从永生代到年轻代的引用被认为是GC的根源。在标记阶段，将忽略从年轻代到永生代的引用。
1. 与通常的看法相反，Minor GC确实触发了世界（stop-the-world）暂停，从而暂停了应用程序线程。对于大多数应用程序而言，如果可以将Eden中的大多数对象视为垃圾并且永远不会复制到Survivor / Old空间，则暂停的长度在延迟方面可以忽略不计。如果情况正好相反，并且大多数新生对象都不符合收集条件，那么次GC暂停将花费相当多的时间。

因此，定义次要GC很容易–  次要GC可以清理年轻代。

## 主要GC与全GC

应该注意的是，这些术语没有正式的定义，无论是在JVM规范中，还是在垃圾收集研究论文中。不过在我们所知道的关于次要GC清理年轻空间的事实基础上来构建这些定义应该很简单：

* **主要GC**（Major GC）正在清理旧空间。
* **Full GC**（Full GC）正在清理整个堆-无论是旧空间还是旧空间。

首先许多主要GC由次要GC触发，因此在很多情况下不可能将两者分开。另一方面，像G1这样的现代垃圾收集算法执行部分垃圾清理，因此，使用术语“清理”只是部分正确的。

这就引出了一个点，你不必担心GC是被称为Major GC还是Full GC，而应专注于确定当前的GC是停止了所有应用程序线程还是能够与应用程序线程同时进行。

JVM标准工具中甚至内置了这种混淆。我的意思最好通过一个例子来解释。让我们比较在运行并发标记和扫描收集器（-XX:+UseConcMarkSweepGC）的JVM上跟踪GC的两种不同工具的输出

首先尝试通过jstat输出：

```bash
my-precious: me$ jstat -gc -t 4235 1s
```

```bash
Time S0C    S1C    S0U    S1U      EC       EU        OC         OU       MC     MU    CCSC   CCSU   YGC     YGCT    FGC    FGCT     GCT   
 5.7 34048.0 34048.0  0.0   34048.0 272640.0 194699.7 1756416.0   181419.9  18304.0 17865.1 2688.0 2497.6      3    0.275   0      0.000    0.275
 6.7 34048.0 34048.0 34048.0  0.0   272640.0 247555.4 1756416.0   263447.9  18816.0 18123.3 2688.0 2523.1      4    0.359   0      0.000    0.359
 7.7 34048.0 34048.0  0.0   34048.0 272640.0 257729.3 1756416.0   345109.8  19072.0 18396.6 2688.0 2550.3      5    0.451   0      0.000    0.451
 8.7 34048.0 34048.0 34048.0 34048.0 272640.0 272640.0 1756416.0  444982.5  19456.0 18681.3 2816.0 2575.8      7    0.550   0      0.000    0.550
 9.7 34048.0 34048.0 34046.7  0.0   272640.0 16777.0  1756416.0   587906.3  20096.0 19235.1 2944.0 2631.8      8    0.720   0      0.000    0.720
10.7 34048.0 34048.0  0.0   34046.2 272640.0 80171.6  1756416.0   664913.4  20352.0 19495.9 2944.0 2657.4      9    0.810   0      0.000    0.810
11.7 34048.0 34048.0 34048.0  0.0   272640.0 129480.8 1756416.0   745100.2  20608.0 19704.5 2944.0 2678.4     10    0.896   0      0.000    0.896
12.7 34048.0 34048.0  0.0   34046.6 272640.0 164070.7 1756416.0   822073.7  20992.0 19937.1 3072.0 2702.8     11    0.978   0      0.000    0.978
13.7 34048.0 34048.0 34048.0  0.0   272640.0 211949.9 1756416.0   897364.4  21248.0 20179.6 3072.0 2728.1     12    1.087   1      0.004    1.091
14.7 34048.0 34048.0  0.0   34047.1 272640.0 245801.5 1756416.0   597362.6  21504.0 20390.6 3072.0 2750.3     13    1.183   2      0.050    1.233
15.7 34048.0 34048.0  0.0   34048.0 272640.0 21474.1  1756416.0   757347.0  22012.0 20792.0 3200.0 2791.0     15    1.336   2      0.050    1.386
16.7 34048.0 34048.0 34047.0  0.0   272640.0 48378.0  1756416.0   838594.4  22268.0 21003.5 3200.0 2813.2     16    1.433   2      0.050    1.484
```

此片段是从JVM启动后的前17秒中提取的。根据此信息，我们可以得出结论，在12次次要GC运行之后，执行了两次完整GC运行 ，总共运行了50毫秒。你可以通过基于GUI的工具（例如jconsole或 jvisualvm）得到相同结论。

在对这个结论点头之前，让我们看看从同一个JVM启动中收集的垃圾收集日志的输出。显然  `-XX:+PrintGCDetails`  告诉了我们一个不同而更详细的故事：

```bash
java -XX:+PrintGCDetails -XX:+UseConcMarkSweepGC eu.plumbr.demo.GarbageProducer
```

```
3.157: [GC (Allocation Failure) 3.157: [ParNew: 272640K->34048K(306688K), 0.0844702 secs] 272640K->69574K(2063104K), 0.0845560 secs] [Times: user=0.23 sys=0.03, real=0.09 secs] 
4.092: [GC (Allocation Failure) 4.092: [ParNew: 306688K->34048K(306688K), 0.1013723 secs] 342214K->136584K(2063104K), 0.1014307 secs] [Times: user=0.25 sys=0.05, real=0.10 secs] 
... cut for brevity ...
11.292: [GC (Allocation Failure) 11.292: [ParNew: 306686K->34048K(306688K), 0.0857219 secs] 971599K->779148K(2063104K), 0.0857875 secs] [Times: user=0.26 sys=0.04, real=0.09 secs] 
12.140: [GC (Allocation Failure) 12.140: [ParNew: 306688K->34046K(306688K), 0.0821774 secs] 1051788K->856120K(2063104K), 0.0822400 secs] [Times: user=0.25 sys=0.03, real=0.08 secs] 
12.989: [GC (Allocation Failure) 12.989: [ParNew: 306686K->34048K(306688K), 0.1086667 secs] 1128760K->931412K(2063104K), 0.1087416 secs] [Times: user=0.24 sys=0.04, real=0.11 secs] 
13.098: [GC (CMS Initial Mark) [1 CMS-initial-mark: 897364K(1756416K)] 936667K(2063104K), 0.0041705 secs] [Times: user=0.02 sys=0.00, real=0.00 secs] 
13.102: [CMS-concurrent-mark-start]
13.341: [CMS-concurrent-mark: 0.238/0.238 secs] [Times: user=0.36 sys=0.01, real=0.24 secs] 
13.341: [CMS-concurrent-preclean-start]
13.350: [CMS-concurrent-preclean: 0.009/0.009 secs] [Times: user=0.03 sys=0.00, real=0.01 secs] 
13.350: [CMS-concurrent-abortable-preclean-start]
13.878: [GC (Allocation Failure) 13.878: [ParNew: 306688K->34047K(306688K), 0.0960456 secs] 1204052K->1010638K(2063104K), 0.0961542 secs] [Times: user=0.29 sys=0.04, real=0.09 secs] 
14.366: [CMS-concurrent-abortable-preclean: 0.917/1.016 secs] [Times: user=2.22 sys=0.07, real=1.01 secs] 
14.366: [GC (CMS Final Remark) [YG occupancy: 182593 K (306688 K)]14.366: [Rescan (parallel) , 0.0291598 secs]14.395: [weak refs processing, 0.0000232 secs]14.395: [class unloading, 0.0117661 secs]14.407: [scrub symbol table, 0.0015323 secs]14.409: [scrub string table, 0.0003221 secs][1 CMS-remark: 976591K(1756416K)] 1159184K(2063104K), 0.0462010 secs] [Times: user=0.14 sys=0.00, real=0.05 secs] 
14.412: [CMS-concurrent-sweep-start]
14.633: [CMS-concurrent-sweep: 0.221/0.221 secs] [Times: user=0.37 sys=0.00, real=0.22 secs] 
14.633: [CMS-concurrent-reset-start]
14.636: [CMS-concurrent-reset: 0.002/0.002 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
```

根据这些信息，我们可以看到在12次次要GC运行之后，确实开始发生“一些不同的事情”。但是实际上，这个“不同的东西”不是两个完整的GC运行，而是一个在旧世代运行的单个GC，它由不同的阶段组成：

* 初始标记阶段，跨度为0.0041705秒或约4ms。此阶段是一个世界停止（stop-the-world）事件，它将停止所有应用程序线程以进行初始标记。
* 标记和预清理阶段。与应用程序线程并发执行。
* 最终备注阶段，范围为0.0462010秒或大约46ms。此阶段再次是世界停止（stop-the-world）事件。
* 扫描操作是并发执行的，没有停止应用程序线程。

因此，我们从实际的垃圾收集日志中看到的是，实际执行的是一次大型GC清理旧空间，而不是两次Full GC操作。

# GC算法

各个收集器的具体细节各不相同，但通常所有收集器都集中在两个领域。

* 找出所有仍然活着的对象
* 去掉所有其他的东西——那些被认为是死的和未用过的东西。

对活动对象的普查，是在所有收集器中通过标记过程实现的。

## 标记可访问对象

在JVM中使用的每种现代GC算法都是通过找出所有仍然存在的对象来开始工作的。这个概念最好用下面的图片来解释JVM的内存布局：

![](/images/Java-GC-mark-and-sweep.png)

首先，GC定义了一些特定的对象，如垃圾收集根. 这样的GC根的例子是：

* 当前执行方法的局部变量和输入参数
* 活动线程
* 加载类的静态字段
* JNI引用

接下来，GC遍历内存中的整个对象图，从那些垃圾收集根和根目录到其他对象的引用，例如实例字段。GC访问的每个对象是标记就像活着一样。

在上面的图片中，活动对象被表示为蓝色。当标记阶段结束时，每个活对象都被标记。因此，所有其他对象（上面图片上的灰色数据结构）都无法从GC根目录访问，这意味着应用程序不能再使用不可访问的对象。这些对象被认为是垃圾，GC应该在下面的阶段清除它们。

关于标记阶段，需要注意以下重要方面：

* 需要停止应用程序线程以进行标记，因为如果图形一直在变化，你将无法真正遍历图形。 当应用程序线程暂时停止以使JVM可以从事清理活动时，这种情况称为**安全点**，导致Stop The World暂停。 可以出于多种原因触发安全点，但是到目前为止，垃圾回收是引入安全点的最常见原因。
* 暂停的持续时间既不取决于堆中的对象总数，也不取决于堆的大小，而取决于活动对象的数量。 因此，增加堆大小不会直接影响标记阶段的持续时间。

标记阶段完成后，GC可以继续下一步，并开始删除无法访问的对象。

## 移除未使用的对象

对于不同的GC算法，未使用对象的去除有点不同，但是所有这样的GC算法可以被分成三组：扫描、压缩和复制。下一节将更详细地讨论每种算法。

### 扫描（Sweep）

标记和扫描（Mark and Sweep）算法从概念上讲是通过最简单的方法来处理垃圾，只是忽略此类对象。 这意味着在标记阶段完成之后，未访问对象占用的所有空间都被视为空闲空间，因此可以重新用于分配新对象。

该方法需要使用每个空闲区域及其大小的所谓空闲列表记录。 空闲列表的管理增加了对象分配的开销。 内置到此方法中的另一个缺点是-可能存在大量的可用区域，但是如果没有单个区域足够大以容纳分配，则分配仍将失败（Java中出现OutOfMemoryError）。

![Java GC扫描](/images/GC-sweep.png)

### 压缩（Compact）

Mark-Sweep-Compact算法通过将所有标记的对象（因此是活动对象）移动到内存区域的开头，解决了Mark和Sweep的缺点。 这种方法的缺点是增加了GC暂停时间，因为我们需要将所有对象复制到新位置并更新对此类对象的所有引用。 Mark和Sweep的好处也是显而易见的-在进行这种压缩操作之后，通过指针碰撞，新对象的分配再次变得非常便宜。 使用这种方法，总是知道可用空间的位置，也不会触发任何碎片问题。

![JAVA GC标记扫描压缩](/images/GC-mark-sweep-compact.png)

### 复制（Copy）

标记和复制算法与标记和压缩算法非常相似，因为它们也可以重新放置所有活动对象。 重要的区别是，重定位的目标是一个不同的内存区域，作为幸存者的新家。 标记和复制方法具有一些优点，因为复制可以在同一阶段与标记同时进行。 缺点是需要另外一个存储区域，该存储区域应足够大以容纳幸存的对象。

![Java GC标记和复制收集器](/images/GC-mark-and-copy-in-Java.png)

## 算法实现

对于大多数JVM，需要两种不同的GC算法-一种用于清理年轻一代，另一种用于清理老一代。

可以从JVM中捆绑的各种此类算法中进行选择。如果未明确指定垃圾回收算法，则将使用特定于平台的默认值。在本节中将说明每种算法的工作原理。

- **Serial GC [-XX:+UseSerialGC]** -带有年轻一代和老一代垃圾收集（即次要GC和主要GC）的简单标记清除紧凑方法。适用于运行简单的独立客户端计算机应用程序，该应用程序具有较低的内存占用量和较少的CPU能力。
- **Parallel GC [-XX:+UseParallelGC]** -具有多线程的次要GC的mark-sweep-compact方法的并行版本（主要GC仍然以串行方式在单个线程中发生）。`–XX:ParallelGCThreads=n` 选项用于定义运行次要GC时需要产生的并行线程数（通常n=CPU内核数）
- **Parallel Old GC [-XX:+UseParallelOldGC]** -并行版本的次要和主要GC的mark-sweep-compact方法。
- **Concurrent Mark Sweep (CMS) Collector [-XX:+UseConcMarkSweepGC]** -垃圾收集通常在暂停时发生（大型GC需要很长时间），这对于高响应性应用程序（我们无法承受较长的暂停时间）造成了问题。CMS Collector通过在应用程序线程内同时执行大多数垃圾收集工作（即，主要GC）来将这些暂停的影响降到最低（Minor GC仍遵循通常的并行算法，而应用程序线程没有任何并发进度）。**–XX:ParallelCMSThreads=n**选项可用于定义并行线程数。
- **G1 Garbage Collector [-XX:+UseG1GC]** —垃圾优先（G1）收集器将Heap划分为多个大小相等的区域，并且在调用GC时，首先使用较少的实时数据收集该区域（年轻一代和老一代的实现在这里不适用）。该收集器是一个并行处理，并发且增量紧凑的低中断垃圾收集器，旨在替换CMS收集器，这也是JDK9中默认的GC策略。

以下列表对于Java 8来说是正确的。对于较旧的Java版本，可用的组合可能略有不同：

| **新生代Young**       | **老年代Tenured** | **JVM options**                              |
| :-------------------- | :---------------- | :------------------------------------------- |
| Incremental           | Incremental       | -Xincgc                                      |
| **Serial**            | **Serial**        | **-XX:+UseSerialGC**                         |
| Parallel Scavenge     | Serial            | -XX:+UseParallelGC -XX:-UseParallelOldGC     |
| Parallel New          | Serial            | N/A                                          |
| Serial                | Parallel Old      | N/A                                          |
| **Parallel Scavenge** | **Parallel Old**  | **-XX:+UseParallelGC -XX:+UseParallelOldGC** |
| Parallel New          | Parallel Old      | N/A                                          |
| Serial                | CMS               | -XX:-UseParNewGC -XX:+UseConcMarkSweepGC     |
| Parallel Scavenge     | CMS               | N/A                                          |
| **Parallel New**      | **CMS**           | **-XX:+UseParNewGC -XX:+UseConcMarkSweepGC** |
| **G1**                | **-XX:+UseG1GC**  |                                              |

上述内容可能看起来过于复杂，不过实际上表中标粗的才需要了解的。其余的要么不推荐使用，要么不被支持，又或者在现实世界中不可行。

### 串行GC（Serial GC）

此垃圾收集器集合为年轻一代使用  mark-copy， 为老一代使用mark-sweep-compact。顾名思义，这两个收集器都是单线程收集器，无法并行处理当前任务。这两个收集器还触发世界暂停，从而停止所有应用程序线程。

因此，这种GC算法无法利用现代硬件中常见的多个CPU内核。与可用内核的数量无关，JVM在垃圾回收期间仅使用了一个。

通过在JVM启动脚本中指定单个参数来为年轻一代和老一代启用此收集器：

```bash
java -XX:+UseSerialGC com.mypackages.MyExecutableClass
```

此选项很有意义，并且仅对于堆大小为数百兆字节且在单个CPU的环境中运行的JVM才建议使用此选项。对于大多数服务器端部署，这是一种罕见的组合。大多数服务器端部署是在具有多个内核的平台上完成的，从本质上讲，这意味着通过选择串行GC，您可以对系统资源的使用设置人为限制。这导致空闲资源，否则这些资源可用于减少延迟或增加吞吐量。

现在让我们回顾一下使用串行GC时垃圾收集器日志的外观以及可以从中获得哪些有用的信息。为此，我们使用以下参数在JVM上打开了GC日志记录：

```bash
-XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintGCTimeStamps
```

结果输出类似于以下内容：

```
2015-05-26T14:45:37.987-0200: 151.126: [GC (Allocation Failure) 151.126: [DefNew: 629119K->69888K(629120K), 0.0584157 secs] 1619346K->1273247K(2027264K), 0.0585007 secs] [Times: user=0.06 sys=0.00, real=0.06 secs]
2015-05-26T14:45:59.690-0200: 172.829: [GC (Allocation Failure) 172.829: [DefNew: 629120K->629120K(629120K), 0.0000372 secs]172.829: [Tenured: 1203359K->755802K(1398144K), 0.1855567 secs] 1832479K->755802K(2027264K), [Metaspace: 6741K->6741K(1056768K)], 0.1856954 secs] [Times: user=0.18 sys=0.00, real=0.18 secs]
```

来自GC日志的简短代码片段暴露了许多有关JVM内部发生的信息。实际上，在此片段中发生了两次垃圾回收事件，其中一个事件是清理年轻一代，另一个事件是清理整个堆。

### 并行GC（Parallel GC）

垃圾收集器的这种组合在年轻一代中使用标记复制（mark-copy），而在旧一代中使用标记压缩（mark-sweep-compact）。Young和Old集合均触发世界停止事件，从而停止所有应用程序线程执行垃圾回收。两个收集器都使用多个线程来运行标记和复制/压缩阶段，因此命名为“并行”。使用这种方法，可以大大减少收集时间。

垃圾回收期间使用的线程数可以通过命令行参数 ` -XX:ParallelGCThreads=NNN` 进行配置 。默认值等于计算机中的内核数。

通过在JVM启动脚本中指定以下任意参数组合来完成对并行GC的选择：

```bash
java -XX:+UseParallelGC com.mypackages.MyExecutableClass
java -XX:+UseParallelOldGC com.mypackages.MyExecutableClass
java -XX:+UseParallelGC -XX:+UseParallelOldGC com.mypackages.MyExecutableClass
```

如果你的主要目标是提高吞吐量，那么Parallel Garbage Collector适用于多核计算机。通过更有效地利用系统资源，可以实现更高的吞吐量：

* 在收集过程中，所有内核都在并行清理垃圾，从而缩短了暂停时间
* 在垃圾收集周期之间，两个收集器都没有消耗任何资源

另一方面，由于收集的所有阶段必须无间断地进行，因此这些收集器仍然容易长时间停顿，在此期间你的应用程序线程将停止。因此，如果延迟是你的主要目标，则应检查下一个垃圾收集器组合。

### 并行标记和扫描（Concurrent Mark and Sweep）

该垃圾收集器集合的正式名称是“并发的标记和清除垃圾收集器”。它 在Young Generation中使用并行的世界停止标记复制算法， 在Old Generation中使用大多数并发标记清除算法。

该收集器的设计旨在避免在旧一代中进行长时间收集。它通过两种方法来实现。首先，它不会压缩旧版本，而是使用空闲列表来管理回收空间。其次，它在标记和清除阶段与应用程序同时完成大部分工作。这意味着垃圾回收不会显式停止应用程序线程执行这些阶段。但是，应该注意的是，它仍然与应用程序线程竞争CPU时间。默认情况下，此GC算法使用的线程数等于计算机物理内核数的1/4。

可以通过在命令行上指定以下选项来选择此垃圾收集器：

```
java -XX:+UseConcMarkSweepGC com.mypackages.MyExecutableClass
```

如果你的主要目标是延迟，那么在多核计算机上，这种组合是一个不错的选择。减少单个GC暂停的持续时间将直接影响最终用户对您的应用程序的感知方式，从而使他们感到应用程序响应速度更快。由于在大多数情况下，GC至少会消耗一些CPU资源，并且不执行应用程序的代码，因此与CPU绑定的应用程序相比，CMS的吞吐量通常比并行GC差。

### （G1 – Garbage First）

G1的主要设计目标之一是使由于垃圾收集而造成的世界停顿的持续时间和分布可预测和可配置。实际上，Garbage-First是一个软实时垃圾收集器，这意味着您可以为其设置特定的性能目标。你可以要求在任何给定的y毫秒长的时间范围内停止世界暂停不超过x毫秒，例如，在任何给定的秒内不超过5毫秒。垃圾优先GC将尽最大可能（但不能确定，这将是实时的）尽最大努力实现这一目标。

为了实现这一目标，G1建立在许多见解之上。首先，不必将堆分成连续的年轻一代和老一代。取而代之的是，将堆拆分为可以容纳对象的多个较小的堆区域（通常约为2048个）。每个区域可以是伊甸园区域，幸存者区域或旧区域。所有伊甸园地区和幸存者地区的合乎逻辑的联盟是年轻一代，而所有旧地区放在一起的都是老一代：

![G1堆区域](/images/g1-011-1024x325.png)

这使GC可以避免一次收集整个堆，而可以逐步解决问题：一次只考虑区域的一个子集，称为收集集。在每个暂停期间都会收集所有Young区域，但也可能包括一些Old区域：

![G1收藏集](/images/g1-02-1024x325.png)

G1的另一个新颖之处在于，它在并发阶段估计每个区域包含的实时数据量。这用于构建收集集：首先收集包含垃圾最多的区域。因此，名称为：垃圾优先收集。

要在启用了G1收集器的情况下运行JVM，请以

```
java -XX:+UseG1GC com.mypackages.MyExecutableClass
```

# 怎样触发GC

垃圾收集器只会销毁无法访问的对象，这是一个在后台自动发生的过程，通常程序员不应该对此做任何事情。

> 注意：在销毁对象之前，垃圾收集器最多只能在该对象上调用finalize()方法一次（对于任何对象而言，finalize()方法永远不会被调用多次）。默认的finalize()方法具有空的实现，但你可以通过覆盖它执行一些清理动作，例如关闭数据库连接等。一旦finalize()方法完成，垃圾收集器将销毁该对象。

以下带有对象构造函数和finalize()方法的Person类。

```java
class Person {   
    // 存储人（对象）名称
    String name;
     
    public Person(String name) {
        this.name = name;
    }
     
    @Override
    /* 覆盖finalize方法以检查哪个对象被垃圾回收 */
    protected void finalize() throws Throwable {
        // 将打印人（对象）的名称
        System.out.println("Person object - " + this.name + " -> successfully garbage collected");
    }
}
```

如果发生以下情况之一（无需等待堆中的世代老化），对象可能立即变得不可访问。

## 情况1：使引用变量为空

当对象的引用变量更改为NULL时，该对象将变得不可访问且可用于GC。

```java
// 创建一个Person对象
// new运算符为该对象动态分配内存并返回对该对象的引用
Person p1 = new Person("John Doe");
 
// 使用p1时会发生一些有意义的工作
...
...
...
 
// 不再使用p1
// 使p1符合gc条件
p1 = null;
 
// 调用垃圾收集器
System.gc(); // p1将被垃圾收集
```

输出为：

```
Person object - John Doe -> successfully garbage collected
```

## 情况2：重新分配引用变量

当一个对象的引用ID指向（引用）到另一个对象的引用ID时，则先前的对象将不再具有对其的引用。该对象将变得不可访问并且有资格使用GC。

```java
// 创建两个Person对象
// new运算符为一个对象动态分配内存并返回对该对象的引用
Person p1 = new Person("John Doe");
Person p2 = new Person("Jane Doe");
 
// p1处于使用状态时发生一些有意义的工作
...
...
...
 
// p1不再使用
// 使p1符合gc
p1 = p2; 
// p1现在称为p2
 
// 调用垃圾收集器
System.gc(); // p1将被垃圾收集
```

输出为：

```
Person object - John Doe -> successfully garbage collected
```

## 情况3：在方法内部创建的对象

方法以LIFO（后进先出）顺序存储在栈中。从栈中弹出此方法时，其所有成员都会死亡，并且如果在其中创建了一些对象，则这些对象也将变得不可访问，因此有资格使用GC。

```java
class PersonTest {    
    static void createMale() {
        // createMale()完成后，方法内的对象p1无法访问
        Person p1 = new Person("John Doe"); 
        createFemale();
        
        // 调用垃圾收集器
        System.out.println("GC Call inside createMale()");
        System.gc(); // p2将被垃圾回收
    }
 
    static void createFemale() {
        // createFemale()完成后，方法内的对象p2无法访问
        Person p2 = new Person("Jane Doe"); 
    }
     
    public static void main(String args[]) {
        createMale();
         
        // 调用垃圾收集器
        System.out.println("\nGC Call inside main()");
        System.gc(); // p1将被垃圾回收
    }
}
```

输出为：

```
GC Call inside createMale()
Person object - Jane Doe -> successfully garbage collected
 
GC Call inside main()
Person object - John Doe -> successfully garbage collected
```

## 情况4：匿名对象

当对象的引用ID未分配给变量时，该对象将变得不可访问并有资格使用GC。

```java
// 创建一个Person对象
// new运算符为该对象动态分配内存，并返回对该对象的引用
new Person("John Doe");
 
// 对象不能使用，因为没有变量赋值，因此它有资格使用gc
// 调用垃圾回收器
System.gc(); // 对象将被垃圾收集
```

输出为：

```
Person object - John Doe -> successfully garbage collected
```

## 情况5：仅具有内部引用的对象（*孤岛*）

仔细观察以下两个对象在失去其外部引用后，就变成了有资格使用GC的对象。

![孤岛](/images/island_isolation.png)

## 以编程方式调用GC

即使对象符合垃圾收集的条件，它也不会立即被垃圾收集器销毁，因为JVM每隔一定时间运行一次GC。但是，使用以下任何一种方法，我们都可以以编程方式向JVM请求运行垃圾收集器（但仍不能保证这些方法中的任何一种都一定会运行垃圾收集器，因为GC完全由JVM决定）。

- 使用**System.gc()**方法
- 使用**Runtime.getRuntime().gc()**方法

```java
// 创建两个Person对象
// new运算符为一个对象动态分配内存并返回对该对象的引用
Person p1 = new Person("John Doe");
Person p2 = new Person("Jane Doe");
 
// p1为p2时正在使用某些有意义的工作
...
...
...
 
// p1和p2不再使用
// 使p1符合gc
p1 = null; 
 
// 调用垃圾收集器
System.gc(); // p1将被垃圾回收
 
// 使p2符合gc 
p2 = null; 
 
// 调用垃圾收集器 
Runtime.getRuntime().gc(); // p2将被垃圾回收
```

输出为：

```
Person object - John Doe -> successfully garbage collected
Person object - Jane Doe -> successfully garbage collected
```
