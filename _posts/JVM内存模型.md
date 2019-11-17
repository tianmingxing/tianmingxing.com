---
title: JVM内存模型
date: 2019-09-20 23:49:00
tags:
- jvm虚拟机
- 内存模型
categories:
- JVM
---

对于Java开发人员来说，了解Java内存模型是必不可少的知识。

# JVM内存模型

在生产环境上一般会配置JVM参数以充分利用硬件资源，不管是在云主机或容器里面，也就是在启动应用时添加下面的某些配置：

* -XmsSetting —初始堆大小
* -XmxSetting —最大堆大小
* -XX:NewSizeSetting —新一代堆大小
* -XX:MaxNewSizeSetting —最大新一代堆大小
* -XX:MaxPermGenSetting —永久生成的最大大小
* -XX:SurvivorRatioSetting —新的堆大小比率（例如，如果Young Gen大小为10m并且内存开关为–XX:SurvivorRatio=2，则将为Eden空间保留5m，为两个Survivor空间分别保留2.5m，默认值= 8）
* -XX:NewRatio —提供新旧大小的比率（默认值= 2）

就像任何其他软件一样，JVM占用主机OS内存上的可用空间。

![](/images/host_os_memory_and_jvm.png)

在JVM内部存在单独的内存空间（堆、非堆和缓存），用来存储运行时数据和编译后的代码。
<!-- more -->

## 堆内存

![](/images/jvm_heap_memory.png)

* 堆分为两部分：年轻一代（Young Generation）和老一代（Old Generation）
* JVM启动时分配堆（初始大小：-Xms）
* 应用程序运行时堆大小增加/减少
* 最大大小：-Xmx

### 世代假设（Generational Hypothesis）

执行垃圾回收需要完全停止应用程序。同样很明显，当对象越多收集所有垃圾所需的时间就越长。但是，如果我们有可能使用较小的内存区域怎么办？在研究可能性时，一组研究人员观察到，应用程序内部的大多数分配都分为两类：

* 大多数对象很快就变得不被使用
* 那些通常不能存活很长时间的

这些观察结果在弱世代假设中综合在一起。基于此假设，VM内的内存分为所谓的Young Generation和Old Generation。后者有时也称为终身制。

![](/images/object-age-based-on-GC-generation-generational-hypothesis.png)

这并不是说这种方法没有问题。首先，来自不同世代的对象实际上可能相互引用，当收集世代时，它们也被视为“事实上的” GC根。

但最重要的是，世代假设实际上可能不适用于某些应用。由于GC算法针对“年轻”或“可能永远活着”的对象进行了优化，因此JVM对于“预期”寿命为“中等”的对象的表现相当差。

### 年轻一代

![](/images/how-java-garbage-collection-works.png)

* 保留新分配的对象。
* Young Gen包括三个部分-Eden Memory和两个Survivor Memory空间（S0，S1）。
* 大多数新创建的对象进入Eden空间。
* 当Eden空间中充满对象时，将执行Minor GC（又名Young Collection），并将所有幸存者对象移动到其中一个幸存者空间。
* 次要GC还检查幸存者对象并将其移至其他幸存者空间。因此，两个幸存者空间之一始终是空的。
* 在许多次GC循环后仍然存在的对象将移至老一代存储空间。也可以设置年轻一代对象的年龄阈值，当达到阈值后才有资格晋升为老一代。

#### 伊甸园（Eden）

Eden是内存中通常在创建对象时分配对象的区域。由于通常有多个线程同时创建许多对象，因此Eden进一步分为一个或多个驻留在Eden空间中的线程本地分配缓冲区（简称TLAB）。这些缓冲区允许JVM在相应的TLAB中直接在一个线程内分配大多数对象，从而避免了与其他线程的昂贵同步。

如果无法在TLAB内部进行分配（通常是因为那里没有足够的空间），则分配将转移到共享的Eden空间。如果那里没有足够的空间，则会触发Young Generation中的垃圾收集过程以释放更多空间。如果垃圾回收也没有在Eden内部产生足够的可用内存，则该对象将在旧一代中分配。

### 老一代

老一代内存空间的实现要复杂得多。老一代通常更大，并且被不太可能成为垃圾的对象所占据。

* 包含在多轮次要GC中存活下来的长寿命对象。
* 当老一代空间已满时，会执行Major GC（又名Old Collection）（通常需要更长的时间）。

## 非堆内存

* 包括永生代（Permanent Generation），不过从Java 8开始由Metaspace代替。
* Perm Gen存储每个类的结构，例如运行时常量池，字段和方法数据，方法和构造函数的代码以及内部字符串。
* 可以使用-XX:PermSize和-XX:MaxPermSize更改其大小

![](/images/jvm_non_heap.png)

## 缓存

* 包括代码缓存
* 存储由JIT编译器生成的已编译代码（即本地代码），JVM内部结构，已加载的分析器代理代码和数据等。
* 当代码缓存超过阈值时，它将被刷新（对象不会被GC重新定位）。

# 栈与堆

到目前为止，我没有提到任何有关Java Stack内存的信息，因为我想分别强调它的区别。

![](/images/jvm_stack_non_heap.png)

Java Stack内存用于执行线程，它包含方法特定的值以及对Heap中其他对象的引用。将Stack和Heap放在表中，看看它们之间的差异。

![](/images/stack_heap_diff.png)

下面的示程序介绍了Stack和Heap如何执行。

```java
class Person {
    int pid;
    String name;
    // constructor, setters/getters
}

public class Driver {
    public static void main(String[] args) {
        int id = 23;
        String pName = "Jon";
        Person p = null;
        p = new Person(id, pName);
    }
}
```

![](/images/stack_memory_heap_space.jpg)

# 版本变化

上面的Java内存模型是最常讨论的实现。但是最新的JVM版本有不同的变化，例如引入了以下新的内存空间。

* 保留区域 （Keep Area）-Young Generation中的新存储空间，用于包含最近分配的对象。直到下一代Young才执行GC。该区域可防止仅由于在开始Young Generation之前就已分配对象而对其进行升级。
* 元空间（Metaspace） —从Java 8开始，永生代被元空间代替。即使Perm Gen始终具有固定的最大大小，它也可以自动增加其大小（达到基础操作系统所提供的大小）。只要类加载器处于活动状态，元数据就在Metaspace中保持活动状态，并且无法释放。


# 内存相关问题

当存在严重的内存问题时，JVM崩溃并在程序输出中引发错误指示，如下所示。

* java.lang.StackOverFlowError —表示栈内存已满
* java.lang.OutOfMemoryError: Java heap space —表示堆内存已满
* java.lang.OutOfMemoryErrorGC Overhead limit exceeded —表示GC已达到其开销限制
* java.lang.OutOfMemoryError: Permgen space —表示永久生成空间已满
* java.lang.OutOfMemoryError: Metaspace —表示元空间已满（自Java 8开始）
* java.lang.OutOfMemoryError：Unable to create new native thread -表示JVM本机代码不再能够从基础操作系统中创建新的本机线程，因为已经创建了这么多线程，并且它们消耗了JVM的所有可用内存
* java.lang.OutOfMemoryError：request size bytes for reason —表示交换存储空间已被应用程序完全消耗
* java.lang.OutOfMemoryError：Requested array size exceeds VM limit –表示我们的应用程序使用的数组大小大于底层平台允许的大小

需要彻底理解的是，这些输出只能表示JVM的影响，而不能指出实际的错误。实际错误及其根本原因可能会出现在代码中的某个位置（例如，内存泄漏，GC问题，同步问题），资源分配甚至硬件设置。因此，我不建议你简单地增加受影响的资源大小来解决问题。也许你需要监视资源使用情况，分析每个类别，遍历堆转储，检查和调试/优化代码等。
