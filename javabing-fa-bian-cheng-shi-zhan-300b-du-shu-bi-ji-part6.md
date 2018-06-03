# 《Java并发编程实战》读书笔记 part6
## 避免活跃性危险

### 死锁
经典的哲学家进餐问题：五个哲学家围圆桌边吃饭，他们有五只筷子，并且每两个人的中间放一根筷子。哲学家们时而思考，时而进餐，每个人都需要一双筷子才能吃到东西，并在吃完后将筷子放回原处继续思考。

||描述|结果|
|:-:|:---|:---|
|算法1 | 尝试获取两只筷子，发现有一只被旁边的哲学家占用，则放弃已得到的筷子，过段时间再试| 每个哲学家相对能及时吃到东西|
|算法2 | 每个人先抓自己左边的筷子，然后等待自己右边的筷子| 可能导致一些或全部哲学家“饿死”|

抱死（Deadly Embrace）：最简单的死锁形式，线程A持有锁L的同时想获得锁R，线程B持有锁R想获得锁L。
考虑数据库系统监测死锁以及从死锁中恢复，通过在表示等待关系的有向图中搜索循环，选择一个牺牲者并放弃这个事务。
JVM在解决死锁问题并没有这么强大，当一组Java线程死锁事——这些线程永远不能再使用了。恢复他们的唯一方式就是中断并重启它，并希望不要再发生同样的事。
与其他并发危险一样，死锁造成的影响很少会立即显现出来，如果一个类可能发生死锁，并不意味着每次都会发生死锁。当死锁出现时，往往是在最糟糕的时候——在高负载情况下。

锁顺序导致的死锁：
```java
// 一个线程调用leftRight，另一个调rightLeft，很容易出现抱死
// 不能单独分析每条获取锁的代码路径，这时候需要对程序中的加锁行为进行全局分析
public class LeftRightDeadlock {
    private final Object left = new Object();
    private final Object right = new Object();
    
    public void leftRight() {
        synchronized (left) {
            synchronized (right) {
                doSomething();
            }
        }
    }

    public void rightLeft() {
        synchronized (right) {
            synchronized (left) {
                doSomethingElse();
            }
        }
    }
}
```
如果所有线程都以固定的顺序获得锁，那么程序中就不会出现锁顺序死锁问题。

动态的锁顺序死锁：
```java
// toAccount和fromAccount参数是调用者传入的，不同线程可以按照不同顺序传入，和上述的顺序死锁类似
// A: transferMoney(me, you, 10)
// B: transferMoney(you, me, 10)
public static void transferMoney(Account fromAccount, Account toAccount, DollarAmount amount)
            throws InsufficientFundsException {
    synchronized (fromAccount) {
        synchronized (toAccount) {
            if (fromAccount.getBalance().compareTo(amount) < 0)
                throw new InsufficientFundsException();
            else {
                fromAccount.debit(amount);
                toAccount.credit(amount);
            }
        }
    }
}
```
不论怎么传入，我们试图按照相同的顺序顺序获取锁，改进：
```java
private static final Object tieLock = new Object();
// 通过hash函数，使得程序按照一定顺序获得锁
public void transferMoney(final Account fromAcct,
                          final Account toAcct,
                          final DollarAmount amount)
        throws InsufficientFundsException {
    class Helper {
        public void transfer() throws InsufficientFundsException {
            if (fromAcct.getBalance().compareTo(amount) < 0)
                throw new InsufficientFundsException();
            else {
                fromAcct.debit(amount);
                toAcct.credit(amount);
            }
        }
    }
    int fromHash = System.identityHashCode(fromAcct);
    int toHash = System.identityHashCode(toAcct);

    if (fromHash < toHash) {
        synchronized (fromAcct) {
            synchronized (toAcct) {
                new Helper().transfer();
            }
        }
    } else if (fromHash > toHash) {
        synchronized (toAcct) {
            synchronized (fromAcct) {
                new Helper().transfer();
            }
        }
    } else {
	    // 哈希相等的情况使用“加时赛”锁（TieBreaking），就不用顾忌fromAcct和toAcct的顺序了
	    // 如果经常出现这种情况，注意tieLock是静态的，可能会造成并发瓶颈
        synchronized (tieLock) {
            synchronized (fromAcct) {
                synchronized (toAcct) {
                    new Helper().transfer();
                }
            }
        }
    }
}
```
不要小瞧了死锁的发生的风险，在商业产品的应用程序每天可能要执行数十亿次获取—释放锁的操作，只要在这数十亿次里有一次发生错误，就可能导致程序发生死锁，并且这是极难复现的。

在协作对象之间发生的锁死：
```java
// 注意方法定义的内置锁，容易发生死锁：
// Taxi.setLocation和Dispatcher.getImage隐式地获取锁，可能造成的锁顺序死锁
public class CooperatingDeadlock {
    // Warning: deadlock-prone!
    class Taxi {
        @GuardedBy("this") private Point location, destination;
        private final Dispatcher dispatcher;

        public Taxi(Dispatcher dispatcher) {
            this.dispatcher = dispatcher;
        }

        public synchronized Point getLocation() {
            return location;
        }

        public synchronized void setLocation(Point location) {
            this.location = location;
            if (location.equals(destination))
                dispatcher.notifyAvailable(this);
        }

        public synchronized Point getDestination() {
            return destination;
        }

        public synchronized void setDestination(Point destination) {
            this.destination = destination;
        }
    }

    class Dispatcher {
        @GuardedBy("this") private final Set<Taxi> taxis;
        @GuardedBy("this") private final Set<Taxi> availableTaxis;

        public Dispatcher() {
            taxis = new HashSet<Taxi>();
            availableTaxis = new HashSet<Taxi>();
        }

        public synchronized void notifyAvailable(Taxi taxi) {
            availableTaxis.add(taxi);
        }

        public synchronized Image getImage() {
            Image image = new Image();
            for (Taxi t : taxis)
                image.drawMarker(t.getLocation());
            return image;
        }
    }
}
```
如果在持有锁时调用某个外部方法，那么将出现活跃性问题。在这个外部方法中可能会获得其他锁，或者阻塞时间过长，导致其他线程无法及时获得当前被持有的锁。
开放调用（Open Call）：在调用某个方法时不需要持有锁的调用。
使用开放调用改进代码，消除死锁风险，收缩同步代码块：
```java
@ThreadSafe
class Taxi {
@GuardedBy("this") private Point location, destination;
private final Dispatcher dispatcher;
...
public synchronized Point getLocation() {
	return location;
}
public void setLocation(Point location) {
	boolean reachedDestination;
	synchronized (this) {
		this.location = location;
		reachedDestination = location.equals(destination);
	}
	if (reachedDestination)
		dispatcher.notifyAvailable(this);
	}
}
@ThreadSafe
class Dispatcher {
	@GuardedBy("this") private final Set<Taxi> taxis;
	@GuardedBy("this") private final Set<Taxi> availableTaxis;
	...
	public synchronized void notifyAvailable(Taxi taxi) {
		availableTaxis.add(taxi);
	}
	public Image getImage() {
		Set<Taxi> copy;
		synchronized (this) {
			copy = new HashSet<Taxi>(taxis);
		}
		Image image = new Image();
		for (Taxi t : copy)
			image.drawMarker(t.getLocation());
		return image;
	}
}
```
在程序中应该尽量使用开放调用。与那些在持有锁时调用外部方法的程序相比，更容易对依赖于开放调用的程序进行死锁分析。

资源死锁：正如当多个线程相互持有彼此正在等待的锁而又不释放自己持有的锁时会发生死锁，当它们在相同的资源集合上等待时，也会发生死锁：
1. 比如两个数据库的资源连接：线程A持有D~1~请求D~2~，线程B持有D~2~请求D~1~
2. 另一种基于资源的死锁形式是线程饥饿死锁（Thread-Starvation Deadlock）（在资源池的使用时讨论过）
### 死锁的避免与诊断
如果每个程序每次至多获取一个锁，那么就不会产生锁顺序死锁。当然啦，这不现实，那么设计时必须考虑锁顺序，尽量减少潜在的加锁交互数量，将获取锁时需要遵守的协议写入正式文档并始终循序这些协议。
通过两阶段策略（Two-Part Strategy）来检查代码中的死锁：
1. 找出在什么地方将获取多个锁（使这个集合尽量小）
2. 对这些事例进行全局分析，从而确保它们在整个程序中获取锁的顺序都保持一致

支持定时的锁：给显示锁指定超时时限，超时后返回失败信息。当然超时的情况可能是死锁，也可能某个线程在持有锁时错误地进入无线循环，也可能是某个操作的耗时远超预期。
通过线程转储信息来分析死锁：线程转储包括各个运行中的线程的栈追踪信息，加锁信息（每个线程持有哪些锁，在哪些栈帧中获得这些锁，以及被阻塞的线程正在等待获取哪一个锁）
线程转储操作：

1. UNIX：向JVM发送SIGQUIT信号（kill -3）、按下Ctrl-\
2. Windows：按下Ctrl-Break

例子：
```log
Found one Java-level deadlock:
=============================
"ApplicationServerThread":
	waiting to lock monitor 0x080f0cdc (a MumbleDBConnection),
	which is held by "ApplicationServerThread"
"ApplicationServerThread":
	waiting to lock monitor 0x080f0ed4 (a MumbleDBCallableStatement),
	which is held by "ApplicationServerThread"
Java stack information for the threads listed above:
"ApplicationServerThread":
	at MumbleDBConnection.remove_statement
	- waiting to lock <0x650f7f30> (a MumbleDBConnection)
	at MumbleDBStatement.close
	- locked <0x6024ffb0> (a MumbleDBCallableStatement)
	...
"ApplicationServerThread":
	at MumbleDBCallableStatement.sendBatch
	- waiting to lock <0x6024ffb0> (a MumbleDBCallableStatement)
	at MumbleDBConnection.commit
	- locked <0x650f7f30> (a MumbleDBConnection)
	...
```
以上信息中，JDBC驱动程序明显存在一个锁顺序问题：不同调用链通过JDBC驱动程序以不同顺序获取多个锁。
当诊断死锁时，JVM可以帮助我们做许多工作：
* 哪些锁导致这个问题
* 涉及哪些线程
* 它们持有哪些其他的锁，是否间接地给其他线程带来不利条件

### 其他活跃性危险
尽管死锁是最常见的活跃性危险，但在并发程序中还存在一些其他的活跃性危险，包括：饥饿、丢失信号、活锁、糟糕的响应性等待。
**饥饿**（Starvation）：当程序由于无法访问它所需的资源而不能继续执行时，就发生了“饥饿”。
Thread API中定义的线程优先级只是作为调度参考，与平台相关。可能不会起到任何作用，也可能使得某个线程的调度优先级高于其他线程，从而导致饥饿。
要避免使用线程优先级，因为这会增加平台依赖性，并可能导致活跃性问题。在大多数并发应用程序中，都可以使用默认的线程优先级。

**活锁**（Livelock）是另一种形式的活跃性问题：
作用：尽管不会阻塞线程，但也不能继续执行，因为线程将不断重复执行相同操作，而且总会失败。
原因：当多个互相协作的线程都对彼此进行响应从而修改各自状态，并使得任何一个线程都无法执行时，就发生了活锁。
解决：在重试机制中引入随机性。

## 性能与可伸缩性

### 对性能的思考
当操作性能由于某种特定的资源而受到限制时，我们通常将该操作称为资源密集型操作，如CPU密集型、数据库密集型
分解目标：
* 更有效地利用现有处理资源
* 在出现新的处理资源时使程序尽可能利用这些新资源
从性能监视来看，就是使CPU尽肯能忙碌（当然，是有用的忙碌）
性能衡量指标：
* 服务时间
* 延迟时间
* 吞吐率
* 效率
* 可伸缩性：当增加计算资源时（如CPU、内存、存储容量、I/O带宽等），程序的吞吐量或处理能力能相应增加
* 容量
...
这些指标中：
一些指标是运行速度（指定的任务单元“多快”能完成）：服务时间、延迟时间......
另一些是处理能力（计算资源一定的情况能完成“多少”工作）：可伸缩性、吞吐量、生产量......
他们是完全独立的，有时甚至相互矛盾。

提高可伸缩性造成性能损失的例子：我们熟悉的三层程序模型，即在模型中的表现层、业务层、持久化层是彼此独立的，并且可能由不同系统来处理。如果这三层合到**单个应用程序**中，其性能肯定要高于将其分为多层并将不同层次分布到多个系统的性能（比如增加了不同层间任务传递的网络延迟，计算过程分解到不同层、任务排队、线程协调、数据复制等等）。

单个应用程序达到自身处理能力极限时，要进一步提升它的处理能力将非常困难，因此，我们通常接受每个工作单元执行更长的时间或者消耗更多的计算资源，以换取程序在增加更多资源的情况处理更高的负载。

**避免不成熟的优化，首先使程序正确，然后再提高运行速度——如果它还运行得不够快。**
大多数性能决策包含多个变量，并且非常依赖运行环境，在使某个方案比其他方案更快之前，首先问自己一些问题：
- “更快”的含义是什么？
- 该方法在什么条件下运行更快？高/低负载？大/小数据集？能否用测试结果验证你的答案？
- 这些条件在运行环境中的发生频率？能否通过测试结果验证你的答案？
- 在其他不同的环境能否使用这里的代码？
- 在实现这种性能提升需要付出哪些隐含代价：增加开发风险/维护开销？

在对性能进行调优时，一定要有明确地性能需求（知道什么时候要调优，什么时候要停止），当然还要有测试程序、真实配置和负载等环境。**以测试为基准，不要乱猜。**

### Amdahl定律
Amdahl定律：描述在增加计算资源的情况下，程序在理论上能够实现最高加速比。设其中F为必须被串行执行的部分，在包含N个处理器的机器中，最高加速比为：

$$Speedup \le \frac 1 {F + \frac {(1-F)} N} $$ 
当$$N \to \infty$$，$$max(Speedup) \to \frac 1 F$$
因此如果程序有50%的计算需要串行执行，那么最高加速比只能是2。如图：
![](/assets/jcip_note/amdahl_low_figure.png)

在所有并发程序中都包含一些串行部分。如果你认为在你的程序中不存在串行部分，那么可以在仔细检查一遍。

串行部分：比如N个线程正在执行doWork，它们从一个工作队列中取任务进行处理，这些任务不相互依赖，不考虑任务如何进入队列的，如果增加处理器，程序的性能是否会发生变化？
```java
public class WorkerThread extends Thread {
    private final BlockingQueue<Runnable> queue;

    public WorkerThread(BlockingQueue<Runnable> queue) {
        this.queue = queue;
    }

    public void run() {
        while (true) {
            try {
                Runnable task = queue.take();
                task.run();
            } catch (InterruptedException e) {
                break; /* Allow thread to exit */
            }
        }
    }
}
```
注意程序中的串行部分——从队列中获取任务，所有线程共享这个任务队列，需要用某种同步机制维持队列的完整性。不同任务队列的实现，对程序的影响不同，从中也可推测隐藏在程序中的串行部分的影响，如图：
![](/assets/jcip_note/queue_implementations_comparing.png)
### 线程引入的开销

1. 上下文切换：上下文切换的消耗不仅JVM和操作系统的开销，当一个新的线程被切换进来，它所需的数据可能不在当前处理器的本地缓存中，因此上下文切换将导致缓存缺失，因而线程在首次调度运行会更加慢。——所以调度器会为每个可运行线程分配一个最小执行时间。大多数通用处理器的上下文切换开销相当于5k~10k个时钟周期，也就是几微秒。
> 可以使用UNIX的vmstat命令，Windows的perfmon工具报告。**如果内存占用率较高（超过10%），通常表示调度活动发生很频繁**，很可能是由I/O或竞争锁导致的阻塞引起的。
2. 内存同步：同步操作的开销有多个方面：
	1. synchronized和volatile提高可见性保证会使用特殊指令，即内存栅栏（Memory Barrier）。内存栅栏可以刷新缓存，使缓存无效，刷新硬件的写缓存，以及停止执行管道。内存栅栏对性能也会间接带来影响，因为它抑制了编译器的优化。在内存栅栏中，大多数操作都是不能被重排序的。
	2. 有竞争的同步和无竞争的同步：无竞争的同步对程序整体影响微乎其微。
	> 不要过度担心非竞争同步带来的消耗。这个基本的机制已经非常快了，并且JVM还能进行额外的优化以进一步降低或消除开销，因此，**我们应该将优化重点放在那些发生锁竞争的地方**。
	
	比如：
	```java
	// JVM去掉没用的锁：
	synchronized (new Object()) {
		// do something
	}
	// 注意Vector对象操作引起的4次锁获取/释放
	// JVM通过锁消除（Lock Elision）优化去掉的全部的锁获取操作
	// 或者通过锁粒度粗化（Lock Coarsening）将相邻的锁合并起来。
	public String getStoogeNames() {
        List<String> stooges = new Vector<String>();
        stooges.add("Moe");
        stooges.add("Larry");
        stooges.add("Curly");
        return stooges.toString();
    }
	```
3. 阻塞：竞争的同步可能需要操作系统介入，从而增加开销。
对于阻塞较短的，适合采用自旋方式；阻塞较长的，适合采用线程挂起。有些JVM将根据对历史等待时间的分析数据在这两者之间进行操作，但大多数JVM在等待锁时都是简单将线程挂起。

### 减少锁的竞争
在并发程序中，对伸缩性的最主要威胁就是独占方式的资源锁。
影响在锁上发生竞争的可能性因素：
1. 锁的请求频率
2. 每次持有该锁的时间

那么减少锁的竞争可以：

1. 减少锁的持有时间：缩小锁的范围
2. 降低锁的请求频率：减小锁的粒度（锁分解、锁分段）
3. 使用带有协调机制的独占锁，这些机制允许更高的并发性：原子变量、读写锁等

缩小锁的范围：仅当可以将一些“大量”的计算或者阻塞操作从同步代码块中移出时，才应该考虑同步代码块的大小。
锁分段的一个劣势在于：与采用单个锁来实现独占访问相比，要获取多个锁来实现独占访问将更加困难并且开销更高。比如采用锁分段实现的ConcurrentHashMap中计算size()，这里没有获取全部的桶的锁，而是对所有桶的size相加。
采用锁分段的时机：一定要表现出在**锁上的竞争**频率高于在锁保护的**数据上的竞争**频率。
每个操作都请求多个变量时，锁的粒度将很难降低，这是在性能与可伸缩性之间相互制衡的另一方面。一些常见的优化措施：比如将反复计算的结果缓存起来，引入“热点域（Hot Field）”，热点域往往会限制可伸缩性。

- 监测CPU利用率
如果CPU利用率不均匀（有些CPU忙碌，有些却不）：首先找出程序的并行性，因为不均匀的利用率表面大多数计算都是由一小组线程完成，并且应用程序没有理由其他处理器。
如果CPU没有充分利用，通常有以下几种原因：
1. 负载不充足
2. I/O密集：可通过iostat或perfmon判断应用是否是磁盘I/O密集型，或者通过监测网络通信流量级别判断是否需要高带宽。
3. 外部限制：依赖外部服务，比如数据库、web服务
4. 锁竞争：通过线程转储工具，分析包含信息形如“waiting to lock monitor ...”

**向对象池说“不”！通常，对象分配操作的开销比同步的开销更低。**
示例，比较Map的性能，如图：
![](/assets/jcip_note/map_implementations_comparing.png)

### 减少上下文切换的开销
日志的实现的两种方式：
1. 及时写日志到文件：写时锁竞争
2. 通过将I/O写入分离到独立的线程

比喻：救火方案：

1. 每个人都拿着一桶水去救火：每个人可能在水源和着火点存在更大的竞争，并且效率也更低（切换模式：装水、跑步、倒水、跑步......）
2. 所有人排成一队，通过递水桶来救火

## 并发程序的测试
在测试并发程序时，所面临的挑战在于：潜在错误的发生并不具有确定性，而是随机的。要在测试中将这些故障暴露出来，就需要比普通的串行程序测试覆盖更广的范围并且执行更长的时间。
并发测试大致分为两类：
1. 安全性测试：通常会采用测试不变性条件的形式，即判断某个类的行为是否与其规范保持一致。
2. 活跃性测试：包括进展测试、无进展测试——如何验证某个方法是被阻塞了，而不只是运行缓慢？
与活跃性测试相关的是性能测试，性能测试的指标包括：
	1. 吞吐量（Throughput）：指一组并发任务中已完成任务所占的比例
	2. 响应性（Responsiveness）：指请求从出发到完成之间的时间（也称为延迟）
	3. 可伸缩性（Scalability）：指在增加更多资源的情况下（通常指CPU），吞吐量（或者环节短缺）的提升情况

### 正确性测试
为某个并发类设计测试时，首先需要执行与测试串行类相同的分析——找出需要检查的不变性条件和后验条件。
一个基于信号量的有界缓存的实现`SemaphoreBoundedBuffer`：
```java
@ThreadSafe
public class SemaphoreBoundedBuffer <E> {
    private final Semaphore availableItems, availableSpaces;
    @GuardedBy("this") private final E[] items;
    @GuardedBy("this") private int putPosition = 0, takePosition = 0;

    public SemaphoreBoundedBuffer(int capacity) {
        if (capacity <= 0)
            throw new IllegalArgumentException();
        availableItems = new Semaphore(0);
        availableSpaces = new Semaphore(capacity);
        items = (E[]) new Object[capacity];
    }

    public boolean isEmpty() {
        return availableItems.availablePermits() == 0;
    }

    public boolean isFull() {
        return availableSpaces.availablePermits() == 0;
    }

    public void put(E x) throws InterruptedException {
        availableSpaces.acquire();
        doInsert(x);
        availableItems.release();
    }

    public E take() throws InterruptedException {
        availableItems.acquire();
        E item = doExtract();
        availableSpaces.release();
        return item;
    }

    private synchronized void doInsert(E x) {
        int i = putPosition;
        items[i] = x;
        putPosition = (++i == items.length) ? 0 : i;
    }

    private synchronized E doExtract() {
        int i = takePosition;
        E x = items[i];
        items[i] = null;
        takePosition = (++i == items.length) ? 0 : i;
        return x;
    }
}
```
下面我们围绕它来测试。
- 基本的单元测试：
```java
public class TestBoundedBuffer extends TestCase {
    private static final long LOCKUP_DETECT_TIMEOUT = 1000;
    private static final int CAPACITY = 10000;
    private static final int THRESHOLD = 10000;

    void testIsEmptyWhenConstructed() {
        SemaphoreBoundedBuffer<Integer> bb = new SemaphoreBoundedBuffer<Integer>(10);
        assertTrue(bb.isEmpty());
        assertFalse(bb.isFull());
    }

    void testIsFullAfterPuts() throws InterruptedException {
        SemaphoreBoundedBuffer<Integer> bb = new SemaphoreBoundedBuffer<Integer>(10);
        for (int i = 0; i < 10; i++)
            bb.put(i);
        assertTrue(bb.isFull());
        assertFalse(bb.isEmpty());
    }
}
```
这些测试都是串行的，有助于我们在开始分析数据竞争之前就找出与并发性无关的问题。
- 对阻塞进行测试：
在测试阻塞行为时，将引入复杂性：当方法被成功地阻塞后，还必须使方法解除阻塞。
```java
// 测试无缓存时，取缓存操作的阻塞行为
void testTakeBlocksWhenEmpty() {
    final SemaphoreBoundedBuffer<Integer> bb = new SemaphoreBoundedBuffer<Integer>(10);
    Thread taker = new Thread() {
        public void run() {
            try {
                int unused = bb.take();
                fail(); // if we get here, it's an error
            } catch (InterruptedException success) {
	            // expected
            }
        }
    };
    try {
        taker.start();
        Thread.sleep(LOCKUP_DETECT_TIMEOUT);
        taker.interrupt();
        taker.join(LOCKUP_DETECT_TIMEOUT);
        assertFalse(taker.isAlive());
    } catch (Exception unexpected) {
        fail();
    }
}
```
**使用Thread.getState来验证线程是否在一个条件等待上阻塞并不可靠。**被阻塞线程不需要进入WAITING或TIME_WAITING状态，因此JVM可以通过自旋等待来实现阻塞。类似地，由于在Object.wait或Condition.await等方法上存在伪唤醒（Spurious Wakeup），即使一个线程等待条件尚未为真，也可能从WAITING或TIME_WAITING等状态零时性地转换到RUNNABLE状态。
- 安全性测试：
如果要构建一些测试来发现并发类中的安全性错误，那么实际上是个“先有鸡还是先有蛋”的问题：测试程序本身是并发的。
测试BoundedBuffer的生产者-消费者程序：
```java
public class PutTakeTest extends TestCase {
    protected static final ExecutorService pool = Executors.newCachedThreadPool();
    protected CyclicBarrier barrier;
    protected final SemaphoreBoundedBuffer<Integer> bb;
    protected final int nTrials, nPairs;
    protected final AtomicInteger putSum = new AtomicInteger(0);
    protected final AtomicInteger takeSum = new AtomicInteger(0);

    public static void main(String[] args) throws Exception {
        new PutTakeTest(10, 10, 100000).test(); // sample parameters
        pool.shutdown();
    }

    public PutTakeTest(int capacity, int npairs, int ntrials) {
        this.bb = new SemaphoreBoundedBuffer<Integer>(capacity);
        this.nTrials = ntrials;
        this.nPairs = npairs;
        this.barrier = new CyclicBarrier(npairs * 2 + 1);
    }

    void test() {
        try {
            for (int i = 0; i < nPairs; i++) {
                pool.execute(new Producer());
                pool.execute(new Consumer());
            }
            barrier.await(); // 等待所有线程就绪
            barrier.await(); // 等待所有线程执行完成
            assertEquals(putSum.get(), takeSum.get());
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }

	// 大多数RNG（随机数生成器）都是线程安全的，并且会带来额外的同步开销。
	// 这里采用个较为合适的RNG
    static int xorShift(int y) {
        y ^= (y << 6);
        y ^= (y >>> 21);
        y ^= (y << 7);
        return y;
    }

    class Producer implements Runnable {
        public void run() {
            try {
                int seed = (this.hashCode() ^ (int) System.nanoTime());
                int sum = 0;
                barrier.await();
                for (int i = nTrials; i > 0; --i) {
                    bb.put(seed);
                    sum += seed;
                    seed = xorShift(seed);
                }
                putSum.getAndAdd(sum);
                barrier.await();
            } catch (Exception e) {
                throw new RuntimeException(e);
            }
        }
    }

    class Consumer implements Runnable {
        public void run() {
            try {
                barrier.await();
                int sum = 0;
                for (int i = nTrials; i > 0; --i) {
                    sum += bb.take();
                }
                takeSum.getAndAdd(sum);
                barrier.await();
            } catch (Exception e) {
                throw new RuntimeException(e);
            }
        }
    }
}
```
我们用栅栏保证所有线程就绪，忽略掉线程创建和启动的开销时间。
这些测试应该放在多处理器的系统上运行，从而进一步测试更高形式的交替执行。

- 资源管理的测试
到这里之前的测试都侧重类与它的设计规范的一致程度，测试的另一方面是判断类中是否没有做它不应该做的事情，比如资源泄漏。

```java
class Big {
    double[] data = new double[100000];
}

void testLeak() throws InterruptedException {
    SemaphoreBoundedBuffer<Big> bb = new SemaphoreBoundedBuffer<Big>(CAPACITY);
    int heapSize1 = snapshotHeap();
    for (int i = 0; i < CAPACITY; i++)
        bb.put(new Big());
    for (int i = 0; i < CAPACITY; i++)
        bb.take();
    int heapSize2 = snapshotHeap();
    assertTrue(Math.abs(heapSize1 - heapSize2) < THRESHOLD);
}
```
在构造测试案例时，对客户提供的代码进行回调是非常有帮助的，回调函数的执行通常是在对象生命周期的一些已知位置，比如ThreadPoolExecutor中将调用任务的Runnable和ThreadFactory。
```java
class TestingThreadFactory implements ThreadFactory {
    public final AtomicInteger numCreated = new AtomicInteger();
    private final ThreadFactory factory = Executors.defaultThreadFactory();

    public Thread newThread(Runnable r) {
        numCreated.incrementAndGet();
        return factory.newThread(r);
    }
}
```
验证线程池拓展能力的测试：
```java
public class TestThreadPool extends TestCase {
    private final TestingThreadFactory threadFactory = new TestingThreadFactory();
    public void testPoolExpansion() throws InterruptedException {
        int MAX_SIZE = 10;
        ExecutorService exec = Executors.newFixedThreadPool(MAX_SIZE);

        for (int i = 0; i < 10 * MAX_SIZE; i++)
            exec.execute(new Runnable() {
                public void run() {
                    try {
                        Thread.sleep(Long.MAX_VALUE);
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                    }
                }
            });
        for (int i = 0;
             i < 20 && threadFactory.numCreated.get() < MAX_SIZE;
             i++)
            Thread.sleep(100);
        assertEquals(threadFactory.numCreated.get(), MAX_SIZE);
        exec.shutdownNow();
    }
}
```
产生更多的交替操作的办法：
1. Thread.yield：JVM可以将之实现为空
2. Thread.sleep

### 性能测试
性能测试的目标：
1. 衡量典型测试用例中的端到端性能。
2. 根据经验值来调整各种不同的限值：如线程数量、缓存容量。

改造PutTakeTest如下部分，增加计时功能：
> this.timer = new BarrierTimer();
this.barrier = new CyclicBarrier(npairs*2 + 1, timer);

```java
public class BarrierTimer implements Runnable {
    private boolean started;
    private long startTime, endTime;

    public synchronized void run() {
        long t = System.nanoTime();
        if (!started) {
            started = true;
            startTime = t;
        } else
            endTime = t;
    }
    public synchronized void clear() { started = false; }
    public synchronized long getTime() {
        return endTime - startTime;
    }
}
```
测试代码：
```java
public void test() {
    try {
        timer.clear();
        for (int i = 0; i < nPairs; i++) {
            pool.execute(new PutTakeTest.Producer());
            pool.execute(new PutTakeTest.Consumer());
        }
        barrier.await(); // 记录startTime
        barrier.await(); // 记录endTime
        long nsPerItem = timer.getTime() / (nPairs * (long) nTrials);
        System.out.print("Throughput: " + nsPerItem + " ns/item");
        assertEquals(putSum.get(), takeSum.get());
    } catch (Exception e) {
        throw new RuntimeException(e);
    }
}
```
从中我们能学到：
1. 生产者-消费者模式在不同参数组合下的吞吐率
2. 有界缓存在不同线程数量的伸缩性
3. 如何选择缓存大小

主测试驱动代码：
```java
public static void main(String[] args) throws Exception {
    int tpt = 100000; // trials per thread
    for (int cap = 1; cap <= 1000; cap *= 10) {
        System.out.println("Capacity: " + cap);
        for (int pairs = 1; pairs <= 128; pairs *= 2) {
            TimedPutTakeTest t = new TimedPutTakeTest(cap, pairs, tpt);
            System.out.print("Pairs: " + pairs + "\t");
            t.test();
            System.out.print("\t");
            Thread.sleep(1000);
            t.test();
            System.out.println();
            Thread.sleep(1000);
        }
    }
    PutTakeTest.pool.shutdown();
}
```
通过以上运行的结果，绘制如图：
![](/assets/jcip_note/timedputtaketest_with_various_buffrt_capacities.png)

缓存容量分别为1、10、100、1000。当缓存大小为1时，吞吐率非常糟糕，提升到10后，有了极大地提高，但超过10后，所得的收益又开始降低。
谨慎对待从上面数据所得的结论，即在使用有界缓存的生产者-消费者程序中可以添加更多线程。这个测试忽略了许多实际因素，比如生产者不需要任何工作就能生成元素并放入队列，消费者无需太多工作就能获取一个元素。在真实的生产者-消费者应用程序中将有更复杂的操作。这个测试的主要目的是，测量生产者和消费者在通过有界缓存传递数据时，哪些约束条件将对整体吞吐量产生影响。

虽然BoundedBuffer是一种非常合理的实现，并且它的执行性能也不错，但还是没有ArrayBlockingQueue或LinkedBlockingQueue那么好，我们看这两个实现方法的TimedPutTakeTest的版本结果，如图：
![](/assets/jcip_note/blockingqueue_implementations_comparing.png)

测试结果表明，LinkedBlockingQueue的可伸缩性要高于ArrayBlockingQueue。初看起来有点奇怪：链表队列在每次插入元素时，都会分配一个链表节点的对象，这似乎比基于数组的队列执行了更多的工作。然而，虽然它拥有更好的内存分配与GC等开销，但与基于数组的队列相比，链表队列的put和take方法支持并发性更高的访问，因为一些优化方法后的链表队列算法能将队列头节点的更新操作和尾节点的更新操作分离开来。由于内存分配操作通常是线程本地的，因此如果算法能通过多执行一些内存分配操作来降低竞争程度，那么这种算法通常具有更高的可伸缩性。

响应性衡量：测量服务时间的变化情况，使我们能回答一些关于服务质量的问题，比如：“操作在100毫秒内成功执行的百分比是多少？”。
如图，给出了不同TimedPutTakeTest中每个任务的完成时间，其中使用了缓存为1000，并发任务为256，元素数量为1000。其中每个任务使用了非公平信号量（隐蔽栅栏，Shaded Bars）、公平信号量（开放栅栏，Open Bars）两种。
![](/assets/jcip_note/timedputtaketest_completion_time_hist.png)
非公平信号量的完成时间变动范围为104\~8714ms，相差80倍，通过在同步控制中实现更高的公平性，可以缩小变动范围：公平信号量的时间变动范围为38194\~38207，成功降低了变动性，但极大降低了吞吐量。
为了说明公平性开销主要由于线程阻塞造成的，我们可以将缓存大小设为1，再运行测试，结果如图：
![](/assets/jcip_note/timedputtaketest_completion_time_hist_with_single.png)

此时非公平信号量和公平信号量的执行性能基本相当，这种情况下，公平性并不会使平均完成时间变长，或者使变动性变小。

### 避免性能测试的陷阱
- 垃圾回收
有两种策略防止垃圾回收操作对测试的产生的偏差：
1. 确保垃圾回收操作在测试运行的整个过程不会执行（用JVM参数-verbose:gc，判断是否执行了垃圾回收）
2. 确保垃圾回收操作在测试期间执行多次。
- 动态编译
动态编译对测试结果带来的偏差，如图：

防止动态编译产生的偏差：

1. 使程序运行足够长的时间
2. 使代码预先运行一段时间且不测试这段时间的代码性能

- 对代码路径的不真实采样
运行时编译器根据收集到的信息对已编译的代码进行优化，JVM可以与执行过程特定的信息生成更优化的代码，这意味着在编译某个程序的方法M时生成的代码，将可能与编译另一个不同程序中的方法M时生成的代码不同。
在某些情况下，JVM可能会基于一些只是临时有效的假设进行优化，并在这些假设失效时抛弃已编译的代码。
因此，测试程序不仅要大致判断某个典型应用程序的使用模式，还需要尽量覆盖在该应用程序中将执行的代码路径集合。比如，即便只想测试单线程的性能，也应该将单线程的性能测试与多线程的性能测试结合在一起。

- 不真实的竞争程度
如果有N个线程从共享工作队列中获取任务并执行它们，并且这些任务都是计算密集型且运行时间较长，那么这种情况下几乎不存在竞争，吞吐量仅受限于CPU资源的可用性。
要获得更有意义的结果，在并发性能测试中应该尽量模拟典型应用程序中的线程本地计算量以及并发协调开销。

- 无用代码的消除 
优化编译器能找出并消除哪些不会对输出结果产生任何影响的无用代码（Dead Code）。由于基准测试通常不会执行任何计算，因此它们很容易在编译器优化过程中被消除。
在多处理器系统上，无论正式版本还是测试版本，都应该选择-server模式而不是-client模式——只是在测试程序时必须保证它们不会受到无用代码消除优化的影响。

### 其他的测试方法
虽然我们希望“找出所有的错误”，但这是一个不切实际的目标，NASA在测试中投入的资源比任何商业集团投入的都要多（据估计，NASA没雇佣一名开发，就会雇佣20名测试人员），但它们生产的代码仍然存在缺陷。
测试的目的不是更多地发现错误，而是提高代码能按照预期方式工作的可信度。
质量保证（Quality Assurance，QA）的目标应该是在给定测试资源下实现最高的可信度。
- 代码审查：多人参与的代码审查通常是不可替代的。除了发现错误，还能提高描述实现细节的注释的质量
- 静态分析工具：如FindBugs
- 面向方面的测试技术：AOP可以用来确保不变性条件不被破坏，或者与同步策略的某些方面保持一致。
- 分析与检测工具：如内置的JMX代理