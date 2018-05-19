# 《Java并发编程实战》读书笔记 part2

## 构建基础模块

### 同步容器类

#### 同步容器类的问题

竞态条件的例子：

```java
public class UnsafeVectorHelpers {
    public static Object getLast(Vector list) {
        int lastIndex = list.size() - 1;
        return list.get(lastIndex);
    }

    public static void deleteLast(Vector list) {
        int lastIndex = list.size() - 1;
        list.remove(lastIndex);
    }
}
```

Vector 对象的原子性不足以保证getLast或deleteLast中复合操作的原子性，对两方法的不同顺序执行，可能造成ArrayIndexOutOfBoundsException异常

![](/assets/jcip_note/5-1.png)将复合操作用Vector对象锁保护，修改后：

```java
public class SafeVectorHelpers {
    public static Object getLast(Vector list) {
        synchronized (list) {
            int lastIndex = list.size() - 1;
            return list.get(lastIndex);
        }
    }

    public static void deleteLast(Vector list) {
        synchronized (list) {
            int lastIndex = list.size() - 1;
            list.remove(lastIndex);
        }
    }
}
```

在迭代操作中，也会造成类似的问题，比如

```java
for (int i = 0; i < vector.size(); i++)
    doSomething(vector.get(i));
```

调用vector对象的size和get方法之间可能有vector长度变化的情况，如果正好i为最后一个索引，此时其他线程从vector移除了一个对象，那么get方法会抛出ArrayIndexOutOfBoundsException异常。当然可以对整个for循环加锁，防止其他线程在迭代期间操作vector对象，但也降低了并发性。类似：

```java
synchronized (vector) {
    for (int i = 0; i < vector.size(); i++)
        doSomething(vector.get(i));
}
```

#### 迭代器与ConcurrentModificationException

无论是直接迭代还是for-each语法中，对容器类进行迭代的标准方式是使用Iterator，但有其他的线程并发修改容器，即使使用迭代期间也要不能避免的对容器加锁。

在同步类的迭代器设计时并没有考虑到并发修改的问题，并且它们表现出的行为是“及时失败”（fail-first），即当容器发现在迭代过程中被修改时，抛出ConcurrentModificationException异常。

它们的实现方式是将计数器的变化与容器关联起来，如果在迭代期间计数器被修改了，那么hasNext或next将抛出ConcurrentModificationException。这种检测是在没有同步的情况进行的，因此可能会看到失效的计数值，而迭代器没有意识到已经发生了修改，这是一种设计上的权衡，从而降低并发修改操作的检测代码对程序性能带来的影响。

类似如上的代码，对迭代过程加锁，假如调用doSomething也持有一把锁，这可能会造成死锁，即使没有这些风险，长时间地对容器加锁也会降低程序的可伸缩性，持有锁的时间越长，那么在锁上的竞争可能就越激烈，如果许多线程都在等待锁的释放，那么将极大地降低吞吐量和CPU的利用率。

如果不希望在迭代期间加锁，一种方法是“克隆”容器，在副本进行迭代。

#### 隐藏迭代器

小心隐式地迭代器，比如字符串的连接可能抛出ConcurrentModificationException

```java
public class HiddenIterator {
    @GuardedBy("this") private final Set<Integer> set = new HashSet<Integer>();

    public synchronized void add(Integer i) {
        set.add(i);
    }

    public synchronized void remove(Integer i) {
        set.remove(i);
    }

    public void addTenThings() {
        Random r = new Random();
        for (int i = 0; i < 10; i++)
            add(r.nextInt());
        System.out.println("DEBUG: added ten elements to " + set);
    }
}
```

真正问题在于HiddenIterator 类并不是线程安全的。

这里得到的教训是，如果状态与保护它的同步代码相隔越远，那么开发人员就越容易忘记在访问状态时使用正确的同步。

容器的hashCode和equals等方法也会间接地执行迭代操作，所以当容器作为另一个容器的元素或键值是，就会出现这种情况。

同样，containsAll , removeAll 和retainAll，以及把容器作为参数的构造函数，所有这些间接的迭代操作都有可能抛出ConcurrentModificationException。

### 并发容器

#### ConcurrentHashMap

使用一种粒度更细的加锁机制来实现更大程度的共享，这种机制称为分段锁（Lock Striping）。

任意数量的读取线程可以并发地访问Map，执行读取操作的线程和执行写入操作的线程可以并发地访问Map，并且一定数量的写入线程可以并发地修改Map。

ConcurrentHashMap和其他的并发容器一起增强了同步容器类：它们提供的迭代器不会抛出ConcurrentModificationException。

ConcurrentHashMap的迭代器具有弱一致性（Weakly Consistent），而非“及时失败”。

#### 额外的原子Map操作

```java
public interface ConcurrentMap<K, V> extends Map<K, V> {
    // 对key的键值操作，更新为新的值，返回新值
    default V compute(K key, BiFunction<? super K, ? super V, ? extends V> remappingFunction);
    // 仅当K没有相应映射值，对key操作，更新为新的值，返回新值
    default V computeIfAbsent(K key, Function<? super K, ? extends V> mappingFunction);
    // 仅当K存在相应映射值，对key的键值操作，更新为新的值，返回新值
    default V computeIfPresent(K key, BiFunction<? super K, ? super V, ? extends V> remappingFunction);
    // get方法拓展，当key无对应值，返回defaultValue
    default V getOrDefault(Object key, V defaultValue);
    // 当K没有相应映射值，同put(key, value)==value；存在相应映射值，同computeIfPresent(key, remappingFunction)
    // 比如图书计数，merge("book", 1, (k, v) -> v + 1)
    default V merge(K key, V value, BiFunction<? super V, ? super V, ? extends V> remappingFunction);
    // function: 操作#1，#2返回#3，如果是null则移除key，否则更新键值。
    default void replaceAll(BiFunction<? super K, ? super V, ? extends V> function);
    // 遍历
    default void forEach(BiConsumer<? super K, ? super V> action);

    // 仅当K没有相应映射值插入，插入成功返回旧值，否则null
    V putIfAbsent(K key, V value);
    // 仅当key被映射到value时才移除
    boolean remove(Object key, Object value);
    // 仅当key被映射到某个值时才替换为value，返回旧值
    V replace(K key, V value);
    // 仅当key被映射到oldValue时才替换为newValue
    boolean replace(K key, V oldValue, V newValue);
}
```

#### CopyOnWriteArrayList

写时复制（Copy-On-Write）容器的线程安全性在于。只要正确的发布一个事实不可变的对象，那么在访问该对象时就不在需要进一步的同步，在每次修改时，都会创建并重新发布一个新的容器副本。

### 阻塞队列和生产者- 消费者模式

阻塞队列提供可阻塞的put和take方法，已经支持定时的offer和poll方法。

阻塞队列支持生产者- 消费者模式，该模式将“找出需要完成的工作”与“执行工作”这两个过程分开，并把工作放到一个“待完成”列表以便在后续处理，而不是找出后立即处理。此外，该模式还将生产数据的过程与使用数据的过程解耦开来以简化工作负载的管理，因为这两个过程在处理数据的速率上有所不同。

BlockingQueue简化了生产者-消费者设计的实现过程，一种最常见的生产者-消费者设计模式就是**线程池与工作队列**的组合。

阻塞队列简化了消费者程序的编码，因为take操作会一直阻塞到有可用的数据。并发类库的使用可以参考[http://tutorials.jenkov.com/java-util-concurrent/index.html](http://tutorials.jenkov.com/java-util-concurrent/index.html)

如果生产者生成工作的速率比消费者的工作速率要快，那么工作项会在队列中累积起来，最终耗尽内存，同样put方法的阻塞特性也极大简化了生产者的编码，因为当队列满了，生产者将阻塞并不能继续工作，而消费者就有时间赶上工作进度。

在构建高可靠的应用程序时，有界队列是一种强大的资源管理工具：它们能抑制并防止产生过多的工作项，使应用在负荷过载的情况下变得更加健壮。

虽然生产者-消费者模式能够将生产者和消费者的代码彼此解耦开来，但它们的行为仍然通过共享工作队列间接耦合在一起。开发者总假设消费者处理工作速率能赶上生产者生成工作的速率，因此不会为工作队列的大小设置边界值，但这将导致在之后需要重新设计系统架构。因此，应尽早地通过阻塞队列在设计中构建资源管理机制——这件事做得越早，就越容易。

类库中包含BlockingQueue的多种实现，其中LinkedBlockingQueue和ArrayBlockingQueue是FIFO，二者分别于LinkedList和ArrayList类似，但比同步List拥有更好的并发性能。PriorityBlockingQueue是一个按优先级排序的队列，当你希望按照某种顺序执行任务时将非常有用。

![](/assets/jcip_note/jdk_BlokingQueue_impls.png)

BlockingQueue接口的使用：

|  | **Throws Exception** | **Special Value** | **Blocks** | **Times Out** |
| :--- | :--- | :--- | :--- | :--- |
| **Insert** | `add(o)` | `offer(o)` | `put(o)` | `offer(o, timeout, timeunit)` |
| **Remove** | `remove(o)` | `poll()` | `take()` | `poll(timeout, timeunit)` |
| **Examine** | `element()` | `peek()` |  |  |

SynchronousQueue实际上不是一个真正的队列，因为它不会为队列元素维护存储空间，与其他队列不同的是，它维护一组线程，这些线程在等待元素加入或移出队列。当交付被接受时，生产者就知道了消费者已经得到了任务，而不是简单地把任务放入一个队列——这种区别就好比将文件直接交给同事，还是放进他的邮箱并希望他能尽快拿到文件。仅当有足够多的消费者，并且总是有一个消费者准备好获取交付的工作时，才适合使用同步队列。

其他的实现：[http://www.cs.rochester.edu/u/scott/papers/2009\_Scherer\_CACM\_SSQ.pdf](http://www.cs.rochester.edu/u/scott/papers/2009_Scherer_CACM_SSQ.pdf)，其中sync代表资源是否有效。

```java
public class HansonSQ<E> {
    E item = null;
    Semaphore sync = new Semaphore(0);
    Semaphore send = new Semaphore(1);
    Semaphore recv = new Semaphore(0);

    Public E take() {
        recv.acquire();
        E x = item;
        sync.release();
        send.release();
        return x;
    }

    public void put(E x) {
        send.acquire();
        item = x;
        recv.release();
        sync.acquire();
    }
}
```

#### 串行线程封闭

在juc中实现的各种阻塞队列中都包含足够的内部同步机制，从而安全地将对象从生产者线程发布到消费者线程，这是一种控制权的转移，被线程封闭的对象从生产者线程发布后，生产者线程就不再访问该对象了，之后的消费者线程可以任意修改该对象。

#### 双端队列与工作密取

Deque和BlockingDeque分别对于Queue和BlockingQueue进行了拓展，Deque是一个双端的队列，具体实现包括ArrayDeque和LinkedBlockingDeque。

正如阻塞队列适用于生产者-消费者模式，双端队列同样适合另一种相关的模式：工作密取（Work Stealing），每个消费者都有各自的双端队列，如果自己的任务完成了，可以从其他消费者的队列的队尾获取任务。相比传统的生产者-消费者模型，密取模式的每个消费者从自己的队列获取任务极大减少了竞争，当需要访问另一个队列时，从队列尾部而不是头部获取任务，进一步降低了队列上的竞争。

工作密取非常适合既是消费者也是生产者问题——当执行某个工作时可能导致出现更多的工作，比如网页爬取，垃圾回收的标记-清除算法。

### 阻塞方法和中断方法

#### 

### 同步工具类

#### 闭锁

#### FutureTask

#### 信号量

#### 栅栏

#### 

### 构建高效且可伸缩的结果缓存



