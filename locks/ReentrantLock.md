### ReentrantLock
什么是可重入锁，顾名思义，就是支持重进入的锁，它表示该锁支持一个线程对资源的重复加锁。它是如何实现重入性的呢？
为什么在线程再次获取锁之后不会被锁阻塞的呢？
1) 锁需要去识别获取锁的线程是否为当前独占锁线程，如果是，则再次成功获取
2) 成功获取锁的线程再次获取锁，只是增加了同步状态值，在释放同步状态的时候减小同步状态值则表示当前锁的释放。
可重入锁又分为公平锁以及非公平锁，公平和非公平锁的选择可在创建可重入锁时传入boolean值。
#### 公平锁
每个锁的竞争机会是相同的，新来的线程需要排队，按照顺序等待队列的上一个线程释放锁之后获取锁。
```
 final void lock() {
            //排队获取锁
            acquire(1);
        }

```
#### 非公平锁
对于非公平锁来说，新来一个的线程可以在允许的时间内去抢占锁，如果抢占成功即获得锁，抢占不到则继续排序。
```
 final void lock() {
            //CAS交换如果成功则设置当前线程为独占锁线程，加锁成功
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
                //线程没有抢占到锁时，需要排队
                acquire(1);
        }
```
上述公平锁和非公平锁的方法中非公平锁比公平锁多一个抢占锁，没有成功的话调用了和公平锁一样都调用了acquire(1);这个方法，这个方法是AQS(AbstractQueuedSynchronizer)
中，
```
public final void acquire(int arg) {
        //尝试获取锁失败并且排队成功
        //如果是公平锁调用公平锁的tryAcquire方法
        //非公平锁调用非公平锁的tryAcquire方法
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            //中断当前线程
            selfInterrupt();
    }
```
那么线程是如何排队的呢?先来说说AQS，AQS它是什么?队列同步器，是用来构建锁或者其他同步组件的基础框架，它使用一个int成员变量state来表示同步状态，通过内置的
FIFO队列(一个FIFO双向队列)来完成资源获取线程的排队工作。当前线程获取锁失败之后时，AQS会将当前线程以及等待状态构造成一个节点(Node)并将其加入到同步队列中，同时会阻塞当前线程。
在jdk中的实现如下:
```
private Node addWaiter(Node mode) {
        //新建一个node
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        //获取尾部节点
        Node pred = tail;
        //如果尾部节点不为null
        if (pred != null) {
            //设置node的前驱节点为pred
            node.prev = pred;
            //交换tail节点为node
            if (compareAndSetTail(pred, node)) {
                //前尾部节点的后驱节点为node
                pred.next = node;
                return node;
            }
        }
        enq(node);
        return node;
    }
```
添加到等待队列之后,什么时候获取锁呢?当同步状态释放后，会把首节点中的线程唤醒，使其再次尝试获取同步状态。在jdk中的实现:
```
final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            //死循环自旋
            for (;;) {
                //获取当前节点的前驱节点
                final Node p = node.predecessor();
                //如果前驱节点是头节点并且尝试获取锁成功
                if (p == head && tryAcquire(arg)) {
                    //设置当前节点为头节点
                    setHead(node);
                    p.next = null;  // help GC
                    failed = false;
                    return interrupted;
                }
                //判断获取锁失败后是否可以park之后并且park成功
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt()) {
                    interrupted = true;
                }
            }
        } finally {
            //如果失败，则取消排队,删除该节点
            if (failed)
                cancelAcquire(node);
        }
    }
```
获取锁成功之后，如何释放锁呢？非公平锁的释放和公平锁的释放是否又一样呢？
在调用了unlock之后，会调用AQS的release方法，该方法执行时，先调用tryRelease()方法释放同步状态(state)成功之后，会唤醒头节点的后续节点线程。在JDK中的实现如下:
```
 public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            //头节点不为null并且等待状态不为0
            if (h != null && h.waitStatus != 0)
                //唤醒头节点
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```
ReentrantLock的tryRelease方法实现如下:
```
 protected final boolean tryRelease(int releases) {
            int c = getState() - releases;
            //判断当前线程是不是独占锁，不是则抛出异常
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            //如果是0的话表示锁已经被释放
            if (c == 0) {
                free = true;
                //设置当前独占锁为null
                setExclusiveOwnerThread(null);
            }
            //设置最新内存值
            setState(c);
            return free;
        }
```
如上为ReentrantLock的获取锁和释放锁的过程，ReentrantLock的默认实现为什么是非公平锁呢？
公平锁和非公平锁分别解决什么问题呢？什么时候使用公平锁？什么时候使用非公平锁呢？
公平锁保证了锁的获取遵循了FIFO原则，减少了线程饥饿发生的概率，等待越久的请求越是能够得到优先满足，而它的代价是进行大量的线程切换。
非公平锁虽然会造成线程长时间可能抢占不到锁，造成线程饥饿，但是它极少的线程切换，保证了其更大的吞吐量。