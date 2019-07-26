## Java 并发编程基础

### 线程简介

#### 什么是线程

现代操作系统调度的最小单元是线程，也叫轻量级进程，在一个进程里可以创建多个线程，这些线程都拥有各自的计数器、堆栈和局部变量等属性，并且能够访问共享的内存变量。处理器在这些线程上高速切换，让使用者感觉到这些线程在同时执行。

#### 为什么要使用多线程

- 更多的处理器核心
- 更快的响应时间
- 更好的编程模型

#### 线程优先级

线程优先级不能作为程序正确性的依赖，因为操作系统可以完全不用理会 Java 线程对于优先级的设定。

#### 线程的状态

- NEW。初始状态，线程被构建，但是还没有调用 start() 方法
- RUNNABLE。运行状态，Java 线程将操作系统中的就绪和运行两种状态笼统地称作 “运行中”
- BLOCKED。阻塞状态，表示线程阻塞于锁
- WAITING。等待状态，表示线程进入等待状态，进入该状态表示当前线程需要等待其他线程做出一些特定动作（通知或中断）
- TIME_WAITING。超时等待状态，该状态不同于 WAITING ，它是可以在指定的时间自行返回的
- TERMINATED。终止状态，表示当前线程已经执行完毕

#### Daemon 线程

Daemon 线程是一种支持型线程，因为它主要被用作线程中后台调度以及支持性工作。但是在 Java 虚拟机推出时 Daemon 线程中的 finally 块并不一定会执行，因此不能依靠 finally 块中的内容来确保执行关闭或清理资源的逻辑。

### 启动和终止线程

#### 构造线程

一个新构造的线程对象是由其 parent 线程来进行空间分配的，而 child 线程继承了 parent 是否为 Daemon ，优先级和加载资源的 contextClassLoader 以及可继承的 ThreadLocal，同时还会分配一个唯一的 ID 来标识这个 child 线程。

#### 启动线程

线程 start() 方法的含义是：当前线程（即 parent 线程）同步告知 Java 虚拟机，只要线程规划器空闲，应立即启动调用 start() 方法的线程。

#### 理解中断

中断表示一个运行中的线程是否被其他线程进行了中断操作。

线程通过 isInterrupted() 来进行判断是否被中断，可以调用静态方法 Thread.interrupted() 对当前线程的中断标识位来进行复位。如果该线程已处于终结状态，即使该线程被中断过，在调用该线程对象的 isInterrupted() 时依旧会返回 false 。

在抛出 InterruptedException 之前，Java 虚拟机会先将该线程的中断标识位清除，然后抛出 InterruptedException ，此时调用 isInterrupted() 方法将会返回 false 。

#### 安全地终止线程

通过标识位或者中断操作的方式能够使线程在终止时有机会去清理资源，因此这种终止线程的做法显得更加安全和优雅。

### 线程间通信

#### volatile 和 synchronized 关键字

volatile 保证所有线程对变量访问的可见性。

synchronized 保证了线程对变量访问的可见性和排他性。对于同步块的实现使用了 monitorenter 和 monitorexit 指令，而同步方法则是依靠方法修饰符上的 ACC_SYNCHRONIZED 来完成的。

#### 等待 / 通知机制

```java
public class WaitNofify {
    static boolean flag = true;
    static Object lock = new Object();
    
    public static void main(String[] args) throws Exception {
        Thread waitThread = new Thread(new Wait(), "WaitThread");
        waitThread.start();
        TimeUnit.SECONDS.sleep(1);
        Thread notifyThread = new Thread(new Notify(), "NotifyThread");
        notifyThread.start();
    }
    
    static class Wait implements Runnable {
        public void run {
            // 加锁，拥有lock的Monitor
            synchronized (lock) {
                // 当条件不满足时，继续wait，同时释放了lock的锁
                while (flag) {
                    try {
                        System.out.println(
                                Thread.currentThread()
                                + " flag is true. wait@ " 
                                + new SimpleDateFormat("HH:mm:ss").format(new Date()));
                        lock.wait();
                    } catch (InterruptedException e) {
                    }
                }
                // 当条件满足时，完成工作
                System.out.println(
                                Thread.currentThread()
                                + " flag is false. running@ " 
                                + new SimpleDateFormat("HH:mm:ss").format(new Date()));
            }
        }
    }
    
    static class Notify implements Runnable {
        public void run() {
            // 加锁，拥有lock的Monitor
            synchronized (lock) {
                // 获取lock的锁，然后进行通知，通知时不会释放lock的锁
                // 直到当前线程释放了lock后，WaitThread才能从wait方法中返回
                System.out.println(
                                Thread.currentThread()
                                + " hold lock. notify@ " 
                                + new SimpleDateFormat("HH:mm:ss").format(new Date()));
                lock.notifyAll();
                flag = false;
                SleepUtils.second(5);
            }
            // 再次加锁
            synchronized (lock) {
                System.out.println(
                                Thread.currentThread()
                                + " hold lock again. sleep@ " 
                                + new SimpleDateFormat("HH:mm:ss").format(new Date()));
                SleepUtils.second(5);
            }
        }
    }
}
```

上述例子主要说明了调用 wait() 、notify() 和 notifyAll() 时需要注意的细节：

- 使用 wait() 、notify() 和 notifyAll() 时需要先对调用对象加锁
- 调用 wait() 方法后，线程状态由 RUNNING 变为 WAITING，并将当前线程放置到对象的等待队列
- notify() 或 notifyAll() 方法调用后，等待线程依旧不会从 wait() 返回，需要调用 notify() 或 notifyAll() 的线程释放锁之后，等待线程才有机会从 wait() 返回
- notify() 方法将等待队列中的一个等待线程移到同步队列中，而 notifyAll() 方法则是将等待队列中所有的线程全部移到同步队列中，被移动的线程状态由 WAITING 变为 BLOCKING
- 从 wait() 方法返回的前提是获得了调用对象的锁

#### 等待 / 通知的经典范式

等待方需遵循：

- 获取对象的锁
- 如果条件不满足，那么调用对象的 wait() 方法，被通知后仍要检查条件
- 条件满足则执行对应的逻辑

通知方需遵循：

- 获取对象的锁
- 改变条件
- 通知所有等待在对象上的线程

#### 管道输入 / 输出流

主要用于线程之间的数据传输，传输媒介为内存。

主要包括了 4 种具体实现：PipedOutputStream 、PipedInputStream 、PipedReader 和 PipedWriter ，前两种面向字节，后两种面向字符。

#### Thread.join() 的使用

如果一个线程 A 执行了 thread.join() 语句，其含义是：当前线程 A 等待 thread 线程终止之后才会从 thread.join() 返回。

#### ThreadLocal 的使用

以 ThreadLocal 对象为键、任意对象为值的存储结构。