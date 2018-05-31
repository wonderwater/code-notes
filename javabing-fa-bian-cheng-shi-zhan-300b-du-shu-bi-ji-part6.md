# 《Java并发编程实战》读书笔记 part6
## 避免活跃性危险

### 死锁
经典的哲学家进餐问题：五个哲学家围圆桌边吃饭，他们有五只筷子，并且每两个人的中间放一根筷子。哲学家们时而思考，时而进餐，每个人都需要一双筷子才能吃到东西，并在吃完后将筷子放回原处继续思考。

||描述|结果|
|:--:|:--|:--|
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

### 正确性测试
### 性能测试
### 避免性能测试的陷阱
### 其他的测试方法
