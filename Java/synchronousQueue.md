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
    static final class QNode {} // 节点信息
    transient volatile QNode head; // 头节点指针
    transient volatile QNode tail; // 尾节点指针
    transient volatile QNode cleanMe; // 对被取消节点的引用
    TransferQueue() { // 初始化一个空节点，头尾指针都指向此空节点
        QNode h = new QNode(null, false); 
        head = h;
        tail = h;
    }
}
```

```java
// 核心方法
E transfer(E e, boolean timed, long nanos) {
    QNode s = null; // 定义一个QNode，用于后面构造
    boolean isData = (e != null); // 判断是生产者还是消费者
    for (;;) { // 循环
        QNode t = tail;
        QNode h = head;
        if (t == null || h == null) {continue;} // 如果头或尾为空，则继续
        // 队列为空 或者 队列中线程与当前线程模式相同（队列中只存在一种模式的线程）
        if (h == t || t.isData == isData) { 
            QNode tn = t.next;
            if (t != tail) {continue;} // 如果头跟尾不相等，则有其他线程修改了连表，则继续循环              
            if (tn != null) { // 如果尾节点不为null，则有这个过程中有其他线程插入队列             
                advanceTail(t, tn); // 通过CAS尝试修改tail指向新的节点
                continue; // 继续循环
            }
            if (timed && nanos <= 0L) {return null;} // 如果非阻塞操作，说明没有可匹配的线程，则直接返回null，退出循环
            // 初始化一个新的节点，并通过CAS将tail的next节点设置成新加入的节点，如果失败，则说明tail.next!=null，有其他线程已经修改了tail.next属性，继续循环
            if (s == null) {s = new QNode(e, isData);}
            if (!t.casNext(null, s)) {continue;}
            // 通过CAS尝试修改tail指向新的节点
            advanceTail(t, s);
            // 自旋或者阻塞，直到传进来的数据不是当前数据（s.item != e）
            Object x = awaitFulfill(s, e, timed, nanos);
            // 请求被取消
            if (x == s) {   
                // 用cleanMe保存了t,等待别的线程clean()再根据情况清除                
                clean(t, s);
                return null; //返回
            }
            // 匹配成功 如果当前请求还在队列中
            if (!s.isOffList()) {       
                advanceHead(t, s);  // 更新头指针       
                if (x != null) {    
                    s.item = s; // 更新尾节点信息
                    s.waiter = null;
                }
            }
            //x != null: 消费者请求拿到了生产者的item,返回拿到的item，x == null: 生产者的item被拿走了,返回自己的item
            return (x != null) ? (E)x : e; 
        } else { // 队列不为空                       
            QNode m = h.next; // 获取头节点              
            if (t != tail || m == null || h != head) {continue;} // 头尾节点发生变化或者获取到的头结点尾空，则继续循环
            Object x = m.item; // 获取头节点的数据
            // 1.当前结点为生产者请求,isData = true,此时头结点为消费者请求,但已经取消了x != null
            // 2.当前结点为消费者请求,isData = false,此时头结点为生产者请求,但已经被匹配x == null
            // x == m 头结点已经取消	
            // 1.如果m为生产者请求 x == 生产的数据,当前消费者请求e == null,把生产者请求的item设置为null
            // 2.如果m为消费者请求 x == null,当前生产者者请求e == 生产的数据,把消费者请求的item设置为生产的数据
            if (isData == (x != null) || x == m || !m.casItem(x, e)) {
                advanceHead(h, m);
                continue;
            }
            // 更新头指针,唤醒匹配结点的线程
            advanceHead(h, m);              
            LockSupport.unpark(m.waiter);
            //x != null: 消费者请求拿到了生产者的item,返回拿到的item，x == null: 生产者的item被拿走了,返回自己的item
            return (x != null) ? (E)x : e;
        }
    }
}
```

```java
Object awaitFulfill(QNode s, E e, boolean timed, long nanos) {
    // 截止时间
    final long deadline = timed ? System.nanoTime() + nanos : 0L;
    Thread w = Thread.currentThread(); // 获取当前线程
    // 获取自旋次数，只有头节点才需要自旋，因为FIFO
    int spins = (head.next == s) ? (timed ? MAX_TIMED_SPINS : MAX_UNTIMED_SPINS) : 0;
    for (;;) {
        if (w.isInterrupted()) { // 当前线程被中断
            s.tryCancel(e); // 取消该节点请求
        }
        Object x = s.item; 
        // s当前是e包装的一个尾节点，当前请求匹配成功（FIFO，头节点匹配成功则尾节点的指针则会发生变化s.item != e）或者请求被取消的时候，则返回尾节点的信息
        if (x != e) {return x;}
        // 是否配置超时
        if (timed) {
            // 计算时间，达到超时时间则取消该节点的请求，取消的时候x 及s.item为null
            nanos = deadline - System.nanoTime();
            if (nanos <= 0L) {
                s.tryCancel(e);
                continue;
            }
        }
        // 自旋次数，继续自旋
        if (spins > 0) {
            --spins;
            Thread.onSpinWait();
        } else if (s.waiter == null) { // 没有自旋次数，如果尾节点的等待线程尾null，则设置尾节点的等待线程为当前线程
            s.waiter = w;
        } else if (!timed) {
            LockSupport.park(this); // 挂起当前线程
        } else if (nanos > SPIN_FOR_TIMEOUT_THRESHOLD) {
            LockSupport.parkNanos(this, nanos); // 根据指定时间挂起当前线程
        }
    }
}
```

## 非公平模式下的转移栈
```java
static final class TransferStack<E> extends Transferer<E> {
    static final class SNode {} // 节点信息
    volatile SNode head; // 头节点
    static final int REQUEST    = 0; // 消费模式
    static final int DATA       = 1; // 生产模式
    static final int FULFILLING = 2; // 当前节点请求匹配中
    static final class SNode {
        volatile SNode next;        // 栈内的下个节点
        volatile SNode match;       // 与当前节点匹配的节点
        volatile Thread waiter;     // 当前等待的线程
        Object item;                // 数据
        int mode;                   // 当前结点的模式:DATA/REQUEST/FULFILLING
    }
}
```

### 惯例上图：
![SyncQueue-transferStack](/assets/image/SynchronousQueue-TransferStack.png)

### 代码示例：
```java
public class SynchronousStackDemo {
    private static final ThreadPoolExecutor PRODUCT_THREAD = new ThreadPoolExecutor(5,12,30,TimeUnit.SECONDS, new ArrayBlockingQueue<>(10000), new NamedThreadFactory("拍卖线程"));
    private static final ThreadPoolExecutor CONSUMER_THREAD = new ThreadPoolExecutor(1,12,30,TimeUnit.SECONDS, new ArrayBlockingQueue<>(10000), new NamedThreadFactory("竞拍线程"));
    private static final Integer SIZE = 5;
    private static final String[] WP = {"《Java从入门到入院》", "《Python从入门到入院》", "《C++从入门到入院》", "《Go从入门到入院》", "《JS从入门到入院》", "《Php从入门到入院》"};

    public static void main(String[] args) throws InterruptedException {
        SynchronousQueue<String> synchronousQueue = new SynchronousQueue<>(false);
        for (int i = 0 ; i < SIZE; i ++) {
            int finalI = i;
            PRODUCT_THREAD.execute(() -> {
                try {
                    System.out.println(Thread.currentThread().getName() + "：开始拍卖物品：" + WP[finalI]);
                    synchronousQueue.offer(WP[finalI], 200, TimeUnit.SECONDS);
                } catch (InterruptedException e) {
                }
            });
        }
        TimeUnit.SECONDS.sleep(10);
        for (int i = 0 ; i < SIZE; i ++) {
            CONSUMER_THREAD.execute(() -> {
                try {
                    System.out.println(Thread.currentThread().getName() + "：竞拍到物品：" + synchronousQueue.poll(500, TimeUnit.SECONDS));
                } catch (InterruptedException e) {
                }
            });
        }
    }
}
```

> 拍卖线程-5：开始拍卖物品：《JS从入门到入院》
拍卖线程：开始拍卖物品：《Java从入门到入院》
拍卖线程-4：开始拍卖物品：《Go从入门到入院》
拍卖线程-3：开始拍卖物品：《C++从入门到入院》
拍卖线程-2：开始拍卖物品：《Python从入门到入院》
竞拍线程：竞拍到物品：《Python从入门到入院》
竞拍线程：竞拍到物品：《C++从入门到入院》
竞拍线程：竞拍到物品：《Go从入门到入院》
竞拍线程：竞拍到物品：《Java从入门到入院》
竞拍线程：竞拍到物品：《JS从入门到入院》