---
title: SynchronousQueue
layout: page
nav_order: 3
parent: Java
permalink: java/syncQueue
---

# {{ page.title }}  

---
> 一个一对一资源交换的阻塞队列，队列里不会存储资源，一个线程从队列中poll/offer资源，必须等待另一个线程进行与之相反的操作。该队列支持对等待生产者和消费者线程进行排序的可选公平策略。默认情况下，此排序不保证。但是，使用公平设置为 true 的队列构造函数将按 FIFO 顺序授予线程访问权限。
  

本文将一行一行代码分析SynchronousQueue的现实原理
## 代码结构
![syncQueue](/assets/image/syncQueue.png)
```java
public class SynchronousQueue<E> extends AbstractQueue<E> implements BlockingQueue<E>, Serializable {
    public SynchronousQueue() { this(false); }
    public SynchronousQueue(boolean fair) { transferer = fair ? new TransferQueue<E>() : new TransferStack<E>();}
    abstract static class Transferer<E> {
        abstract E transfer(E e, boolean timed, long nanos);
    }
    static final int NCPUS = Runtime.getRuntime().availableProcessors();
    static final int maxTimedSpins = (NCPUS < 2) ? 0 : 32;
    static final int maxUntimedSpins = maxTimedSpins * 16;
    static final long spinForTimeoutThreshold = 1000L;
    static final class TransferStack<E> extends Transferer<E> {...}
    static final class TransferQueue<E> extends Transferer<E> {...}
    private transient volatile Transferer<E> transferer;
    static class WaitQueue implements Serializable {}
    static class LifoWaitQueue extends WaitQueue {}
    static class FifoWaitQueue extends WaitQueue {}
    private ReentrantLock qlock;
    private WaitQueue waitingProducers;
    private WaitQueue waitingConsumers;
}
```