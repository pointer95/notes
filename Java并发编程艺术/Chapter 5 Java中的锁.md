## Java 中的锁

### Lock 接口

Lock 接口提供的 synchronized 关键字不具备的主要特性：

- 尝试非阻塞地获取锁
- 能被中断地获取锁
- 超时获取锁

### 队列同步器

锁是面向使用者的，它定义了使用者与锁交互对的接口，隐藏了实现细节；同步器面向的是锁的实现者，它简化了锁的实现方式，屏蔽了同步状态管理、线程的排队、等待与唤醒等底层操作。

#### 队列同步器的接口与示例

重写同步器指定的方法时，需要使用同步器提供的如下 3 个方法来访问或修改同步状态：

- getState()
- setState()
- compareAndSetState(int expect, int update)

同步器提供的模板方法分为：独占式获取与释放同步状态、共享式获取与释放同步状态和查询同步队列中的等待线程情况。

```java
class Mutex implements Lock {
    // 静态内部类，自定义同步器
    private static class Sync extends AbstractQueuedSynchronizer {
        // 是否处于占用状态
        protected boolean isHeldExclusively() {
            return getState() == 1;
        }
        // 当状态为0的时候获取锁
        public boolean tryAcquire(int acquires) {
            if (compareAndSetState(0, 1)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }
        // 释放锁，将状态设置为0
        protected boolean tryRelease(int release) {
            if (getState() == 0) {
                throw new IllegalMonitorStateException();
            }
            setExclusiveOwnerThread(null);
            setState(0);
            return true;
        }
        // 返回一个 Condition ，每个 condition 都包含了一个 condition 队列
        Condition newCondition() {
            return new ConditionObject();
        }
    }
    // 仅需要将操作代理到 Sync 上即可
    private final Sync sync = new Sync();
    public void lock() {
        sync.acquire(1);
    }
    public tryLock() {
        return sync.tryAcquire(1);
    }
    public void unlock() {
        sync.release(1);
    }
    public Condition newCondition() {
        return sync.newCondition();
    }
    public boolean isLocked() {
        return sync.isHeldExclusively();
    }
    public boolean hasQueuedThreads() {
        return sync.hasQueuedThreads();
    }
    public void lockInterruptibly() throws InterruptedException {
        sync.acquireInterruptibly(1);
    }
    public boolean tryLock(long timeout, Timeunit unit) throws InterruptedException {
        return sync.tryAcquireNanos(1, unit.toNanos(timeout));
    }
}
```

#### 队列同步器的实现分析

- 同步队列

同步器依赖内部的同步队列（一个 FIFO 双向队列）来完成同步状态的管理，当前线程获取同步状态失败时，同步器会将当前线程以及等待状态等信息构造成为一个节点（Node）并将其加入同步队列，同时会阻塞当前线程，当同步状态释放时，会把首节点中的线程唤醒，使其在此尝试获取同步状态。

同步队列中的节点用来保存获取同步状态失败的线程引用、等待状态以及前驱节点和后继节点。

- int waitStatus 等待状态

    - CANCELLED，值为 1，取消等待

    - SIGNAL，值为 -1，当前节点的线程如果释放了同步状态或者被取消，将会通知后继节点
    - CONDITION，值为 -2，节点线程等待在 Condition 上，当其他线程对 Condition 调用了 signal() 方法后，该节点将会从等待队列中转移到同步队列中，加入到对同步状态的获取中
    - PROPAGATE，值为 -3，表示下一次共享式同步状态获取将会无条件地被传播下去
    - INITIAL，值为 0，初始状态

- Node prev 前驱节点

- Node next 后继节点

- Node nextWaiter 等待队列中的后继节点或是一个 SHARED 常量

- Thread thread 获取同步状态的线程

同步器拥有首节点（head）和尾节点（tail），没有成功获取同步状态的线程将会成为节点加入该队列的尾部。

节点加入到同步队列时的过程必须要保证线程安全，同步器提供了一个基于 CAS 的设置尾节点的方法：compareAndSetTail(Node expect, Node update) 。

首节点时获取同步状态成功的节点，首节点的线程在释放同步状态时，将会唤醒后继节点，而后继节点将会在获取同步状态成功时将自己设置成为首节点。

- 独占式同步状态与释放

通过调用同步器的 acquire(int arg) 方法可以获取同步状态。

```java
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```

首先调用自定义同步器实现的 tryAcquire(int arg) 方法，该方法保证线程安全的获取同步状态，如果同步状态获取失败，则构造同步节点（独占式 Node.EXCLUSIVE ）并通过 addWaiter(Node mode) 方法将该节点加入到同步队列的尾部，最后调用 acquireQueued(Node node, int arg) 方法，使得该节点以 “死循环” 的方式获取同步状态。如果获取不到则阻塞节点中的线程，而被阻塞线程的唤醒主要依靠前驱节点的出队或阻塞线程被中断来实现。

```java
    private Node addWaiter(Node mode) {
        Node node = new Node(mode);

        for (;;) {
            Node oldTail = tail;
            if (oldTail != null) {
                node.setPrevRelaxed(oldTail);
                if (compareAndSetTail(oldTail, node)) {
                    oldTail.next = node;
                    return node;
                }
            } else {
                initializeSyncQueue();
            }
        }
    }
```

节点进入同步队列后，就进入了一个自旋的过程。

```java
    final boolean acquireQueued(final Node node, int arg) {
        boolean interrupted = false;
        try {
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    return interrupted;
                }
                if (shouldParkAfterFailedAcquire(p, node))
                    interrupted |= parkAndCheckInterrupt();
            }
        } catch (Throwable t) {
            cancelAcquire(node);
            if (interrupted)
                selfInterrupt();
            throw t;
        }
    }
```

通过调用同步器的 release(int arg) 方法可以释放同步状态，该方法在释放了同步状态之后，会唤醒其后继节点。

```java
    public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```

- 共享式同步状态获取与释放

共享式获取可以在同一时刻有多个线程同时获取到同步状态。共享式的访问资源时，其他共享式的访问均被允许，而独占式访问被阻塞；独占式访问资源时，同一时刻其他访问均被阻塞。

通过调用同步器的 acquireShared(int arg) 方法可以共享式地获取同步状态。

```java
    public final void acquireShared(int arg) {
        if (tryAcquireShared(arg) < 0)
            doAcquireShared(arg);
    }

    private void doAcquireShared(int arg) {
        final Node node = addWaiter(Node.SHARED);
        boolean interrupted = false;
        try {
            for (;;) {
                final Node p = node.predecessor();
                if (p == head) {
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        return;
                    }
                }
                if (shouldParkAfterFailedAcquire(p, node))
                    interrupted |= parkAndCheckInterrupt();
            }
        } catch (Throwable t) {
            cancelAcquire(node);
            throw t;
        } finally {
            if (interrupted)
                selfInterrupt();
        }
    }
```

通过调用 releaseShared(int arg) 方法可以释放同步状态。

```java
    public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }

    private void doReleaseShared() {
        for (;;) {
            Node h = head;
            if (h != null && h != tail) {
                int ws = h.waitStatus;
                if (ws == Node.SIGNAL) {
                    if (!h.compareAndSetWaitStatus(Node.SIGNAL, 0))
                        continue;            // loop to recheck cases
                    unparkSuccessor(h);
                }
                else if (ws == 0 &&
                         !h.compareAndSetWaitStatus(0, Node.PROPAGATE))
                    continue;                // loop on failed CAS
            }
            if (h == head)                   // loop if head changed
                break;
        }
    }
```

- 独占式超时获取同步状态

- 自定义同步组件——TwinsLock

设计一个同步工具：该工具在同一时刻只允许至多两个线程同时访问，超过两个线程的访问将被阻塞。

```java
public class TwinsLock implements Lock {
    private final Sync sync = new Sync(2);
    private final static class Sync extends AbstractQueuedSynchronizer {
        Sync(int count) {
            if (count <= 0) {
                throw new IllegalArgumentException("count must large than zero.");
            }
            setState(count);
        }
        public int tryAcquireShared(int reduceCount) {
            for (;;) {
                int current = getState();
                int newCount = current - reduceCount;
                if (newCount < 0 || compareAndSetState(current, newCount)) {
                    return newCount;
                }
            }
        }
        public boolean tryReleaseShared(int returnCount) {
            for (;;) {
                int current = getState();
                int newCount = current + returnCount;
                if (compareAndSetState(current, newCount)) {
                    return true;
                }
            }
        }
    }
    public void lock() {
        sync.acquireShared(1);
    }
    public void unlock() {
        sync.releaseShared(1);
    }
}
```

### 重入锁

支持重进入的锁，表示该锁能够支持一个线程对资源的重复加锁。

#### 实现重进入

需解决：

- **线程再次获取锁**。锁需要去识别获取锁的线程是否为当前占据锁的线程，如果是，则再次成功获取。
- **锁的最终释放**。线程重复 n 次获取了锁，随后在第 n 次释放该锁后，其他线程能够获取到该锁。

```java
        final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }

        protected final boolean tryRelease(int releases) {
            int c = getState() - releases;
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
        }
```

成功获取锁的线程再次获取锁，只是增加了同步状态值，在释放同步状态时减少同步状态值。如果该锁被获取了 n 次，那么前 n-1 次释放锁必须返回 false 。

#### 公平与非公平获取锁的区别

```java
        protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```

hasQueuedPredecessors() 方法表示判断加入了同步队列中当前节点是否有前驱节点，如果有线程比当前线程更早地请求获取锁，需要等待前驱线程获取并释放锁之后才能继续获取锁。

公平性锁保证了锁的获取按照 FIFO 原则，而代价是进行了大量的线程切换。非公平性锁虽然可能造成线程饥饿，但极少的线程切换保证了更大的吞吐量。

### 读写锁

在读多于写的情况下，读写锁能够提供比排它锁更好的并发性和吞吐量。

ReentrantReadWriteLock 的特性：

- 支持非公平和公平的锁获取方式
- 支持重进入
- 支持锁降级

#### 读写锁的接口与示例

```java
public interface ReadWriteLock {
    Lock readLock();
    Lock writeLock();
}
```

ReetrantReadWriteLock 还提供了：

- int getReadLockCount() 返回当前读锁被获取的次数
- int getReadHoldCount() 返回当前线程获取锁的次数
- boolean isWriteLocked() 判断写锁是否被获取
- int getWriteHoldCount() 返回当前写锁被获取的次数

#### 读写锁的实现分析

- 读写状态的设计

读写锁将一个变量切分成了两个部分，高 16 位表示读，低 16 位表示写

- 写锁的获取与释放

如果当前线程已经获取了写锁，则增加写状态。如果当前线程在获取写锁时，读锁已经被获取或者该线程不是已经获取写锁的线程，则当前线程进入等待状态。

```java
        protected final boolean tryAcquire(int acquires) {
            Thread current = Thread.currentThread();
            int c = getState();
            int w = exclusiveCount(c);
            if (c != 0) {
                // (Note: if c != 0 and w == 0 then shared count != 0)
                if (w == 0 || current != getExclusiveOwnerThread())
                    return false;
                if (w + exclusiveCount(acquires) > MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                // Reentrant acquire
                setState(c + acquires);
                return true;
            }
            if (writerShouldBlock() ||
                !compareAndSetState(c, c + acquires))
                return false;
            setExclusiveOwnerThread(current);
            return true;
        }
```

该方法除了重入条件之外，增加了一个读锁是否存在的判断。如果读锁存在，则写锁不能被获取，因为读写锁要确保写锁的操作对读锁可见，只有等待其他读咸亨都释放了读锁，写锁才能被当前线程获取。

写锁释放类似 ReetrantLock 的释放过程。

- 读锁的获取与释放

在没有其他写线程访问时，读锁总会被成功地获取，而所做的也只是增加读状态。如果当前线程已经获取了读锁，则增加读状态。如果当前线程在读取读锁时，写锁已被其他线程获取，则进入等待状态。

读锁的每次释放均减少读状态，减少的值是（1<<16）。

- 锁降级

锁降级指的是写锁降级成为读锁，即把持住写锁，再获取到读锁，随后释放先前拥有的写锁的过程。

```java
public void processData() {
    readLock.lock();
    if (!update) {
        // 必须先释放读锁
        readLock.unlock();
        // 从获取到写锁开始
        writeLock.lock();
        try {
            if (!update) {
                update = true;
            }
            readLock.lock();
        } finally {
            writeLock.unlock();
        }
        // 锁降级完成
    }
    try {
        // 使用数据的流程
    } finally {
        readLock.unlock();
    }
}
```

### LockSupport 工具

LockSupport 定义了一组以 park 开头的方法用来阻塞当前咸亨，以及 unpark(Thread thread) 方法来唤醒一个被阻塞的线程。

### Condition 接口

```java
public interface Condition {
    void await() throws InterruptedException;
    void awaitUninterruptibly();
    long awaitNanos(long nanosTimeout) throws InterruptedException;
    boolean await(long time, TimeUnit unit) throws InterruptedException;
    boolean awaitUntil(Date deadline) throws InterruptedException;
    void signal();
    void signalAll();
}
```

一般会讲 Condition 对象作为成员变量。当调用 await() 方法后，当前线程会释放锁并在此等待，而其他线程调用 Condition 对象的 signal() 方法，通知当前线程后，当前线程才从 await() 方法返回，并且在返回前已经获取了锁。

#### Condition 的实现分析

- 等待队列

等待队列是一个 FIFO 的队列，在队列中的每个节点都包含了一个线程引用，该线程就是在 Condition 对象上等待的线程，如果一个线程调用了 Condition.await() 方法，那么该线程将会释放锁、构造成节点从尾部加入等待队列并进入等待状态。

节点的定义复用了同步器中节点的定义。Condition 拥有首节点（firstWaiter）和尾节点（lastWaiter）。

在并发包中的 Lock 拥有一个同步队列和多个等待队列。

- 等待

当调用 await() 方法时，相当于同步队列的首节点移动到了 Condition 的等待队列中。该方法会讲当前线程构造成节点并加入到等待队列中，然后释放同步状态，唤醒同步队列中的后继节点，然后当前线程会进入等待状态。

当等待队列中的节点被唤醒，则唤醒节点的线程开始尝试获取同步状态。如果不是通过其他线程调用 signal() 方法而是对等待线程进行中断，则会抛出 InterruptedException 。

```java
        public final void await() throws InterruptedException {
            if (Thread.interrupted())
                throw new InterruptedException();
            Node node = addConditionWaiter();
            int savedState = fullyRelease(node);
            int interruptMode = 0;
            while (!isOnSyncQueue(node)) {
                LockSupport.park(this);
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null) // clean up if cancelled
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
        }
```

- 通知

调用 Condition 的 signal() 方法，将会唤醒在等待队列中等待时间最长的节点，即首节点，在唤醒之前，会讲节点移到同步队列中。

```java
        public final void signal() {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            Node first = firstWaiter;
            if (first != null)
                doSignal(first);
        }

        private void doSignal(Node first) {
            do {
                if ( (firstWaiter = first.nextWaiter) == null)
                    lastWaiter = null;
                first.nextWaiter = null;
            } while (!transferForSignal(first) &&
                     (first = firstWaiter) != null);
        }
```

isHeldExclusively() 检查当前线程是否获取了锁的线程。

被唤醒后的线程，将从 await() 方法中的 while 循环中退出（isOnSyncQueue(Node node) 方法返回 true，节点已经在同步队列中），进而调用同步器的 acquireQueue() 方法加入到同步状态的竞争中。

成功获取同步状态之后，被唤醒的线程将从先前调用的 await() 方法返回。

Condition 的 signal() 方法，相当于对等待队列中的每个节点均执行一次 signal() 方法，效果就是将等待队列中所有节点全部移动到同步队列中，并唤醒每个节点的线程。