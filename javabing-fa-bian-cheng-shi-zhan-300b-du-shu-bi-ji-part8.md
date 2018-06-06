# 《Java并发编程实战》读书笔记 part8
## 构建自定义的同步工具

创建状态依赖类的最简单方法通常是在类库中现有状态依赖类的基础上进行构造，如果类库没有提供所需要的功能，那么还可以使用Java语言和类库提供的底层机制构造自己的同步机制，包括内置的条件队列、显示的Condition对象、AbstractQueuedSynchronizer框架。

### 状态依赖性的管理
可阻塞的状态依赖操作的形式如下：
```java
acquire lock on object state
while (precondition does not hold) {
	release lock
	wait until precondition might hold
	optionally fail if interrupted or timeout expires
	reacquire lock
}
perform action
release lock
```
构成前提条件的状态变量必须由对象的锁保护，从而使它们在测试前提条件的同时保持不变。如果条件不满足时，就必须释放锁，以便其他线程可以修改对象的状态，再次测试前提条件之前，再重新获取锁。

在生产者-消费者的设计中常会用到像ArrayBlockingQueue这样的有界缓存。
接下来介绍有界缓存的实现，这是基类：
```java
@ThreadSafe
public abstract class BaseBoundedBuffer <V> {
    @GuardedBy("this") private final V[] buf;
    @GuardedBy("this") private int tail;
    @GuardedBy("this") private int head;
    @GuardedBy("this") private int count;

    protected BaseBoundedBuffer(int capacity) {
        this.buf = (V[]) new Object[capacity];
    }

    protected synchronized final void doPut(V v) {
        buf[tail] = v;
        if (++tail == buf.length)
            tail = 0;
        ++count;
    }

    protected synchronized final V doTake() {
        V v = buf[head];
        buf[head] = null;
        if (++head == buf.length)
            head = 0;
        --count;
        return v;
    }

    public synchronized final boolean isFull() {
        return count == buf.length;
    }

    public synchronized final boolean isEmpty() {
        return count == 0;
    }
}
```
在子类中，我们将实现put和take操作。
- 第一个实现：将前提条件失败传递给调用者
```java
@ThreadSafe
public class GrumpyBoundedBuffer <V> extends BaseBoundedBuffer<V> {
	public GrumpyBoundedBuffer(int size) { super(size); }
	
    public synchronized void put(V v) throws BufferFullException {
        if (isFull())
            throw new BufferFullException();
        doPut(v);
    }

    public synchronized V take() throws BufferEmptyException {
        if (isEmpty())
            throw new BufferEmptyException();
        return doTake();
    }
}
```
这个实现很简单，但使用起来并非如此，**异常应该用于发生异常条件的情况中。**
“缓存已满”并不是有界缓存的一个异常条件，就像“红灯”不代表交通信号灯出现异常。
调用take的客户端代码的示例如下——并不是很漂亮：
```java
while (true) {
    try {
        String item = buffer.take();
        // use item
        break;
    } catch (BufferEmptyException e) {
        Thread.sleep(SLEEP_GRANULARITY);
    }
}
```
另一种方法是，当缓存处于某种错误的状态时返回一个错误值，这只是一种改进，因为并没有放弃异常机制，没有解决根本问题：调用者必须自行处理前提条件失败的情况。
客户端代码中可以不进入休眠，而是直接重新调用take方法，这样的方法称为忙等待或自旋等待。如果缓存在长时间不会发生变化，那么这样会消耗过多的CPU时间；但如果缓存状态在刚调用sleep就立即发生变化，那么将不必要地休眠一段时间。
因此，客户代码必须要在二者之间进行选择：

 1. 要么容忍自旋导致的CPU时钟周期的浪费
 2. 要么容忍由于休眠导致的低响应性
 
 还有一种办法：Thread.yield


- 第二个实现：通过轮询与休眠来实现简单的阻塞
```java
@ThreadSafe
public class SleepyBoundedBuffer <V> extends BaseBoundedBuffer<V> {
    public SleepyBoundedBuffer(int size) { super(size); }

    public void put(V v) throws InterruptedException {
        while (true) {
            synchronized (this) {
                if (!isFull()) {
                    doPut(v);
                    return;
                }
            }
            Thread.sleep(SLEEP_GRANULARITY);
        }
    }

    public V take() throws InterruptedException {
        while (true) {
            synchronized (this) {
                if (!isEmpty())
                    return doTake();
            }
            Thread.sleep(SLEEP_GRANULARITY);
        }
    }
}
```
缓存代码必须在持有锁的时候才能进行测试相应的状态条件，因为表示状态条件的变量是有缓存锁保护的。从调用者的角度看，如果某个方法可以执行，那么立即执行，否则阻塞，无需处理失败和重试。
SleepyBoundedBuffer对调用者提出了一个新的需求：处理InterruptedException。通过中断可支持取消操作。
- 第三个实现：条件队列（Condition queues）
“条件队列”这个名字来源于：它使得一组线程（称之为等待线程集合）能够通过某种方式来等待特定条件变成真。
正如每个Java对象都可以作为一个锁，每个对象同样可以作为一个条件队列，并且Object中的wait、notify、notifyAll方法构成内部条件队列的API。
```java
@ThreadSafe
public class BoundedBuffer <V> extends BaseBoundedBuffer<V> {
    // CONDITION PREDICATE: not-full (!isFull())
    // CONDITION PREDICATE: not-empty (!isEmpty())
    public BoundedBuffer(int size) { super(size); }

    // BLOCKS-UNTIL: not-full
    public synchronized void put(V v) throws InterruptedException {
        while (isFull())
            wait();
        doPut(v);
        notifyAll();
    }

    // BLOCKS-UNTIL: not-empty
    public synchronized V take() throws InterruptedException {
        while (isEmpty())
            wait();
        V v = doTake();
        notifyAll();
        return v;
    }

    // BLOCKS-UNTIL: not-full
    // Alternate form of put() using conditional notification
    public synchronized void alternatePut(V v) throws InterruptedException {
        while (isFull())
            wait();
        boolean wasEmpty = isEmpty();
        doPut(v);
        if (wasEmpty)
            notifyAll();
    }
}
```
与使用“休眠”的有界缓存相比，条件队列并没有改变原来的语义。
### 使用条件队列
- 条件谓词：首先找出对象在哪个条件谓词上等待。
条件谓词是使某个操作成为状态依赖操作的前提条件，对take来说，它的条件谓词就是“缓存不为空”。
> 将与条件队列相关联的条件谓词以及在这些条件谓词上等待操作都写入文档。

 条件等待中存在一种重要的三元关系：包括加锁、wait方法、一个条件谓词。
每一次wait调用都会隐式地与特定的条件谓词关联起来。当调用某个特定条件谓词的wait时，调用者必须已经持有与条件队列相关的锁，并且这个锁必须保护着构成条件谓词的状态变量。

- 过早唤醒问题
wait方法的返回并不一定意味着线程正在等待的条件谓词已经变成真。
1. 内置条件队列可以与多个条件谓词一起使用。当一个线程由于调用notifyAll而醒来时，并不意味着该线程**正在等待的条件谓词**已经变成真。（比如烤面包机和咖啡机共用一个铃声，响铃时，必须再查看是哪个设备响了）
2. 另外，wait方法还可以“假装”返回，也许发出通知的线程调用notifyAll时，条件谓词已经变成真，但重新获取锁时可能将再次变成假。因为在线程被唤醒带wait重新获取锁这段时间内有其他线程已获取这把锁，并修改了对象的状态。

状态依赖方法的标准形式：
```java
void stateDependentMethod() throws InterruptedException {
	// condition predicate must be guarded by lock
	synchronized(lock) {
		while (!conditionPredicate())
			lock.wait();
		// object is now in desired state
	}
}
```
当使用条件等待时（比如Object.wait或Condition.await）：

    1. 通常都有一个条件谓词——包括一些对象状态的测试，线程在执行前必须首先通过这些测试。
    2. 在调用wait之前测试条件谓词，并且从wait返回时再进行测试。
    3. 在一个循环中调用wait。
    4. 确保使用与条件队列相关的锁来保护构成条件谓词的各个状态变量。
    5. 当调用wait、notify、notifyAll等方法时，一定要持有与条件队列相关的锁。
    6. 在检测条件谓词之后以及开始执行相应的操作之前，不要释放锁。

- 丢失的信号

除了死锁、活锁，还有一种活跃性故障是丢失的信号。
丢失的信号：线程必须等待一个已经为真的条件，但在开始等待之前没有检查条件谓词。现在，线程将等待一个已经发生过的事件。按状态依赖方法的标准形式设计条件等待，那么就不会发生信号丢失。

- 通知
> 每当在等待一个条件时，一定要确保在条件谓词变为真时通过某种方式发出通知。

由于多个线程可以基于不同条件谓词在同一个条件队列上等待，因此如果使用notify而不是notifyAll，那么将是一种危险的操作。因为单一的通知很容易导致类似于信号丢失的问题（被不该唤醒的线程“劫持”了信号）。

只有同时满足以下两个条件，才可以用单一的notify而不是notifyAll：
1. 所有等待线程的类型相同。只有一个条件谓词与条件队列相关，并且每个线程在从wait返回后将执行相同的操作。
2. 单进单出。在条件谓词变量上的每次通知，最多只能唤醒一个线程来执行。

大多数类并不满足这些需求，因此普遍认可的做法是优先使用notifyAll而不是notify。
有些开发者不并赞同这种“普遍认可的做法”。比如当只有一个线程可以执行时，10个线程在一个条件队列上等待，那么调用notifyAll将唤醒每个线程，并使得它们在锁上发生竞争，然后，它们中大多数或全部又都回到休眠状态，因而，在每个线程执行一个事件的同时，将出现大量的上下文切换操作以及发生竞争的锁获取操作。（最坏的情况下，在使用notifyAll时将导致`O(n^2)`次唤醒操作，而实际只需要`n`次唤醒）。这是“性能考虑因素与安全性考虑因素相互矛盾”的另一种情况。

我们可以优化`BoundedBuffer.put`的使用条件通知：
```java
// BLOCKS-UNTIL: not-full
// Alternate form of put() using conditional notification
public synchronized void put(V v) throws InterruptedException {
    while (isFull())
        wait();
    boolean wasEmpty = isEmpty();
    doPut(v);
    if (wasEmpty)
        notifyAll();
}
```
一个可重新开关的阀门类：
`ThreadGate`可以打开和关闭，并提供await方法，该方法能一直阻塞到阀门打开。
```java
@ThreadSafe
public class ThreadGate {
    // CONDITION-PREDICATE: opened-since(n) (isOpen || generation>n)
    @GuardedBy("this") private boolean isOpen;
    @GuardedBy("this") private int generation;

    public synchronized void close() {
        isOpen = false;
    }

    public synchronized void open() {
        ++generation;
        isOpen = true;
        notifyAll();
    }

    // BLOCKS-UNTIL: opened-since(generation on entry)
    public synchronized void await() throws InterruptedException {
        int arrivalGeneration = generation;
        while (!isOpen && arrivalGeneration == generation)
            wait();
    }
}
```

使用条件通知或单次通知时，一些约束条件使得子类化过程变得更加复杂：
1. 对于状态依赖的类，要么将其等待和通知等协议完全向子类公开（并写入正式文档），要么完全阻止子类参与到等待和通知等过程中。
2. 完全禁止子类化，例如将类声明为final类型，或者将条件队列，锁和状态变量等隐藏起来。

### 显式的Condition对象

Condition接口定义：
```java
public interface Condition {
	void await() throws InterruptedException;
	boolean await(long time, TimeUnit unit)
	throws InterruptedException;
	long awaitNanos(long nanosTimeout) throws InterruptedException;
	void awaitUninterruptibly();
	boolean awaitUntil(Date deadline) throws InterruptedException;
	void signal();
	void signalAll();
}
```
> 注意区别内置条件队列的wait、notify、notifyAll方法。

内置条件队列的问题：每个内置锁都只能有一个相关联的条件队列，因而多个线程可能在同一个条件队列上等待不同的条件谓词，并且在最常见的加锁模式下公开条件队列对象。
与内置条件队列不同的是，对于每个Lock，可以有任意数量的Condition对象。
使用显示条件变量的有界缓存实现：
```java
@ThreadSafe
public class ConditionBoundedBuffer <T> {
    protected final Lock lock = new ReentrantLock();
    // CONDITION PREDICATE: notFull (count < items.length)
    private final Condition notFull = lock.newCondition();
    // CONDITION PREDICATE: notEmpty (count > 0)
    private final Condition notEmpty = lock.newCondition();
    private static final int BUFFER_SIZE = 100;
    @GuardedBy("lock") private final T[] items = (T[]) new Object[BUFFER_SIZE];
    @GuardedBy("lock") private int tail, head, count;

    // BLOCKS-UNTIL: notFull
    public void put(T x) throws InterruptedException {
        lock.lock();
        try {
            while (count == items.length)
                notFull.await();
            items[tail] = x;
            if (++tail == items.length)
                tail = 0;
            ++count;
            notEmpty.signal();
        } finally {
            lock.unlock();
        }
    }

    // BLOCKS-UNTIL: notEmpty
    public T take() throws InterruptedException {
        lock.lock();
        try {
            while (count == 0)
                notEmpty.await();
            T x = items[head];
            items[head] = null;
            if (++head == items.length)
                head = 0;
            --count;
            notFull.signal();
            return x;
        } finally {
            lock.unlock();
        }
    }
}
```

### Synchronizer剖析
ReentrantLock和Semaphore这两个接口存在许多共同点。
1. 都可用做一个“阀门”，即每次只允许一定数量的线程通过，当线程到达阀门，可以通过或者等到，还可以取消。
2. 支持可中断、不可中断、可限时获取操作
3. 支持公平、非公平队列操作
事实上，可以通过锁来实现信号量：
```java
@ThreadSafe
public class SemaphoreOnLock {
    private final Lock lock = new ReentrantLock();
    // CONDITION PREDICATE: permitsAvailable (permits > 0)
    private final Condition permitsAvailable = lock.newCondition();
    @GuardedBy("lock") private int permits;

    SemaphoreOnLock(int initialPermits) {
        lock.lock();
        try {
            permits = initialPermits;
        } finally {
            lock.unlock();
        }
    }

    // BLOCKS-UNTIL: permitsAvailable
    public void acquire() throws InterruptedException {
        lock.lock();
        try {
            while (permits <= 0)
                permitsAvailable.await();
            --permits;
        } finally {
            lock.unlock();
        }
    }

    public void release() {
        lock.lock();
        try {
            ++permits;
            permitsAvailable.signal();
        } finally {
            lock.unlock();
        }
    }
}
```
事实上，它们在实现时都使用了一个共同的基类，即AbstractQueuedSynchronizer（AQS），除了ReentrantLock、Semaphore基于AQS构建，还有CountDownLatch、ReentrantReadWriteLock、SynchronousQueue、FutureTask等。

### AbstractQueuedSynchronizer
在基于AQS构建的同步器类中，最基本的操作包括各种形式的获取操作和释放操作。
获取操作是一种依赖于状态的操作，并通常会阻塞：

1. 使用锁或信号量，获取的是锁或许可，并且调用者可能会一直等待到同步器类处于可获取的状态。
2. 使用CountDownLatch，“获取”操作意味者“等待直到闭锁到达结束状态”
3. 使用FutureTask，“获取”操作意味着“等待并直到任务已经完成”
释放操作不是一个可阻塞的操作，执行“释放”时，所有在请求时被阻塞的线程都会开始执行。

如果一个类想成为状态依赖的类，那么它必须拥有一些状态。AQS负责管理同步容器类的状态，它管理了一个整数状态信息，可以通过getState、setState、compareAndSetState等protected类型方法进行操作。

1. ReentrantLock用它表示所有者线程已经重复获取该锁的次数
2. Semaphore用它表示剩余许可数量
3. FutureTask用它表示任务的状态（尚未开始、正在运行、已完成、已取消）

AQS中获取操作和释放操作的标准形式：
```java
boolean acquire() throws InterruptedException {
	while (state does not permit acquire) { // 当前状态不允许获取操作
		if (blocking acquisition requested) { // 需要阻塞获取请求
			enqueue current thread if not already queued // 如果当前线程不在队列中，则将其插入队列
			block current thread // 阻塞当前线程
		}
		else
			return failure // 返回失败
	}
	possibly update synchronization state // 可能更新同步器的状态
	dequeue thread if it was queued // 如果线程位于队列中，则将其移出队列
	return success // 返回成功
}
void release() {
	update synchronization state // 更新同步器的状态
	if (new state may permit a blocked thread to acquire) // 新的状态允许某个被阻塞的线程获取成功
		unblock one or more queued threads // 解除队列中一个或多个线程的阻塞状态
}
```

一个简单二元闭锁的实现：
```java
@ThreadSafe
public class OneShotLatch {
    private final Sync sync = new Sync();

    public void signal() {
        sync.releaseShared(0);
    }

    public void await() throws InterruptedException {
        sync.acquireSharedInterruptibly(0);
    }

    private class Sync extends AbstractQueuedSynchronizer {
        protected int tryAcquireShared(int ignored) {
            // Succeed if latch is open (state == 1), else fail
            return (getState() == 1) ? 1 : -1;
        }

        protected boolean tryReleaseShared(int ignored) {
            setState(1); // Latch is now open
            return true; // Other threads may now be able to acquire

        }
    }
}
```
OneShotLatch可以通过扩展AQS来实现，而不是将一些功能委托给AQS，但这种做法并不合理：
1. 这样将破坏OnShotLatch接口（只有两方法）的简洁性。
2. 虽然AQS的公共方法不允许调用者破坏闭锁状态，但调用者仍可以很容易误用它们。

### java.util.concurrent同步类中的AQS

- ReentrantLock

只支持独占方式的获取操作，因此它实现了tryAcquire、tryRelease、isHeldExclusively。
非公平版本的tryAcquire的实现：
```java
protected boolean tryAcquire(int ignored) {
	final Thread current = Thread.currentThread();
	int c = getState();
	if (c == 0) {
		if (compareAndSetState(0, 1)) {
			owner = current;
			return true;
		}
	} else if (current == owner) {
		setState(c+1);
		return true;
	}
	return false;
}
```
使用owner区分获取操作是重入的，还是竞争的。
用同步状态保存获取计数

- Semaphore与CountDownLatch

Semaphore用同步状态保存当前的可用许可。tryAcquireShared和tryReleaseShared实现如下：
```java
protected int tryAcquireShared(int acquires) {
	while (true) {
		int available = getState();
		int remaining = available - acquires;
		if (remaining < 0 || compareAndSetState(available, remaining))
			return remaining;
	}
}
// 返回值表示在这次释放操作中解除了其他线程的阻塞
protected boolean tryReleaseShared(int releases) {
	while (true) {
		int p = getState();
		if (compareAndSetState(p, p + releases))
			return true;
	}
}
```
CountDownLatch使用AQS的方式和Semaphore很像：在同步状态中保存的是当前的计数值。countDown调用release，从而导致计数值递减，当计数值为0时，解除所有等待线程的阻塞。await调用acquire，当计数值为0时，acquire立即返回，否则阻塞。

- FutureTask

初看上去FutureTask不像一个同步器，但Future.get的语义类似于闭锁的语义——如果发生了某个事件（由FutureTask表示的任务执行完成或被取消），那么线程就可以恢复执行，否则这些线程将停留在队列中并直到该事件发生。
FutureTask用AQS的同步状态保存任务的状态（正在运行、已完成、已取消）。还有些额外的变量，比如保存计算结果或抛出的异常。还维护了一个引用，指向正在执行的计算任务线程（如果它当前处于运行状态），因而如果任务取消，该线程就会中断。
- ReentrantReadWriteLock

ReentrantReadWriteLock表示存在两个锁：一个读取锁和一个写入锁。当个AQS子类同时管理读取加锁和写入加锁，用一个16位的状态表示写入锁的计数，用另一个16位状态表示写入锁的计数。在读取锁上的操作将使用共享的获取、释放方法，在写入锁的操作使用独占的获取、释放方法。
AQS在内部维护了一个等待线程队列，其中记录了某个线程请求的是独占访问还是共享访问。在ReentrantReadWriteLock中，当锁可用时，如果队列头部的线程执行的是写入操作，那么线程会得到这个锁，如果位于队列头部的线程执行读取方法，那么队列中在第一个写入线程之前的所有线程都将获取这个锁。

