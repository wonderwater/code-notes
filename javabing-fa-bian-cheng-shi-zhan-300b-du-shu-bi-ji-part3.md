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
	// 操作#1，#2返回#3
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

#### 串行线程封闭

#### 双端队列与工作密取

#### 

### 阻塞方法和中断方法

#### 

### 同步工具类

#### 闭锁

#### FutureTask

#### 信号量

#### 栅栏

#### 

### 构建高效且可伸缩的结果缓存



