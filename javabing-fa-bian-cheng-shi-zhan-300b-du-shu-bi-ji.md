# 《Java并发编程实战》读书笔记

## 一、简介

线程的**优势**：
--
1. 发挥多处理器的强大能力
2. 建模的简单性
3. 异步时间的简单处理
4. 响应更灵敏的用户界面

线程的**风险**：
--
* 安全性问题
如果错误的估计程序的执行顺序，可能带来各种危险
``` java
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
> solution: `public synchronized int getNext`, `private volatile int value;`
> water书上的说明没有加`volatile`?

* 活跃性问题

比如死锁、饥饿、活锁
> 安全性是希望不发生错误的事情，比如上面的代码因为多线程执行顺序可能出现和预期不一致的情况。
> 活跃性是当某个操作无法继续下去而发生的，可能运行的好好的，突然间就处于一种奇妙的状态，也不退出，也不执行业务代码，可能也几乎不耗资源。

* 性能问题
多线程的切换影响：保存和恢复执行上下文，丢失局部性，cpu的线程调度消耗

## 线程安全性

线程安全性：核心问题是正确性（某个类的行为与其规范完全一致）
 
  
    
xx
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
xx
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
xx
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x
x