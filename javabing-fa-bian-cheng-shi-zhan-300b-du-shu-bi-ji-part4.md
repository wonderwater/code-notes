# 《Java并发编程实战》读书笔记 part4

## 任务执行

大多数并发程序都是围绕任务执行（Task Execution）来构造的，任务通常是一些抽象的且离散的工作单元。

### 在线程中执行任务

首要的是找出清晰的任务边界。理想情况下，各个任务之间应该相互独立：任务并不依赖于其他任务的状态、结果、边界效应。

独立有助于实现并发，因为如果存在足够多的处理资源，那么这些独立地任务都可以并行执行。

正常负载下，服务器应用程序应该同时表现出良好的吞吐量和快速地响应性。应用程序提供商希望程序支持尽可能多的用户，从而降低每个用户的服务成本，而用户希望获得尽快的响应，当负载增大时，应用程序的性能应该是逐渐降低的，而不是直接失败。要实现上述目标，应该选择清晰的任务边界以及明确地任务执行策略。

串行地执行任务，每次只能执行一个用户请求，性能很糟，响应速度慢，不能充分利用资源：

```java
public class SingleThreadWebServer {
    public static void main(String[] args) throws IOException {
        ServerSocket socket = new ServerSocket(80);
        while (true) {
            Socket connection = socket.accept();
            handleRequest(connection);
        }
    }

    private static void handleRequest(Socket connection) {
        // request-handling logic here
    }
}
```

显式地为任务创建线程：

```java
public class ThreadPerTaskWebServer {
    public static void main(String[] args) throws IOException {
        ServerSocket socket = new ServerSocket(80);
        while (true) {
            final Socket connection = socket.accept();
            Runnable task = new Runnable() {
                public void run() {
                    handleRequest(connection);
                }
            };
            new Thread(task).start();
        }
    }

    private static void handleRequest(Socket connection) {
        // request-handling logic here
    }
}
```

在代码结构上类似单线程程序，区别在于对于每个请求都独立创建一个线程运行。我们希望的的结论：

* 任务处理过程从主线程中分离出来，从而主线程可以更快重新接受下一个请求。
* 任务可以并行处理，从而同时服务多个请求。
* 处理程序必须线程安全。

在请求的到达速率不超过服务器请求处理能力，这种方法可以带来更快的响应性和更高的吞吐率。

这种无限制创建线程也有问题，尤其是需要大量的线程的时候：

* 线程生命周期的开销非常高
* 资源消耗：线程调度，内存资源等
* 稳定性：可创建的线程数存在限制，依据平台的不同而不同。

我们需要限制可创建线程的数量。

## Executor框架

任务执行的主要抽象不是Thread，而是Executor

```java
public interface Executor {
    void execute(Runnable command);
}
```

Executor将任务的提交过程和执行过程解耦开，它的提供了对线程声明周期的支持，以及统计信息收集、应用程序管理机制和性能监视等机制。可将任务的提交操作和执行操作分别看出生产者，消费者。

执行策略中定义了任务执行的“what/where/when/how”等方面，包括

* 在什么（what）线程中执行？
* 任务按照什么（what）顺序执行？
* 有多少（how many）个任务能并发执行？
* 在队列中有多少（how many）个任务在等待执行？
* 如果系统由于负载而需要拒绝一个任务，那么应该选哪个（which）？另外，如何（how）通知应用程序有任务被拒绝？
* 在执行一个任务的前后，应该进行哪些（what）动作？

我们引入线程池，管理一组同构工作线程的资源池，它与工作队列（work queue）密不可分，在工作队列中保存所有等待执行的任务。工作者线程（worker thread）的任务很简单，从工作队列中取任务执行，然后返回线程池并等待下一个任务。

类库提供的静态工厂方法：

* newFixedThreadPool  固定长度的线程池。
* newCachedThreadPool 工作者线程结束后，等待60s取任务，无线程大小限制，没有队列。
* newSingleThreadExecutor  单线程，无限等待队列。
* newScheduledThreadPool 以延迟或定时的方式执行任务。

Executor的的关闭：

```java
public interface ExecutorService extends Executor {
    void shutdown(); // 停止接受任务
    List<Runnable> shutdownNow(); // 停止接受任务并中断当前的所有任务
    boolean isShutdown(); // 是否是不再接受任务
    boolean isTerminated(); // 是否全部关闭了
    boolean awaitTermination(long timeout, TimeUnit unit) throws InterruptedException; // 试着等待一段时间，直到关闭
    // ... additional convenience methods for task submission
}
```

上面提到了newScheduledThreadPool，和Timer类类似，但是Timer存在一些缺陷：

* Timer只创建一个线程，如果某个任务执行时间过长，将破坏其他TimerTask的准确性。
* 如果TimerTask抛出了一个未检查的异常，那么Timer将不捕获，直接抛出终止定时线程，并且不会恢复线程的执行。

```java
public class OutOfTime {
    public static void main(String[] args) throws Exception {
        Timer timer = new Timer();
        timer.schedule(new ThrowTask(), 1);
        SECONDS.sleep(1);
        timer.schedule(new ThrowTask(), 1); // 抛出异常 java.lang.IllegalStateException: Timer already cancelled.
        SECONDS.sleep(5);
    }

    static class ThrowTask extends TimerTask {
        public void run() { throw new RuntimeException(); }
    }
}
```

这个问题称为“线程泄漏”（Thread Leakage）

## 找出可利用的并行性示例

html页面渲染：渲染文字，下载图片，显示图片，代码如下：

```java
public abstract class SingleThreadRenderer {
    void renderPage(CharSequence source) {
        renderText(source);
        List<ImageData> imageData = new ArrayList<ImageData>();
        for (ImageInfo imageInfo : scanForImageInfo(source))
            imageData.add(imageInfo.downloadImage());
        for (ImageData data : imageData)
            renderImage(data);
    }
}
```

图片下载过程大部分是等待I/O完成的时间，cpu几乎不做任何工作。将如上过程分为两个任务：渲染所有文本和下载所有图片，其中一个是CPU密集型，一个是I/O密集型，因此即使单CPU的系统也能有性能提升，代码如下。

```java
public abstract class FutureRenderer {
    private final ExecutorService executor = Executors.newCachedThreadPool();

    void renderPage(CharSequence source) {
        final List<ImageInfo> imageInfos = scanForImageInfo(source);
        Callable<List<ImageData>> task =
                new Callable<List<ImageData>>() {
                    public List<ImageData> call() {
                        List<ImageData> result = new ArrayList<ImageData>();
                        for (ImageInfo imageInfo : imageInfos)
                            result.add(imageInfo.downloadImage());
                        return result;
                    }
                };

        Future<List<ImageData>> future = executor.submit(task);
        renderText(source);

        try {
            List<ImageData> imageData = future.get();
            for (ImageData data : imageData)
                renderImage(data);
        } catch (InterruptedException e) {
            // Re-assert the thread's interrupted status
            Thread.currentThread().interrupt();
            // We don't need the result, so cancel the task too
            future.cancel(true);
        } catch (ExecutionException e) {
            throw launderThrowable(e.getCause());
        }
    }
}
```

CompletionService将Executor和BlockingQueue的功能融合在一起，我们使用后，将图像的串行下载过程转为并行过程，减少下载所有图像的总时间，另外每完成一幅图像立即渲染，提高界面响应性，代码如下。

```java
public abstract class Renderer {
    private final ExecutorService executor;

    Renderer(ExecutorService executor) {
        this.executor = executor;
    }

    void renderPage(CharSequence source) {
        final List<ImageInfo> info = scanForImageInfo(source);
        CompletionService<ImageData> completionService =
                new ExecutorCompletionService<ImageData>(executor);
        for (final ImageInfo imageInfo : info)
            completionService.submit(new Callable<ImageData>() {
                public ImageData call() {
                    return imageInfo.downloadImage();
                }
            });

        renderText(source);
        try {
            for (int t = 0, n = info.size(); t < n; t++) {
                Future<ImageData> f = completionService.take();
                ImageData imageData = f.get();
                renderImage(imageData);
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } catch (ExecutionException e) {
            throw launderThrowable(e.getCause());
        }
    }
}
```

## 为任务设置时限

示例，渲染广告，超时则放弃。

```java
Page renderPageWithAd() throws InterruptedException {
    long endNanos = System.nanoTime() + TIME_BUDGET;
    Future<Ad> f = exec.submit(new FetchAdTask());
    // Render the page while waiting for the ad
    Page page = renderPageBody();
    Ad ad;
    try {
        // Only wait for the remaining time budget
        long timeLeft = endNanos - System.nanoTime(); // 可能是负数，在juc库中所有与时限相关的方法都将负数视为0，不必额外处理
        ad = f.get(timeLeft, NANOSECONDS);
    } catch (ExecutionException e) {
        ad = DEFAULT_AD;
    } catch (TimeoutException e) {
        ad = DEFAULT_AD;
        f.cancel(true);
    }
    page.setAd(ad);
    return page;
}
```

时限的场景：从每个公司获取酒店报价，只显示在指定时间内收到的信息，超时的忽略掉。使用Executor.invokeAll方法

```java
class QuoteTask implements Callable<TravelQuote> {
    private final TravelCompany company;
    private final TravelInfo travelInfo;
    ...
    public TravelQuote call() throws Exception {
        return company.solicitQuote(travelInfo);
    }
}

public List<TravelQuote> getRankedTravelQuotes(TravelInfo travelInfo, Set<TravelCompany> companies,
        Comparator<TravelQuote> ranking, long time, TimeUnit unit)
        throws InterruptedException {
    List<QuoteTask> tasks = new ArrayList<QuoteTask>();
    for (TravelCompany company : companies)
    tasks.add(new QuoteTask(company, travelInfo));

    List<Future<TravelQuote>> futures = exec.invokeAll(tasks, time, unit);

    List<TravelQuote> quotes = new ArrayList<TravelQuote>(tasks.size());
    Iterator<QuoteTask> taskIter = tasks.iterator();
    for (Future<TravelQuote> f : futures) {
        QuoteTask task = taskIter.next();
        try {
            quotes.add(f.get());
        } catch (ExecutionException e) {
            quotes.add(task.getFailureQuote(e.getCause()));
        } catch (CancellationException e) {
            quotes.add(task.getTimeoutQuote(e));
        }
    }
    Collections.sort(quotes, ranking);
    return quotes;
}
```

## 取消与关闭
要使任务能够安全、快速、可靠地停止下来，并不是一件容易的事。Java没有提供任何机制来安全地终止线程。但它提供了中断（Interruption），这是一种协作机制，能够使一个线程安全地终止另一个线程的当前工作。

### 任务取消
取消某个操作的原因很多：
1. 用户请求取消
2. 有时间限制地操作
3. 应用程序事件：比如某个算法执行中，对任务进行剪枝，取消掉没有必要执行的任务
4. 错误：磁盘满、网络超时等
5. 关闭：服务关闭时，必须对等待处理或正在处理的任务执行操作，可能等待任务完成，或者立即取消任务

取消的方式举例，一种协作机制能设置某个“已请求取消（Cancellation Requested）”标志：
```java
@ThreadSafe
public class PrimeGenerator implements Runnable {
    private static ExecutorService exec = Executors.newCachedThreadPool();
    @GuardedBy("this") private final List<BigInteger> primes
            = new ArrayList<BigInteger>();
    private volatile boolean cancelled;

    public void run() {
        BigInteger p = BigInteger.ONE;
        while (!cancelled) {
            p = p.nextProbablePrime();
            synchronized (this) {
                primes.add(p);
            }
        }
    }
    public synchronized List<BigInteger> get() {
        return new ArrayList<BigInteger>(primes);
    }
    public void cancel() { cancelled = true; }
}
```
一个可取消的任务必须拥有取消策略（Cancellation Policy），在这个策略中将详细定义取消操作的“How”、“When”、“What”，即其他代码如何（How）取消该任务，任务在何时（When）检查是否已请求了取消，以及在响应取消请求时应该执行哪些（What）操作。
如果任务中使用了阻塞方法，任务可能永远不会检查取消标记，因此永远不会结束。
```java
class BrokenPrimeProducer extends Thread {
    private final BlockingQueue<BigInteger> queue;
    private volatile boolean cancelled = false;

    BrokenPrimeProducer(BlockingQueue<BigInteger> queue) {
        this.queue = queue;
    }

    public void run() {
        try {
            BigInteger p = BigInteger.ONE;
            while (!cancelled)
                queue.put(p = p.nextProbablePrime()); // put方法阻塞，cancelled取消标志可能永远不会被检查
        } catch (InterruptedException consumed) {
        }
    }

    public void cancel() { cancelled = true; }
}
```
在Java API或语言规范中，并没有将中断与任何取消语义关联起来，但实际上，如果在取消之外的其他操作中使用中断都是不合适的，并且很难支撑起更大的应用。通常，中断是实现取消的最合理的方式。

Thread中的中断方法：
```java
public static boolean interrupted() { ... }  // 返回当前线程是否中断，并清除中断标记
public boolean isInterrupted() { ... }  // 该线程是否中断
public void interrupt() { ... } // 中断该线程
```
阻塞库方法，例如`Thread.sleep`和Object.wait等都会检查线程何时中断，并且做如下操作：
1. 清除中断状态
2. 抛出InterruptedException
我们通过中断解决阻塞方法的取消问题：

```java
public class PrimeProducer extends Thread {
    private final BlockingQueue<BigInteger> queue;

    PrimeProducer(BlockingQueue<BigInteger> queue) {
        this.queue = queue;
    }

    public void run() {
        try {
            BigInteger p = BigInteger.ONE;
            while (!Thread.currentThread().isInterrupted())
                queue.put(p = p.nextProbablePrime());
        } catch (InterruptedException consumed) {
            /* Allow thread to exit */
        }
    }

    public void cancel() { interrupt(); }
}
```
像任务中包含取消策略，线程同样应该包含中断策略，规定线程如何解释某个中断请求——当发现中断请求时，应该做哪些工作，哪些工作单元对于中断请求时原子操作，以及以多快的速度来响应中断。
任务在某个服务拥有的线程中执行，应该小心地保存中断状态，这样拥有线程的代码才能对中断做出响应。
如上提到处理InterruptedException的两种实用策略
1. 传递异常
2. 恢复中断状态
一些重新尝试的方法里，的保存中断状态，在返回前注意中断状态：
```java
public class NoncancelableTask {
    public Task getNextTask(BlockingQueue<Task> queue) {
        boolean interrupted = false;
        try {
            while (true) {
                try {
                    return queue.take();
                } catch (InterruptedException e) {
                    interrupted = true;
                    // fall through and retry
                }
            }
        } finally {
            if (interrupted)
                Thread.currentThread().interrupt();
        }
    }
}
```

计时运行的示例：
```java
public class TimedRun1 {
    private static final ScheduledExecutorService cancelExec = ...

    public static void timedRun(Runnable r,
                                long timeout, TimeUnit unit) {
        final Thread taskThread = Thread.currentThread();
        cancelExec.schedule(new Runnable() {
            public void run() {
                taskThread.interrupt();
            }
        }, timeout, unit);
        r.run();
    }
}
```
这段代码有几个问题：
1. timedRun调用者所在的线程可能被其他线程中断，提前返回
2. 任务可能在中断前执行完成，为后续的执行带来风险
3. 如果任务不响应中断，那么timeRun要在任务结束后返回，可能超出时限

改进，在专门的线程中做中断任务
```java
public class TimedRun2 {
    private static final ScheduledExecutorService cancelExec = newScheduledThreadPool(1);

    public static void timedRun(final Runnable r,
                                long timeout, TimeUnit unit)
            throws InterruptedException {
        class RethrowableTask implements Runnable {
            private volatile Throwable t;

            public void run() {
                try { r.run(); } 
                catch (Throwable t) { this.t = t; }
            }

            void rethrow() {
                if (t != null)
                    throw launderThrowable(t);
            }
        }

        RethrowableTask task = new RethrowableTask();
        final Thread taskThread = new Thread(task);
        taskThread.start();
        cancelExec.schedule(new Runnable() {
            public void run() {
                taskThread.interrupt();
            }
        }, timeout, unit);
        taskThread.join(unit.toMillis(timeout)); // Thread.join的设计缺陷，不能判断因为等待线程结束还是超时而返回
        task.rethrow();
    }
}
```
通过Future来实现取消：
```java
public class TimedRun {
    private static final ExecutorService taskExec = Executors.newCachedThreadPool();

    public static void timedRun(Runnable r,
                                long timeout, TimeUnit unit)
            throws InterruptedException {
        Future<?> task = taskExec.submit(r);
        try {
            task.get(timeout, unit);
        } catch (TimeoutException e) {
            // task will be cancelled below
        } catch (ExecutionException e) {
            // exception thrown in task; rethrow
            throw launderThrowable(e.getCause());
        } finally {
            // Harmless if task already completed
            task.cancel(true); // 中断正在执行的任务，false——取消未待执行的任务
        }
    }
}
```
Java库中，许多可阻塞的方法都是通过提前返回或者抛出InterruptedException来响应中断的，然而并非所有的可阻塞方法或阻塞机制都能响应中断。
* Java.io包中的同步Socket I/O：虽然InputStream和OutputStream的read和write方法都不会响应中断，但通过关闭底层套接字，可以使得由于执行read或write等方法而被阻塞的线程抛出SocketException。
* Java.nio包中的同步I/O：当中断一个正在InterruptibleChannel上等待的线程，将抛出ClosedByInterruptException并关闭链路（使得其他在这条链路上阻塞的线程同样抛出ClosedByInterruptException）。当关闭一个InterruptibleChannel时，将导致所有在链路操作上阻塞的线程抛出AsynchronousCloseException。
* Selector的异步I/O：如果在一个线程调用Selector.select方法阻塞了，调用close或者wakeup方法将使线程抛出ClosedSelectorException并提前返回。
* 获取某个锁：等待某个内置锁而阻塞将无法响应中断，因为线程认为它肯定能获得锁，所以不会理会中断请求。Lock类中提供了lockInterruptibly，运行在等待一把锁的同时仍可以响应中断。
封装非标准操作例子：
```java
public class ReaderThread extends Thread {
    private final Socket socket;
    private final InputStream in;

    public ReaderThread(Socket socket) throws IOException {
        this.socket = socket;
        this.in = socket.getInputStream();
    }

    public void interrupt() {
        try {
            socket.close();
        } catch (IOException ignored) {
        } finally {
            super.interrupt();
        }
    }

    public void run() {
        try {
            byte[] buf = new byte[BUFSZ];
            while (true) {
                int count = in.read(buf);
                if (count < 0)
                    break;
                else if (count > 0)
                    processBuffer(buf, count);
            }
        } catch (IOException e) { /* Allow thread to exit */ }
    }
}
```
ThreadPoolExecutor提供newTaskFor方法封装非标准操作：
```java

interface CancellableTask <T> extends Callable<T> {
    void cancel();

    RunnableFuture<T> newTask();
}


@ThreadSafe
class CancellingExecutor extends ThreadPoolExecutor {
    protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
        if (callable instanceof CancellableTask)
            return ((CancellableTask<T>) callable).newTask();
        else
            return super.newTaskFor(callable);
    }
}
public abstract class SocketUsingTask <T> implements CancellableTask<T> {
    @GuardedBy("this") private Socket socket;
    protected synchronized void setSocket(Socket s) { socket = s; }

    public synchronized void cancel() {
        try {
            if (socket != null)
                socket.close();
        } catch (IOException ignored) {
        }
    }

    public RunnableFuture<T> newTask() {
        return new FutureTask<T>(this) {
            public boolean cancel(boolean mayInterruptIfRunning) {
                try {
                    SocketUsingTask.this.cancel();
                } finally {
                    return super.cancel(mayInterruptIfRunning);
                }
            }
        };
    }
}
```
### 停止基于线程的服务
对于持有线程的服务，只要服务的存在时间大于创建线程的方法的存在时间，那么就该提供生命周期方法。比如创建线程池的方法比线程池要提前结束，线程池提供关闭方法。
一个基于生产者-消费者的日志服务：
```java
public class LogWriter {
    private final BlockingQueue<String> queue;
    private final LoggerThread logger;

    public LogWriter(Writer writer) {
        this.queue = new LinkedBlockingQueue<String>(CAPACITY);
        this.logger = new LoggerThread(writer);
    }

    public void start() {
        logger.start();
    }

    public void log(String msg) throws InterruptedException {
        queue.put(msg);
    }

    private class LoggerThread extends Thread {
        private final PrintWriter writer;

        public LoggerThread(Writer writer) {
            this.writer = new PrintWriter(writer, true); // autoflush
        }

        public void run() {
            try {
                while (true)
                    writer.println(queue.take());
            } catch (InterruptedException ignored) {
            } finally {
                writer.close();
            }
        }
    }
}
```
为了使像LogWriter这样的服务在软件产品中发挥实际作用，还需要实现一种终止日志线程的方法。
我们注意到，中断消费者线程即可停止服务，但有几个问题：
1. 会丢失哪些正在等待被写入到日志的信息
2. 其他线程在调用log被阻塞了（可能队列已满），那么这些线程无法解除阻塞状态。

我们加入“已请求关闭”标志，避免生产者进一步提交日志：
```java
public void log(String msg) throws InterruptedException {
	if (!shutdownRequested)
		queue.put(msg);
	else
		throw new IllegalStateException("logger is shut down");
}
```
这是典型的“先判断后执行”的代码序列，存在竞态条件：实现者发现服务未关闭，因此在服务关闭后仍然会将日志放入队列中，这同样会使得生产者在调用log是阻塞并无法解除阻塞状态。
可以一些技巧降低这种情况的概率：在宣布队列被清空前，然消费者等待数秒（尽可能使得消费者清空时，生产者都已结束把日志放入队列的操作），但没有从本质上解决问题。
改进：注意同步的对象不是队列。
```java
public class LogService {
    private final BlockingQueue<String> queue;
    private final LoggerThread loggerThread;
    private final PrintWriter writer;
    @GuardedBy("this") private boolean isShutdown;
    @GuardedBy("this") private int reservations;

    public LogService(Writer writer) {
        this.queue = new LinkedBlockingQueue<String>();
        this.loggerThread = new LoggerThread();
        this.writer = new PrintWriter(writer);
    }

    public void start() { loggerThread.start(); }

    public void stop() {
        synchronized (this) {
            isShutdown = true;
        }
        loggerThread.interrupt();
    }

    public void log(String msg) throws InterruptedException {
        synchronized (this) {
            if (isShutdown)
                throw new IllegalStateException(/*...*/);
            ++reservations;
        }
        queue.put(msg);
    }

    private class LoggerThread extends Thread {
        public void run() {
            try {
                while (true) {
                    try {
                        synchronized (LogService.this) {
                            if (isShutdown && reservations == 0)
                                break;
                        }
                        String msg = queue.take();
                        synchronized (LogService.this) {
                            --reservations;
                        }
                        writer.println(msg);
                    } catch (InterruptedException e) { /* retry */
                    }
                }
            } finally {
                writer.close();
            }
        }
    }
}
```
ExecutorService提供两种关闭方法：
1. shutdown：不再接收任务调度
2. shutdownNow：不再接收任务调度，并中断所有正在执行的任务

使用这种方式，对`LogService`进行关闭：
```java
public class LogService {
	private final ExecutorService exec = newSingleThreadExecutor();
	...
	public void start() { }
	public void stop() throws InterruptedException {
		try {
			exec.shutdown();
			exec.awaitTermination(TIMEOUT, UNIT);
		} finally {
			writer.close();
		}
	}
	public void log(String msg) {
	try {
		exec.execute(new WriteTask(msg));
	} catch (RejectedExecutionException ignored) { }
	}
}
```
另一种关闭生产者-消费者服务的方式是使用“毒丸（Poison Pill）”对象：“毒丸”是指一个放入在队列的对象，其含义是：当得到这个对象时，立即停止。
只有在生产者和消费者数量都已知的情况下，才可以使用“毒丸”对象。只有在无界队列中，“毒丸”对象才能可靠地工作。

shutdownNow方法尝试取消正在执行的任务，并返回所有已提交但尚未开始的任务，从而将这些任务写入日志或者保存起来以便之后进行处理。
shutdownNow的局限性：我们无法通过常规方法找出那些任务已经开始但尚未结束。
跟踪关闭后被取消的任务，通过代理方式：
```java
public class TrackingExecutor extends AbstractExecutorService {
    private final ExecutorService exec;
    private final Set<Runnable> tasksCancelledAtShutdown =
            Collections.synchronizedSet(new HashSet<Runnable>());
	...
    public List<Runnable> getCancelledTasks() {
        if (!exec.isTerminated())
            throw new IllegalStateException(/*...*/);
        return new ArrayList<Runnable>(tasksCancelledAtShutdown);
    }

    public void execute(final Runnable runnable) {
        exec.execute(new Runnable() {
            public void run() {
                try {
                    runnable.run();
                } finally {
                    if (isShutdown()
                            && Thread.currentThread().isInterrupted()) // 已关闭状态；并被中断（shutdownNow引起的）的任务
                        tasksCancelledAtShutdown.add(runnable);
                }
            }
        });
    }
}
```
这种方式存在一个不可编码的竞态条件，从而产生“误报”问题：任务完成了最后一条指令和线程池将任务记为“结束”的两个时刻之间，线程池可能被关闭。

### 处理非正常的线程终止
导致线程提前死亡的主要原因是RuntimeException，默认在控制台输出栈追踪信息，并终止线程。
在以上我们提到的OutOfTime由于遗漏线程造成严重后果：TImer表示的服务将永远无法使用。
任何代码都有可能抛出RuntimeException，每当调用一个方法时，都要对它的行为保持怀疑。
典型的工作者线程结构：
```java
public void run() {
	Throwable thrown = null;
	try {
		while (!isInterrupted())
			runTask(getTaskFromWorkQueue());
	} catch (Throwable e) {
		thrown = e;
	} finally {
		threadExited(this, thrown); // 可能重新补充工作者线程
	}
}
```
Thread API提供UncaughtExceptionHandler处理由于未捕获的异常而终止的情况。
```java
public interface UncaughtExceptionHandler {  
	void uncaughtException(Thread t, Throwable e);  
}
```
可为Thread，ThreadGroup设置该UncaughtExceptionHandler对象
```java
public class Thread implements Runnable {
	// 设置线程全局处理对象
	public static void setDefaultUncaughtExceptionHandler(UncaughtExceptionHandler eh){ ... }
	public void setUncaughtExceptionHandler(UncaughtExceptionHandler eh) { ... }
}
  
public class ThreadGroup implements Thread.UncaughtExceptionHandler {
	public void uncaughtException(Thread t, Throwable e) { ... }
}
```
只有一个UncaughtExceptionHandler对象会被执行，搜索顺序：本线程 -> 线程组 -> 线程组父类 ... -> 线程全局处理对象

### JVM关闭
JVM关闭的操作：
+ 正常关闭（orderly shut down）：最后一个非守护线程结束、System.exit、平台特定方法（发送SIGINT、Ctrl-C）
+ 强行关闭（abrupt shut down）：Runtime.halt、杀死JVM进程（发送SIGKILL）

正常关闭时，JVM首先调用所有已注册的关闭钩子（Shutdown Hook）。关闭钩子是通过Runtime.addShutdownHook注册且尚未开始的线程。JVM不保证关闭钩子的调用顺序。如果有应用线程，那么他们将和关闭进程并发执行。当所有关闭钩子完成时，如果runFinalizersOnExit为true，则JVM可以选择运行终结器，然后停止。JVM不会尝试停止或中断任何在关闭时仍在运行的应用程序线程（当JVM最终停止时，它们会突然终止）。如果关闭钩子或终结器没有完成，那么正常关闭过程会“挂起”，并且JVM将被强行关闭。
强行关闭时，只是关闭JVM，不会运行关闭钩子。
关闭钩子应当是线程安全的，可以实现服务或应用程序的清理工作：如删除临时文件、清除操作系统无法自动清理的资源。
线程可分为普通线程和守护线程，他们的差异仅在于当线程退出时发生的操作，只有在仅存在守护线程时，JVM正常退出——即不会执行finally代码块，也不会执行回卷栈（unwound），而JVM只是直接退出。
上述提到的终结器（定义了finalize方法的对象操作）：避免使用终结器。




