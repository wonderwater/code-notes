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

当`ThisEscape`发布`EventListener`（`source.registerListener`)，也隐式地发布了`ThisEscape.this`，但是`ThisEscape.this`可能没有初始化完成。

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
多线程间不共享数据

#### ad-hoc线程封闭
指维护线程封闭性的职责完全有程序实现来承担。

#### 栈封闭
在栈封闭中，只能通过局部变量才能访问对象



```
public int loadTheArk(Collection<Animal> candidates) {
    SortedSet<Animal> animals;
    int ret = 0;
    // 线程封闭，animals对象不可逸出
    animals = new TreeSet<Animal>();
    animals.addAll(candidates);
    
    for (Animal a : animals) {
        // ...
    }
    return ret;
}
```



但要小心的是，只有编写代码的开发人员才知道那些对象需要封闭到执行线程中，以及被封闭的对象是否是线程安全的。如果没有明确地说明这些需求，那么后续的维护人员很容易错误地使对象逸出。

#### ThreadLocal类

### 不变性

#### Final域

### 安全发布

#### 不正确的发布：正确的对象被破坏

#### 不可变对象与初始化安全性

#### 安全发布的常用模式

#### 事实不可变对象

#### 可变对象

#### 安全地共享对象

## 第四章 对象的组合

### 设计线程安全的类

#### 收集同步需求

#### 依赖状态的操作

#### 状态的所有权

### 实例封闭

#### Java监视器模式

### 线程安全性的委托

#### 独立地状态变量

#### 当委托失效时

#### 发布低层的状态变量

### 在现有的线程安全类中添加功能

#### 客户端加锁机制

#### 组合

### 将同步策略文档化



