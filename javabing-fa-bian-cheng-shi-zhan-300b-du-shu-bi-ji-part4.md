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

### 任务取消

### 停止基于线程的服务

### 处理非正常的线程终止

### JVM关闭



