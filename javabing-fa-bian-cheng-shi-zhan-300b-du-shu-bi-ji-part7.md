# 《Java并发编程实战》读书笔记 part7
## 显示锁

Java5之前协调对共享对象的访问时可以使用的机制只有synchronized和volatile。Java5增加了新的机制：ReentrantLock。

### Lock与ReentrantLock
Lock接口定义了一组抽象的加锁操作：
```java
public interface Lock {
	void lock();
	void lockInterruptibly() throws InterruptedException;
	boolean tryLock();
	boolean tryLock(long timeout, TimeUnit unit) throws InterruptedException;
	void unlock();
	Condition newCondition();
}
```
ReentrantLock实现了Lock接口，并提供了与synchronized相同的互斥性和内存可见性。使用的一般形式：
```java
Lock lock = new ReentrantLock();
...
lock.lock();
try {
	// update object state
	// catch exceptions and restore invariants if necessary
} finally {
	lock.unlock(); // 与内置锁相比，显示锁必须手动清除，因此显示锁也更加“危险”
}
```
ReentrantLock的使用场景：
1. 可定时的与可轮询的锁
	前面提到的死锁的情况，恢复程序的唯一办法就是重启启动程序。
	**可定时的与可轮询的锁**提供了另一种选择：避免死锁的发生。
	- 可轮询锁：使用tryLock，如果锁获取不全，那么就释放已有的锁，并随机等待一定时间重试。
	- 可定时锁：及时失败
2. 可中断的锁操作：lockInterruptibly()
3. 非块结构的加锁：交替锁（Hand-Over-Hand Locking）【例子来自《七周七并发模型》】
	```java
	class ConcurrentSortedList {
	  private class Node {
	    int value;
	    Node prev;
	    Node next;
	    ReentrantLock lock = new ReentrantLock();

	    Node() {}

	    Node(int value, Node prev, Node next) {
	      this.value = value; this.prev = prev; this.next = next;
	    }
	  }

	  private final Node head;
	  private final Node tail;

	  public ConcurrentSortedList() {
	    head = new Node(); tail = new Node();
	    head.next = tail; tail.prev = head;
	  }

	  public void insert(int value) {
	    Node current = head;
	    current.lock.lock(); 
	    Node next = current.next;
	    try {
	      while (true) {
	        next.lock.lock(); 
	        try {
	          if (next == tail || next.value < value) { 
	            Node node = new Node(value, current, next); 
	            next.prev = node;
	            current.next = node;
	            return; 
	          }
	        } finally { current.lock.unlock(); } 
	        current = next;
	        next = current.next;
	      }
	    } finally { next.lock.unlock(); } 
	  }

	  public int size() {
	    Node current = tail;
	    int count = 0;
		
	    while (current.prev != head) {
	      ReentrantLock lock = current.lock;
	      lock.lock();
	      try {
	        ++count;
	        current = current.prev;
	      } finally { lock.unlock(); }
	    }
		
	    return count;
	  }

	  public boolean isSorted() {
	    Node current = head;
	    while (current.next.next != tail) {
	      current = current.next;
	      if (current.value < current.next.value)
	        return false;
	    }
	    return true;
	  }
	}
	```
插入操作，如图：
![](/assets/jcip_note/hand-over-hand_locking.png)

### 性能考虑因素
在Java5，ReentrantLock比内置锁有更好的竞争性能，Java6改进了内置锁的管理算法后，二者性能基本相当，如图：
![](/assets/jcip_note/reentrantLock_perfoance_on_java5_and_java6.png)

当然，性能是一个不断变化的指标，如果昨天的测试基准中发现X比Y更快，那么今天就可能已经过时了。

### 公平性
ReentrantLock的构造函数提供了两种公平的选择：
1. 创建一个非公平的锁（默认）：如果在发出请求的同时该锁的状态变为可用，那么这个线程跳过队列中所有等待的线程并获得这个锁。（允许“插队”）
2. 创建一个公平的锁：线程将按照它们发出请求的顺序来获得锁。

比较Map的性能测试：由公平和非公平的ReentrantLock包装的HashMap的性能，如图：
![](/assets/jcip_note/lock_fair_nonfair.png)

在激烈竞争的情况下，非公平的性能高于公平锁的性能的一个原因是：在恢复一个被挂起的线程与该线程真正开始运行之间存在着严重的延迟。
当持有锁的时间相对较长，或者请求锁的平均时间间隔较长，那么应该使用公平锁。在这种情况下，“插队”带来的吞吐量提升则可能不会出现。

与默认的ReentrantLock一样，内置锁并不会提供确定的公平性保证，但在大多数情况下，在锁实现统计上的公平性保证已经足够了。

### 在synchronized和ReentrantLock之间进行选择
ReentrantLock在加锁和内存上提供的语义与内置锁相同，此外它还提供了一些其他的功能，包括定时的锁等待、可中断的锁等待、公平性、实现非块结构加锁。
但内置锁仍然具有很大的优势：
1. 开发人员熟悉，并且简洁紧凑，而且大多数代码都已经使用了内置锁。
2. 在线程转储中能给出在哪些调用帧中获得了哪些锁，并且能够检测和识别发生死锁的线程。

在一些内置锁无法满足需求的情况下，ReentrantLock可以作为一种高级工具。当需要一些高级功能时才应该使用ReentrantLock，这些功能包括：可定时的、可轮询的、可中断的锁获取操作，公平以及非块结构的锁。否则，还是应该优先使用synchronized。

未来更可能会提升synchronized而不是ReentrantLock的性能。因synchronized是内置属性，它能执行一些优化比如对线程封闭的锁对象的锁消除优化，通过增加锁的粒度来消除内置锁的同步，而如果基于类库锁来实现这些优化，则可能性不大。

### 读—写锁
对于维护数据的完整性来说，互斥通常是一种过于强硬的加锁规则，因此也不必要的限制了并发性。只要每个线程都确保能读取到最新的数据，并且在读取数据时不会有其他线程修改数据，那么可以使用读/写锁：一个资源可以被多个读操作访问，或者被一个写操作访问，但二者不可同时进行。
ReadWriteLock接口定义：
```java
public interface ReadWriteLock {
	Lock readLock();
	Lock writeLock();
}
```
读写锁是一种优化措施，在实际情况中，对于多处理器系统被频繁读取的数据结构，读-写锁能提高性能，在其他情况下，读-写锁被独占锁性能要略差一些。

ReadWriteLock 中一些可选实现：
1. 释放优先：释放写锁时，优先选择读线程、写线程、还是最早请求的线程？
2. 读线程插队：读持续持有锁，写线程在等待时，新到达的读线程能否立即获得锁，还是应该在写线程后面等待？如果允许读线程插队到写线程前，能提高并发性，但可能造成写线程发生饥饿
3. 重入性
4. 降级：持有写锁的线程能否不释放该锁而获得读锁？这可能使得写锁被“降级”为读锁，同时不允许其他写线程修改被保护资源。
5. 升级：读锁能否优先其他在等待的读/写线程升级为写锁？大多数实现不支持升级，因为容易造成死锁（比如两个读锁同时请求升级为写锁）
ReentrantReadWriteLock为两种锁都提供了可重入的加锁语义，和ReentrantLock类似，有公平、非公平两种构造：
- 公平：等待时间最长优先获得锁。
- 非公平：线程获取访问许可的顺序是不确定的，写线程可降级为读线程，但读线程不能升级为写线程。
大多数情况下ConcurrentHashMap性能已经足够好了，但如果需要非散列的Map实现方式，可以用读-写锁包装，用ReentrantReadWriteLock包装Map：
```java
public class ReadWriteMap <K,V> {
    private final Map<K, V> map;
    private final ReadWriteLock lock = new ReentrantReadWriteLock();
    private final Lock r = lock.readLock();
    private final Lock w = lock.writeLock();

    public ReadWriteMap(Map<K, V> map) {
        this.map = map;
    }

    public V put(K key, V value) {
        w.lock();
        try {
            return map.put(key, value);
        } finally {
            w.unlock();
        }
    }
    public V get(Object key) {
        r.lock();
        try {
            return map.get(key);
        } finally {
            r.unlock();
        }
    }
	...
}
```
下面给出分别用ReentrantReadWriteLock和ReentrantLock来分装的ArrayList的吞吐量对比（测试操作：每个操作随机选择一个值并在容器中查找这个值，并且只有少量的操作会修改容器的内容）。如图：
![](/assets/jcip_note/read-write_lock_performance.png)

