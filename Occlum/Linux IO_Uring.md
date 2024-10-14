# Linux IO_Uring 

IO_Uring 在 Linux 内核 5.1 中推出（2019 年 3 月），为 Linux 中 AIO 提供了内核解决方案，本文主要从网络 IO 的角度解读 Linux  IO_Uring 框架，并与其他 IO 模型一起比较学习。

## 一、网络 IO

下图是一个 UDP 连接中 socket 的 IO 流程：

<img src="https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20241013163621280.png" alt="image-20241013163621280" style="zoom: 33%;" />

可以看到，在每次调用 recvfrom 时，由于网络数据传输的时延缘故，Client/Server 都需要一定程度的等待，如果不做特殊的处理，这里就只能阻塞进程，被迫放弃 CPU，内核调度其他的进程运行。在内核中，IO 更为具体的流程如下：

<img src="https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20241013173929488.png" alt="image-20241013173929488" style="zoom:67%;" />

1. 用户进程通过 socket 系统调用，创建 socket
2. 在进程 socket 配置之后，发起 recvfrom 系统调用陷入内核。此时，由于 socket 的数据接收队列还没有数据到来，所以内核将用户进程阻塞，且将进程描述符添加到进程等待队列，之后调度其他进程执行
3. 在其他进程执行的过程中，数据到达网卡
4. 网卡通过 DMA 将数据拷贝到内存环形缓冲区 RingBuffer
5. 网卡随后就向 CPU 发出硬件中断
6. CPU 向内核中断进程发出软中断
7. 中断进程首先通过 RingBuffer 中的数据（IP + 端口），找到对应的 socket
8. 将 RingBuffer 中的数据拷贝到 socket 中的接收队列，并唤醒用户进程
9. 用户进程进入运行队列，继续从之前被阻塞的 recvfrom 处执行，此时接收队列有数据，将数据拷贝进用户空间的 buffer。

根据上面的流程可知，在 Client/Server 执行一次网络 IO 的过程中，需要等待两次：

- 在调用 recvfrom 陷入内核时，由于 socket 的接收队列没有数据，所以需要等待数据从网卡输入最终进入 socket 的接收队列
- 在 socket 的接收队列准备好数据，原用户进程被唤醒以后，还需要等待将数据从内核中的 socket 接收队列拷贝进用户空间

其中，根据第一次等待需不需要发生，将 IO 分为阻塞 IO 和 非阻塞 IO。

### 1. 阻塞 IO（BIO）

上文中，清晰地展示了阻塞 IO 模型。当进程调用 recvfrom 时，由于此时 socket 中数据为空，进程阻塞。只有当内核将数据准备好，并且拷贝到用户空间后，进程才能继续执行，recvfrom 系统调用返回。

![图片](https://raw.githubusercontent.com/charming-c/image-host/master/img/640)

在阻塞 IO 中，在调用 recvfrom 时，会发生阻塞，导致进程切换，数据需要经历从网卡到内核，从内核到用户空间的两次转换。一次进程切换或者或者是数据拷贝均会导致许多不必要的 CPU 开销。

### 2. 非阻塞 IO（NIO）

非阻塞 IO 即不发生阻塞的网络 IO。Linux 中为 socket 实现了 non_blocking 选项，这样在调用 recvfrom 时，socket 接收队列没有准备好数据时，进程不会阻塞，没有上下文切换，而是直接返回。因此，在使用是，需要轮询数据是否就绪，如下图：

![图片](https://raw.githubusercontent.com/charming-c/image-host/master/img/640-20241013183939246)

虽然不会发生阻塞，但当数据已经到达内核空间的 socket 的接收队列后，用户进程依然要等待 recvfrom() 函数将数据从内核空间拷贝到用户空间，才会从 recvfrom() 系统调用函数中返回。

**小结：**

- NIO 解决了 BIO 的进程阻塞，上下文切换的问题，但是需要频繁的系统调用，消耗系统资源
- 在 NIO 和 BIO 中，都需要两次数据的拷贝，一次从网卡到内核空间，一次从内核空间到用户空间。即便对于 NIO 来说，当 socket 中的数据已经就绪，仍然需要等待数据从内核到用户的时间开销，这个过程对数据来说是同步的，也因此，他们都是同步 IO。

### 3. IO 多路复用

为了解决非阻塞 IO 频繁系统调用的问题，引出 IO 多路复用机制，一次性管理多个连接的 IO。我们不用频繁的系统调用，而可以非阻塞的处理多个连接。不过，在没有连接就绪时，我们仍然需要阻塞进程进行等待，并且数据还是需要进行两轮拷贝。由于 IO 多路复用不是本文的重点，这里忽略详细介绍，具体请看 [这里](https://mp.weixin.qq.com/s/5xj42JPKG8o5T7hjXIKywg)。

### 4. 异步 IO（AIO）

在之前的 IO 模型中，当数据在内核态就绪时，在内核态拷贝用用户态的过程中，仍然会进行短暂时间的阻塞等待。而 AIO 的使用则是为了避免这一点：内核态拷贝数据到用户态交给内核来实现，不由用户线程完成。这样，当数据在内核态就绪时，进程可以直接在用户态读取数据并处理，没有额外的等待时间。

## 二、IO_Uring 框架结构

### 1. SQ && CQ

### 2. io_uring_setup

### 3. io_uring_register

### 4. io_uring_enter

## 三、IO_Uring 与 IO_Multiplexing 对比

## 四、liburing 学习

## 参考文档

- [Efficient IO with io_uring](https://kernel.dk/io_uring.pdf)
- [How io_uring and eBPF Will Revolutionize Programming in Linux](https://thenewstack.io/how-io_uring-and-ebpf-will-revolutionize-programming-in-linux/)
- [An Introduction to the io_uring Asynchronous I/O Framework](https://medium.com/oracledevs/an-introduction-to-the-io-uring-asynchronous-i-o-framework-fad002d7dfc1)
- [深入学习IO多路复用 select/poll/epoll 实现原理](https://mp.weixin.qq.com/s/5xj42JPKG8o5T7hjXIKywg)
- [网络 IO 演变发展过程和模型介绍](https://mp.weixin.qq.com/s/EDzFOo3gcivOe_RgipkTkQ)

