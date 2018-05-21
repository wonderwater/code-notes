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

* newFixedThreadPool  
* newCachedThreadPool 
* newSingleThreadExecutor  
* newScheduledThreadPool 



## 找出可利用的并行性

## 为任务设置时限







