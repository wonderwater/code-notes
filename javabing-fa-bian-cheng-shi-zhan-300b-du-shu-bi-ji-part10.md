# 《Java并发编程实战》读书笔记 part10

# Java内存模型

### 什么是内存模型，为什么需要它
假设一个线程为变量aVariable赋值：
```java
aVariable = 3;
```
内存模型要解决的这个问题：
> 在什么条件下，读取aVariable的线程将看到这个值为3？

很多操作都会使得一个线程无法看到变量的最新值，并且会导致其他线程中的内存操作似乎在乱序执行：
1. 在编译器中生成的指令顺序，可以与源码顺序不同
2. 编译器会把变量保存在寄存器中，而不是内存
3. 处理器可以采用乱序或并行等方式执行指令
4. 缓存可能会改变 将写入变量提交到主内存的次序
5. 保存在处理器本地缓存中的值，对其他处理器是不可见的

Java语言规范要求JVM在线程中维护一种类似串行地语言：只要程序的最终结果与在严格串行环境中执行的结果相同，那么上述所有操作都是允许的。

- 平台的内存模型：在共享内存的多处理器架构中，每个处理器都拥有自己的缓存，并且定期地与主内存进行协调，在不同的处理器架构中提供了不同级别的缓存一致性（Cache Coherence）。Java提供自己的内存模型，并且JVM通过在适当的位置上插入内存栅栏来屏蔽JMM与底层平台内存模型之间的差异。
- 重排序：各种使操作延迟或者看似乱序执行的不同原因，都可以归为重排序

Java内存模型是通过各种操作来定义的，包括对变量的读/写操作，监视器的加锁和释放操作，已经线程的启动和合并操作。JMM为程序中所有的操作定义了一个偏序关系（比如自然数），称之为Happens-Before。想要保证执行操作B的线程看到操作A的结果，那么A和B必须满足Happens-Before关系。如果两个操作缺乏Happens-Before关系，那么JVM可以对它们任意重排序。

Happens-Before规则包括：
1. 程序顺序规则。程序中操作A在操作B前，那么线程中操作A在操作B前执行。
2. 监视器锁规则。监视器上的解锁操作必须在同一个监视器上的加锁操作之前执行。
3. volatile变量规则。对volatile变量的写入操作必须在对该变量的读取操作之前执行。
4. 线程启动规则。在线程上对Thread.start的调用必须在该线程中执行任何操作之前执行。即Thread.start这个调用必须在被调起的线程中的任何操作之前执行。
5. 线程结束规则。线程中的任何操作必须在其他线程检测到该线程已经结束之前执行，或者从Thread.join返回，或者调用Thread.isAlive返回false。
6. 中断规则。当一个线程在另一个线程上调用interrupt时，必须在被中断线程检测到interrupt调用之前执行。
7. 终结器规则。对象的构造函数必须在启动该对象的终结器之前执行完成。
8. 传递性规则。如果操作A在操作B之前执行，且操作B在操作C之前执行，那么操作A必须在操作C之前执行。

例子：
线程A内部的所有操作都按照它们在源程序中的先后顺序来排序，在线程B内部的操作也是如此。由于A释放了锁M，并且B随后获得了锁M，**因此A中所有在释放锁之前的操作，也就位于B中请求锁之后的所有操作之前**。如果这两个线程在不同的锁上进行同步的，那么就不能推断它们之间的动作顺序，因为在这两个线程的操作之间并不存在Happens-Before关系。
如图：
![](/assets/jcip_note/happens-before_in_jmm.png)

类库中提供的Happens-Before排序包括：
1. 将一个元素放入一个线程安全容器的操作将在另一个线程从该容器中获得这个元素的操作之前执行。
2. 在CountDownLatch上的倒数操作将在线程从闭锁上的await方法中返回之前执行。
3. 释放Semaphore许可的操作将在从该Semaphore上获取一个许可之前执行。
4. Future表示的任务的所有操作将在从Future.get中返回之前执行。
5. 向Executor提交一个Runnable或Callable的操作将在任务开始执行之前执行。
6. 一个线程到达CyclicBarrier或Exchanger的操作将在其他到达该栅栏或交换点的线程被释放之前执行。如果CyclicBarrier使用一个栅栏操作，那么到达栅栏的操作将在栅栏操作之前执行，而栅栏操作又会在线程从栅栏中释放之前执行。

### 发布
造成不正确发布的真正原因：**“发布一个共享对象”与“另一个线程访问该对象”之前缺少一种Happens-Before排序**
例如：
```java
@NotThreadSafe
public class UnsafeLazyInitialization {
    private static Resource resource;

    public static Resource getInstance() {
        if (resource == null)
            resource = new Resource(); // unsafe publication
        return resource;
    }
}
```
两个线程A和B，线程A写入resource的操作与线程B读取resource的操作之间不存在Happens-Before关系。在发布对象时存在数据竞争问题，因此B并不一定能看到resource的正确状态。
当新分配一个Resource时，Resource的构造函数将把新实例中的各个域由默认值（由Object构造函数写入的）修改为它们的初始值。

由于在两个线程中都没有使用同步
 -> 线程B看到线程A中的操作顺序，可能与线程A执行这些操作时的顺序并不相同。
 -> 即使线程A初始化Resource实例之后再将resource设置为指向它，线程B就可能看到对resource的写入操作将在对Resource各个域的写入操作之前发生。
 -> 线程B就可能看到一个被部分构造的Resource实例，该实例可能处于无效状态，并在随后该实例的状态可能出现无法预料的变化。

除了不可变对象以外，使用被另一个线程初始化的对象通常都是不安全的，除非对象的发布操作是在使用该对象的线程开始之前执行。
事实上，Happens-Before比安全发布提供了更强可见性和顺序保证。如果将X从A安全地发布到B，那么这种安全发布可以保证X状态的可见性，但无法保证A访问的其他变量的状态可见性。然而，如果A将X置入队列的操作在线程B从队列中获取X的操作之前执行，那么B不仅能看到A留下的X状态，而且还能看到A在移交X之前所做的任何操作。
Happens-Before排序是在内存访问级别上操作的，它是一种“并发级汇编语言”，而安全发布的运行级别更接近程序设计。
线程安全的延迟初始化：
```java
@ThreadSafe
public class SafeLazyInitialization {
    private static Resource resource;

    public synchronized static Resource getInstance() {
        if (resource == null)
            resource = new Resource();
        return resource;
    }
}
```
提前初始化：
```java
@ThreadSafe
public class EagerInitialization {
    private static Resource resource = new Resource();

    public static Resource getResource() {
        return resource;
    }
}
```
延迟初始化占位类模式：
```java
@ThreadSafe
public class ResourceFactory {
    private static class ResourceHolder {
        public static Resource resource = new Resource();
    }

    public static Resource getResource() {
        return ResourceFactory.ResourceHolder.resource;
    }
}
```
双重检查加锁（DCL）：
```java
@NotThreadSafe
public class DoubleCheckedLocking {
    private static Resource resource;

	// 问题：线程可能看到一个仅被部分构造的Resource
    public static Resource getInstance() {
        if (resource == null) {
            synchronized (DoubleCheckedLocking.class) {
                if (resource == null)
                    resource = new Resource();
            }
        }
        return resource;
    }
}
```
DCL的真正问题在于：当在没有同步的情况下读取一个共享对象时，可能发生的最糟糕情况只是看到一个失效值，此时DCL方法将通过持有锁的情况再次尝试来避免这种风险。然而，实际情况远比这种情况糟糕——线程可能看到引用的当前值，但对象的状态值却是失效的，这意味着线程可以看到对象处于无效或错误的状态。
在JMM后续的版本（>=Java5.0），如果把resource声明为volatile类型，那么就能启用DCL，并且这种方式对性能影响很小。

### 初始化过程中的安全性
如果能确保初始化过程中的安全性，那么就可以使得被正确构造的**不可变对象**在没有同步的情况下也能安全地在多个线程之间共享，而不管它们是如何发布的。
不可变对象的初始化安全性：
```java
@ThreadSafe
public class SafeStates {
    private final Map<String, String> states;

	// 构造过程中，final域变量states可到达的任意变量将同样对于其他线程时可见的。
    public SafeStates() {
        states = new HashMap<String, String>();
        states.put("alaska", "AK");
        states.put("alabama", "AL");
        /*...*/
        states.put("wyoming", "WY");
    }

    public String getAbbreviation(String s) {
        return states.get(s);
    }
}
```
初始化安全性只能保证通过final域可达的值从构造过程完成时开始的可见性。对于通过非final域可达的值，或者在构造过程完成后可能改变的值，必须采用同步来确保可见性。