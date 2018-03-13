# IO同步异步

[https://segmentfault.com/a/1190000003063859](https://segmentfault.com/a/1190000003063859)

![](/assets/iomodels.png)

> 当一个read操作发生时，会有两个阶段
>
> 1. 等待数据准备 \(Waiting for the data to be ready\)  
> 2. 将数据从内核拷贝到进程中 \(Copying the data from the kernel to the process\)
>
> 正式因为这两个阶段，linux系统产生了下面五种网络模式的方案。  
> - 阻塞 I/O（blocking IO）  
> - 非阻塞 I/O（nonblocking IO）  
> - I/O 多路复用（ IO multiplexing）  
> - 信号驱动 I/O（ signal driven IO）  
> - 异步 I/O（asynchronous IO）

阻塞IO

![](/assets/blockingio.png)

非阻塞IO

![](/assets/nonblockingio.png)

io多路复用

> IO multiplexing就是我们说的select，poll，epoll，有些地方也称这种IO方式为event driven IO。select/epoll的好处就在于单个process就可以同时处理多个网络连接的IO。它的基本原理就是select，poll，epoll这个function会不断的轮询所负责的所有socket，当某个socket有数据到达了，就通知用户进程。

![](/assets/iomultiplexing.png)

异步io

![](/assets/asynchronousio.png)

## blocking和non-blocking的区别

调用blocking IO会一直block住对应的进程直到操作完成，而non-blocking IO在kernel还准备数据的情况下会立刻返回。

## synchronous IO和asynchronous IO的区别

在说明synchronous IO和asynchronous IO的区别之前，需要先给出两者的定义。POSIX的定义是这样子的：

> A synchronous I/O operation causes the requesting process to be blocked until that I/O operation completes;  
> An asynchronous I/O operation does not cause the requesting process to be blocked;

两者的区别就在于synchronous IO做”**IO operation**”的时候会将process阻塞。按照这个定义，之前所述的blocking IO，non-blocking IO，IO multiplexing都属于synchronous IO。

有人会说，non-blocking IO并没有被block啊。这里有个非常“狡猾”的地方，定义中所指的”IO operation”是指真实的IO操作，就是例子中的recvfrom这个system call。non-blocking IO在执行recvfrom这个system call的时候，如果kernel的数据没有准备好，这时候不会block进程。但是，当kernel中数据准备好的时候，recvfrom会将数据从kernel拷贝到用户内存中，这个时候进程是被block了，在这段时间内，进程是被block的。

而asynchronous IO则不一样，当进程发起IO 操作之后，就直接返回再也不理睬了，直到kernel发送一个信号，告诉进程说IO完成。在这整个过程中，进程完全没有被block。



