# 《Java并发编程实战》读书笔记 part9
## 原子变量与非阻塞同步机制
近年来，在并发算法领域的大多数研究都侧重于非阻塞算法，这种算法底层的原子机器指令（比如比较并交换指令）代替锁来确保数据在并发访问中的一致性。非阻塞算法被广泛地用于操作系统和JVM中实现线程、进程调度机制、垃圾回收、锁和其他数据结构。
与基于锁的方案相比，非阻塞算法在设计和实现上要复杂得多，但在可伸缩性和活跃性上有着巨大优势。

### 锁的劣势

锁的作用：通过使用一致的锁定协议对共享状态的访问，可以确保无论哪个线程持有守护变量的锁，都能采用独占方式来访问这些变量，并且对变量的任何修改对随后获得这个锁的其他线程都是可见的。
- 在挂起和恢复线程等过程中存在很大的开销，并且通常存在较长时间的中断。
- 如果基于锁的类中包含细粒度的操作（如同步容器类，大多数方法只有少量操作），那么在锁上存在激烈的竞争时，调度开销和工作开销的比值会很高。
- 当一个线程在等待锁时，它不能做任何其他事情。如果持有锁的线程被延迟执行（比如发生缺页错误、调度延迟等），那么所有需要这个锁的其他线程都无法执行下去，尤其是持有锁的线程相比其他线程优先级要低时，这是个严重的问题——也称为优先级反转（Priority Inversion）。
- 锁定方式对细粒度的操作（比如递增计数器）来说仍然是一种高开销机制。

### 硬件对并发的支持
独占锁是一项悲观技术——它假设最坏情况，并且只有在确保其他线程不会造成干扰的情况下才能执行下去。
对细粒度操作，有种更高效乐观的方法，可以在不发生干扰的情况下完成更新操作。
在针对多处理器而设计的处理器提供了一些特殊指令，用于管理对共享数据的并发访问。在早期的处理器支持，比如
- 测试并设置（Test-and-Set）
- 获取并递增（Fetch-and-Increment）
- 交换（Swap）

现在几乎所有处理器都包含某种形式的原子读-写-改指令，比如
- 比较并交换（Compare-and-Swap，即CAS）
- 关联加载/条件存储（Load-Linked/Store-Conditional）

大多数处理器架构（包括IA32和Sparc）采用的方法时CAS指令；
其他处理器（比如PowerPC）采用一对指令实现相同功能：关联加载与条件存储。

CAS包含三个操作数：
V：需要读写的内存位置
A：进行比较的值
B：拟写入的新值

CAS指令的模拟：
```java
@ThreadSafe
public class SimulatedCAS {
    @GuardedBy("this") private int value;

    public synchronized int get() { return value; }

    public synchronized int compareAndSwap(int expectedValue,
                                           int newValue) {
        int oldValue = value;
        if (oldValue == expectedValue)
            value = newValue;
        return oldValue;
    }

    public synchronized boolean compareAndSet(int expectedValue,
                                              int newValue) {
        return (expectedValue
                == compareAndSwap(expectedValue, newValue));
    }
}
```

CAS的典型使用模式是：首先从V中读取值A，并根据A计算新值B，然后在通过CAS以原子方式将V中的值由A变成B。由于CAS能检测其他线程的干扰，因此即使不使用锁也能实现原子的读-改-写操作序列。
基于CAS的非阻塞计数器：
```java
@ThreadSafe
public class CasCounter {
    private SimulatedCAS value;

    public int getValue() {
        return value.get();
    }

    public int increment() {
        int v;
        do {
            v = value.get();
        } while (v != value.compareAndSwap(v, v + 1));
        return v + 1;
    }
}
```
看上去它比基于锁的计数器性能应该更差一些，因为它需要更多的操作和更复杂的控制流，并且还依赖看似复杂的CAS操作。但实际上，当竞争程度不高是，基于CAS的计数器在性能上远远超过基于锁的计数器。CAS大多数情况下都能执行成功，因此硬件能够正确地预测while循环中的分支，从而把复杂控制逻辑的开销降至最低。

通常，反复重试是一种合理的策略，但在一些竞争激烈的情况，更好的方式是在重试前首先等待一段时间或者回退，从而避免造成活锁问题。

CAS的问题：
1. 它将使调用者处理竞争问题（通过重试、回退、放弃），而锁能自动处理竞争问题。
2. CAS的性能随着处理器数量的不同而变化很大。一个很管用的经验法则是：在大多数处理器上，在无竞争的锁获取和释放的“快速代码路径”上的开销，大约是CAS开销的两倍。

### 原子变量类
原子变量类相当于一种泛化的volatile变量，能够支持原子的和有条件的读-改-写操作。
共有12个原子变量类，可分为4组：
1. 标量类（Scalar）：AtomicInteger、AtomicLong、AtomicBoolean、AtomicReference。（模拟其他基本类型的原子变量，可以将short、byte等与int类型进行转换，使用floatToIntBits或doubleToLongBits来转换浮点数）
2. 更新器类
3. 数组类
4. 复合变量类
5. 
原子的标量类扩展了Number类，但并没有扩展一些基本类型的包装类，比如Integer，实际上，也做不到，因为基本类型的包装类是不可修改的，而原子变量是可修改的，因此原子变量类中同样没有重新定义hashCode或equals方法。
通过CAS来维持包含多个变量的不变性条件：
```java
@ThreadSafe
public class CasNumberRange {
    @Immutable
    private static class IntPair {
        // INVARIANT: lower <= upper
        final int lower;
        final int upper;
        ...
    }

    private final AtomicReference<IntPair> values =
            new AtomicReference<IntPair>(new IntPair(0, 0));

    public int getLower() { return values.get().lower; }

    public int getUpper() { return values.get().upper; }

    public void setLower(int i) {
        while (true) {
            IntPair oldv = values.get();
            if (i > oldv.upper)
                throw new IllegalArgumentException("Can't set lower to " + i + " > upper");
            IntPair newv = new IntPair(i, oldv.upper);
            if (values.compareAndSet(oldv, newv))
                return;
        }
    }
	...
}
```

- 性能比较：锁和原子变量
为了说明锁和原子变量之间的可伸缩性差异。构造几种伪随机数生成器的实现。
一种是基于ReentrantLock，另一种基于AtomicInteger。
```java
@ThreadSafe
public class ReentrantLockPseudoRandom extends PseudoRandom {
    private final Lock lock = new ReentrantLock(false);
    private int seed;

    ReentrantLockPseudoRandom(int seed) {
        this.seed = seed;
    }

    public int nextInt(int n) {
        lock.lock();
        try {
            int s = seed;
            seed = calculateNext(s);
            int remainder = s % n;
            return remainder > 0 ? remainder : remainder + n;
        } finally {
            lock.unlock();
        }
    }
}
```
```java
@ThreadSafe
public class AtomicPseudoRandom extends PseudoRandom {
    private AtomicInteger seed;

    AtomicPseudoRandom(int seed) {
        this.seed = new AtomicInteger(seed);
    }

    public int nextInt(int n) {
        while (true) {
            int s = seed.get();
            int nextSeed = calculateNext(s);
            if (seed.compareAndSet(s, nextSeed)) {
                int remainder = s % n;
                return remainder > 0 ? remainder : remainder + n;
            }
        }
    }
}
```
在竞争程度较高情况下的Lock与AtomicInteger的性能比较，如图：
![](/assets/jcip_note/lock_and_atomicInteger_performance_under_high_contention.png)

在竞争程度适中情况下的Lock与AtomicInteger的性能比较，如图：
![](/assets/jcip_note/lock_and_atomicInteger_performance_under_moderate_contention.png)
在高度竞争的情况下，锁的性能将超过原子变量的性能，因为锁在发生竞争时会挂起线程，从而降低了CPU的使用率和共享内存总线上的同步信号量，另一方面，AtomicPseudoRandom在遇到竞争时将立即重试，这通常是一种正确的方法，但在激烈竞争环境却导致更多的竞争。
任何一个真实的程序都不会除了竞争锁或原子变量，其他什么都不做，在实际情况中，原子变量在可伸缩性要高于锁，因为在应对常见的竞争程度时，原子变量的效率会更高。

图中包含的第三条曲线，它是一个使用ThreadLocal来保存PRNG状态的PseudoRandom。这种实现方法改变了类的行为，即每个线程都只能看到自己私有的伪随机数字序列，而不是所有线程共享一个随机序列。这说明，如果能够避免使用共享状态，那么开销会更小。我们可以提高处理竞争的效率来提高可伸缩性，但只有完全消除竞争，才能实现真正的可伸缩性。

### 非阻塞算法
如果在某种算法中，一个线程的失败或者挂起不会导致其他线程也失败或挂起，那么这种算法就被称为非阻塞算法。如果在算法的每个步骤都存在某个线程能够执行下去，那么这种算法也被称为无锁（Lock-Free）算法。
如果算法中仅将CAS用于协调线程之间的操作，并且能正确地实现，那么它既是一种无阻塞算法，又是一种无锁算法。

- 非阻塞的栈
```java
@ThreadSafe
public class ConcurrentStack <E> {
    AtomicReference<Node<E>> top = new AtomicReference<Node<E>>();

    public void push(E item) {
        Node<E> newHead = new Node<E>(item);
        Node<E> oldHead;
        do {
            oldHead = top.get();
            newHead.next = oldHead;
        } while (!top.compareAndSet(oldHead, newHead));
    }

    public E pop() {
        Node<E> oldHead;
        Node<E> newHead;
        do {
            oldHead = top.get();
            if (oldHead == null)
                return null;
            newHead = oldHead.next;
        } while (!top.compareAndSet(oldHead, newHead));
        return oldHead.item;
    }

    private static class Node <E> {
        public final E item;
        public Node<E> next;

        public Node(E item) {
            this.item = item;
        }
    }
}
```
安全性保证：compareAndSet既能提供原子性，又能提供可见性。

- 非阻塞的链表
Michael-Scott提出的非阻塞连接队列算法的插入部分：
```java
@ThreadSafe
public class LinkedQueue <E> {

    private static class Node <E> {
        final E item;
        final AtomicReference<LinkedQueue.Node<E>> next;

        public Node(E item, LinkedQueue.Node<E> next) {
            this.item = item;
            this.next = new AtomicReference<LinkedQueue.Node<E>>(next);
        }
    }

    private final LinkedQueue.Node<E> dummy = new LinkedQueue.Node<E>(null, null);
    private final AtomicReference<LinkedQueue.Node<E>> head
            = new AtomicReference<LinkedQueue.Node<E>>(dummy);
    private final AtomicReference<LinkedQueue.Node<E>> tail
            = new AtomicReference<LinkedQueue.Node<E>>(dummy);

    public boolean put(E item) {
        LinkedQueue.Node<E> newNode = new LinkedQueue.Node<E>(item, null);
        while (true) {
            LinkedQueue.Node<E> curTail = tail.get();
            LinkedQueue.Node<E> tailNext = curTail.next.get();
            if (curTail == tail.get()) {
                if (tailNext != null) {
                    // Queue in intermediate state, advance tail
                    tail.compareAndSet(curTail, tailNext);
                } else {
                    // In quiescent state, try inserting new node
                    if (curTail.next.compareAndSet(null, newNode)) {
                        // Insertion succeeded, try advancing tail
                        tail.compareAndSet(curTail, newNode);
                        return true;
                    }
                }
            }
        }
    }
}
```

插入示意图：
处于稳定状态并包含两个元素的队列：
![](/assets/jcip_note/queue_with_two_elements_in_quiescent_state.png)

在插入过程中处于中间状态的对立：
![](/assets/jcip_note/queue_in_intermediate_state_during_insertion.png)

在插入操作完成后，队列再次处于稳定状态：
![](/assets/jcip_note/queue_again_in_quiescent_state_after_insertion_is_complete.png)

- 原子域的更新器
实际的ConcurrentLinkedQueue中没有使用原子引用来表示每个Node，而是使用普通的volatile类型引用，并通过基于反射的AtomicReferenceFieldUpdater来进行更新：

```java
private class Node<E> {
	private final E item;
	private volatile Node<E> next;
	public Node(E item) {
		this.item = item;
	}
}
private static AtomicReferenceFieldUpdater<Node, Node> nextUpdater
	= AtomicReferenceFieldUpdater.newUpdater(
		Node.class, Node.class, "next");
```
原子的域更新器类表示现有volatile域的一种基于反射的“视图”，从而能够在已有的volatile域使用CAS。域更新器提供的原子性保证比普通原子类更弱一些，因为无法保证底层的域不被直接修改——compareAndSet以及其他算术方法只能确保其他使用原子更新器方法的线程的原子性。
使用AtomicReferenceFieldUpdater完全是为了提高性能：对于一些频繁分配并且生命周期短暂的对象，例如队列的链接节点，如果能去掉每个Node的AtomicReference的创建过程，那么将极大地降低插入操作的开销。然而，大多数情况下，普通原子变量的性能都很不错，只有在很少的情况下才需要使用原子的域更新器。（如果在执行原子更新的同时还要维持现有类的序列化形式【比如保持原生类型】，那么原子的域更新器将非常有用）。

- ABA问题
在某些算法中，如果V的值首先由A变成B，再由B变成A，那么仍然被认为是发生了变化，并需要重新执行算法中的某些步骤。
相对简单的解决方案：更新两个值，包括一个引用和一个版本号。AtomicStampedReference（以及AtomICMarkableReference）支持在两个变量上执行原子的条件更新。
