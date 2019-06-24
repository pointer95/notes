## 线程的基本机制 

### 定义任务 

实现 Runnable 接口并编写 run() 方法。 

run() 被写成无限循环的形式，除非有某个条件使得 run() 终止，否则它将永远运行下去。 

在 run() 中对静态方法 Thread.yield() 的调用是对 *线程调度器* 的一种建议，声明执行完生命周期中最重要的部分，可以切换给其它线程来执行。 

```java
// 显示发射之前的倒计时
// Demonstration of the Runnable interface.
public class LiftOff implements Runnable {

    protected int countDown = 10;
    private static int taskCount = 0;
    private final int id = taskCount ++;
    
    public LiftOff() {
    }
    
    public LiftOff(int countDown) {
        this.countDown = countDown;
    }
    
    public String status() {
        return "#" + id + "(" +
          (countDown > 0 ? countDown : "LiftOff!") + "), ";
    }
    
    public void run() {
        while(countDown-- > 0) {
            System.out.print(status());
            Thread.yield();
        }
    }
}

public class MainThread {
    public static void main(String[] args) {
        LiftOff launch = new LiftOff();
        launch.run();
    }
}

/* Output:
#0(9), #0(8), #0(7), #0(6), #0(5), #0(4), #0(3), #0(2), #0(1), #0(LiftOff!), 
*/
```

### Thread 类 

将 Runnable 对象转变为工作任务的传统方式是把它提交给一个 Thread 构造器。 

Thread 构造器只需要一个 Runnable 对象。调用 Thread 对象的 start() 方法为该线程执行必需的初始化操作，然后调用 Runnable 的 run() 方法，以便在这个新线程中启动该任务。 

```java
// Adding more threads.
public class MoreBasicThreads {
    public static void main(String[] args) {
        for (int i = 0; i < 5; i ++) {
            new Thread(new LiftOff()).start();
        }
        System.out.println("Wating for LiftOff");
    }
}
```

### 使用Executor

管理多个异步任务的执行，而无需程序员显式地管理线程的生命周期。 

- CachedThreadPool 为每个任务都创建都创建一个线程。 

- FixedThreadPool 可以一次性预先执行代价高昂的线程分配，因而也就可以限制线程的数量了。 

- SingleThreadExecutor 相当于线程数量为1的 FixedThreadPool. 

```java
public class CachedThreadPool {
    public static void main(String[] args) {
        ExecutorService exec = Executors.newCachedThreadPool();
        for (int i = 0; i < 5; i ++) {
            exec.execute(new LiftOff());
        }
        exec.shutdown();
    }
}
```

```java
public class FixedThreadPool {
    public static void main(String[] args) {
        ExecutorService exec = Executors.newFixedThreadPool(5);
        for (int i = 0; i < 5; i ++) {
            exec.execute(new LiftOff());
        }
        exec.shutdown();
    }
}
```

```java
public class SingleThreadExecutor {
    public static void main(String[] args) {
        ExecutorService exec = Executors.newSingleThreadExecutor();
        for (int i = 0; i < 5; i ++) {
            exec.execute(new LiftOff());
        }
        exec.shutdown();
    }
}
```

### 从任务中产生返回值

实现 Callable 接口可以在任务完成时能够返回一个值。

```java
class TaskWithResult implements Callable<String> {
    
    private int id;
    
    public TaskWithResult(int id) {
        this.id = id;
    }
    
    public String call() {
        return "result of TaskWithResult " + id;
    }
}

public class CallableDemo {
    public static void main(String[] args) {
        ExecutorService exec = Executors.newCacherdThreadPool();
        ArrayList<Future<String>> results = new ArrayList<Future<String>>();
        for (int i = 0; i < 10; i ++) {
            results.add(exec.submit(new TaskWithResult(i)));
        }
        for (Future<String> fs : results) {
            try {
                // get() blocks until completion:
                System.out.println(fs.get());
            } catch(InterruptedException e) {
                System.out.println(e);
                return;
            } catch(ExecutionException e) {
                System.out.println(e);
            } finally {
                exec.shutdown();
            }
        }
    }
}

/* Output:
result of TaskWithResult 0
result of TaskWithResult 1
result of TaskWithResult 2
result of TaskWithResult 3
result of TaskWithResult 4
result of TaskWithResult 5
result of TaskWithResult 6
result of TaskWithResult 7
result of TaskWithResult 8
result of TaskWithResult 9
*/
```

### 休眠

调用 sleep() 使任务中止执行给定的时间。

对 sleep() 的调用可能抛出 InterruptedException 异常，因为异常不能跨线程传播回 main()，所以必须在本地处理所有在任务内部产生的异常。

### 优先级

在绝大多数时间里，所有线程都应该以默认的优先级运行。试图操纵线程优先级通常是一种错误。

getPriority() 来读取现有线程的优先级，并且可以通过 setPriority() 在任何时刻修改它。

### 让步

上文提到的 yield() 。

### 后台线程

后台（daemon）线程是指在程序运行的时候在后台提供一种通用服务的线程，不属于程序中不可或缺的部分。

当所有非守护线程结束时，程序也就终止，同时会杀死所有守护线程。

main() 属于非守护线程。

```java
// Daemon threads don't prevent the program from ending.
public class SimpleDaemons implements Runnable {
    public void run() {
        try {
            while (true) {
                TimeUtil.MILLSECONDS.sleep(100);
                System.out.println(Thread.currentThread() + " " + this);
            }
        } catch(InterruptedException e) {
            System.out.println("sleep() interrupted");
        }
    }
    
    public static void main(String[] args) throws Exception {
        for (int i = 0; i < 10; i ++) {
            Thread daemon = new Thread(new SimpleDaemons());
            daemon.setDaemon(true);
            daemon.start();
        }
        System.out.println("All Daemons started");
        TimeUtil.MILLSECONDS.sleep(175);
    }
}

/* Output: (Sample)
All daemons started
Thread[Thread-0,5,main] SimpleDaemons@530daa
...
Thread[Thread-10,5,main] SimpleDaemons@83cc67
*/
```



通过编写定制的 ThreadFactory 可以定制由 Executor 创建的线程的属性（后台、优先级、名称）。然后用一个新的 DaemonThreadFactory 作为参数传递给 Executor.newCachedThreadPool() 。

```java
public class DaemonThreadFactory implements ThreadFactory {
    public Thread newThread(Runnable r) {
        Thread t = new Thread(r);
        t.setDaemon(true);
        return t;
    }
}
```



可以通过调用 isDaemon() 方法来确定线程是否是一个后台线程。如果是一个后台线程，那么它创建的任何线程将被自动设置成后台线程。

### 编码的变体

继承 Thread。

实现 Runnable 接口可以继承另一个不同的类，而从 Thread 继承将不行。

```java
// Inheriting directly from the Thread class.
public class SimpleThread extends Thread {
    
    private int countDown = 5;
    private static int threadCount = 0;
    
    public SimpleThread() {
        // store the thread name
        super(Interger.toString(++ threadCount));
        start();
    }
    
    public String toString() {
        return "#" + getName() + "(" + countDown() + "), ";
    }
    
    public void run() {
        while(true) {
            System.out.print(this);
            if (-- countDown == 0) {
                return;
            }
        }
    }
    
    public static void main(String[] args) {
        for (int i = 0; i < 5; i ++) {
            new SimpleThread();
        }
    }
}
```

### 加入一个线程

一个线程在其他线程之上调用 join() 方法，其效果是等待一段时间直到第二个线程结束才继续执行。

```java
// Understanding join() .
class Sleeper extends Thread {
    
    private int duration;
    
    public Sleeper(String name, int sleepTime) {
        super(name);
        duration = sleepTime;
        start();
    }
    
    public void run() {
        try {
            sleep(duration);
        } catch(InterruptedException e) {
            System.out.println(getName() + " was interrupted. " +
                               "isInterrupted(): " + isInterrupted());
            return;
        }
        System.out.println(getName() + "has awakened");
    }    
}

class Joiner extends Thread {
    
    private Sleeper sleeper;
    
    public Joiner(String name, Sleeper sleeper) {
        super(name);
        this.sleeper = sleeper;
        start();
    }
    
    public void run() {
        try {
            sleeper.join();
        } catch(InterruptedException e) {
            System.out.print("Interruped");
        }
        System.out.println(getName() + " join completed");
    }
    
    public class Joining {
        public static void main(String[] args) {
            Sleeper 
                sleepy = new Sleeper("Sleepy", 1500),
            	grumpy = new Sleeper("Grumpy", 1500);
            Joiner
                dopey = new Joiner("Dopey", sleepy),
            	doc = new Joiner("Doc", grumpy);
            grumpy.interrupt();
        }
    }
}

/* Output:
Grumpy was interrupted. isInterrupted(): false
Doc join completed
Sleepy has awakened
Doepy join completed
*/
```

在 catch 子句中，根据 isInterrupted() 的返回值报告这个中断，然而异常被捕获时将清理这个标志。所以在 catch 子句中异常被捕获的时候这个标志总是为 false 。

## 共享受限资源

### 解决共享资源竞争

- Synchronized
- ReentrantLock

### 原子性与易变性

*原子操作*是不能被线程调度机制中断的操作；一旦操作开始，那么它一定可以在可能发生的"上下文切换"之前执行完毕。

原子性可应用于除 long 和 double 之外的所有基本类型。

定义 long 和 double 变量时使用 **volatile** 关键字就会获得原子性。

volatile 还确保了应用中的可视性。将一个域声明为 volatile 的，那么只要对这个域产生了写操作，那么所有的读操作就都可以看到这个修改。即使使用了本地缓存，情况也是如此，volatile域会立即被写入到主存中，而读取操作就发生在主存中。

使用 volatile 而不是 synchronized 的唯一安全的情况时类中只有一个可变的域。

### 原子类

Atomic类

### 临界区

只是希望防止多个线程同时访问方法内部的部分代码而不是防止访问整个方法，通过这种方式分离出来的代码段被称为*临界区*。

### 线程本地存储

线程本地存储可以为使用相同变量的每个不同的线程都创建不同的存储。

```java
// Automatically giving each thread its own storage.
class Accessor implements Runnable {
    
    private final int id;
    
    public Accessor(int idn) {
        id = idn;
    }
    
    public void run() {
        while(!Thread.currentThread().isInterruped()) {
            ThreadLocalVariableHolder.increment();
            System.out.println(this);
            Thread.yield();
        }
    }
    
    public String toString() {
        return "#" + id + ": " + ThreadLocalVariableHolder.get();
    }
}

public class ThreadLocalVariableHolder {
    private static ThreadLocal<Interger> value = new ThreadLocal<Interger>() {
        private Random rand = new Random(47);
        protected synchronized Interger initialValue() {
            return rand.nextInt(10000);
        }
    };
    
    public static void increment() {
        value.set(value.get() + 1);
    }
    
    public static int get() {
        return value.get();
    }
    
    public static void main(String[] args) throws Exception {
        ExecutorService exec = Executors.newCachedThreadPool();
        for (int i = 0; i < 5; i ++) {
            exec.execute(new Accessor(i));
        }
        TimeUnit.SECONDS.sleep(3);
        exec.shutdown();
    }
}

/* Output: (Sample)
#0: 9259
#1: 556
#2: 6694
#3: 1862
#4: 962
#0: 9260
#1: 557
#2: 6695
#3: 1863
#4: 963
*/
```

## 终结任务

### 在阻塞时终结

线程状态：

- 新建
- 就绪
- 阻塞
- 死亡

进入阻塞状态原因：

- 调用 sleep(milliseconds) 使任务进入休眠状态。在这种情况下，任务在指定的时间内不会运行。
- 调用 wait() 使线程挂起。直到线程得到了 notify() 和 notifyAll() 消息，线程才会进入就绪状态。
- 任务在等待某个输入/输出完成。
- 任务试图在某个对象上调用其同步控制方法，但是对象锁不可用，因为另一个任务已经获取了这个锁。

suspend() 和 resume() 用来阻塞和唤醒线程，现在已不再使用，因为可能会导致死锁。stop() 也不再使用，因为它不释放线程获得的锁，并且如果线程处于不一致的状态（受损状态），其他任务可以在这种状态下浏览并修改它们。

### 中断

调用 Thread 类的 interrupt() 方法。如果一个线程已经被阻塞，或者试图执行一个阻塞操作，那么设置这个线程的中断状态将抛出 InterruptedException。

在 Executor 上调用 shutdownNow() ，那么它将发送一个interrupt() 调用给它启动的所有线程。通过调用 submit() 来启动任务，就可以持有该任务的上下文。submit() 将返回一个范型 Future<?>，通过调用该对象的 cancel() 来中断某个特定任务。

能够中断对 sleep() 的调用，但不能中断正在试图获取 synchronized 锁或者试图执行I/O操作的线程。

## 线程之间的协作

### wait() 与 notifyAll()

有两种形式的 wait() 。第一种接受毫秒数作为参数，可以通过 notify()、notifyAll()，或者令时间到期，从 wait() 中恢复执行。第二种 wait() 不接受任何参数，这种 wait() 将无限等待下去，直到线程接收到notify() 或者notifyAll() 消息。

这些方法是基类 Object 的一部分，而不是属于 Thread 的一部分。

只能在同步控制方法里调用这些方法，否则程序在运行的时候会抛出 IllegalMonitorStateException 异常。

当一个任务在方法里遇到了对 wait() 的调用的时候，线程的执行被挂起，对象的上的锁被释放。

```java
// Basic task cooperation.
class Car {

    private boolean waxOn = false;

    public synchronized void waxed() {
        waxOn = true; // Ready to buff
        notifyAll();
    }

    public synchronized void buffed() {
        waxOn = false; // Ready to another coat of wax
        notifyAll();
    }

    public synchronized void waitForWaxing() throws InterruptedException {
        while (!waxOn) {
            wait();
        }
    }

    public synchronized void waitForBuffing() throws InterruptedException {
        while (waxOn) {
            wait();
        }
    }
}

public class WaxOMatic {

    public static void main(String[] args) throws Exception {
        Car car = new Car();
        ExecutorService exec = Executors.newCachedThreadPool();
        exec.execute(() -> {
            try {
                while (!Thread.interrupted()) {
                    car.waitForWaxing();
                    System.out.print("Wax Off! ");
                    TimeUnit.MILLISECONDS.sleep(200);
                    car.buffed();
                }
            } catch (InterruptedException e) {
                System.out.println("Exiting via interrupt");
            }
            System.out.println("Ending Wax On Task");
        });
        exec.execute(() -> {
            try {
                while (!Thread.interrupted()) {
                    System.out.print("Wax On! ");
                    TimeUnit.MILLISECONDS.sleep(200);
                    car.waxed();
                    car.waitForBuffing();
                }
            } catch (InterruptedException e) {
                System.out.println("Exiting via interrupt");
            }
            System.out.println("Ending Wax On Task");
        });
        TimeUnit.SECONDS.sleep(5);
        exec.shutdownNow();
    }
}

/* Output: (95% match)
Wax On! Wax Off! Wax On! Wax Off! Wax On! Wax Off! Wax On! Wax Off! Wax On! Wax Off! Wax On! Wax Off! Wax On! Wax Off! Wax On! Wax Off! Wax On! Wax Off! Wax On! Wax Off! Wax On! Wax Off! Wax On! Wax Off! Wax On! Exiting via interrupt
Ending Wax On Task
Exiting via interrupt
Ending Wax On Task
*/
```



如果一个线程 T1 先执行 notify() ，另一个线程 T2 再进入 wait() ，此时 notify() 将错失，而 T2 也将无限地等待这个已经发送过的信号，从而产生死锁。

### notify() 与 notifyAll()

在技术上，可能会有多个任务在单个对象上处于 wait() 状态，因此调用 notifyAll() 比只调用 notify() 要更安全。

使用 notify() 时，在众多等待同一个锁的任务中只有一个会被唤醒，因此如果使用 notify() ，就必须保证被唤醒的是恰当的任务，并且所有任务必须等待相同的条件。当条件发生变化时，必须只有一个任务能够从中受益。最后这些限制对所有可能存在的子类都必须总是起作用的。

### 使用显式的 Lock 和 Condition 对象

juc 类库中提供了使用互斥并允许任务挂起的 Condition 类，可以在 Condition 上调用 await() 方法来挂起一个任务，其它线程调用 signal() 或 signalAll() 方法唤醒挂起的任务。

```java
// Using Lock and Condition objects.
class Car {

    private Lock lock = new ReentrantLock();
    private Condition condition = lock.newCondition();
    private boolean waxOn = false;

    public void waxed() {
        lock.lock();
        try {
            waxOn = true;
            condition.signalAll();
        } finally {
            lock.unlock();
        }
    }

    public void buffed() {
        lock.lock();
        try {
            waxOn = false;
            condition.signalAll();
        } finally {
            lock.unlock();
        }
    }

    public void waitForWaxing() throws InterruptedException {
        lock.lock();
        try {
            while (!waxOn) {
                condition.await();
            }
        } finally {
            lock.unlock();
        }
    }

    public void waitForBuffing() throws InterruptedException {
        lock.lock();
        try {
            while (waxOn) {
                condition.await();
            }
        } finally {
            lock.unlock();
        }
    }
}

public class WaxOMatic2 {
    public static void main(String[] args) throws Exception {
        Car car = new Car();
        ExecutorService exec = Executors.newCachedThreadPool();
        exec.execute(() -> {
            try {
                while (!Thread.interrupted()) {
                    car.waitForWaxing();
                    System.out.print("Wax Off! ");
                    TimeUnit.MILLISECONDS.sleep(200);
                    car.buffed();
                }
            } catch (InterruptedException e) {
                System.out.println("Exiting via interrupt");
            }
            System.out.println("Ending Wax On Task");
        });
        exec.execute(() -> {
            try {
                while (!Thread.interrupted()) {
                    System.out.print("Wax On! ");
                    TimeUnit.MILLISECONDS.sleep(200);
                    car.waxed();
                    car.waitForBuffing();
                }
            } catch (InterruptedException e) {
                System.out.println("Exiting via interrupt");
            }
            System.out.println("Ending Wax On Task");
        });
        TimeUnit.SECONDS.sleep(5);
        exec.shutdownNow();
    }
}
```

### 生产者-消费者队列

java.util.concurrent.BlockingQueue 接口中提供了同步队列来解决任务协作问题。

- LinkedBlockingQueue 无界队列
- ArrayBlockingQueue 固定尺寸

如果消费者任务试图从队列中获取对象，而该队列此时为空，那么这些队列还可以挂起消费者任务，并且当有更多的元素可用时恢复消费者队列。

```java
// RunByHand
class LiftOffRunner implements Runnable {
    
    private BlockingQueue<LiftOff> rockets;
    
    public LiftOffRunner(BlockingQueue<LiftOff> queue) {
        rockets = queue;
    }
    
    public void add(LiftOff lo) {
        try {
            rockets.put(lo);
        } catch(InterruptedException e) {
            System.out.println("Interrupted during put()");
        }
    }
    
    public void run() {
        try {
            while (!Thread.interrupted()) {
                LiftOff rocket = rockets.take();
                rocket.run();
            }
        } catch(InterruptedException e) {
            System.out.println("Waking from take()");
        }
        System.out.println("Exiting LiftOffRunner");
    }
}

public class TestBlockingQueues {
    static void getKey() {
        try {
            new BufferedReader(new InputStreamReader(System.in)).readLine();
        } catch (java.io.IOException e) {
            throw new RuntimeException(e);
        }
    }

    static void getKey(String message) {
        System.out.println(message);
    }

    static void test(String msg, BlockingQueue<LiftOff> queue) {
        System.out.println(msg);
        LiftOffRunner runner = new LiftOffRunner(queue);
        Thread t = new Thread(runner);
        t.start();
        for (int i = 0; i < 5; i ++) {
            runner.add(new Liftoff(5));
        }
        getKey("Press 'Enter' (" + msg + ")");
        t.interrupt();
        System.out.println("Finished " + msg + " test");
    }
    
    public static void main(String[] args) {
        test("LinkedBlockingQueue",  // Unlimited Size
             new LinkedBlockingQueue<LiftOff>());
        test("ArrayBlockingQueue",   // Fixed Size
             new ArrayBlockingQueue<LiftOff>());
        test("SynchronousQueue",     // Size of 1
             new SynchronousQueue<LiftOff>());
    }
}
```

### 死锁

某个任务在等待另一个任务，而后者又等待别的任务，这样一直下去，直到这个链条上的任务又在等待第一个任务释放锁。这得到了一个任务之间相互等待的连续循环，没有哪个线程能继续。折旧被称之为*死锁*。

### 新类库中的构件

- CountDownLatch

CountDownLatch 被用来同步一个或多个任务，强制它们等待由其他任务执行的一组操作完成。

可以向 CountDownLatch 对象设置一个初始计数值，任何在这个对象上调用 wait() 的方法都将阻塞，直到这个计数值到达0。其他任务在结束工作时，可以在该对象上调用 countDown() 来减小这个计数值。

CountDownLatch 被设计为只触发一次，计数值不能被重置。

```java
import java.util.Random;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

// Performss some portion of a task:
class TaskPortion implements Runnable {

    private static int counter = 0;
    private final int id = counter ++;
    private static Random rand = new Random(47);
    private final CountDownLatch latch;

    public TaskPortion(CountDownLatch latch) {
        this.latch = latch;
    }

    public void run() {
        try {
            doWork();
            latch.countDown();
        } catch (InterruptedException e) {
            // Acceptable way to exit
        }
    }

    public void doWork() throws InterruptedException {
        TimeUnit.MILLISECONDS.sleep(rand.nextInt(2000));
        System.out.println(this + "completed");
    }

    public String toString() {
        return String.format("%1$-3d ", id);
    }
}

// Waits on the CountDownLatch:
class WaitingTask implements Runnable {

    private static int counter = 0;
    private final int id = counter ++;
    private final CountDownLatch latch;

    public WaitingTask(CountDownLatch latch) {
        this.latch = latch;
    }

    public void run() {
        try {
            latch.await();
            System.out.println("Latch barrier passed for " + this);
        } catch (InterruptedException e) {
            System.out.println(this + " interrupted");
        }
    }

    public String toString() {
        return String.format("%1$-3d ", id);
    }
}

public class CountDownLatchDemo {

    static  final int SIZE = 100;

    public static void main(String[] args) throws Exception {
        ExecutorService exec = Executors.newCachedThreadPool();
        // all must share a single CountDownLatch object:
        CountDownLatch latch = new CountDownLatch(SIZE);
        for (int i = 0; i < 10; i ++) {
            exec.execute(new WaitingTask(latch));
        }
        for (int i = 0; i < SIZE; i ++) {
            exec.execute(new TaskPortion(latch));
        }
        System.out.println("Latched all tasks");
        exec.shutdown();
    }
}
```

- CyclicBarrier

CyclicBarrier 适用于一组任务并行地执行工作，然后在进行下一个步骤之前等待，直至所有任务都完成。它使得所有的并行任务都将在栅栏处列队，因此可以一致地向前移动。相比 CountDownLatch ，CyclicBarrier 可以多次重用。

```java
// Using CyclicBarrier.

import java.util.ArrayList;
import java.util.List;
import java.util.Random;
import java.util.concurrent.*;

class Horse implements Runnable {

    private static int counter = 0;
    private final int id = counter ++;
    private int strides = 0;
    private static Random rand = new Random(47);
    private static CyclicBarrier barrier;

    public Horse(CyclicBarrier b) {
        barrier = b;
    }

    public synchronized int getStrides() {
        return strides;
    }

    public void run() {
        try {
            while (!Thread.interrupted()) {
                synchronized (this) {
                    strides += rand.nextInt(3);
                }
                barrier.await();
            }
        } catch (InterruptedException e) {
            // A legitimate way to exit.
        } catch (BrokenBarrierException e) {
            // This one we want to know about.
            throw new RuntimeException();
        }
    }

    public String toString() {
        return "Horse " + id + " ";
    }

    public String tracks() {
        StringBuilder s = new StringBuilder();
        for (int i = 0; i < getStrides(); i ++) {
            s.append("*");
        }
        s.append(id);
        return s.toString();
    }
}

public class HorseRace {

    static final int FINISH_LINE = 75;
    private List<Horse> horses = new ArrayList<Horse>();
    private ExecutorService exec = Executors.newCachedThreadPool();
    private CyclicBarrier barrier;

    public HorseRace(int nHorses, final int pause) {
        barrier = new CyclicBarrier(nHorses, () -> {
            StringBuilder s = new StringBuilder();
            for (int i = 0; i < FINISH_LINE; i ++) {
                s.append("=");
            }
            System.out.println(s);
            for (Horse horse : horses) {
                System.out.println(horse.tracks());
            }
            for (Horse horse : horses) {
                if (horse.getStrides() >= FINISH_LINE) {
                    System.out.println(horse + "won");
                    exec.shutdownNow();
                    return;
                }
            }
            try {
                TimeUnit.MILLISECONDS.sleep(pause);
            } catch (InterruptedException e) {
                System.out.println("barrier-action sleep interrupted");
            }
        });
        for (int i = 0; i < nHorses; i ++) {
            Horse horse = new Horse(barrier);
            horses.add(horse);
            exec.execute(horse);
        }
    }

    public static void main(String[] args) {
        int nHourses = 7;
        int pause = 200;
        if (args.length > 0) {
            int n = new Integer(args[0]);
            nHourses = n > 0 ? n : nHourses;
        }
        if (args.length > 1) {
            int p = new Integer(args[1]);
            pause = p > -1 ? p : pause;
        }
        new HorseRace(nHourses, pause);
    }
}
```

- DelayQueue
- PriorityBlockingQueue
- ScheduledThreadPoolExecutor
- Semaphore

*计数信号量*允许n个任务同时访问同一项资源

```java
package semaphore_demo;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.Semaphore;

// Using a Semaphore inside a Pool, to restrict the number of tasks that can use a resource.
public class Pool<T> {

    private int size;
    private List<T> items = new ArrayList<T>();
    private volatile boolean[] checkedOut;
    private Semaphore available;

    public Pool(Class<T> classObject, int size) {
        this.size = size;
        checkedOut = new boolean[size];
        available = new Semaphore(size, true);
        // Load pool with objects that can be checked out:
        for (int i = 0; i < size; i ++) {
            try {
                // Assumes a default constructor:
                items.add(classObject.newInstance());
            } catch (Exception e) {
                throw new RuntimeException();
            }
        }
    }

    public T checkOut() throws InterruptedException {
        available.acquire();
        return getItem();
    }

    public void checkIn(T x) {
        if (releaseItem(x)) {
            available.release();
        }
    }

    private synchronized T getItem() {
        for (int i = 0; i < size; i ++) {
            if (!checkedOut[i]) {
                checkedOut[i] = true;
                return items.get(i);
            }
        }
        return null; // Semaphore prevents reaching here.
    }

    private synchronized boolean releaseItem(T item) {
        int index = items.indexOf(item);
        if (index == -1) {
            // Not in the list
            return false;
        }
        if (checkedOut[index]) {
            checkedOut[index] = false;
            return true;
        }
        return false; // Wasn't checked out
    }
}
```

```java
package semaphore_demo;

// Objects that are expensive to create
public class Fat {

    private volatile double d;
    private static int counter = 0;
    private final int id = counter ++;

    public Fat() {
        // Expensive, interruptible operation:
        for (int i = 0; i < 10000; i ++) {
            d += (Math.PI + Math.E) / (double)i;
        }
    }

    public void operation() {
        System.out.println(this);
    }

    public String toString() {
        return "Fat id:" + id;
    }
}
```

```java
package semaphore_demo;
// Test the Pool class

import java.sql.Time;
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;
import java.util.concurrent.TimeUnit;

// A task to check a resource out of a pool
class CheckoutTask<T> implements Runnable {

    private static int counter = 0;
    private final int id = counter ++;
    private Pool<T> pool;

    public CheckoutTask(Pool<T> pool) {
        this.pool = pool;
    }

    public void run() {
        try {
            T item = pool.checkOut();
            System.out.println(this + "cheked out " + item);
            TimeUnit.SECONDS.sleep(1);
            System.out.println(this + "cheked in " + item);
            pool.checkIn(item);
        } catch (InterruptedException e) {
            // Available way to terminate
        }
    }

    public String toString() {
        return "CheckoutTask " + id + " ";
    }
}

public class SemaphoreDemo {

    final static int SIZE = 25;
    public static void main(String[] args) throws Exception {
        final Pool<Fat> pool = new Pool<Fat>(Fat.class, SIZE);
        ExecutorService exec = Executors.newCachedThreadPool();
        for (int i = 0; i < SIZE; i ++) {
            exec.execute(new CheckoutTask<Fat>(pool));
        }
        System.out.println("All CheckoutTasks created");
        List<Fat> list = new ArrayList<Fat>();
        for (int i = 0; i < SIZE; i ++) {
            Fat f = pool.checkOut();
            System.out.println(i + ": main() thread checked out ");
            f.operation();
            list.add(f);
        }
        Future<?> blocked = exec.submit(() -> {
            try {
                // Semaphore prevents additional checkout.
                // so call is blocked:
                pool.checkOut();
            } catch (InterruptedException e) {
                System.out.println("checkOut() Interrupted");
            }
        });
        TimeUnit.SECONDS.sleep(2);
        blocked.cancel(true);
        System.out.println("Checking in objects in " + list);
        for (Fat f : list) {
            pool.checkIn(f);
        }
        for (Fat f : list) {
            pool.checkIn(f); // Second checkIn ignored
        }
        exec.shutdown();
    }
}
```

- Exchanger

### 免锁容器