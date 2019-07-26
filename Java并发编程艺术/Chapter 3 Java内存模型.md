## Java 内存模型

### Java 内存模型的基础

#### 并发编程模型的两个关键问题

- 线程之间如何通信
- 线程之间如何同步

通信是指线程之间以何种机制来交换信息。在命令式编程中，通信机制有两种：共享内存和消息传递。

同步是指程序中用于控制不同线程间操作发生的相对顺序的机制。

#### Java 内存模型的抽象结构

线程之间的共享变量存储在主内存中，每个线程都有一个私有的本地内存，其中存储了该线程以读 / 写共享变量的副本。

如果线程 A 与线程 B 之间要通信的话，必须要经历 2 个步骤：

- 线程 A 把本地内存 A 中更新过的共享变量刷新到主内存中去。
- 线程 B 到主内存中去读线程 A 之前已更新过的共享变量。

#### 从源代码到指令序列的重排序

为了提高性能，编译器和处理器常常会对指令做重排序。重排序分 3 种类型：

- 编译器优化的重排序。编译器在不改变单线程程序语义的前提下，可以重新安排语句的执行顺序。
- 指令级并行的重排序。现代处理器采用了指令级并行技术来将多条指令重叠执行。如果不存在数据依赖性，处理器可以改变对应机器指令的执行顺序。
- 内存系统的重排序。由于处理器使用缓存和读 / 写缓冲区，这使得加载和存储操作看上去可能是乱序执行。

从 Java 源代码到最终实际执行的指令序列，会分别经历编译器优化的重排序、指令级并行的重排序和内存系统的重排序。这些重排序可能会导致多线程程序出现内存可见性问题。

对于编译器，JMM 的编译器重排序规则会禁止特定类型的编译器重排序。对于处理器重排序，JMM 的处理器重排序规则会要求 Java 编译器在生成指令序列时，插入特定类型的内存屏障指令，通过内存屏障指令来禁止特定类型的重排序。

#### happens-before 简介

无论是在一个线程之内还是不同线程之间，如果一个操作执行的结构需要对另一个操作可见，那么这两个操作之间必须要存在 happens-before 关系。

- 程序顺序规则：一个线程中的每个操作，happens-before 于该线程中的任意后续操作
- 监视器锁规则：对于一个锁的解锁，happens-before 于随后对这个锁的加锁
- volatile 变量规则：对一个 volatile 域的写，happens-before 于任意后续对这个 volatile 域的读
- 传递性：如果 A happens-before B，且 B happens-before C ，那么 A happens-before C

一个 happens-before 规则对应一个或多个编译器和处理器重排序规则。它避免为了理解 JMM 提供的内存可见性保证而去学习复杂的重排序规则以及这些规则的具体实现方法。

### 重排序

重排序是指编译器和处理器为了优化程序性能而对指令序列进行重新排序的一种手段。

#### 数据依赖性

如果两个操作访问同一个变量，且这两个操作中有一个为写操作，此时这两个操作之间就存在数据依赖性。

- 写后读
- 写后写
- 读后写

#### as-if-serial 语义

不管怎么重排序，单线程程序的执行结果不能被改变。因此编译器和处理器不会对存在数据依赖关系的操作做重排序。

#### 程序顺序规则

happens-before 仅要求前一个操作对后一个操作可见，且前一个操作按顺序排在第二个操作之前，并不意味着一个操作必须要在后一个操作之前执行。

#### 重排序对多线程的影响

在多线程程序中，对存在控制依赖的操作重排序，可能会改变程序的执行结果。

### 顺序一致性

#### 数据竞争与顺序一致性

数据竞争是指：

> 在一个线程中写一个变量，
>
> 在另一个线程读一个变量，
>
> 而且写和读没有通过同步来排序。

如果程序是正确同步的，程序的执行将具有顺序一致性——即程序的执行结果与程序在顺序一致性内存模型中的执行结果相同。

#### 顺序一致性内存模型

特性：

- 一个线程中的所有操作必须按照程序的顺序来执行
- 所有线程都只能看到一个单一的操作执行顺序。在顺序一致性内存模型中，每个操作都必须原子执行且立刻对所有线程可见。
- 保证对所有的内存读 / 写操作都具有原子性

#### 同步程序和顺序一致性效果

JMM 的具体实现为，在正确同步的程序执行结果的前提下，尽可能地为编译器和处理器的优化打开方便之门。

#### 未同步程序的执行特性

JMM 不保证对 64 位的 long 型和 double 型变量的写操作具有原子性。

### volatile 的内存语义

#### volatile 的特性

- 可见性。对一个 volatile 变量的读，总是能看到（任意线程）对这个 volatile 变量最后的写入。
- 原子性。对任意单个 volatile 变量的读 / 写具有原子性。

#### volatile 写 - 读建立的 happens-before 关系

从 JDK5 开始，volatile 变量的写 - 读可以实现线程之间的通信。

从内存语义的角度来说，volatile 的写 - 读与锁的释放 - 获取有相同的内存效果。

volatile 规则可以得到 happens-before 关系

#### volatile 的写 - 读的内存语义

- 线程 A 写一个 volatile 变量，实质上是线程 A 向接下来将要读这个 volatile 的某个线程发出了消息
- 线程 B 读一个 volatile 变量，实质上是线程 B 接收了之前某个线程发出的消息

- 线程 A 写一个 volatile 变量，随后线程 B 读这个 volatile 变量，这个过程实质上是线程 A 通过主内存向线程 B 发送消息。

#### volatile 内存语义的实现

为了实现 volatile 内存语义，JMM 会分别限制编译器重排序和处理器重排序。

- 当第二个操作是 volatile 写时，不管第一个操作是什么，都不能重排序。这个规则确保 volatile 写之前的操作不会被编译器重排序到 volatile 写之后。
- 当第一个操作是 volatile 读时，不管第二个操作是什么，都不能重排序。这个规则确保 volatile 读之后的操作不会被编译器重排序到 volatile 读之前。
- 当第一个操作是 volatile 写，第二个操作是 volatile 读时，不能重排序。

### 锁的内存语义

#### 锁的释放 - 获取建立的 happens-before 关系

监视器锁规则可以得到 happens-before 关系。

#### 锁的释放和获取的内存语义

- 线程 A 释放一个锁，实质上是线程 A 向接下来将要获取这个锁的某个线程发出了消息
- 线程 B 获取一个锁，实质上是线程 B 接收了之前某个线程发出的消息
- 线程 A 释放锁，随后线程 B 获取这个锁，这个过程实质上是线程 A 通过主内存向线程 B 发送消息

#### 锁内存语义的实现

- 公平锁和非公平锁释放时，最后都要写一个 volatile 变量 state
- 公平锁获取时，首先会去读 volatile 变量
- 非公平锁获取时，首先会用 CAS 更新 volatile 变量，这个操作同时具有 volatile 读和 volatile 写的内存语义

#### concurrent 包的实现

首先，声明共享变量为 volatile 。

然后，使用 CAS 的原子条件更新来实现线程之间的同步。

同时，配合以 volatile 的 读 / 写和 CAS 所具有的 volatile 读和写的内存语义来实现线程之间的通信。

### final 域的内存语义

#### final 域的重排序规则

- 在构造函数内对一个 final 域的写入，与随后把这个被构造对象的引用赋值给一个引用变量，这两个操作之间不能重排序
- 初次读一个包含 final 域的对象的引用，与随后初次读这个 final 域，这两个操作之间不能重排序

```java
public class FinalExample {
    int i;
    final int j;
    static FinalExample obj;
    
    public FinalExample() {
        i = 1;
        j = 2;
    }
    
    public static void writer() {
        obj = new FinalExample();
    }
    
    public static void reader() {
        FinalExample object = obj;
        int a = object.i;
        int b = object.j;
    }
}
```

#### 写 final 域的重排序规则

在对象引用为任意线程可见之前，对象的 final 域已经被正确初始化过了，而普通域不具有这个保障。在读线程看到对象引用 obj 时，很可能 obj 对象还没构造完成，因为对普通域 i 的写操作被重排序到构造函数之外。

#### 读 final 域的重排序规则

在读一个对象的 final 域之前，一定会先读包含这个 final 域的对象的引用。即先读 obj 对象，再读 j 。

#### final 域为引用类型

在构造函数内对一个 final 引用的对象的成员域的写入，与随后在构造函数外把这个被构造对象的引用赋值给一个引用变量，这两个操作之间不能重排序。

#### 为什么 final 引用不能从构造函数内逸出

```java
public class FinalReferenceEscapeExample {
    final int i;
    static FinalReferenceEscapeExample obj;
    
    public FinalReferenceEscapeExample() {
        i = 1;
        obj = this; // this 引用逸出
    }
    
    public static void writer() {
        new FinalReferenceEscapeExample();
    }
    
    public static void reader() {
        if (obj != null) {
            int temp = obj.i;
        }
    }
}
```

在构造函数返回前，被构造对象的引用不能为其他线程所见，因为此时的 final 域可能还没有被初始化。

### happens-before

#### JMM 的设计

- 会改变程序的重排序，JMM 要求编译器和处理器必须禁止这种重排序
- 不会改变程序的重排序，JMM 对编译器和处理器不做要求

#### happens-before 的定义

- 如果一个操作 happens-before 另一个操作，那么第一个操作的执行结果将对第二个操作可见，而且第一个操作的执行顺序排在第二个操作之前
- 两个操作之间存在 happens-before 关系，并不意味着 Java 平台的具体实现必须要按照 happens-before 关系指定的顺序来执行

#### happens-before 规则

- 程序顺序规则：一个线程中的每个操作，happens-before 于该线程中的任意后续操作
- 监视器锁规则：对于一个锁的解锁，happens-before 于随后对这个锁的加锁
- volatile 变量规则：对一个 volatile 域的写，happens-before 于任意后续对这个 volatile 域的读
- 传递性：如果 A happens-before B，且 B happens-before C ，那么 A happens-before C
- start() 规则：如果线程 A 执行操作 ThreadB.start() ，那么线程 A 的 ThreadB.start() 操作 happens-before 于线程 B 中的任意操作
- join() 规则：如果线程 A 执行操作 ThreadB.join() 并成功返回，那么线程 B 中的任意操作 happens-before 于线程 A 从 ThreadB.join() 操作成功返回

### 双重检查锁定与延迟初始化

在 Java 多线程程序中，有时候需要采用延迟初始化来降低初始化类和创建对象的开销。

双重检查锁定是常见的延迟初始化技术，但它是一个错误的用法。

#### 双重检查锁定的由来

```java
public class DoubleCheckedLocking {
    private static Instance instance;
    
    public static Instance getInstance() {
        if (instance == null) {
            synchronized (DoubleCheckedLocking.class) {
                if (instance == null) {
                    instance = new Instance();
                }
            }
        }
        return instance;
    }
}
```

代码第一次读取到 instance 不为 null 时，instance 引用的对象有可能还没有完成初始化。

#### 问题的根源

在单线程程序中，初始化对象和设置 instance 指向内存空间被重排序，但不会影响执行结果。而在多线程程序中，其他线程很有可能会看到一个还没有完全初始化的对象。

解决方法：

- 不允许初始化对象和设置 instance 指向内存空间重排序
- 允许初始化对象和设置 instance 指向内存空间被重排序，但不允许其他线程看到这个重排序

#### 基于 volatile 的解决方案

```java
public class SafeDoubleCheckedLocking {
    private volatile static Instance instance;
    
    public static Instance getInstance() {
        if (instance == null) {
            synchronized (DoubleCheckedLocking.class) {
                if (instance == null) {
                    instance = new Instance();
                }
            }
        }
        return instance;
    }
}
```

#### 基于类初始化的解决方案

JVM 在类的初始化阶段，会获取一个锁，这个锁可以同步多个线程对同一个类的初始化。

```java
public class InstanceFactory {
    private static class InstanceHolder {
        public static Instance instance = new Instance();
    }
    
    public static Instance getInstance() {
        return InstanceHolder.instance;
    }
}
```

在大多数情况下，正确的初始化要优于延迟初始化。基于 volatile 的延迟初始化的方案适用于对实例字段初始化；基于类的延迟初始化方案适用于对静态字段初始化。