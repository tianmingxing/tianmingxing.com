---
title: 介绍JVM中OOM的8种类型
date: 2019-11-17 16:00:00
tags:
- JVM OOM
- OutOfMemoryError
categories:
- JVM
---

![OutOfMemoryError](/images/OutOfMemoryError.png)

# java.lang.OutOfMemoryError

## Java heap space

Java应用程序仅允许使用有限的内存，此限制是在应用程序启动期间指定的。Java内存被分为两个不同的区域，这些区域称为Heap（堆空间）和Permgen（永久代）：

![](/images/java-lang-outofmemoryerror-java-heap-space.png)

这些区域的大小是在Java虚拟机（JVM）启动期间设置的，可以通过指定JVM参数-Xmx和-XX：MaxPermSize进行自定义。如果未明确设置大小，将使用特定于平台的默认值。

应用程序尝试添加更多的数据放入堆空间区域，但没有足够的空间供它，可能会有大量的物理内存可用，但当JVM达到堆大小限制时，都会引发Java堆空间错误。

### 是什么原因造成的？

因为你尝试将XXL大的对象放入S大小的Java堆空间中。也就是说，该应用程序需要更多的Java堆空间才能正常运行。

* 用量/数据量激增。该应用程序旨在处理一定数量的用户或一定数量的数据。当用户数量或数据量突然达到峰值并超过预期阈值时，在峰值之前正常运行的操作将停止运行并触发java.lang.OutOfMemoryError: Java heap space。
* 内存泄漏。一种特殊类型的编程错误将导致您的应用程序不断消耗更多的内存。每次使用应用程序的泄漏功能时，都会将某些对象留在Java堆空间中。随着时间的流逝，泄漏的对象会消耗所有可用的Java堆空间。
<!-- more -->
### 示例

#### 简单示例

以下Java代码尝试分配2M整数数组。当你对其进行编译并以12MB的Java堆空间（java -Xmx12m OOM）启动时，它会失败并显示java.lang.OutOfMemoryError: Java heap space。Java堆空间为13MB，程序运行正常。

```java
class OOM {
  static final int SIZE = 2 * 1024 * 1024;
  public static void main(String[] a) {
    int[] i = new int[SIZE];
  }
}
```

#### 内存泄漏示例

在Java中，当开发人员创建和使用新对象（例如new Integer(5)）时，他们不必自己分配内存-Java虚拟机（JVM）可以解决这一问题。在应用程序的生命周期中，JVM会定期检查内存中哪些对象仍在使用，哪些未使用。可以丢弃未使用的对象，并回收和再次使用内存。此过程称为垃圾收集。JVM中负责收集的相应模块称为垃圾收集器（GC）。

Java的自动内存管理依靠GC定期查找未使用的对象并将其删除。简单地说，可以说Java中的内存泄漏是指某些对象不再被应用程序使用，而Garbage Collection无法识别它的情况。结果，这些未使用的对象将无限期地保留在Java堆空间中。该堆积最终将触发java.lang.OutOfMemoryError: Java heap space错误。

构造满足内存泄漏定义的Java程序非常容易：

```java
class KeylessEntry {
 
   static class Key {
      Integer id;
 
      Key(Integer id) {
         this.id = id;
      }
 
      @Override
      public int hashCode() {
         return id.hashCode();
      }
   }
 
   public static void main(String[] args) {
      Map m = new HashMap();
      while (true)
         for (int i = 0; i < 10000; i++)
            if (!m.containsKey(new Key(i)))
               m.put(new Key(i), "Number:" + i);
   }
}
```

当执行上面的代码时，你可能希望它可以永久运行而不会出现任何问题，假设缓存解决方案仅将基础Map扩展到10,000个元素，除此之外，所有键都已经存在于HashMap中。但是，实际上，元素将继续添加，因为Key类在其hashCode()旁边不包含适当的equals()实现。

结果，随着时间的流逝，随着泄漏代码的不断使用，“缓存”结果最终会占用大量Java堆空间。并且，当泄漏的内存填满堆区域中的所有可用内存，而Garbage Collection无法清除它时。

该解决方案很容易–添加与下面的方法类似的equals()方法的实现。

```java
@Override
public boolean equals(Object o) {
   boolean response = false;
   if (o instanceof Key) {
      response = (((Key)o).id).equals(this.id);
   }
   return response;
}
```

### 解决办法

在某些情况下，分配给JVM的堆数量不足以满足在JVM上运行的应用程序的需求。在这种情况下，你应该分配更多的堆。

但是，在许多情况下，提供更多的Java堆空间将无法解决问题。例如，如果应用程序包含内存泄漏，则添加更多的堆只会推迟错误。此外，增加Java堆空间的数量也往往会增加GC暂停的时间，从而影响应用程序的吞吐量或延迟。

如果希望解决Java堆空间的根本问题而不是掩盖症状，则需要弄清楚代码的哪一部分负责分配最多的内存。换句话说，你需要清楚以下问题：

1. 哪些对象占据堆的大部分
1. 这些对象在源代码中的分配位置

在分配Java堆空间时你可以对GB使用g或G，对于MB使用m或M，对于KB使用k或K。例如，以下几种形式表示的最大Java堆空间为1GB：

```bash
java -Xmx1073741824 com.mycompany.MyClass
java -Xmx1048576k com.mycompany.MyClass
java -Xmx1024m com.mycompany.MyClass
java -Xmx1g com.mycompany.MyClass
```

## GC overhead limit exceeded

Java运行时环境包含一个内置的垃圾收集（GC）进程。在许多其他编程语言中，开发人员需要手动分配和释放内存区域，以便可以重新使用释放的内存。

另一方面，Java应用程序仅需要分配内存。每当不再使用内存中的特定空间时，称为垃圾收集的单独进程就会为它们清除内存。

在显示 `java.lang.OutOfMemoryError: GC overhead limit exceeded` 错误信息时表示应用程序已经耗尽了几乎所有的可用内存并且GC一直未能回收它。

### 是什么原因造成的？

应用程序花费太多的时间做垃圾收集太少的结果JVM的方式。默认情况下，如果JVM花费的时间超过总时间的98％，并且在GC之后仅恢复了不到2％的堆，则将其配置为引发此错误。

![](/images/OOM-example-schema3.png)

如果此GC开销限制不存在，将会发生什么？请注意，只有在几个GC周期后释放2％的内存时，才会抛出错误。这意味着GC能够清理的少量堆可能会很快再次被填满，从而迫使GC再次重新启动清理过程。这形成了一个恶性循环，其中CPU 100％忙于GC，无法进行任何实际工作。应用程序的最终用户将面临极慢的速度-通常在几毫秒内完成的操作需要几分钟才能完成。

因此，“ java.lang.OutOfMemoryError: GC overhead limit exceeded ”是一个很好的使用快速失败原理的例子。

### 示例

在以下示例中，我们通过初始化Map并在不终止的循环中将键值对添加到Map中来创建错误：

```java
class Wrapper {
  public static void main(String args[]) throws Exception {
    Map map = System.getProperties();
    Random r = new Random();
    while (true) {
      map.put(r.nextInt(), "value");
    }
  }
}
```

然后用下面的参数启动：

```bash
java -Xmx100m -XX:+UseParallelGC Wrapper
```

### 解决办法

如果希望解决Java堆空间的根本问题而不是掩盖症状，则需要弄清楚代码的哪一部分负责分配最多的内存。换句话说，需要搞清楚以下问题：

1. 哪些对象占据堆的大部分
1. 这些对象在源代码中的分配位置

更改JVM启动配置，并在启动脚本中仅添加（或增加该值，如果存在的话）一个参数：

```bash
java -Xmx1024m com.yourcompany.YourClass
```

在上面的示例中，为Java进程分配了1GB的堆空间。

## Permgen space

Java应用程序仅允许使用有限的内存。在应用程序启动期间，将指定特定应用程序可以使用的具体内存量。Java内存被划分为不同的区域，如下图所示：

![](/images/java.lang_.outofmemoryerror-permgen-space.png)

所有这些区域的大小，包括permgen区域，都是在JVM启动期间设置的。如果未自行设置大小，则将使用特定于平台的默认值。

该错误表示永久代的内存区域被耗尽。

### 是什么原因造成的?

要了解 ` java.lang.OutOfMemoryError: PermGen space` 的原因，我们需要了解此特定内存区域的用途。

出于实际目的，永久生成主要由装入并存储到PermGen中的类声明组成。这包括类的名称和字段，带有方法字节码的方法，常量池信息，与类关联的对象数组和类型数组以及即时编译器优化。

根据上面的定义，可以得出PermGen大小要求取决于加载的类的数量以及此类声明的大小。因此，可以说造成该错误的主要原因是永久区中装入了太多的类或太大的类。

### 示例

#### 简单示例

PermGen空间的使用与加载到JVM中的类的数量密切相关。以下代码是最简单的示例：

```java
import javassist.ClassPool;

public class MicroGenerator {
  public static void main(String[] args) throws Exception {
    for (int i = 0; i < 100_000_000; i++) {
      generate("eu.plumbr.demo.Generated" + i);
    }
  }

  public static Class generate(String name) throws Exception {
    ClassPool pool = ClassPool.getDefault();
    return pool.makeClass(name).toClass();
  }
}
```

在此示例中，源代码遍历循环并在运行时生成类。javassist库正在处理类生成的复杂性。

启动上面的代码将继续生成新的类，并将其定义加载到Permgen空间中，直到该空间被充分利用并且抛出java.lang.OutOfMemoryError: Permgen space异常。

#### 重新部署示例

对于更复杂，更实际的示例，让我们逐步介绍一下在应用程序重新部署期间发生的Permgen空间错误。重新部署应用程序时，你希望垃圾回收会摆脱引用所有先前加载的类的加载器，并被加载新类的类加载器取代。

不幸的是，许多第三方库以及对线程，JDBC驱动程序或文件系统句柄等资源的不良处理使得无法卸载以前使用的类加载器。反过来，这意味着在每次重新部署期间，所有先前版本的类仍将驻留在PermGen中，从而在每次重新部署期间生成数十兆的垃圾。

让我们想象一个使用JDBC驱动程序连接到关系数据库的示例应用程序。启动应用程序时，初始化代码将加载JDBC驱动程序以连接到数据库。对应于规范，JDBC驱动程序向java.sql.DriverManager进行注册。该注册包括将对驱动程序实例的引用存储在DriverManager的静态字段中。

现在，当从应用程序服务器取消部署应用程序时，java.sql.DriverManager仍将保留该引用。我们最终获得了对驱动程序类的实时引用，而驱动程序类又保留了用于加载应用程序的java.lang.Classloader实例的引用。反过来，这意味着垃圾回收算法无法回收空间。

而且该java.lang.ClassLoader实例仍引用应用程序的所有类，通常在PermGen中占据数十兆字节。这意味着只需少量重新部署即可填充通常大小的PermGen。

### 解决办法

#### 1. 解决初始化时的OutOfMemoryError

在应用程序启动期间触发由于PermGen耗尽导致的OutOfMemoryError时，解决方案很简单。该应用程序仅需要更多空间才能将所有类加载到PermGen区域，因此我们只需要增加其大小即可。为此，更改你的应用程序启动配置并添加（或增加，如果存在）-XX:MaxPermSize参数，类似于以下示例：

```bash
java -XX:MaxPermSize=512m com.yourcompany.YourClass
```

上面的配置将告诉JVM，PermGen可以增长到512MB。

#### 2. 解决重新部署时的OutOfMemoryError

重新部署应用程序后立即发生OutOfMemoryError时，应用程序会遭受类加载器泄漏的困扰。在这种情况下，解决问题的最简单，继续进行堆转储分析–使用类似于以下命令的重新部署后进行堆转储：

```bash
jmap -dump:format=b,file=dump.hprof <process-id>
```

然后使用你最喜欢的堆转储分析器打开转储（Eclipse MAT是一个很好的工具）。在分析器中可以查找重复的类，尤其是那些正在加载应用程序类的类。从那里，你需要进行所有类加载器的查找，以找到当前活动的类加载器。

对于非活动类加载器，你需要通过从非活动类加载器收集到GC根的最短路径来确定阻止它们被垃圾收集的引用。有了此信息，你将找到根本原因。如果根本原因是在第三方库中，则可以进入Google/StackOverflow查看是否是已知问题以获取补丁/解决方法。

#### 3. 解决运行时OutOfMemoryError

第一步是检查是否允许GC从PermGen卸载类。在这方面，标准的JVM相当保守-类是天生的。因此，一旦加载，即使没有代码在使用它们，类也会保留在内存中。当应用程序动态创建许多类并且长时间不需要生成的类时，这可能会成为问题。在这种情况下，允许JVM卸载类定义可能会有所帮助。这可以通过在启动脚本中仅添加一个配置参数来实现：

```bash
-XX:+CMSClassUnloadingEnabled
```

默认情况下，此选项设置为false，因此要启用此功能，你需要在Java选项中显式设置。如果启用CMSClassUnloadingEnabled，GC也会扫描 PermGen并删除不再使用的类。请记住，只有同时使用UseConcMarkSweepGC时此选项才起作用。

```bash
-XX:+UseConcMarkSweepGC
```

在确保可以卸载类并且问题仍然存在之后，你应该继续进行堆转储分析–使用类似于以下命令的方法进行堆转储：

```bash
jmap -dump:file=dump.hprof,format=b <process-id>
```

然后，使用你最喜欢的堆转储分析器（例如Eclipse MAT）打开转储，然后根据已加载的类数查找最昂贵的类加载器。从此类加载器中，你可以继续提取已加载的类，并按实例对此类进行排序，以使可疑对象排在首位。

然后，对于每个可疑者，就需要你手动将根本原因追溯到生成此类的应用程序代码。

## Metaspace

Java应用程序只能使用有限的内存。在应用程序启动期间，将指定特定应用程序可以使用的确切内存量。为了使事情更复杂，Java内存被划分为不同的区域，如下图所示：

![](/images/OOM-example-metaspace.png)

可以在JVM启动期间指定所有这些区域的大小，包括元空间区域。如果您自己不确定大小，将使用特定于平台的默认值。

所述java.lang.OutOfMemoryError：元空间中消息指示所述元空间区域在存储器中被耗尽。

### 是什么原因造成的？

如果您不是Java领域的新手，那么您可能熟悉Java内存管理中另一个称为PermGen的概念。从Java 8开始，Java中的内存模型发生了重大变化。引入了一个名为Metaspace的新存储区，并删除了Permgen。做出此更改是由于多种原因，包括但不限于：

所需的冷气量很难预测。它导致预配置不足触发java.lang.OutOfMemoryError：Permgen大小错误，或者预配置过多导致资源浪费。
改进了GC性能，实现了并发类数据的重新分配，而无需GC暂停和元数据上的特定迭代器
支持进一步的优化，例如G1并发类卸载。
因此，如果您熟悉PermGen，那么您需要了解的所有知识就是– Java 8之前PermGen中的内容（类的名称和字段，具有方法字节码的类方法，常量池，JIT优化等） ）–现在位于Metaspace中。

如您所见，元空间的大小要求取决于装入的类的数量以及此类声明的大小。所以很容易看到的主要原因java.lang.OutOfMemoryError：元空间是：太多的级别或过大的类加载到元空间。

### 示例

元空间的使用与加载到JVM中的类的数量密切相关。以下代码是最简单的示例：

```java
public class Metaspace {
	static javassist.ClassPool cp = javassist.ClassPool.getDefault();

	public static void main(String[] args) throws Exception{
		for (int i = 0; ; i++) { 
			Class c = cp.makeClass("eu.plumbr.demo.Generated" + i).toClass();
		}
	}
}
```

在此示例中，源代码遍历循环并在运行时生成类。所有这些生成的类定义最终都会消耗Metaspace。类生成的复杂性由javassist库处理。

该代码将继续生成新的类，并将其定义加载到Metaspace中，直到该空间被充分利用且抛出java.lang.OutOfMemoryError: Metaspace为止。

### 解决方法

当由于元空间而面临OutOfMemoryError时，第一个解决方案应该是显而易见的。如果应用程序耗尽了内存中的Metaspace区域，则应增加Metaspace的大小。更改应用程序启动配置并增加以下内容：

```bash
-XX:MaxMetaspaceSize=512m
```

上面的配置示例告诉JVM，允许Metaspace增长到512 MB。

另一种解决方案甚至更简单。你可以通过删除此参数来完全解除对Metaspace大小的限制，JVM默认对Metaspace的大小没有限制。但是请注意以下事实：这样做可能会导致大量交换或达到本机物理内存而分配失败。

## Unable to create new native thread

Java应用程序本质上是多线程的。这意味着用Java编写的程序可以一次（看似）做几件事。例如，即使在只有一个处理器的机器上，当你将内容从一个窗口拖动到另一个窗口时，在后台播放的电影也不会因为你一次执行多个操作而停止。

思考线程的一种方法是将线程视为可以向其提交要执行的任务的工人。如果你只有一名工人，那么他或她当时只能执行一项任务。但是，当你有十几个工人可用时，他们可以同时执行你的几个命令。

现在，与物理世界中的工作人员一样，JVM中的线程需要一定的空间来执行被召唤来处理的工作。当线程多于内存时：

![](/images/java-lang-outofmemoryerror-unable-to-create-new-native-thread.png)

java.lang.OutOfMemoryError: Unable to create new native thread意味着Java应用程序已达到其可以启动线程数的限制。

### 是什么原因造成的？

每当JVM从操作系统请求新线程时，都无法创建新的本机线程。只要基础操作系统无法分配新的本机线程，就会抛出该OutOfMemoryError。本机线程的确切限制非常依赖于平台，因此建议通过运行类似于以下示例的测试来找出这些限制。但是，通常导致java.lang.OutOfMemoryError的情况：无法创建新的本机线程需要经历以下阶段：

1. JVM内部运行的应用程序请求新的Java线程
1. JVM本机代码代理为操作系统创建新本机线程的请求
1. 操作系统尝试创建一个新的本机线程，该线程需要将内存分配给该线程
1. 操作系统将拒绝本机内存分配，原因是32位Java进程大小已耗尽其内存地址空间（例如，已达到（2-4）GB进程大小限制）或操作系统的虚拟内存已完全耗尽
1. 该java.lang.OutOfMemoryError: Unable to create new native thread引发错误。

### 示例

在循环中创建并启动新线程。运行代码时，将很快达到操作系统限制，并显示java.lang.OutOfMemoryError: Unable to create new native thread。

```java
while(true){
    new Thread(new Runnable(){
        public void run() {
            try {
                Thread.sleep(10000000);
            } catch(InterruptedException e) { }        
        }    
    }).start();
}
```

具体本机线程限制取决于平台，例如，在Windows，Linux和Mac OS X上进行的测试表明：

* 64位Mac OS X 10.9，Java 1.7.0_45 –创建＃2031线程后，JVM死亡
* 64位Ubuntu Linux，Java 1.7.0_45 –创建＃31893线程后，JVM死亡
* 64位Windows 7，Java 1.7.0_45 –由于操作系统使用的线程模型不同，因此在此特定平台上似乎不会抛出此错误。在线程250,000上，该过程仍然有效，即使交换文件已增长到10GB并且应用程序面临极端性能问题。

### 解决办法

可以通过增加操作系统级别的限制来绕过无法创建新的本机线程问题。例如，如果限制了JVM可在用户空间中产生的进程数，则应检查出并可能增加该限制：

```bash
[root@dev ~]# ulimit -a
core file size          (blocks, -c) 0
--- cut for brevity ---
max user processes              (-u) 1800
```

通常，OutOfMemoryError对新的本机线程的限制表示编程错误。当应用程序产生数千个线程时，很可能出了一些问题—很少有应用程序可以从如此大量的线程中受益。

解决问题的一种方法是开始进行线程转储以了解情况。

## Out of swap space

Java应用程序在启动期间会获得有限的内存。此限制是通过-Xmx和其他类似的启动参数指定的。在JVM请求的总内存大于可用物理内存的情况下，操作系统开始将内容从内存换出到硬盘驱动器。

![](/images/outofmemoryerror-out-of-swap-space.png)

错误表明交换空间也已用尽，并且由于缺少物理内存和交换空间，新的尝试分配失败。

### 是什么原因造成的？

当来自本机堆的字节分配请求失败并且本机堆快要用尽时，JVM会抛出该异常。该消息表示分配失败的大小（以字节为单位）以及内存请求的原因。

在Java进程已开始交换的情况下会出现问题，回想一下Java是一种垃圾回收语言已经不是一个好情况。现代GC算法可以很好地完成工作，但是当遇到由交换引起的延迟问题时，GC暂停通常会增加到大多数应用程序无法承受的水平。

交换空间不足？通常是由操作系统级别的问题引起的，例如：

* 操作系统配置的交换空间不足。
* 系统上的另一个进程正在消耗所有内存资源。

应用程序也可能由于本机泄漏而失败，例如，如果应用程序或库代码连续分配内存但未将其释放给操作系统。

### 解决办法

首先，通常最简单的解决方法是增加交换空间。这样做的方法是特定于平台的，例如在Linux中，可以使用以下示例命令序列来实现，这些命令创建并附加一个大小为640MB的新交换文件：

```bash
swapoff -a 
dd if=/dev/zero of=swapfile bs=1024 count=655360
mkswap swapfile
swapon swapfile
```

由于垃圾回收会清除内存内容，因此通常对于Java进程而言，交换是不希望的。在交换的分配上运行垃圾回收算法可能会使GC暂停的时间增加几个数量级，因此在跳到简单的解决方案之前，你应该三思而后行。

如果将应用程序部署在JVM需要与之竞争资源的“嘈杂邻居”旁边，则应将服务隔离到单独的（虚拟）计算机上。

在许多情况下，唯一真正可行的选择是升级计算机以包含更多内存，或者优化应用程序以减少其内存占用。

## Requested array size exceeds VM limit

Java对程序可以分配的最大数组大小有限制。确切的限制是特定于平台的，但通常在1到21亿个元素之间。

![](/images/java.lang_.outofmemoryerror-array-size-exceeds-vm-limit.png)

当遇到请求的数组大小超出VM限制时，这意味着因错误而崩溃的应用程序正试图分配一个大于Java虚拟机可以支持的数组。

### 是什么原因造成的？

该错误由JVM中的本机代码引发。当JVM执行特定于平台的检查时，会在为数组分配内存之前发生这种情况：分配的数据结构在此平台中是否可寻址。此错误不如你最初想象的普遍。

你很少遇到此错误的原因是Java数组由int索引。Java中的最大正整数为2 ^ 31 – 1 = 2,147,483,647。而且特定于平台的限制实际上可以接近此数字-例如，在Java 1.7上的64位MB Pro上，最多包含2,147,483,645或Integer.MAX_VALUE-2元素。

将数组的长度增加1到Integer.MAX_VALUE-1将导致熟悉的OutOfMemoryError：

```bash
Exception in thread "main" java.lang.OutOfMemoryError: Requested array size exceeds VM limit
```

但是限制可能不会那么高–在分配带有约11亿个元素的数组时，你已经达到“ java.lang.OutOfMemoryError: Requested array size exceeds VM limit ”。

### 示例

尝试创建请求的数组大小超出VM限制错误时，请看以下代码：

```java
for (int i = 3; i >= 0; i--) {
	try {
		int[] arr = new int[Integer.MAX_VALUE-i];
		System.out.format("Successfully initialized an array with %,d elements.\n", Integer.MAX_VALUE-i);
	} catch (Throwable t) {
		t.printStackTrace();
	}
}
```

该示例进行四次迭代，并在每个回合中初始化一个长原语数组。该程序尝试初始化的数组的大小在每次迭代时都增加一，最终达到Integer.MAX_VALUE。现在，在具有Hotspot 7的64位Mac OS X上启动代码段时，应该获得类似于以下内容的输出：

```bash
java.lang.OutOfMemoryError: Java heap space
	at eu.plumbr.demo.ArraySize.main(ArraySize.java:8)
java.lang.OutOfMemoryError: Java heap space
	at eu.plumbr.demo.ArraySize.main(ArraySize.java:8)
java.lang.OutOfMemoryError: Requested array size exceeds VM limit
	at eu.plumbr.demo.ArraySize.main(ArraySize.java:8)
java.lang.OutOfMemoryError: Requested array size exceeds VM limit
	at eu.plumbr.demo.ArraySize.main(ArraySize.java:8)
```

请注意，在面对所请求的数组大小在最后两次尝试中超过VM限制之前，分配失败的原因是更加熟悉的java.lang.OutOfMemoryError: Java heap space消息。发生这种情况是因为你试图腾出空间的2 ^ 31-1 int原语需要8G内存，该内存小于JVM使用的默认值。

此示例还说明了为什么错误如此罕见的原因–为了看到VM受到阵列大小的限制，你需要分配一个大小介于平台限制和Integer.MAX_INT之间的阵列。当我们的示例在具有Hotspot 7的64位Mac OS X上运行时，只有两个这样的数组长度：Integer.MAX_INT-1和Integer.MAX_INT。

### 解决办法

请求阵列大小超过限制VM可以表现为任一下列情况的结果：

* 你的阵列太大，最终导致其大小介于平台限制和Integer.MAX_INT之间
* 你故意尝试分配大于2 ^ 31-1元素的数组以尝试使用限制。

在第一种情况下，检查你的代码库以查看是否真的需要那么大的数组。也许可以减小数组的大小并使用它来完成。或将阵列分成较小的批量，并批量加载所需的数据以适合您的平台限制。

在第二种情况下–记住Java数组是由int索引的。因此，在平台内使用标准数据结构时，不能超出数组中的2 ^ 31-1个元素。实际上，在这种情况下，您已经被编译器阻止，并在编译过程中宣布“error: integer number too large ”。

但是，如果你使用真正的大型数据集，则需要重新考虑你的选择。可以分批加载需要使用的数据，而仍然使用标准Java工具，否则可能超出标准实用程序的范围。实现此目的的一种方法是查看sun.misc.Unsafe类。这使你可以像在C语言中一样直接分配内存。

## Kill process or sacrifice child

为了理解此错误，我们需要弥补操作系统的基础知识。操作系统是建立在流程概念之上的。这些进程由几个内核作业负责，其中一个名为“内存不足的杀手（Out of memory killer）”在我们这种特殊情况下是我们感兴趣的。

该内核作业可以在极低的内存条件下消灭你的进程。当检测到这种情况时，将激活“内存不足杀手”并选择要杀死的进程。使用一组对所有过程进行评分的启发式方法选择目标，然后选择得分最差的目标来杀死目标。杀进程或牺牲子进程不由JVM代理，而是内置到操作系统内核的安全网。

![](/images/out-of-memory-kill-process-or-sacrifice-child.png)

杀进程或牺牲子进程时可用虚拟内存（包括交换）被产生误差被消耗到整个操作系统稳定性被投入风险的程度。在这种情况下，内存不足杀手选择恶意进程并将其杀死。

### 是什么原因造成的？

默认情况下，Linux内核允许进程请求的内存比系统中当前可用的内存更多。考虑到大多数进程实际上从未真正使用过它们分配的所有内存，因此这在世界范围内都是有意义的。与这种方法最简单的比较是宽带运营商。他们向所有消费者提供100Mbit的下载承诺，远远超出了他们网络中的实际带宽。再次押注的事实是，用户将不会同时全部使用其分配的下载限制。因此，一个10Gbit链路可以成功服务超过我们的简单数学所允许的100个用户。

如果你的某些程序正在耗尽系统内存的路径上，则可以看到这种方法的副作用。这可能会导致极低的内存状况，在这种情况下无法分配任何页面进行处理。你可能已经遇到过这样的情况，即使没有root帐户也无法杀死有问题的任务。为防止此类情况，杀手启动并识别出将被杀的流氓进程。

现在我们有了上下文，你如何知道是什么触发了“杀手”并在凌晨5点将你唤醒？激活的一种常见触发因素隐藏在操作系统配置中。在/proc/sys/vm/overcommit_memory中检查配置时，会得到第一个提示-此处指定的值指示是否允许所有malloc()调用成功。请注意，proc文件系统中参数的路径取决于受更改影响的系统。

过量使用的配置允许为此流氓进程分配越来越多的内存，这最终可能触发“ 内存不足杀手 ”来准确执行其意图。

### 示例

在Linux上编译并启动以下Java代码段时：

```java
public class OOM {

public static void main(String[] args){
	java.util.List<int[]> l = new java.util.ArrayList();
	for (int i = 10000; i < 100000; i++) {
			try {
				l.add(new int[100_000_000]);
			} catch (Throwable t) {
				t.printStackTrace();
			}
		}
	}
}
```

在系统日志（我们的示例中为/var/log/kern.log）中遇到类似于以下错误：

```bash
Jun  4 07:41:59 plumbr kernel: [70667120.897649] Out of memory: Kill process 29957 (java) score 366 or sacrifice child
Jun  4 07:41:59 plumbr kernel: [70667120.897701] Killed process 29957 (java) total-vm:2532680kB, anon-rss:1416508kB, file-rss:0kB
```

请注意，可能需要调整交换文件和堆大小，在我们的测试案例中使用了-Xmx2g指定的2g堆，并具有以下交换配置：

```bash
swapoff -a 
dd if=/dev/zero of=swapfile bs=1024 count=655360
mkswap swapfile
swapon swapfile
```

### 解决办法

有几种方法可以处理这种情况。解决该问题的第一个也是最直接的方法是将系统迁移到具有更多内存的实例。

其他可能性包括微调OOM杀手，在几个小实例上水平扩展负载或减少应用程序的内存需求。
