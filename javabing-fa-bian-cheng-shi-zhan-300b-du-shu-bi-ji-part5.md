# 《Java并发编程实战》读书笔记 part5

## 线程池的使用

线程池配置与调优的高级选项分析和示例

### 在任务与执行策略之间的隐性耦合

Executor框架将任务的提交和任务的执行策略解耦开来。虽然Executor框架为定制和修改执行策略提供了相当大的灵活性，但并非所有任务都能适用所有的执行策略，比如：  
1. 依赖性任务：如果提交给线程池的任务需要依赖其他任务，那么就隐含地给执行策略带来了约束，此时必须小心维持这些执行策略以避免产生活跃性问题。  
2. 适用线程封闭机制的任务：比如任务由于缺少同步等，要求执行所在的Executor是单线程的  
3. 对响应时间敏感的任务：比如GUI应用程序，如果多个运行时间较长的任务提交到只包含少量线程池中，会降低Executor管理服务的响应性。  
4. 使用ThreadLocal的任务：条件允许，Executor会重用线程，可能会使得新的任务异常。

只有当任务都是同类型的并且相互独立时，线程池的性能才能达到最佳。  
如果将运行时间较长和较短的任务混在一起，那么除非线程非常大，否则可能造成“拥堵”。  
如果提交的任务依赖于其他任务，除非线程非常大，否则可能造成死锁。  
线程任务死锁：

```java
public class ThreadDeadlock {
    ExecutorService exec = Executors.newSingleThreadExecutor();

    public class RenderPageTask implements Callable<String> {
        public String call() throws Exception {
            Future<String> header, footer;
            header = exec.submit(new LoadFileTask("header.html"));
            footer = exec.submit(new LoadFileTask("footer.html"));
            String page = renderBody();
            // Will deadlock -- task waiting for result of subtask
            return header.get() + page + footer.get();
            // 在线程池容量有限，这两个新提交的任务不会被调度，因此正执行的本任务也不会结束，造成死锁。
        }
    }
}
```

如果使用单线程Executor，那么ThreadDeadlock会经常发生死锁。同样，如果线程池不够大，那么多个任务通过栅栏（Barrier）机制执行来协调时，将导致线程饿死死锁。（栅栏机制要求指定数量任务在栅栏wait，有可能其中一些任务由于缺少线程调度使整体无法通过栅栏）。  
运行时间较长的任务会影响整体的调度时间，对于这些任务可以限定等待资源时间，通过一些方法，比如：Thread.join、BlockingQueue.put、CountDownLatch.await、Selector.select等。超时的任务标记为失败，终止或重新放回队列，将线程释放出来执行一些更快完成的任务。

### 设置线程池的大小

线程池设置过小：导致许多空闲的处理器无法执行工作，从而降低吞吐率。  
线程池设置过大：大量的线程将在相对较少的CPU和内存资源竞争，导致更大的开销。  
设置线程池大小，要分析计算环境、资源预算、任务特性，比如：  
1. 部署系统的CPU数？  
2. 多大内存？  
3. 任务是计算密集型还是I/O密集型还是二者皆可？  
4. 是否需要像JDBC连接这样的稀缺资源？  
...

**如果需要执行不同类型的任务，并且他们之间的行为相差很大，那么应该考虑使用多个线程，从而使每个线程池根据各自工作负载来调整**  
计算密集型任务，在拥有$$N_{cpu}$$个处理器的系统上， 当线程池大小为$$N_{cpu}+1$$时，通常能实现最优利用率。（即使当密集型线程偶尔由于缺页故障或者其他原因而暂停时，这个额外的线程也能确保CPU时钟周期不被浪费）。  
I/O操作或阻塞操作的任务，由于线程不会一直执行，因此线程池应该设更大。考虑估算任务等待时间与计算时间的比值，或者在某一个基准负载下，分别设置不同大小的线程来运行应用程序，并观察CPU利用率水平。

$$N_{cpu}$$：CPU数量，`Runtime.getRuntime().availableProcessors();`  
$$U_{cpu}$$：目标CPU利用率，取值区间\[0,1\]  
$$\frac WC$$：等待时间 / 计算时间  
使得处理器达到期望使用率，线程池的最优大小为：

$$N_{threads}=N_{cpu} * U_{cpu}*(1+\frac WC)$$

当然，CPU周期并不是唯一影响线程池大小的资源，还包括内存、文件句柄、套接字句柄、数据库连接等，衡量这些资源的影响：计算每个任务对该资源的需求量，然后用该资源的可用量总数除以每个任务需求量，所得结果就是线程大小上限。

### 配置ThreadPoolExecutor

ThreadPoolExecutor为Executor提供了基本实现，ThreadPoolExecutor的构造函数：

```java
public ThreadPoolExecutor(
    int corePoolSize,    // 基本大小
    int maximumPoolSize, // 最大大小
    long keepAliveTime, // 存活时间
    TimeUnit unit, 
    BlockingQueue<Runnable> workQueue,// 工作队列
    ThreadFactory threadFactory, // 线程工厂，默认新建非守护线程，主要用于收集统计、调试信息
    RejectedExecutionHandler handler // 拒绝执行器，提供饱和策略
                        ) { ... }
```

线程池的基本大小（Core Pool Size）、最大大小（Maximum Pool Size）以及存活时间等因素共同负责线程的创建与销毁。

* 基本大小：线程池目标大小，只有在工作队列满了，才会超出这个大小
* 最大大小：同时活动的线程数量的上限
* 存活时间：某个线程的空闲时间超出存活时间，将被标记为可回收，当线程池大小超出基本大小，这个线程被终止（Java6新增allowCoreThreadTimeOut，允许在基本大小内回收可回收线程）

工作队列对于任务排队的方法有三种：  
1. 无界队列：比如newFixedThreadPool、newSingleThreadPool使用了无界的LinkedBlockingQueue  
2. 有界队列：队列被填满时，通过饱和策略处理这种情况  
3. 同步移交：比如通过SynchronousQueue，这不是一个真正队列，而是线程间移交的机制，比如在newCachedThreadPool的使用

有界队列被填满时，通过饱和策略处理这种情况（如果某个任务被提交到一个已关闭的Executor，也会用到饱和策略）  
JDK提供了几种不同的饱和策略RejectedExecutionHandler的实现：

1. AbortPolicy：抛出RejectedExecutionException异常。默认的策略。
2. CallerRunsPolicy：在调用者线程执行任务。比如一个socket服务，当线程池满了，由于该饱和策略，主线程一段时间内不accept新的请求，不会有更多工作进来，减轻整体的压力，实现在高负载下平缓的性能降低。
3. DiscardPolicy：直接放弃任务。
4. DiscardOldestPolicy：放弃下一个任务，在尝试重新加入任务，注意队列的特性，有可能优先级最高的任务被放弃了。  
   我们可以通过限制任务的到达率，避免工作队列被填满：（题外话：springbatch的简单step运行实现的线程控制也类似）

   ```java
   @ThreadSafe
   public class BoundedExecutor {
    private final Executor exec;
    private final Semaphore semaphore;

    public BoundedExecutor(Executor exec, int bound) {
        this.exec = exec;
        this.semaphore = new Semaphore(bound);
    }
    public void submitTask(final Runnable command)
            throws InterruptedException {
        semaphore.acquire();
        try {
            exec.execute(new Runnable() {
                public void run() {
                    try {
                        command.run();
                    } finally {
                        semaphore.release();
                    }
                }
            });
        } catch (RejectedExecutionException e) {
            semaphore.release();
        }
    }
   }
   ```

   构造完成后的ThreadPoolExecutor仍可以通过Setter修改，比如线程池基本大小、最大大小、存活时间、线程工厂、拒绝执行器等。通过unconfigurableExecutorService包装可屏蔽这些Setter方法。

   ### 扩展ThreadPoolExecutor

ThreadPoolExecutor提供了几个可在子类改写的方法：

* beforeExecute：如果抛出InterruptException，任务不被执行，afterExecute也不被调用
* afterExecute：任务正常或者抛出异常而返回都会调用，（？存疑：但如果任务完成后带有Error，则不会调用）
* terminated：所有任务完成、所有工作者线程已关闭后调用
  可添加日志、计时、监视、统计信息收集功能

### 递归算法的并行化

当循环中的迭代操作都是独立地，不需要等待所有迭代操作完成再继续，可用通过Executor将串行循环转为并行循环：

```java
  void processSequentially(List<Element> elements) {
      for (Element e : elements)
          process(e);
  }
// == 转换 ==>
  void processInParallel(Executor exec, List<Element> elements) {
      for (final Element e : elements)
          exec.execute(new Runnable() {
              public void run() {
                  process(e);
              }
          });
  }
```

如果提交任务需要等待他们全部完成，可以使用ExecutorService.invokeAll等待所有结果返回，或超时。  
如果想逐步获取已完成的结果，可以使用CompletionService。  
应用：深度优先方法来遍历树结构：

```java
public<T> void sequentialRecursive(List<Node<T>> nodes,
    Collection<T> results) {
    for (Node<T> n : nodes) {
        results.add(n.compute());
        sequentialRecursive(n.getChildren(), results);
    }
}
public<T> void parallelRecursive(final Executor exec, List<Node<T>> nodes, final Collection<T> results) {
    for (final Node<T> n : nodes) {
        exec.execute(new Runnable() {
            public void run() {
                results.add(n.compute());
            }
        });
        parallelRecursive(exec, n.getChildren(), results);
    }
}
```

等待所有结果完成：

```java
public<T> Collection<T> getParallelResults(List<Node<T>> nodes)
        throws InterruptedException {
    ExecutorService exec = Executors.newCachedThreadPool();
    Queue<T> resultQueue = new ConcurrentLinkedQueue<T>();
    parallelRecursive(exec, nodes, resultQueue);
    exec.shutdown();
    exec.awaitTermination(Long.MAX_VALUE, TimeUnit.SECONDS);
    return resultQueue;
}
```

示例：谜题框架  
一些谜题需要找出一系列的操作从初始状态转换到目标状态，例如类似“搬箱子”、“Hi-Q”、“四色方柱”等  
谜题定义：包含一个初始位置、一个目标位置、以及用于判断是否有效移动的规则集。规则集包含两个部分：计算从指定位置开始的所有合法移动，以及每次移动的位置结果。  
谜题抽象类：

```java
// P，M分别是位置类、移动类，表示位置、移动信息
public interface Puzzle<P, M> {
    P initialPosition();
    boolean isGoal(P position);
    Set<M> legalMoves(P position);
    P move(P position, M move);
}
```

链表节点：

```java
@Immutable
static class Node<P, M> {
    final P pos;
    final M move;
    final Node<P, M> prev;
    Node(P pos, M move, Node<P, M> prev) {...}

    List<M> asMoveList() {
        List<M> solution = new LinkedList<M>();
        for (Node<P, M> n = this; n.move != null; n = n.prev) // 沿着节点回溯即可找出解
            solution.add(0, n.move);
        return solution;
    }
}
```

一个谜题框架的串行解决方案：

```java
public class SequentialPuzzleSolver<P, M> {
    private final Puzzle<P, M> puzzle;
    private final Set<P> seen = new HashSet<P>();
    public SequentialPuzzleSolver(Puzzle<P, M> puzzle) {
        this.puzzle = puzzle;
    }
    public List<M> solve() {
        P pos = puzzle.initialPosition();
        return search(new Node<P, M>(pos, null, null));
    }
    private List<M> search(Node<P, M> node) {
        if (!seen.contains(node.pos)) {
            seen.add(node.pos);
            if (puzzle.isGoal(node.pos))
                return node.asMoveList();
            for (M move : puzzle.legalMoves(node.pos)) {
                P pos = puzzle.move(node.pos, move);
                Node<P, M> child = new Node<P, M>(pos, move, node);
                List<M> result = search(child);
                if (result != null)
                    return result;
            }
        }
        return null;
    }
    static class Node<P, M> { ... }
}
```

改进的措施：  
1. 很多节点存在重复计算 =&gt; 保存曾经计算过的状态，剪枝  
2. 引入并发

```java
public class ConcurrentPuzzleSolver<P, M> {
    private final Puzzle<P, M> puzzle;
    private final ExecutorService exec;
    // 保存曾经计算过的状态
    private final ConcurrentMap<P, Boolean> seen;
    final ValueLatch<Node<P, M>> solution = new ValueLatch<Node<P, M>>();
    ...
    public List<M> solve() throws InterruptedException {
        try {
            P p = puzzle.initialPosition();
            exec.execute(newTask(p, null, null));
            // block until solution found
            Node<P, M> solnNode = solution.getValue();
            return (solnNode == null) ? null : solnNode.asMoveList();
        } finally {
            exec.shutdown();
        }
    }

    protected Runnable newTask(P p, M m, Node<P,M> n) {
        return new SolverTask(p, m, n);
    }

    class SolverTask extends Node<P, M> implements Runnable {
        ...
        public void run() {
            if (solution.isSet() || seen.putIfAbsent(pos, true) != null)
                return; // already solved or seen this position
            if (puzzle.isGoal(pos))
                solution.setValue(this);
            else
                for (M m : puzzle.legalMoves(pos))
                    exec.execute(newTask(puzzle.move(pos, m), m, this));
        }
    }
}
```

相比串行版本的深度优先，并行版本执行的是广度优先，更不会受到栈大小的限制。  
再来看获取结果的阻塞的实现：

```java
@ThreadSafe
public class ValueLatch<T> {
    @GuardedBy("this") private T value = null;
    private final CountDownLatch done = new CountDownLatch(1);
    public boolean isSet() {
        return (done.getCount() == 0);
    }
    public synchronized void setValue(T newValue) {
        if (!isSet()) {
            value = newValue;
            done.countDown();
        }
    }
    public T getValue() throws InterruptedException {
        done.await();
        synchronized (this) { // 这把内置锁应该是简单地解决可见性
            return value;
        }
    }
}
```

最后，处理谜题问题无解的情况：

```java
public class PuzzleSolver<P,M> extends ConcurrentPuzzleSolver<P,M> {
    ...
    private final AtomicInteger taskCount = new AtomicInteger(0);
    protected Runnable newTask(P p, M m, Node<P,M> n) {
        return new CountingSolverTask(p, m, n);
    }
    class CountingSolverTask extends SolverTask {
        CountingSolverTask(P pos, M move, Node<P, M> prev) {
            super(pos, move, prev);
            taskCount.incrementAndGet();
        }
        public void run() {
            try {
                super.run();
            } finally {
                // 最后一个任务执行结束，尝试设为无解，以唤醒可能的由于调用getValue()而阻塞的线程
                if (taskCount.decrementAndGet() == 0)
                    solution.setValue(null);
            }
        }
    }
}
```

## 图形用户界面应用程序

使用Swing编写GUI应用程序，为了维持安全性，一些特定的任务必须运行在Swing的事件线程中。  
在事件线程中不应该执行耗时较长的操作，以免失去响应。  
由于Swing的数据结构不是线程安全的，因此必须将它们限制在事件线程中。  
几乎所有的GUI工具包（包括Swing和SWT）都被实现为单线程子系统，即所有GUI操作都被限制在单个线程中。

### 为什么GUI是单线程的

单线程的GUI框架不仅限于Java，在Qt、NexiStep、MacOS Cocoa、X Windows以及其他环境的GUI框架都是单线程的。  
曾经有很多尝试多线程的GUI框架，但最终都由于竞态条件和死锁导致的稳定性问题而重新回到单线程的事件队列模块。（比如最初很大程度支持多线程的AWT，在吸取经验和教训后，Swing的实现决定采用单线程模型）。

多线程的GUI框架中更容易引发死锁问题的原因分析：  
1. 在输入事件的处理过程与GUI组件面向对象模型之间会存在错误的交互。比如，用户的点击事件，通过操作系统的捕获包装给应用程序处理（类似气泡上升），而应用程序引发修改背景颜色的请求，类似“气泡下沉”的方式将请求转发给操作系统，因此，一方面这两组操作以完全相反的顺序访问相同的GUI对象（气泡方向完全不一样）；另一方面又要确保每个对象都是线程安全的，从而导致不一致的锁定顺序，并引发死锁。  
2. 常见的MVC设计模型的问题：C调用M，M可以将其变化通知给V，C同样可以调用V，V可以调回M查询状态，这将导致不一致锁定顺序并出现死锁。

Swing中的线程封闭机制：所有Swing组件（如JButton）和数据模型对象（如TableModel）都被封闭在事件线程中，因此任何访问它们的代码都必须在事件线程中运行，即Swing中的组件及模型都会只能在这个事件分发线程中进行创建、修改、查询。当然也有例外：

1. SwingUtilities.isEventDispatchThread：判断当前是否是事件线程
2. SwingUtilities.invokeLater：将一个Runnable任务调度到事件线程中执行
3. SwingUtilities.invokeAndWait：将一个Runnable任务调度到事件线程中执行并阻塞到任务完成
4. 所有重绘（Repaint）请求或重新生效（Revalidation）请求请求插入队列的方法
5. 所有添加或移除监听器的方法

我们用Executor来模拟Swing中SwingUtilities的实现：

```java
public class SwingUtilities {
    private static final ExecutorService exec =
            Executors.newSingleThreadExecutor(new SwingThreadFactory());
    private static volatile Thread swingThread;

    private static class SwingThreadFactory implements ThreadFactory {
        public Thread newThread(Runnable r) {
            swingThread = new Thread(r);
            return swingThread;
        }
    }

    public static boolean isEventDispatchThread() {
        return Thread.currentThread() == swingThread;
    }

    public static void invokeLater(Runnable task) {
        exec.execute(task);
    }

    public static void invokeAndWait(Runnable task)
            throws InterruptedException, InvocationTargetException {
        Future f = exec.submit(task);
        try {
            f.get();
        } catch (ExecutionException e) {
            throw new InvocationTargetException(e);
        }
    }
}
```

可以将Swing的事件线程视为一个单线程的Executor，由它处理事件队列的任务。我们的实现如下：

```java
public class GuiExecutor extends AbstractExecutorService {
    // Singletons have a private constructor and a public factory
    private static final GuiExecutor instance = new GuiExecutor();

    private GuiExecutor() {
    }

    public static GuiExecutor instance() {
        return instance;
    }

    public void execute(Runnable r) {
        if (SwingUtilities.isEventDispatchThread())
            r.run();
        else
            SwingUtilities.invokeLater(r);
    }

    public void shutdown() {
        throw new UnsupportedOperationException();
    }
    ...
}
```

### 短时间的GUI任务

事件在事件线程中产生，并通过“气泡上升”的方式传递给应用程序提供的监听器，而监听器根据收到的事件执行一些计算来修改表现对象，短时间的任务可以把整个操作都放在监听器所在的事件线程中执行里。  
一个按钮监听器：

```java
final Random random = new Random();
final JButton button = new JButton("Change Color");
...
button.addActionListener(new ActionListener() {
    // 只要任务是短时间的，并且只访问GUI对象（或其他线程封闭或线程安全的应用程序对象）即可在此执行。
    public void actionPerformed(ActionEvent e) {
        button.setBackground(new Color(random.nextInt()));
    }
});
```

点击时的执行控制流：

![](/assets/jcip_note/gui_click_control_flow.png)

Swing将大多数可视化组件分为两个对象：  
1. 模型对象：保存的是将被显示的数据  
2. 视图对象：保存控制显示方式的规则  
视图对象订阅模型对象的变化。当模型对象被改变，触发`fireXxx`方法，调用视图对象的监听器  
它们间的控制流：

![](/assets/jcip_note/gui_click_model_control_flow.png)

### 长时间的GUI任务

一些复杂的耗时任务必须在另一个线程中运行，才能使得GUI在运行时保持高响应性，比如：拼写检查、后台编辑、获取远程资源等。  
只有GUI应用程序很少会发起大量的长时间任务，因此即使线程池可以无限制地增长也不会有太大风险。  
从一个简单的例子开始：

```java
ExecutorService backgroundExec = Executors.newCachedThreadPool();
...
button.addActionListener(new ActionListener() {
    // Fire and Forget，事件提交后，就没有后续的反馈了，比如完成指示、进度指示等
    public void actionPerformed(ActionEvent e) {
        backgroundExec.execute(new Runnable() {
            public void run() { doBigComputation(); }
        });
}});
```

我们加上可视化的反馈：

```java
// 点击后立即禁用按钮，执行任务完成后恢复按钮
button.addActionListener(new ActionListener() {
    public void actionPerformed(ActionEvent e) {
        button.setEnabled(false);
        label.setText("busy");
        backgroundExec.execute(new Runnable() {
            public void run() {
                try {
                    doBigComputation();
                } finally {
                    GuiExecutor.instance().execute(new Runnable() {
                        public void run() {
                            button.setEnabled(true);
                            label.setText("idle");
                        }
                    });
                }
            }
        });
    }
});
```

在按下按钮是触发的任务包含3个连续子任务，分别是  
1. 更新用户界面并启动第二个子任务  
2. 执行任务结束后把第三个任务交回事件线程中运行  
3. 更新用户界面来表示操作完成。  
以上1、2、3三个任务“线程接力”是处理长时间任务的典型方法。

我们再来加上任务取消操作：

```java
Future<?> runningTask = null; // 被封闭在事件线程中，无需同步
...
startButton.addActionListener(new ActionListener() {
    public void actionPerformed(ActionEvent e) {
        if (runningTask == null) {
            runningTask = backgroundExec.submit(new Runnable() {
                public void run() {
                    while (moreWork()) {
                        // 检查中断状态，及时响应取消操作，退出任务执行
                        if (Thread.currentThread().isInterrupted()) {
                            cleanUpPartialWork();
                            break;
                        }
                        doSomeWork();
                    }
                }
            });
    };
}});
cancelButton.addActionListener(new ActionListener() {
    public void actionPerformed(ActionEvent event) {
        if (runningTask != null)
            runningTask.cancel(true);
}});
```

加上进度标识和完成标识的版本：

```java
abstract class BackgroundTask<V> implements Runnable, Future<V> {
    private final FutureTask<V> computation = new Computation();

    private class Computation extends FutureTask<V> {
        public Computation() {
            super(new Callable<V>() {
                public V call() throws Exception {
                    return BackgroundTask.this.compute();
                }
            });
        }
        @Override // FutureTask任务结束的回调
        protected final void done() {
            GuiExecutor.instance().execute(new Runnable() {
                public void run() {
                    V value = null;
                    Throwable thrown = null;
                    boolean cancelled = false;
                    try {
                        value = get();
                    } catch (ExecutionException e) {
                        thrown = e.getCause();
                    } catch (CancellationException e) {
                        cancelled = true;
                    } catch (InterruptedException consumed) {
                    } finally {
                        onCompletion(value, thrown, cancelled);
                    }
                };
            });
        }
    }

    protected void setProgress(final int current, final int max) {
        GuiExecutor.instance().execute(new Runnable() {
            public void run() { onProgress(current, max); }
        });
    }
    public void run() { computation.run(); }

    // 把以下三个方法聚在同一个类中：耗时任务、进度和完成回调
    // Called in the background thread
    protected abstract V compute() throws Exception;
    // Called in the event thread
    protected void onCompletion(V result, Throwable exception, boolean cancelled) { }
    protected void onProgress(int current, int max) { }
    // Other Future methods forwarded to computation
}
```

### 共享数据模型

最简单情况下，数据模型中的数据由用户来输入或者有应用程序在启动时静态地从文件或其他数据源加载，这时除了事件线程外的其它线程都不可能访问数据。当数据在应用程序进出时，有多个线程都可以访问这些数据，要注意数据模型的线程安全。  
分解数据模型设计：从GUI的角度，Swing的表格模型类，如TableModel，都是保存将要显示的数据。然而，这些模型对象本身通常都是应用程序中其他对象的“视图”，如果在程序中既包含用于表示的数据模型，又包含应用程序特定的数据模型，那么这种应用程序就被称为拥有一种分解模型设计。  
在分解模型中，表现模型被封闭在事件线程中，而其他模型，即共享模型，是线程安全的。  
表现模型会注册共享模型的监听器，从而得到更新时通知，然后，表现模型可以从共享模型中得到更新：通过将相关的快照嵌入到更新消息中，或者由表现模型在收集到更新事件时直接从共享模型中获取数据。（这里其实是订阅者模式，推/拉的实现）  
如果一个数据模型必须被多个线程共享，而且由于阻塞、一致性或者复杂度等原因而无法实现一个线程安全的模型时，可以考虑使用分解模型设计。

### 其他形式的单线程子系统

有时候，当程序员无法避免同步或死锁等问题时，也将不得不使用线程封闭，例如，一些原生库（Native Library）要求：所有对库的访问，甚至当通过System.loadLibrary来加载库时，都必须放在同一个线程中执行。

