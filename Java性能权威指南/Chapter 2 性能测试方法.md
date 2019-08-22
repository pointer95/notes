## 性能测试方法

### 原则 1：测试真实应用

应该在产品实际使用的环境中进行性能测试。

#### 微基准测试

用来测量微小代码单元的性能，包括调用同步方法的用时与非同步方法的用时比较，创建线程的代价与使用线程池的代价，执行某种算法的耗时与其代替实现的耗时等。

```java
/**
 * 计算出第50个斐波那契数
 */
public void doTest() {
    double l;
    long then = System.currentTimeMillis();
    for (int i = 0; i < nLoops; i++) {
        l = fibImpl(50);
    }
    long now = System.currentTimeMillis();
    System.out.println("Elapsed time: " + (now - then));
}

private double fibImpl(int n) {
    if (n < 0) throw new IllegalArgumentException("Must be > 0");
    if (n == 0) return 0d;
    if (n == 1) return 1d;
    double d = fibImpl(n - 1) + fidImpl(n - 2);
    if (Double.isInfinite(d)) throw new ArithmeticException("Overflow");
    return d;
}
```

这段测试代码看起来很简单，但存在很多问题：

1. 必须使用被测试的结果

   这段代码最大的问题是，实际上它永远都不会改变程序的任何状态。因为斐波那契的计算结果从未使用过，所以编译器可以很放心地去除计算结果。智能的编译器最终执行的是以下代码：

   ```java
   long then = System.currentTimeMillis();
   long now = System.currentTimeMillis();
   System.out.println("Elapsed time: " + (now - then));
   ```

   将局部变量 l 的定义改为实例变量并用 volatile 声明就能测试这个方法的性能了。

   本示例是单线程微基准测试，也必需使用 volatile 变量。对于多线程微基准测试，当若干个线程同时执行小段代码时，极有可能会产生同步瓶颈。

2. 不要包括无关的操作

   上述代码只有一个操作：计算第 50 个斐波那契数。其中有些迭代时多余的，编译器可能会少执行几次迭代。

   另外，fibImpl(1000) 和 fibImpl(1) 的性能差距可能会很大。如果目的是为了比较不同实现的性能，测试的输入应该考虑用一系列数据。

3. 必须输入合理的参数

   任意选择的随机输入值对于这段被测代码的用法来说不具有代表性。在这个测试用例中，合理的输入范围是 0~1476 之间。

综合所有因素，正确的微基准测试代码看起来应该是这样的：

```java
public class FibonacciTest {
    private volatile double l;
    private int nloops;
    private int[] inputl;
    
    public static void main(String[] args) {
        FibonacciTest ft = new FibonacciTest(Integer.parseInt(args[0]));
        ft.doTest(true);
        ft.doTest(false);
    }
    
    private FibonacciTest(int n) {
        nLoops = n;
        input = new int[nLoops];
        Random r = new Random();
        for (int i = 0; i < nLoops; i++) {
            input[i] = r.nextInt(100);
        }
    }
    
    private void doTest() {
    	long then = System.currentTimeMillis();
    	for (int i = 0; i < nLoops; i++) {
    	    l = fibImpl(input[i]);
    	}
        if (!isWarmup) {
            long now = System.currentTimeMillis();
    		System.out.println("Elapsed time: " + (now - then));
        }
	}
    
    private double fibImpl(int n) {
    	if (n < 0) throw new IllegalArgumentException("Must be > 0");
    	if (n == 0) return 0d;
    	if (n == 1) return 1d;
    	double d = fibImpl(n - 1) + fidImpl(n - 2);
    	if (Double.isInfinite(d)) throw new ArithmeticException("Overflow");
    	return d;
	}
}
```

调用 fibImpl() 的循环和方法以及 volatile 变量都会带来额外的开销。

#### 宏基准测试

衡量应用性能最好的事物就是应用本身，以及它所用到的外部资源。应用本身必须在完整真实配置的环境中测试。但随着应用规模的增长，上述准则愈加重要也更难达到。

#### 介基准测试

介基准测试微基准测试相比隐患更少，又比宏基准测试容易。

### 原则 2：理解批处理流逝时间、吞吐量和响应时间

#### 批处理流逝时间

虚拟机会花几分钟或更长时间全面优化代码并以最高性能执行。由于这个以及其他原因，研究 Java 性能优化就要密切注意代码优化的热身期：大多数时候，应该在运行代码执行足够长时间，已经编译并优化之后再测量性能。

许多情况下应用从开始到结束的整体性能更为重要。

#### 吞吐量测试

吞吐量是所有客户端所完成的操作总量。通常这个数字是每秒完成的操作量，这个指标常被称作每秒事务数（TPS）、每秒请求数（RPS）或每秒操作次数（OPS）。

#### 响应时间测试

响应时间是指从客户端发送请求至收到相应之间的流逝时间。

响应时间测试和吞吐量测试之间的差别是，响应时间测试的客户端会在操作之间休眠一段时间。这被称为思考时间。

衡量响应时间有两种方法：平均值和百分位请求。平均值会受到离群值影响。

### 原则 3：用统计方法应对性能变化

性能测试的结果会随时间而变。因为有很多因素会影响程序的运行。

好的基准测试不会每次都处理相同的数据集，而是会在测试中制造一些随机行为以模拟真实的世界。

正确判定测试结果见的差异需要统计分析，通过统计分析才能确定这些差异是不是归因于随机因素。

### 原则 4：尽早频繁测试

遵循以下准则，可以使得尽早频繁测试变得有用

- 自动化一切

  所有的性能测试都应该脚本化。全部环境都必须通过脚本安装和配置新代码，然后用脚本运行测试集。

- 测试一切

  必须自动收集能想象到的每一点数据，以便进行后续分析。这些数据包括整个运行过程中采集的系统信息：CPU 使用率、磁盘使用率、网络使用率和内存使用率。数据还包括应用的日志——应用产生的日志，以及垃圾收集器的日志。

- 在真实系统上运行