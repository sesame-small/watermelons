---
title: Virtual Thread
layout: page
nav_order: 1
parent: Java
permalink: java/virtualthread
---

# {{ page.title }}  

--- 
> 虚拟线程是Java21引入的一种基于[Project Loom](https://github.com/openjdk/loom){:target="_blank"}技术实现的轻量级线程，是一种在用户空间中运行的线程，它们可以在操作系统级别的线程上高效地并行执行，降低了线程上下文切换的开销，极大的增强了Java语言对高并发的实现。  

### 平台线程 与 虚拟线程  

| 属性 | 平台线程（Platform Thread） | 虚拟线程（Virtual Thread） |
| :--- | :--- | :--- |
| 级别 | 操作系统级别 | 用户空间级别 |
| 实现方式 | 由操作系统直接管理，一个平台线程对应一个OS线程 | 由JVM内部实现，在用户空间中运行，并可以在多个OS线程之间调度 |
| 创建成本 | 较高，涉及到操作系统的资源分配 | 较低，只需要很小的内存开销即可创建大量虚拟线程 |
| 上下文切换 | 成本较高，因为涉及到了硬件级别的上下文切换 | 成本较低，由于是在用户空间内进行，可以更高效地执行 |
| 可伸缩性 | 受到操作系统限制，数量有限 | 更好地支持高并发场景，可以创建大量虚拟线程来处理任务 |
| 适用场景 | 对性能要求极高且需要精确控制资源的应用程序 | 需要处理大量并发任务且对延迟不敏感的应用程序 |
  

### 创建虚拟线程
```java
public static void main(String[] args) {
    // 使用Thread.ofVirtual()创建虚拟线程
    Thread helloWorld = Thread.ofVirtual().start(() -> System.out.println("Thread.ofVirtual create"));
    // 使用静态构造器
    Thread startVirtualThread = Thread.startVirtualThread(() -> System.out.println("Thread.startVirtualThread create"));
    // 使用虚拟线程工程
    ThreadFactory factory = Thread.ofVirtual().name("Thread.ofVirtual", 0).factory();
    factory.newThread(() -> System.out.println("ThreadFactory create")).start();
    // 使用Executors创建虚拟线程
    ExecutorService executorService = Executors.newVirtualThreadPerTaskExecutor();
    executorService.submit(() -> System.out.println("executors create"));
}
```

### 虚拟线程的质变
```java
public class VirtualThreadApplication {
    private static final int echo = 100000;
    public static void main(String[] args) throws InterruptedException {
        long start = System.currentTimeMillis();
        Random random = new Random();
        Runnable runnable = () -> { double i = random.nextDouble(1000); };
        for (int i = 0; i < echo; i++){
            Thread t = new Thread(runnable);
            t.start();
        }
        long finish = System.currentTimeMillis();
        long timeElapsed = finish - start;
        System.out.println("Platform Thread Run time: " + timeElapsed);

        long timeMillis = System.currentTimeMillis();
        for (int i = 0; i < echo; i++){
            Thread startVirtualThread = Thread.startVirtualThread(runnable);
        }
        long endTimeMillis = System.currentTimeMillis();
        long endTimeElapsed = endTimeMillis - timeMillis;
        System.out.println("Virtual Thread Run time: " + endTimeElapsed);
    }
}
Platform Thread Run time: 25357
Virtual Thread Run time: 485
```

### 调度虚拟线程
![虚拟线程](/assets/image/VirtualThread.png)
为了完成有用的工作，线程需要被调度，也就是被分配到处理器核心上执行。对于作为操作系统线程实现的平台线程，JDK依赖于操作系统中的调度器。相比之下，对于虚拟线程，JDK有自己的调度器。不是直接将虚拟线程分配给处理器，而是由JDK的调度器将虚拟线程分配给平台线程（这是之前提到的虚拟线程的M:N调度）。然后，平台线程像往常一样由操作系统进行调度。

JDK的虚拟线程调度器是一个采用FIFO模式的操作窃取型ForkJoinPool。调度器的并行度是用于调度虚拟线程的平台线程数量。默认情况下，它等于可用处理器的数量，但可以通过系统属性jdk.virtualThreadScheduler.parallelism进行调整。这个ForkJoinPool与用于实现并行流等的通用池不同，通用池采用LIFO模式运行。

调度器分配虚拟线程的平台线程被称为虚拟线程的载体。在虚拟线程的生命周期中，它可以被调度到不同的载体上；换句话说，调度器不维护虚拟线程与任何特定平台线程之间的亲和性。从Java代码的角度来看，运行中的虚拟线程在逻辑上与其当前载体独立：

- 载体的身份对虚拟线程不可用。Thread.currentThread()返回的值始终是虚拟线程本身。

- 载体和虚拟线程的堆栈跟踪是分开的。在虚拟线程中抛出的异常不会包含载体的堆栈帧。线程转储不会在虚拟线程的堆栈中显示载体的堆栈帧，反之亦然。

- 载体的线程局部变量对虚拟线程不可用，反之亦然。

此外，从Java代码的角度来看，虚拟线程及其载体暂时共享一个操作系统线程的事实是不可见的。相比之下，从本机代码的角度来看，虚拟线程及其载体都在同一个本机线程上运行。因此，多次在同一虚拟线程上调用的本机代码可能会在每次调用时观察到不同的操作系统线程标识符。
调度器目前尚未为虚拟线程实现时间共享。时间共享是指强制抢占消耗了分配的CPU时间的线程。虽然在平台线程数量相对较少且CPU利用率达到100%时，时间共享可以有效地降低某些任务的延迟，但在存在百万个虚拟线程的情况下，时间共享是否同样有效尚不清楚。


### 参考文档：
+ [Java SE 21官网文档](https://docs.oracle.com/en/java/javase/21/core/virtual-threads.html#GUID-DC4306FC-D6C1-4BCC-AECE-48C32C1A8DAA){:target="_blank"}
+ [OpenJdk loom](https://github.com/openjdk/loom){:target="_blank"}