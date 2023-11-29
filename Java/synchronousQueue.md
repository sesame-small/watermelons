---
title: SynchronousQueue
layout: page
nav_order: 3
parent: Java
permalink: java/syncQueue
---

# {{ page.title }}  

---
> 一个一对一资源转移的阻塞队列，队列里不会存储资源，一个线程从队列中(poll/take)/(offer/put)资源，必须等待另一个线程进行与之相反的操作。该队列支持对等待生产者和消费者线程进行排序的可选公平策略。默认情况下，此排序不保证。但是，使用公平设置为 true 的队列构造函数将按 FIFO 顺序授予线程访问权限。
  

本文将一行一行代码分析SynchronousQueue的现实原理
## 代码结构
![syncQueue](/assets/image/syncQueue.png)
```java
public class SynchronousQueue<E> extends AbstractQueue<E> implements BlockingQueue<E>, Serializable {
    // 构造器 参数判断是否公平策略
    public SynchronousQueue() { this(false); }
    public SynchronousQueue(boolean fair) { transferer = fair ? new TransferQueue<E>() : new TransferStack<E>();}
    // 抽象内部类，提供一个抽象的资源转移方法，由具体子类实现
    abstract static class Transferer<E> {
        abstract E transfer(E e, boolean timed, long nanos);
    }
    // CPU的数量
    static final int NCPUS = Runtime.getRuntime().availableProcessors();
    // 最大的限时旋转次数 及 最大不限时旋转次数
    static final int maxTimedSpins = (NCPUS < 2) ? 0 : 32;
    static final int maxUntimedSpins = maxTimedSpins * 16;
    // 自旋超时阈值
    static final long spinForTimeoutThreshold = 1000L;
    // 非公平模式下的实现类转移堆栈
    static final class TransferStack<E> extends Transferer<E> {...}
    // 公平模式下的实现类转移队列
    static final class TransferQueue<E> extends Transferer<E> {...}
    // volatile修饰的transferer
    private transient volatile Transferer<E> transferer;
    // 对外封装的方法
    public boolean offer(E e, long timeout, TimeUnit unit){}
    public void put(E e){}
    public E take(){}
    public E poll(long timeout, TimeUnit unit){}
}
```

> **SynchronousQueue**没有使用synchronized、reentrantlock等锁，通过**CAS**与**自旋**实现线程安全，并且直接对线程进行**阻塞（park）与唤醒**。

## 公平模式下的转移队列
### 惯例上图：
![SyncQueue-transferQueue](/assets/image/SynchronousQueue-TransferQueue.png)
### 代码示例：
可以修改消费线程的sleep时长、生产线程offer的超时时间、消费线程poll的超时时间来观察转移队列一对一转移的效果。
```java
public class SynchronousQueueDemo {
    private static final ThreadPoolExecutor PRODUCT_THREAD = new ThreadPoolExecutor(2,12,30,TimeUnit.SECONDS, new ArrayBlockingQueue<>(10000), new NamedThreadFactory("拍卖线程"));
    private static final ThreadPoolExecutor CONSUMER_THREAD = new ThreadPoolExecutor(2,12,30,TimeUnit.SECONDS, new ArrayBlockingQueue<>(10000), new NamedThreadFactory("竞拍线程"));
    private static final Integer SIZE = 5;
    private static final String[] WP = {"《Java从入门到入院》", "《Python从入门到入院》", "《C++从入门到入院》", "《Go从入门到入院》", "《JS从入门到入院》", "《Php从入门到入院》"};
    public static void main(String[] args) throws InterruptedException {
        SynchronousQueue<String> synchronousQueue = new SynchronousQueue<>(true);
        for (int i = 0 ; i < SIZE; i ++) {
            int finalI = i;
            PRODUCT_THREAD.execute(() -> {
                try {
                    System.out.println(Thread.currentThread().getName() + "：开始拍卖物品：" + WP[finalI] + ":" + synchronousQueue.offer(WP[finalI], 5, TimeUnit.SECONDS));
                } catch (InterruptedException e) {
                }
            });
        }
        for (int i = 0 ; i < SIZE; i ++) {
            CONSUMER_THREAD.execute(() -> {
                try {
                    TimeUnit.SECONDS.sleep(1);
                    System.out.println(Thread.currentThread().getName() + "：竞拍到物品：" + synchronousQueue.poll(5, TimeUnit.SECONDS));
                } catch (InterruptedException e) {
                }
            });
        }
    }
}
```
### 源码解读
```java
static final class TransferQueue<E> extends Transferer<E> {
    static final class QNode {} // 队列节点信息
    transient volatile QNode head; // 头节点指针
    transient volatile QNode tail; // 尾节点指针
    transient volatile QNode cleanMe; 
    TransferQueue() {
        QNode h = new QNode(null, false); // initialize to dummy node.
        head = h;
        tail = h;
    }
}
```

## 非公平模式下的转移堆栈
```java
static final class TransferStack<E> extends Transferer<E> {
    static final class SNode {}
    volatile SNode head;
    static final int REQUEST    = 0;
    static final int DATA       = 1;
    static final int FULFILLING = 2;
}
```