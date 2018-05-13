# 《Java并发编程实战》读书笔记 part2

## 对象的共享

### 可见性

#### 失效数据

ReaderThread线程访问的ready一直为false，程序永远不会停下来

可能有重排序的情况，输出0

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

从Java的内存模型讲起，《深入理解Java虚拟机》关于现代处理器和内存的一般交互架构

![](/assets/jcip_note/jvm_memory_hard.png)

Java内存模型的类似抽象如图，其中工作内存和上图的高速缓存类比：![](/assets/jcip_note/jvm_memory_ab.png)也就是说工作内存（高速缓存）的值并没有因为主内存的值更新了而失效。

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

### 发布与逸出

### 线程封闭

#### ad-hoc线程封闭

#### 栈封闭

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

## 对象的组合

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



