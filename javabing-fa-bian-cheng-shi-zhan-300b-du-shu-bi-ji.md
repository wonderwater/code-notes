# 《Java并发编程实战》读书笔记 part1

## 一、简介

### 线程的**优势**：

1. 发挥多处理器的强大能力
2. 建模的简单性
3. 异步时间的简单处理
4. 响应更灵敏的用户界面

### 线程的**风险**：

* 安全性问题
  
  如果错误的估计程序的执行顺序，可能带来各种危险

```java
@NotThreadSafe
public class UnsafeSequence {
  private int value;

  /**
   * Returns a unique value.
   */
  public int getNext() {
      return value++;
  }
}
```

> 以上代码至少存在两个问题：  
> 1. `value++`操作的原子性无法保证  
> 2. `value`并非内存可见性，对于某个线程可能存在未能及时更新的情况  
> solution: `public synchronized int getNext` 
> `synchronized`的语义不仅保证了同步，也保证了可见性 

* 活跃性问题

比如死锁、饥饿、活锁

> 安全性是希望不发生错误的事情，比如上面的代码因为多线程执行顺序可能出现和预期不一致的情况。  
> 活跃性是当某个操作无法继续下去而发生的，可能运行的好好的，突然间就处于一种奇妙的状态，也不退出，也不执行业务代码，可能也几乎不耗资源。

* 性能问题

  多线程的切换影响：保存和恢复执行上下文，丢失局部性，cpu的线程调度消耗

### 线程安全性

* 线程安全性

核心问题是正确性（某个类的行为与其规范完全一致）。

在多个线程访问某个类时，这个类始终都能表现正确的行为，则称这个类是线程安全的。

推论：无状态的类一定是线程安全的，比如

```java
@ThreadSafe
public class StatelessFactorizer extends GenericServlet implements Servlet {

    public void service(ServletRequest req, ServletResponse resp) {
        BigInteger i = extractFromRequest(req);
        BigInteger[] factors = factor(i);
        encodeIntoResponse(resp, factors);
    }
}
```

### 原子性

* 竞态条件

  最常见的形式：先检查后执行（Check-Then-Act）
  比如延迟初始化：

```java
@NotThreadSafe
public class LazyInitRace {
    private ExpensiveObject instance = null;

    public ExpensiveObject getInstance() {
        if (instance == null)
            instance = new ExpensiveObject();
        return instance;
    }
}

class ExpensiveObject { }
```

* 复合操作

假定两个操作A和B，如果从执行A的线程来看，当一个线程执行B时，要么B全部执行完，要么B完全不执行，那么A和B对彼此来说原子的。  
原子操作是指，对于访问同一个状态的所有操作来说，这个操作是一个以原子方式执行的操作。

> 代换理解：  
> 假定两个操作  
> A：`v++`  
> B：`v--`  
> 如果从执行A的线程T1来看，当线程T2执行B时，要么B全部执行完，要么B完全不执行，那么A和B对彼此来说原子的。  
> water？T1看T2？看什么？什么时候看？

### 加锁机制

要注意是否有足够的原子性保证，比如

```java
@NotThreadSafe
public class UnsafeCachingFactorizer extends GenericServlet implements Servlet {
    private final AtomicReference<BigInteger> lastNumber
            = new AtomicReference<BigInteger>();
    private final AtomicReference<BigInteger[]> lastFactors
            = new AtomicReference<BigInteger[]>();

    public void service(ServletRequest req, ServletResponse resp) {
        BigInteger i = extractFromRequest(req);
        // 典型的竞态条件问题，先检查后执行
        if (i.equals(lastNumber.get()))
            encodeIntoResponse(resp, lastFactors.get());
        else {
            BigInteger[] factors = factor(i);
            // 虽然这两个变量各自都能保证原子性，但我们需要的效果是复合操作保证原子性
            lastNumber.set(i);
            lastFactors.set(factors);
            encodeIntoResponse(resp, factors);
        }
    }
}
```

* 内置锁（Intrinsic Lock）/监视器锁（Monitor Lock）

同步代码块包括两部分  
一个作为锁的引用对象，一个作为有这个锁保护的代码块

```java
synchronized(lock){
// 访问或修改由锁包含的共享状态
}
```

以关键字synchronized修饰的方法就是一种横跨整个方法的同步代码块第一部分就是方法调用所在的对象，静态方法就是Class对象。

* 重入

某个线程尝试获取一个自身已持有的锁，那么这个请求就会成功。

比如`loggingWidget.doSomething()`：

```java
class Widget {
    public synchronized void doSomething() {
    }
}

class LoggingWidget extends Widget {
    public synchronized void doSomething() {
        System.out.println(toString() + ": calling doSomething");
        super.doSomething();
    }
}
```

### 用锁来保护状态

对于可能被多个线程同时访问的可变状态变量，在访问它时都需要持有同一把锁，在这种情况下，我们称状态变量是有这个锁保护的。

**对象的内置锁与其状态之间没有内在关联**。虽然大多数类都将内置锁用作一种有效的加锁机制，但对象的域并不一定要通过内置锁来保护。比如即使获得和对象关联的锁，并不能阻止其他线程访问该对象，只能阻止其他线程获得同一把锁。

之所以每个对象都有内置锁，只是为了免去显式创建锁对象。（作者吐槽：回想起来，这种设计决策或许比较糟糕，不仅会引起混乱，而且还迫使JVM需要在对象大小与加锁性能之间进行权衡）

**每个共享和可变的变量都应该只由一把锁来保护，从而使维护人员知道是哪一把锁。**
一般的加锁约定：将所有的可变状态都封锁在对象内部：比如Vector, HashTable...


### 活跃性与性能

> 简单性：少出现同步声明
并发性：少进入同步块
简单性比如直接将方法声明成`synchronized`，并发性则在方法体内细粒度的加同步块。

分得太细要注意原子性，避免破坏代码的正确性。

在简单性与并发性，简单性和性能的权衡。
分得太细要注意原子性，避免破坏安全性。

**当执行时间较长的计算或者可能无法快速完成的操作时（如：网络I/O，控制台I/O），一定不要持有锁。**


## 唠唠
这本书依然有价值吗？
https://stackoverflow.com/a/10214606