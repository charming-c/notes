# Linux IO_Uring 

IO_Uring 在 Linux 内核 5.1 中推出（2019 年 3 月），为需要进行 AIO 但是希望内核执行 IO 的应用程序提供了低延迟和功能丰富的接口。

## 一、同步 IO 和 异步 IO

### 同步 IO

### 异步 IO

## 二、IO_Uring 框架结构

### io_uring_setup

### io_uring_register

### io_uring_enter

## 三、IO_Uring 与 IO_Multiplexing 对比

## 四、liburing 学习

## 参考文档

- [Efficient IO with io_uring](https://kernel.dk/io_uring.pdf)

- [How io_uring and eBPF Will Revolutionize Programming in Linux](https://thenewstack.io/how-io_uring-and-ebpf-will-revolutionize-programming-in-linux/)

- [An Introduction to the io_uring Asynchronous I/O Framework](https://medium.com/oracledevs/an-introduction-to-the-io-uring-asynchronous-i-o-framework-fad002d7dfc1)

