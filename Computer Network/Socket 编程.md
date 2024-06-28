# Socket 编程

## 一、什么是 socket

Socket 是应用层与TCP/IP协议族通信的中间软件抽象层，它是一组接口。在设计模式中，Socket其实就是一个门面模式，它把复杂的TCP/IP协议族隐藏在 Socket 接口后面，对用户来说，一组简单的接口就是全部，让Socket去组织数据，以符合指定的协议。在传统的计算机网络分层模型中，socket 的位置大致如下图所示：

![socket位置](https://raw.githubusercontent.com/charming-c/image-host/master/img/20171114161918044.png)

## 二、socket 通信原理

套接字（socket），在Linux环境下，用于表示进程间通信的特殊文件类型（伪文件）。我们知道，在TCP/IP协议中：

- IP地址：在网络环境中唯一标识一台主机

- 端口号：在主机中唯一标识一个进程

- IP地址+端口号：在网络环境中唯一标识一个进程

这个在网络中被唯一标识的进程，被称为socket。在网络通信中，套接字一定是成对出现的。这两个socket组成的socket pair就唯一标识一个连接。

![socket通信原理](https://raw.githubusercontent.com/charming-c/image-host/master/img/20171114161609619.png)

在进行 tcp 或者 UDP 通信时，其通信过程入下图所示：

<img src="https://raw.githubusercontent.com/charming-c/image-host/master/img/20171114200747792.png" alt="TCP" style="zoom: 67%;" />            <img src="https://raw.githubusercontent.com/charming-c/image-host/master/img/20171114200805172.png" alt="img" style="zoom:75%;" />




## 三、socket() 函数

socket 函数提供的系统调用接口是`int socket(int domain, int type, int protocol);`

- domain 表示连接的域
- type 表示 socket 的类型
- protocol 表示 socket 连接所采用的协议

其中常用的 socket type 大致分为三种： SOCK_STREAM、SOCK_DGRAM、SOCK_RAW，同时不同的 type 对应支持不同的 protocol，函数的传参拥有一定的限制。

1. Stream Sockets

该Socket类型提供有序的、可靠的、双向的和基于连接的字节流，使用带外数据传送机制，为Internet地址族使用TCP。 保证了数据交互的可靠性。可以确保数据可靠的发送到对方。但是Stream Socket所占的资源更多。 TCP传送的是一个**无记录边界**概念的字节流。可以总结为TCP中没有**用户可见**的"分组"概念，它只是传送了一个字节流，我们无法准确地预测在一个特定的读操作中会返回多少字节。最最简单的例子来说，你通过send函数send 3次的数据可能在另一端接受的时候 2次就读取完了。其传输的过程如上图所示。

2. Datagram Sockets

该Socket类型支持无连接的、不可靠的和使用固定大小（通常很小）缓冲区的数据服务，为Internet地址族使用UDP。 因为是使用UDP来实现数据通讯，因此它不能保证数据能够到达目的地，但是由于它不需要专用的网络链接，所以它所需的资源相对少的多。相对于TCP而言， TCP 协议在传输数据的时候，如果数据发生错误，那么将重新传输该错误的部分。但是这样以来常常会浪费很多时间，在一些讲究实时性的通讯过程中，这样做有些不切实际。例如我们在观看网络视频的时候，少量的数据丢失并不会有很严重的影响，因此我们就会用到UDP 这样的协议。其传输的过程如上图所示。

3. Raw Sockets

该Socket类型为用户提供了访问底层通信协议的能力，该类型不适于一般的用户，它主要是用于有想法开发新的协议或者对现有协议进行自定义操作的研发人员。也就是说一般常规的 TCP 和 UDP 请求会采用上面的 socket，而如果想要拿到 IP 层或者更底层的数据，则要选择使用 raw socket。以下是它的一些简单使用。

```c
socket(PF_INET, SOCK_RAW, IPPROTO_TCP|IPPROTO_UDP|IPPROTO_ICMP)//发送接收ip数据包
socket(PF_PACKET, SOCK_RAW, htons(ETH_P_IP|ETH_P_ARP|ETH_P_ALL))//发送接收以太网数据帧
```

