# 《Java并发编程实战》读书笔记 part2

## 第三章 对象的共享

### 可见性

#### 失效数据

以下代码可能出现两种情况：

1. ReaderThread线程访问的ready一直为false，程序永远不会停下来（可见性问题）
2. 程序的输出为0（重排序问题）

```java
public class NoVisibility {
    private static boolean ready;
    private static int number;
    private static class ReaderThread extends Thread {
    public void run() {
        while (!ready)
        Thread.yield();
        System.out.println(number);
        }
    }
    public static void main(String[] args) {
        new ReaderThread().start();
        number = 42;
        ready = true;
    }
}
```

从Java的内存模型讲起，《深入理解Java虚拟机》关于现代处理器和内存的架构

![](/assets/jcip_note/jvm_memory_hard.png)

Java内存模型的类似抽象如图，其中工作内存和上图的高速缓存类比：![](/assets/jcip_note/jvm_memory_ab.png)也就是说工作内存（高速缓存）的值并没有因为主内存的值更新了而失效，从而出现可见性问题。

#### 非原子的64位操作

> 如果有多个线程共享一个并未声明为volatile的long或double类型的变量，并且同时对他们进行读取和修改操作，那么某些线程可能会读取到一个既非原值，也不是其他线程修改值的代表了“半个变量”的数值。（目前在商用Java虚拟机中不会出现）
>
> 虽然Java内存模型允许虚拟机不把long和double变量的读写实现成原子操作，但允许虚拟机选择把这些操作实现为具有原子性的操作，而且还“强烈建议”虚拟机这样做。目前各个平台下的商用虚拟机几乎都选择把64位数据的读写操作作为原子操作对待，因此在我们编写代码时不需要把用到的long或double变量专门声明为volatile。
>
> -- 《深入理解Java虚拟机》

#### 加锁与可见性

加锁的含义不仅仅局限于互斥行为，还包括内存可见性

#### volatile变量

建议的使用场景，满足所有条件：

1. 确保只有单个线程更新变量的值
2. 该变量不会与其他状态变量一起纳入不变性条件中
3. 在访问变量不需要加锁

   * 不变性条件指的是无论怎样改变或转换，都维持固有的条件。

比如：某个方法的入参必须大于0，或者某个方法的入参必须非null，账户余额必须大于100美元......

参考：[https://www.quora.com/What-are-invariants-in-Java](https://www.quora.com/What-are-invariants-in-Java)

### 发布与逸出

发布（Publish）一个对象是指，使对象能够在当前作用域之外的代码使用。

错误的发布称为逸出（Escape）。

不建议内部的可变对象逸出：

```java
class UnsafeStates {
    private String[] states = new String[] {
        "AK", "AL" ...
    };
    public String[] getStates() { return states; }
}
```

无论其他线程是否用了逸出的对象，其实不重要，因为误用的风险始终存在。就好比银行密码放到微博上了，无论是否有人利用，都不安全了。

隐式地this引用逸出：

```java
public class ThisEscape {
    public ThisEscape(EventSource source) {
        source.registerListener(
            new EventListener() {
                public void onEvent(Event e) {
                    doSomething(e);
                }
            });
    }
}
```

当`ThisEscape`发布`EventListener`（`source.registerListener`\)，也隐式地发布了`ThisEscape.this`，但是`ThisEscape.this`可能没有初始化完成。

**不要在构造过程中使this引用逸出，即使在构造函数最后一行逸出也不行。**

改进：用工厂方法来防止this引用在构造过程中逸出：

```java
public class SafeListener {
    private final EventListener listener;
    private SafeListener() {
        listener = new EventListener() {
            public void onEvent(Event e) {
                doSomething(e);
            }
        };
    }
    public static SafeListener newInstance(EventSource source) {
        SafeListener safe = new SafeListener();
        source.registerListener(safe.listener);
        return safe;
    }
}
```

### 线程封闭

多线程间不共享数据。

1. Swing的可视化组件和数据模型对象
2. JDBC的connection对象

#### ad-hoc线程封闭

指维护线程封闭性的职责完全有程序实现来承担。

#### 栈封闭

在栈封闭中，只能通过局部变量才能访问对象

```java
public int loadTheArk(Collection<Animal> candidates) {
    int ret = 0;
    // 线程封闭，animals对象不可逸出
    SortedSet<Animal> animals = new TreeSet<Animal>();
    animals.addAll(candidates);

    for (Animal a : animals) {
        // ...
    }
    return ret;
}
```

但要小心的是，只有编写代码的开发人员才知道那些对象需要封闭到执行线程中，以及被封闭的对象是否是线程安全的。如果没有明确地说明这些需求，那么后续的维护人员很容易错误地使对象逸出。

#### ThreadLocal类

注意ThreadLocal其实类似全局变量，他能降低代码的可重用性，并在类间引入隐含的耦合性，因此使用时要格外小心。

### 不变性

使用不可变对象，对象状态不可变了，就没有同步问题了。

如果某个对象在创建之后其状态就不能再被修改，这个对象就称为不可变对象。

在Java语言里，不可变对象的充分条件：  
1. 对象创建以后其状态就不能修改了  
2. 对象的所有域都是final类型  
3. 对象是正确创建的（在创建期间，this引用没有逸出）

第二条，从技术上说，不需要，比如String，成员hash并不是final的。

```
public class String{
    private int hash;
    public int hashCode() {
        int h = hash;
        if (h == 0 && value.length > 0) {
            char val[] = value;

            for (int i = 0; i < value.length; i++) {
                h = 31 * h + val[i];
            }
            hash = h;
        }
        return h;
    }
}
```

多线程调用`hashCode()`，因为在对象实例中`hash`这个值每次计算出来都是个定值，看起来就像没有变化一样。

#### Final域

final类型的域是不能修改的，在Java内存模型中，final还有特殊的语义，final域能确保初始化过程的安全性，从而可以不受限制地访问不可变对象，并在共享这些对象时无需同步。

除非需要更高的访问可见性，否则应将所有域都声明为私有域。  
除非需要某个域是可变的，否则应将其声明为final域。

### 安全发布

以上重点讨论了确保对象不发布，但在某些情况下，我们希望可以在多个线程间共享对象。

如果只是这样保存到公有域中，并不够安全。

```java
// Unsafe publication
public Holder holder;

public void initialize() {
    holder = new Holder(42);
}
```

由于可见性问题，其他线程看到Holder对象将处于不一致的状态，即使在该对象的构造函数中已经正确地构建了不变性条件。这种不正确的发布导致其他线程看到部分构造的对象（partially constructed object）。

#### 不正确的发布：正确的对象被破坏

按照上一段代码的方式发布，另一个线程在调用`assertSanity()`时可能抛出异常。

```java
public class Holder {
    private int n;
    public Holder(int n) { this.n = n; }
    public void assertSanity() {
        if (n != n)
            throw new AssertionError("This statement is false.");
    }
}
```

由于没有使用同步确保Holder对象对其他线程可见，Holder称为“未被正确发布”（not properly published）  
这个未被正确发布的对象存在两个问题：  
1. 除了发布线程，其他线程看到的Holder对象是个失效值  
2. 线程看到Holder的引用是最新的，但是Holder的状态是失效的，可能第一次读和第二次读不一致，导致上述异常的抛出。

#### 不可变对象与初始化安全性

如果Holder对象是不可变的（`public final Holder holder`），Holder即使没被正确发布，调用`assertSanity()`也不会抛出异常。（final的语义保证初始化的安全性）

#### 安全发布的常用模式

一个正确构造的对象可以通过以下方式安全发布：

1. 在静态初始化函数中初始化一个对象引用：比如`public static Holder holder = new Holder(1);`  
2. 将对象的引用保存到volatile类型的域或者AtomicReference对象  
3. 将对象的引用保存到某个正确构造对象的final类型域中：比如`public final Holder holder = new Holder(1);`  
4. 将对象的引用保存到一个由锁保护的域中：比如放入Vector，ConcurrentMap对象

#### 事实不可变对象

从技术上看是可变的，但其状态在发布后不会再改变，那么把这种对象称为“事实不可变对象”（effectively immutable），常用的比如Date对象

#### 可变对象

安全发布只能确保“发布当时”状态可见性，发布之后需要同步。

#### 安全地共享对象

并发程序中对象的使用和共享策略：

1. 线程封闭：对象由线程独有
2. 只读共享：不可变对象和事实不可变对象
3. 线程安全共享：对象内部同步
4. 保护对象：特定锁保护

## 第四章 对象的组合

### 设计线程安全的类

在设计线程安全类的过程中，需要包含以下三个要素：  
1. 找出构成对象状态的所有变量，包括变量引用的其他对象  
2. 找出约束状态变量的不变性条件  
3. 建立对象状态的并发访问策略

比如：

```java
@ThreadSafe
public final class Counter {
    @GuardedBy("this") private long value = 0;
    public synchronized long getValue() {
        return value;
    }
    public synchronized long increment() {
        if (value == Long.MAX_VALUE)
            throw new IllegalStateException("counter overflow");
        return ++value;
    }
}
```

以下部分将对这段代码进行上述三点要素的分析  
第一点：Counter类的所有变量只有一个：value

同步策略（ synchronization policy）定义如何在不违背对象不变性条件或后验条件的情况下对其状态的访问操作进行协同。要确保开发人员可以对这个类进行分析和维护，就必须将同步策略写为正式文档，文档内容包括如何将不可变性、线程封闭和加锁机制等结合起来以维护线程的安全性、哪些变量由哪些锁来保护。

#### 收集同步需求

第二点：类中的不变性条件。value域是long型的，其状态为Long.MIN\_VALUE到LONG.MAX\_VALUE，但是其取值存在一个限制，即不能为负数。  
操作中包含一些后验条件用于判断状态迁移是否有效。如果Counter的当前状态为17，那么下一个有效状态只能是18，当下一个状态需要依赖当前状态时，这个操作必须是一个复合操作，第三点，我们用了同步块。

如果不了解对象的不变性条件和后验条件，那么就不能确保线程安全性。

#### 依赖状态的操作

类的不变性条件和后验条件约束了在对象上有哪些状态和状态转换是有效的，在某些对象的方法中还包含一些基于状态的先验条件，比如，不能从空队列中移除一个元素，删除前，队列必须是非空的状态。

如果在某个操作中包含有基于状态的先验条件，那么这个操作就称为依赖状态的操作。

#### 状态的所有权

许多情况下，所有权和封装性总是相互关联的：对象封装它拥有的状态，反之也成立，即对它封装的状态拥有所有权。状态变量的所有者将决定用何种加锁协议来维持变量状态的完整性。所有权意味着控制权。如果发布了某个变量，就不再独占控制权，最多是“共享控制权”。

### 实例封闭

**将数据封装在对象内部，可以将数据的访问限制在对象的方法上，从而更容易确保线程在访问数据时总能持有正确的锁。**

```java
@ThreadSafe
public class PersonSet {
    @GuardedBy("this")
    private final Set<Person> mySet = new HashSet<Person>();
    public synchronized void addPerson(Person p) {
        mySet.add(p);
    }
    public synchronized boolean containsPerson(Person p) {
        return mySet.contains(p);
    }
}
```

实例封闭是构建线程安全类的一个最简单的方式（分析类的线程安全性无需检查整个程序），它还在锁策略的选择上拥有更多的灵活性（封装的好处）。

#### Java监视器模式

从线程封闭原则及其逻辑推论可以得出Java监视器模式。遵循Java监视器模式的对象会把对象的所有可变状态都封装起来，并由对象的**内置锁**来保护。比如：Vector，HashTable...

Java监视器模式只是一种编码约定，对于任何一种锁对象，只要始终都使用该锁对象，都可以用来保护对象的状态。比如通过一个私有锁来保护状态：

```java
public class PrivateLock {
private final Object myLock = new Object();
    @GuardedBy("myLock") Widget widget;
    void someMethod() {
        synchronized(myLock) {
            // Access or modify the state of widget
        }
    }
}
```

私有锁对象可以将锁封装起来，使客户端无法得到锁，但是内置锁就不行了。

### 线程安全性的委托

章节2.4的代码如下。在无状态的类中增加一个AtomicLong类型的域，并且得到的组合对象依然是线程安全的。由于CountingFactorizer的状态就是AtomicLong的状态，而AtomicLong是线程安全的，而且CountingFactorizer没有对count的状态施加额外的有效性约束（书里的翻译有误，非因果关系，这里改成“而且”），所以很容易知道CountingFactorizer是线程安全的。[^1]

```java
@ThreadSafe
public class CountingFactorizer extends GenericServlet implements Servlet {
    private final AtomicLong count = new AtomicLong(0);

    public long getCount() { return count.get(); }

    public void service(ServletRequest req, ServletResponse resp) {
        BigInteger i = extractFromRequest(req);
        BigInteger[] factors = factor(i);
        count.incrementAndGet();
        encodeIntoResponse(resp, factors);
    }

    void encodeIntoResponse(ServletResponse res, BigInteger[] factors) {}
    BigInteger extractFromRequest(ServletRequest req) {return null; }
    BigInteger[] factor(BigInteger i) { return null; }
}
```

#### 独立地状态变量

除了以上的AtomicLong对象的例子，我们可以将线程安全性委托给多个状态变量，只要这些变量是彼此独立地，即组合而成的类并不会在其包含的多个状态变量上增加任何不变性条件。

```java
public class VisualComponent {
    private final List<KeyListener> keyListeners
            = new CopyOnWriteArrayList<KeyListener>();
    private final List<MouseListener> mouseListeners
            = new CopyOnWriteArrayList<MouseListener>();

    public void addKeyListener(KeyListener listener) {
        keyListeners.add(listener);
    }

    public void addMouseListener(MouseListener listener) {
        mouseListeners.add(listener);
    }

    public void removeKeyListener(KeyListener listener) {
        keyListeners.remove(listener);
    }

    public void removeMouseListener(MouseListener listener) {
        mouseListeners.remove(listener);
    }
}
```

#### 当委托失效时

大多数对象的组合都不会想上述例子那样简单，比如

```java
public class NumberRange {
    // 不变性条件: lower <= upper
    private final AtomicInteger lower = new AtomicInteger(0);
    private final AtomicInteger upper = new AtomicInteger(0);

    public void setLower(int i) {
        // Warning -- unsafe check-then-act
        if (i > upper.get())
            throw new IllegalArgumentException("can't set lower to " + i + " > upper");
        lower.set(i);
    }

    public void setUpper(int i) {
        // Warning -- unsafe check-then-act
        if (i < lower.get())
            throw new IllegalArgumentException("can't set upper to " + i + " < lower");
        upper.set(i);
    }

    public boolean isInRange(int i) {
        return (i >= lower.get() && i <= upper.get());
    }
}

```

仅靠委托给AtomicInteger 保证安全性是不够的，这个类必须提供自己的加锁机制以保证这些复合操作都是原子的。

#### 发布低层的状态变量

### 在现有的线程安全类中添加功能

#### 客户端加锁机制

#### 组合

### 将同步策略文档化

[^1]: 原文：Since the state of CountingFactorizer is the state of the thread-safe AtomicLong , and since CountingFactorizer imposes no additional validity constraints on the state of the counter, it is easy to see that CountingFactorizer is thread-safe.

