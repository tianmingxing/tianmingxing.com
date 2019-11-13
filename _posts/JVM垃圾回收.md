---
title: JVM垃圾回收
date: 2019-11-13 00:13:00
tags:
- java虚拟机
- 垃圾回收
categories:
- JVM
---

> 单从字面上看垃圾回收应该是查找并丢弃垃圾，但实际上它所做的恰恰相反。垃圾回收会跟踪所有仍在使用的对象，并将其余对象标记为垃圾。

## 了解GC的重要性

GC使用更少的样板代码（无需手动分配和释放内存）实现更快的开发，并消除了与内存相关的问题。但实际上JVM通过创建和删除太多对象来执行垃圾回收，这会导致严重的性能问题。为了有效地管理JVM中的垃圾回收和内存问题，你需要了解垃圾回收在幕后的工作原理。

# GC如何工作

Java Garbage Collector作为守护线程运行（即，低优先级线程在后台运行，以向用户线程提供服务或执行JVM任务）。它会不时地查看堆内存中的所有对象，并标识不再由程序任何部分引用的对象（应用程序代码不再可以访问这些未引用的对象）。然后将所有这些未引用的对象销毁，并为新创建的对象回收空间。

我们可以使用以下简单方法来解释Java垃圾回收过程。

1. **标记**：标识当前正在使用和未使用的对象
2. **正常删除**：删除未使用的对象并回收可用空间
3. **压缩删除**：将所有幸存的对象移动到一个幸存者空间（以提高向较新对象分配内存的性能）

但是，这种方法存在以下问题。

- 效率不高，因为本质上大多数新创建的对象将变为未使用状态
- 长寿命的对象也很有可能会在未来的GC周期中使用

为了解决上述问题，新对象以每个世代空间指示其中存储对象的寿命的方式存储在堆的单独的世代空间中。然后，垃圾收集会在两个主要阶段（分别称为“ **次要GC”**和“ **主要GC”）中进行**，并在完全删除之前扫描对象并在世代空间之间移动对象。

![img](/images/jvm_heap_memory.png)

## 标记和扫描模型

Mark＆Sweep Model是Java垃圾回收中的基础实现。它分为两个主要阶段。

1. **标记**：标识并标记仍在使用且可访问的所有对象引用（从GC根开始）（又称活动对象），其余的则视为垃圾。
2. **扫描**：遍历堆并在活动对象之间找到未占用的空间，这些空间记录在空闲列表中，可用于将来的对象分配。

## Java垃圾收集根

当没有对对象的引用时，应用程序代码将无法访问该对象，因此也有资格进行垃圾回收。等等，那甚至意味着什么？参考什么？如果是这样，第一个参考是什么？首先我也有同样的问题。让我解释一下这些引用和可访问性是如何在后台进行的。

为了使应用程序代码可以到达一个对象，应该有一个根对象，该根对象已连接到你的对象，并且可以从堆外部进行访问。从堆外部可以访问的此类根对象称为**垃圾收集（GC）根**。GC根有几种类型，例如局部变量，Active Java线程，静态变量，JNI引用等。这里需要学习的是**只要这些GC根目录之一直接或间接引用了我们的对象并且GC根目录仍然存在，我们的对象就可以视为可访问对象。一旦我们的对象失去对GC根的引用，它就变得不可访问，因此有资格进行垃圾回收**。

![img](/images/gc_root.png)

*GC根是本身被JVM引用的对象，因此可以防止其他所有对象被垃圾回收*

# GC资格

垃圾收集器只会销毁无法访问的对象，这是一个在后台发生的自动过程，通常程序员不应该对此做任何事情。

> 注意：在销毁对象之前，垃圾收集器最多只能在该对象上调用finalize()方法一次（对于任何对象而言，finalize()方法永远不会被调用多次）。默认的finalize()方法具有空的实现，但你可以通过覆盖它执行一些清理动作，例如关闭数据库连接等。一旦finalize()方法完成，垃圾收集器将销毁该对象。

考虑以下带有对象构造函数和finalize()方法的Person类。

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

![img](/images/island_isolation.png)

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

# GC执行策略

以下配置选项在JVM启动时可以调整选择GC策略。

- **Serial GC [-XX:+UseSerialGC]** -带有年轻一代和老一代垃圾收集（即次要GC和主要GC）的简单标记清除紧凑方法。适用于运行简单的独立客户端计算机应用程序，该应用程序具有较低的内存占用量和较少的CPU能力。
- **Parallel GC [-XX:+UseParallelGC]** -具有多线程的次要GC的mark-sweep-compact方法的并行版本（主要GC仍然以串行方式在单个线程中发生）。`–XX:ParallelGCThreads=n` 选项用于定义运行次要GC时需要产生的并行线程数（通常n=CPU内核数）
- **Parallel Old GC [-XX:+UseParallelOldGC]** -并行版本的次要和主要GC的mark-sweep-compact方法。
- **Concurrent Mark Sweep (CMS) Collector [-XX:+UseConcMarkSweepGC]** -垃圾收集通常在暂停时发生（大型GC需要很长时间），这对于高响应性应用程序（我们无法承受较长的暂停时间）造成了问题。CMS Collector通过在应用程序线程内同时执行大多数垃圾收集工作（即，主要GC）来将这些暂停的影响降到最低（Minor GC仍遵循通常的并行算法，而应用程序线程没有任何并发进度）。**–XX:ParallelCMSThreads=n**选项可用于定义并行线程数。
- **G1 Garbage Collector [-XX:+UseG1GC]** —垃圾优先（G1）收集器将Heap划分为多个大小相等的区域，并且在调用GC时，首先使用较少的实时数据收集该区域（年轻一代和老一代的实现在这里不适用）。该收集器是一个并行处理，并发且增量紧凑的低中断垃圾收集器，旨在替换CMS收集器。

# 内存泄漏

这是学习GC如何工作的主要目的。Java垃圾收集专用于跟踪活动对象，删除未使用的对象并为将来的对象释放堆，这是JVM中最重要的内存管理机制。程序员可以通过保留对未使用对象的引用来使此自动过程无效，从而使其仍然可以访问，因此无法获得使用GC的资格。此类未使用但仍引用的对象的累积会无效率地消耗Heap内存，这种情况称为内存泄漏。

![img](/images/gc_root_memory_leak.png)

*GC根目录可能存在内存泄漏*

对于大型应用程序而言，检测这种类型的逻辑内存泄漏可能是开发人员最头疼的事情。好在当前有这类分析工具和，不过它们只能排查出可疑对象。
