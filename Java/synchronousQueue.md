---
title: SynchronousQueue
layout: page
nav_order: 3
parent: Java
permalink: java/syncQueue
---

# {{ page.title }}  

---
> 一个一对一资源转移的阻塞队列，队列里不会存储资源，一个线程从队列中poll/offer资源，必须等待另一个线程进行与之相反的操作。该队列支持对等待生产者和消费者线程进行排序的可选公平策略。默认情况下，此排序不保证。但是，使用公平设置为 true 的队列构造函数将按 FIFO 顺序授予线程访问权限。
  

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
    // 非公平模式下的实现类转移栈
    static final class TransferStack<E> extends Transferer<E> {...}
    // 公平模式下的实现类转移队列
    static final class TransferQueue<E> extends Transferer<E> {...}
    // volatile修饰的transferer
    private transient volatile Transferer<E> transferer;
}
```

## 公平模式下的转移队列
```java
static final class TransferQueue<E> extends Transferer<E> {
    static final class QNode {
        volatile QNode next;          // next node in queue
        volatile Object item;         // CAS'ed to or from null
        volatile Thread waiter;       // to control park/unpark
        final boolean isData;
        QNode(Object item, boolean isData) {
            this.item = item;
            this.isData = isData;
        }
        boolean casNext(QNode cmp, QNode val) {
            return next == cmp && QNEXT.compareAndSet(this, cmp, val);
        }
        boolean casItem(Object cmp, Object val) {
            return item == cmp && QITEM.compareAndSet(this, cmp, val);
        }
        void tryCancel(Object cmp) {
            QITEM.compareAndSet(this, cmp, this);
        }
        boolean isCancelled() {
            return item == this;
        }
        boolean isOffList() {
            return next == this;
        }
        private static final VarHandle QITEM;
        private static final VarHandle QNEXT;
        static {
            try {
                MethodHandles.Lookup l = MethodHandles.lookup();
                QITEM = l.findVarHandle(QNode.class, "item", Object.class);
                QNEXT = l.findVarHandle(QNode.class, "next", QNode.class);
            } catch (ReflectiveOperationException e) {
                throw new ExceptionInInitializerError(e);
            }
        }
    }

    transient volatile QNode head;
    transient volatile QNode tail;
    transient volatile QNode cleanMe;
    TransferQueue() {
        QNode h = new QNode(null, false); // initialize to dummy node.
        head = h;
        tail = h;
    }
    void advanceHead(QNode h, QNode nh) {
        if (h == head && QHEAD.compareAndSet(this, h, nh))
            h.next = h; // forget old next
    }
    void advanceTail(QNode t, QNode nt) {
        if (tail == t) {
            QTAIL.compareAndSet(this, t, nt);
        }
    }
    boolean casCleanMe(QNode cmp, QNode val) {
        return cleanMe == cmp && QCLEANME.compareAndSet(this, cmp, val);
    }
    E transfer(E e, boolean timed, long nanos) {
        QNode s = null; // constructed/reused as needed
        boolean isData = (e != null);
        for (;;) {
            QNode t = tail;
            QNode h = head;
            if (t == null || h == null)         // saw uninitialized value
                continue;                       // spin
            if (h == t || t.isData == isData) { // empty or same-mode
                QNode tn = t.next;
                if (t != tail)                  // inconsistent read
                    continue;
                if (tn != null) {               // lagging tail
                    advanceTail(t, tn);
                    continue;
                }
                if (timed && nanos <= 0L)       // can't wait
                    return null;
                if (s == null)
                    s = new QNode(e, isData);
                if (!t.casNext(null, s))        // failed to link in
                    continue;

                advanceTail(t, s);              // swing tail and wait
                Object x = awaitFulfill(s, e, timed, nanos);
                if (x == s) {                   // wait was cancelled
                    clean(t, s);
                    return null;
                }
                if (!s.isOffList()) {           // not already unlinked
                    advanceHead(t, s);          // unlink if head
                    if (x != null)              // and forget fields
                        s.item = s;
                        s.waiter = null;
                    }
                return (x != null) ? (E)x : e;
            } else {                            // complementary-mode
                QNode m = h.next;               // node to fulfill
                if (t != tail || m == null || h != head)
                    continue;                   // inconsistent read
                Object x = m.item;
                if (isData == (x != null) || x == m || !m.casItem(x, e)) {         // lost CAS
                    advanceHead(h, m);          // dequeue and retry
                    continue;
                }
                advanceHead(h, m);              // successfully fulfilled
                LockSupport.unpark(m.waiter);
                return (x != null) ? (E)x : e;
            }
        }
    }

    Object awaitFulfill(QNode s, E e, boolean timed, long nanos) {
            /* Same idea as TransferStack.awaitFulfill */
        final long deadline = timed ? System.nanoTime() + nanos : 0L;
        Thread w = Thread.currentThread();
        int spins = (head.next == s) ? (timed ? MAX_TIMED_SPINS : MAX_UNTIMED_SPINS) : 0;
        for (;;) {
            if (w.isInterrupted())
                s.tryCancel(e);
            Object x = s.item;
            if (x != e)
                return x;
            if (timed) {
                nanos = deadline - System.nanoTime();
                if (nanos <= 0L) {
                    s.tryCancel(e);
                    continue;
                }
            }
            if (spins > 0) {
                --spins;
                Thread.onSpinWait();
            } else if (s.waiter == null) {
                    s.waiter = w;
            } else if (!timed) {
                    LockSupport.park(this);
            } else if (nanos > SPIN_FOR_TIMEOUT_THRESHOLD) {
                LockSupport.parkNanos(this, nanos);
            }
        }

        void clean(QNode pred, QNode s) {
            s.waiter = null; // forget thread
            while (pred.next == s) { // Return early if already unlinked
                QNode h = head;
                QNode hn = h.next;   // Absorb cancelled first node as head
                if (hn != null && hn.isCancelled()) {
                    advanceHead(h, hn);
                    continue;
                }
                QNode t = tail;      // Ensure consistent read for tail
                if (t == h)
                    return;
                QNode tn = t.next;
                if (t != tail)
                    continue;
                if (tn != null) {
                    advanceTail(t, tn);
                    continue;
                }
                if (s != t) {        // If not tail, try to unsplice
                    QNode sn = s.next;
                    if (sn == s || pred.casNext(s, sn))
                        return;
                }
                QNode dp = cleanMe;
                if (dp != null) {    // Try unlinking previous cancelled node
                    QNode d = dp.next;
                    QNode dn;
                    if (d == null ||               // d is gone or
                        d == dp ||                 // d is off list or
                        !d.isCancelled() ||        // d not cancelled or
                        (d != t &&                 // d not tail and
                         (dn = d.next) != null &&  //   has successor
                         dn != d &&                //   that is on list
                         dp.casNext(d, dn)))       // d unspliced
                        casCleanMe(dp, null);
                    if (dp == pred)
                        return;      // s is already saved node
                } else if (casCleanMe(null, pred))
                    return;          // Postpone cleaning s
            }
        }

        // VarHandle mechanics
        private static final VarHandle QHEAD;
        private static final VarHandle QTAIL;
        private static final VarHandle QCLEANME;
        static {
            try {
                MethodHandles.Lookup l = MethodHandles.lookup();
                QHEAD = l.findVarHandle(TransferQueue.class, "head",
                                        QNode.class);
                QTAIL = l.findVarHandle(TransferQueue.class, "tail",
                                        QNode.class);
                QCLEANME = l.findVarHandle(TransferQueue.class, "cleanMe",
                                           QNode.class);
            } catch (ReflectiveOperationException e) {
                throw new ExceptionInInitializerError(e);
            }
        }
    }
}

```